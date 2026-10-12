---
title: "空のUDPデータグラムでOpenTelemetry Collectorが落ちる件をGoのbytes.Buffer.WriteToから読む"
emoji: "📡"
type: "tech"
topics: [go, opentelemetry, debugging, opensource, udp]
published: true
---

OpenTelemetry Collector Contrib の Jaeger receiver で、**長さ0のUDPデータグラムを1つ送るだけで collector プロセス全体が panic で落ちる**というバグ（[issue #51973](https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/51973)）の調査・修正の機会があったためまとめます。修正自体は数行ですが、原因には Go 標準ライブラリ `bytes.Buffer` の `WriteTo` の仕様が絡んでいて、「APIの契約を正しく読む」ことがデバッグの鍵になった良い例だと思います。

なお本バグには先行PR（[#51976](https://github.com/open-telemetry/opentelemetry-collector-contrib/pull/51976)）が存在し、私が出した修正PR [#51974](https://github.com/open-telemetry/opentelemetry-collector-contrib/pull/51974) は重複としてクローズされました（2026-10-12時点）。この記事の価値は原因の切り分けプロセスと標準ライブラリ仕様の整理にあります。

## 構成の前提：Jaeger receiver はどう UDP を処理するか

Jaeger receiver（thrift compact/binary プロトコル）のUDP受信経路は3層に分かれています。

1. **`UDPServer`** (`internal/udpserver/udpserver.go`): `net.UDPConn` から `packetReader` 経由で1データグラムを読み、`*bytes.Buffer` に詰めて `dataChan` チャネルへ投入する。バッファは `sync.Pool` で再利用。
2. **`ThriftProcessor.processBuffer`** (`thrift_processor.go`): `dataChan` から `*bytes.Buffer` を受け取り、プロトコルオブジェクト（プールされた `thrift.TProtocol`）のトランスポートへ `buf.WriteTo(...)` で書き込む。
3. **`TBufferedReadTransport`** (`buffered_read_transport.go`): thrift の `TTransport` 実装。`Write(buf)` で渡されたバイト列を内部の `readBuf *bytes.Buffer` に持ち、`Read(buf)` でそれを thrift プロトコル層が読む。

問題の核心は、この 1→2→3 の受け渡しのうち「**空のデータグラム**」の扱いです。

## パニックの再現とスタックトレース

collector を起動して、UDPソケットに長さ0のデータグラムを1つ送ると、サーバ側では次の順で処理が進みます。

- `packetReader.readPacket` は `buf.ReadFrom(r)` を通じて1パケットを読む。**長さ0のパケットは `n=0, err=nil` で正常終了**する
- それを `dataChan` へ投入（`err == nil` のパスを通るため）
- `processBuffer` がこの `*bytes.Buffer`（中身0バイト）を受け取る

`processBuffer` はこう実装されています（修正前の該当箇所）。

```go
func (s *ThriftProcessor) processBuffer() {
	for buf := range s.server.DataChan() {
		protocol := s.protocolPool.Get().(thrift.TProtocol)
		_, _ = buf.WriteTo(protocol.Transport()) // writes to memory transport don't fail
		// ... handler.Process ...
	}
}
```

ここで `buf.WriteTo(protocol.Transport())` を呼ぶと、`TBufferedReadTransport.Write(buf)` が呼ばれる想定ですが、実は**空のバッファでは呼ばれません**。その結果、プールから取り出した `TBufferedReadTransport` の `readBuf` は `nil` のままで、続く thrift プロトコルのデコードが `Read` を呼ぶと nil ポインタデリファレンスが発生します。

実行時に出るスタックトレースはこうなります（nil deref のSIGSEGV）。

```text
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x20 ...]

bytes.(*Buffer).Read(...)
    bytes/buffer.go:319
github.com/.../udpserver.(*TBufferedReadTransport).Read(...)
```

さらに厄介なのがゴルーチン構造です。`processBuffer` は `errgroup` 的なワーカー上で動いており、この panic は **non-recover ゴルーチン** のパスを通るため、receiver 単体ではなく collector プロセス全体が落ちます。つまり、UDPで1つ空のデータグラムを送れた攻撃者（あるいは単に間違った監視エージェント）が collector 全体をクラッシュさせられるという意味で、DoS になりうるバグです。

## 原因：`bytes.Buffer.WriteTo` は空バッファで Write を呼ばない

なぜ `Write` が呼ばれないのか、Go標準ライブラリの仕様を確認します。`io.WriterTo` の契約と `bytes.Buffer.WriteTo` の実装です。

- `io.WriterTo` インターフェースは `WriteTo(w io.Writer) (n int64, err error)` です
- `bytes.Buffer.WriteTo` の実装は、内部で `b.Write(w)` をループする一般的な実装ではなく、`b.Len() > 0` の場合のみ実際の書き込みを行う最適化を持っています
- バッファが空の場合は書き込むべきバイトが存在しないため、`w.Write` は一度も呼ばれないまま `(0, nil)` を返す

実際にミニマルな再現コードで確認しました。

```go
package main

import (
	"bytes"
	"fmt"
)

type fakeTransport struct{ called bool }

func (f *fakeTransport) Write(p []byte) (int, error) {
	f.called = true
	return len(p), nil
}

func main() {
	protocol := &fakeTransport{}
	buf := bytes.NewBuffer(nil)
	n, err := buf.WriteTo(protocol)
	fmt.Printf("n=%d err=%v; Write called: %v\n", n, err, protocol.called)
}
```

実行結果:

```text
n=0 err=nil; Write called: false
```

この挙動自体は `io.WriterTo` の契約的に**正しい**ものです（書き込むデータがないので `Write` を呼ばない）。問題は、Jaeger receiver 側が `TBufferedReadTransport.Write` が必ず呼ばれることを暗黙の前提にしていた点にあります。

- `TBufferedReadTransport.Write(buf)` は `p.readBuf = bytes.NewBuffer(buf)` と、**毎回新しいバッファに差し替える**実装です。つまり `Write` が呼ばれて初めて `readBuf` が非nilになる
- 通常の（非空の）データグラムなら `Write` が呼ばれて `readBuf` が初期化されるが、空のデータグラムでは `Write` がスキップされるので `readBuf` が `nil` のまま
- そして thrift プロトコルのデコード側は、データが届いたものとして `Read` を呼ぶ（thrift のトランスポートはnilを許容しない）

結果として、non-recover ゴルーチン上の `Read` で nil dereference → プロセス死、という構図です。

## 修正：nilを許容する形に遅延初期化する

修正方針はシンプルで、`TBufferedReadTransport` の `Read`/`RemainingBytes` が `readBuf == nil` を許容するようにしました（PR [#51974](https://github.com/open-telemetry/opentelemetry-collector-contrib/pull/51974)）。

```go
func (p *TBufferedReadTransport) Read(buf []byte) (int, error) {
	if p.readBuf == nil {
		p.readBuf = bytes.NewBuffer(nil)
	}
	in, err := p.readBuf.Read(buf)
	return in, thrift.NewTTransportExceptionFromError(err)
}

func (p *TBufferedReadTransport) RemainingBytes() uint64 {
	if p.readBuf == nil {
		return 0
	}
	return uint64(p.readBuf.Len())
}
```

`Read` 側は nil なら空バッファに差し替える（つまり `EOF` まで読むバッファとして振る舞う）遅延初期化にしました。`RemainingBytes` はnilのとき0を返すようにしています。この2点で、空データグラムが来ても thrift プロトコル層は「すぐ読み切り」という正常な挙動になり、デコードはエラー（有効なthriftメッセージが無い）で握りつぶされることなく、パニックそのものがなくなるという形です。

なお `Write` メソッド自体は従来どおり `readBuf` を差し替える設計のまま変更していません。ここを拡張しても良いのですが、前提「`Write` が必ず呼ばれる」が崩れている箇所にのみ対処する最小修正としました。

## 再現テストと検証

回帰テストとして、空データグラムを collector 起動後に送り、そのあと普通のバッチが処理されることを確認するテストを追加しました（compact/binary 両プロトコル）。

要点は次の通りです。

- `net.Dial("udp", hostPort)` でクライアントを用意し、`conn.Write([]byte{})` で**長さ0の書き込み**を1回
- その直後に、通常の Jaeger バッチを thrift クライアントから `EmitBatch` で送信
- `TestProcessor_EmptyDatagram` では、空データグラムが先にエンキューされているためプロセッサがこれを先に消費する、という状況を作れる
- 修正前:このテストを走らせると、テストプロセス自体が panic で落ちる（non-recover ゴルーチン上の panic がプロセスを殺すため）
- 修正後:テストは通る。また `go test ./receiver/jaegerreceiver/...` 全体もパス

実際の手元での確認結果:

```text
$ go test -run TestProcessor_EmptyDatagram -count=1 ./internal/udpserver/
ok  github.com/open-telemetry/opentelemetry-collector-contrib/receiver/jaegerreceiver/internal/udpserver  0.031s
```

また、上記の「`Write` が呼ばれない」挙動は独立したミニマルコードでも検証済みです（節「原因」の再現コードと出力を参照）。nil deref が実際に panic に変わることも、別のミニマルコードで確認しました。

## 学び：IOインターフェースの「契約」を読む

このバグから得られる教訓は、次のように整理できます。

- **`io.WriterTo` の契約**: `WriteTo(w)` は「何も書くものがなければ `w.Write` を呼ばない」スタイルの最適化を許されます。つまり `Write` の回数や呼ばれることを前提にした設計は契約違反になりうる
- **必ず呼ばれる前提は自前で担保する**: 中間のトランスポート実装（今回の `TBufferedReadTransport`）が「内部バッファを `Write` で初期化する」設計をしているなら、`Write` 未呼出のケースを nil 安全にしておくべき
- **sync.Pool と初期化の組み合わせに注意**: プールから取り出したオブジェクトは「前回使われた状態」ではなく、今回のケース構造では「まだ `Write` が一度も呼ばれていない初期状態」になりうる。プールを導入すると、この初期化漏れが露呈しやすい
- **panic の影響範囲を追う**: receiver 内の panic がプロセス全体を落とすのは、OpenTelemetry Collector の `errgroup`/`zap` 周りのレイヤーで recover されないため。自分で collector を拡張する際も、non-recover ゴルーチン上で panic を投げないよう注意が必要

## 参考資料

- [Issue #51973: nil pointer dereference in Jaeger UDP receiver on empty datagram](https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/51973)
- [PR #51974（本記事の調査に基づく修正案・クローズ済み）](https://github.com/open-telemetry/opentelemetry-collector-contrib/pull/51974)
- [PR #51976（先行修正PR・本記事執筆時点でopen）](https://github.com/open-telemetry/opentelemetry-collector-contrib/pull/51976)
- [Go: bytes.Buffer.WriteTo の実装（go/src/bytes/buffer.go）](https://cs.opensource.google/go/go/+/master:src/bytes/buffer.go)
- [Go: io.WriterTo インターフェース](https://cs.opensource.google/go/go/+/master:src/io/io.go)
- [OpenTelemetry Collector Contrib リポジトリ](https://github.com/open-telemetry/opentelemetry-collector-contrib)
