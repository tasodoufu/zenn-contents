---
title: "smoltcpのARPレート制限が別ホストの接続まで遅延させる問題を直す"
emoji: "🌐"
type: "tech"
topics: [rust, network, tcpip, embedded, opensource]
published: true
---

組み込み機器や小さなネットワークスタックでは、近隣探索（ARP/NDISC）の送信頻度を抑える必要があります。一方で、レート制限の状態を「キャッシュ全体で1つ」持つと、ある宛先への探索が別の宛先への通信まで遅延させることがあります。

smoltcpのIssue [#1209](https://github.com/smoltcp-rs/smoltcp/issues/1209) とPR [#1210](https://github.com/smoltcp-rs/smoltcp/pull/1210) を題材に、症状の切り分け、最小再現、修正、回帰テストをまとめます。

## 症状：DNSの直後のTCP接続だけ約1秒遅い

Issueの報告環境では、DNSサーバーと接続先サーバーが同じリンク上にあり、DNSサーバーがゲートウェイでした。リンクアップ直後の最初の接続で、次のような遅延が発生していました。

- DNS解決：約20 ms
- その直後のTCP接続：約1.03 s
- DNS応答と接続の間に1.5 s待つと、接続：約0.01 s
- 同じセッション内のキャッシュ済み接続：0〜20 ms

これはTCPの再送遅延とは限りません。DNS問い合わせのためにゲートウェイのARP解決を行った結果、同じLAN上にある接続先のARP解決まで待たされている可能性があります。

## 最小再現：別アドレスまでRateLimitedになる

修正前の`neighbor::Cache`は、レート制限の期限を1つだけ保持していました。

```rust
struct Cache {
    storage: LinearMap<IpAddress, Neighbor, IFACE_NEIGHBOR_CACHE_COUNT>,
    silent_until: Instant,
}
```

ある宛先への要求を送信すると、次のようにキャッシュ全体の期限を更新します。

```rust
fn limit_rate(&mut self, timestamp: Instant) {
    self.silent_until = timestamp + Self::SILENT_TIME;
}
```

`lookup()`もアドレスを区別せず、その期限だけを参照していました。

```rust
if timestamp < self.silent_until {
    Answer::RateLimited
} else {
    Answer::NotFound
}
```

したがって、アドレスAの要求を時刻0に送った後、時刻100 msにアドレスBを調べると、Bについては何も送信していないのに`RateLimited`になります。

実際に、修正前のsmoltcpコードへ「Bは`NotFound`であるべき」というアサーションを追加して実行すると、次の失敗を再現できました。

```text
test iface::neighbor::test::test_hush ... FAILED
assertion `left == right` failed
left: RateLimited
right: NotFound
```

この再現は、ハードウェアや実ネットワークに依存せず、状態管理の誤りだけを確認できます。

## 原因：レート制限の単位が間違っている

「同じ宛先へのARP/NDISC要求を短時間に繰り返さない」という制約を、「キャッシュ内の全宛先について要求を送らない」という状態として実装していました。

レート制限の単位は宛先アドレスです。Aへの要求を抑制しても、Bへの最初の要求を抑制する理由にはなりません。

## 修正：期限を宛先ごとに保持する

PR #1210では、単一の`Instant`をアドレスごとの`LinearMap`に変更しています。

```rust
silent_until: LinearMap<IpAddress, Instant, IFACE_NEIGHBOR_CACHE_COUNT>
```

APIも、どの宛先への要求を送信したか渡す形に変わりました。

```rust
self.neighbor_cache.limit_rate(dst_addr, self.now);
```

`lookup(addr, now)`では、そのアドレスの期限だけを調べます。これにより、Aがレート制限中でも、Bは`NotFound`として通常の近隣探索へ進めます。

エントリ数には固定上限があります。満杯で新しいアドレスを登録する場合は、期限が最も早いエントリを削除してから挿入します。smoltcpのヒープを使わない設計を維持しつつ、既存の固定容量マップに収めています。

また、`flush()`では近隣キャッシュだけでなく、レート制限のマップもクリアします。キャッシュを消去した後に古い期限だけが残る状態を防ぐためです。

## 回帰テスト

PRでは次のテストを追加・拡張しています。

- 1つのアドレスを制限しても、別アドレスは`NotFound`になる
- AとBをそれぞれ制限すると、期限は独立して動作する
- `flush()`後は制限状態も消える

パッチを適用したフレッシュなsmoltcpクローンで、次の結果を確認しました。

```text
cargo test
675 passed; 0 failed

cargo test --lib neighbor
12 passed; 0 failed

Doc-tests smoltcp
7 passed; 0 failed
```

smoltcpのPRページでも、stable、MSRV、clippy、fmt、fuzz、ネットワークシミュレーションを含む14チェックがすべて成功しています。実機のESP32-S3での再計測はこの記事の検証範囲には含めていません。1.03秒という実環境の測定値はIssue報告者の記録であり、ローカル検証では状態遷移と回帰テストを確認しました。

## この問題から得られること

### 1. タイムアウトの共有範囲を確認する

「レート制限」「クールダウン」「再試行待ち」は、実際に何を単位として制限するのかを先に決めます。宛先ごとの制約を全体状態で表すと、独立した処理まで直列化されます。

### 2. 実ネットワークの遅延を状態機械へ分解する

DNSが速くても、その後のTCP接続が遅い場合があります。名前解決、近隣探索、ルーティング、TCP接続の各段階を分けて観測すると、DNSそのものではなくARP/NDISCの抑制が原因だと切り分けやすくなります。

### 3. 回帰テストは「別のキー」を含める

同じ宛先を2回調べるテストだけでは、全体状態とキー単位状態の違いを検出できません。Aを操作した後にBを調べるテストを入れることで、状態のスコープを検証できます。

### 4. 固定容量でも独立性は保てる

組み込み向けコードでは、無制限のハッシュマップに置き換えるのが解決策とは限りません。固定容量のまま、期限の早いエントリを退避する設計でも、必要なアドレス単位の独立性を実現できます。

## 参考資料

- [smoltcp Issue #1209: arp: resolving one address delays another by up to 1s](https://github.com/smoltcp-rs/smoltcp/issues/1209)
- [smoltcp Pull Request #1210: arp,ndisc: rate-limit neighbor discovery per destination address](https://github.com/smoltcp-rs/smoltcp/pull/1210)
- [smoltcp公式リポジトリ](https://github.com/smoltcp-rs/smoltcp)
