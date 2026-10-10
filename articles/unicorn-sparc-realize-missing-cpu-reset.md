---
title: "Unicorn EngineのSPARCエミュレーションでRIP=0に落ちる：CPU stateのreset忘れを2行で直す"
emoji: "🦄"
type: "tech"
topics: ["emulator", "c", "oss", "debugging", "sparc"]
published: true
---

## はじめに

CPUエミュレータ [Unicorn Engine](https://www.unicorn-engine.org) は、内部的にQEMUのCPUモデルを流用してゲストコードを実行します。SPARCアーキテクチャを指定して`uc_open()`した場合、エミュレータのCPU実体化処理（`realizefn`）の末尾でCPU状態のリセットが行われず、未初期化のまま残り、非常に短いひと命令でホストプロセスごとクラッシュする問題があります。

実際に[Issue #2425](https://github.com/unicorn-engine/unicorn/issues/2425)で報告されている内容を再現し、修正PR [#2429](https://github.com/unicorn-engine/unicorn/pull/2429)を出しました。この記事はその調査・修正・検証の記録です。再現自体はゲストSPARCの機械語4バイトと、UnicornのC APIを数行書くだけで完結するため、「エミュレータのCPU初期化漏れがどうクラッシュに化けるか」を追う題材としても読めるはずです。

**修正の現状（2026-10-11時点）: PR #2429はopen、マージは未です。** 内容は検証済みのコミット `a469af7` に基づきます。

## 現象：たった4バイトのゲストコードでホストごとSIGSEGV

再現コードは次のCプログラムだけです。ゲストSPARCの機械語4バイトを`poc.bin`として用意し、それをSPARC32ビッグエンディアンとしてマップして実行します。記事掲載のコードは、PR作成時に実際に使用した再現ドライバ（PR本文でも同内容を公開）です。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unicorn/unicorn.h>
int main(int argc, char **argv) {
    unsigned char code[4096];
    FILE *f = fopen(argc > 1 ? argv[1] : "poc.bin", "rb");
    size_t len = fread(code, 1, sizeof(code), f); fclose(f);
    uc_engine *uc = NULL;
    if (uc_open(UC_ARCH_SPARC, UC_MODE_BIG_ENDIAN | UC_MODE_SPARC32, &uc)) return 1;
    uint64_t ADDR = 0x100000;
    uc_mem_map(uc, ADDR, 4 * 1024 * 1024, UC_PROT_ALL);
    uc_mem_write(uc, ADDR, code, len);
    uc_err err = uc_emu_start(uc, ADDR, ADDR + len, 500000, 200000);
    printf("[*] returned %d (%s)\n", err, uc_strerror(err));
    uc_close(uc); return 0;
}
```

`poc.bin`の中身（ビッグエンディアン4バイト）：

```text
B0 42 1C 06
```

この表記はSPARCの`ADDX %o0, %g6, %i0`に相当します。ビットフィールドに分解すると：

- op = 10（2）、op3 = 01000（`ADDX`）
- rs1 = 8（`%o0`）
- rs2 = 6（`%g6`）
- rd = 11000（24、`%i0`）

実行すると、修正前のUnicornではホストプロセスごと落ちます（手元の実行でも `Segmentation fault`、終了コード139 を再現）。[Issue #2425](https://github.com/unicorn-engine/unicorn/issues/2425) の報告では、クラッシュは `RIP = 0`、すなわちNULL関数ポインタの間接呼び出しとして観測されています：

「エミュレータプロセス自体が落ちる」ので、SPARC用のゲストコードを渡したユーザーにとっては検出しにくい、致命的な形です。ゲストコードを1命令実行しただけで、エミュレーションエラー（`UC_ERR_*`）ではなくホストプロセスのSIGSEGVになる点が、報告されていたIssueの核心です。

## 原因追究：RIP=0から表まで遡る

### 最初の手がかり

手元で再現したSIGSEGV（終了コード139）がアドレス0への飛びであれば、NULL関数ポインタの間接呼び出しが濃厚です。この場合、通常はクラッシュしたスレッドのバックトレースから、どの構造体の関数ポインタテーブルを経由したかを辿ります。Issue報告の `RIP=0` と合致することから、表（関数ポインタテーブル）経由のNULL呼び出しを第一の仮定として調べました。

SPARCターゲットでは、条件コード（キャリー、ゼロなど）の計算は関数ポインタテーブル `icc_table` に委譲されています。テーブル定義は次のように、キャリー計算用の `compute_c` エントリにデフォルトのNULLを許容する作りになっています（以下は `qemu/target/sparc/cc_helper.c` から該当部分）。

```c
typedef struct CCTable {
    uint32_t (*compute_all)(CPUSPARCState *env); /* return all the flags */
    uint32_t (*compute_c)(CPUSPARCState *env);  /* return the C flag */
} CCTable;

static const CCTable icc_table[CC_OP_NB] = {
    /* CC_OP_DYNAMIC should never happen */
    [CC_OP_FLAGS] = { compute_all_flags, compute_C_flags },
    [CC_OP_DIV]   = { compute_all_div,   compute_C_div },
    [CC_OP_ADD]   = { compute_all_add,   compute_C_add },
    [CC_OP_ADDX]  = { compute_all_addx,  compute_C_addx },
    ...
};
```

ポイントは次の2点です。

1. `icc_table` の `CC_OP_DYNAMIC` エントリは **初期化子に明示的に書かれておらず、NULLのまま** 。直前のコメントが「CC_OP_DYNAMIC should never happen」、つまり来るはずがないという前提の設計です。
2. フラグ計算は実行時に `icc_table[env->cc_op].compute_c(env)` として **間接呼び出しされる** 。

つまり「来るはずがない」と想定された `CC_OP_DYNAMIC` が**実際に来てしまう**と、NULLのメンバがそのまま呼び出され、Issue報告の `RIP=0` と整合する形で落ちます。これはQEMU本体（システムエミュレーションの`target/sparc`）由縁のコードで、UnicornがQEMUの`target/sparc`を組み込む形で利用している構造そのものに起因します。

### なぜ未初期化のまま残るのか

`CC_OP_DYNAMIC` は列挙値が `0` で、`CPUSPARCState` は`uc_open()`時点でゼロクリアされたメモリ上に展開されます。結果、 **`env->cc_op` は初期化処理を一切経ずに `CC_OP_DYNAMIC`（0）のまま** になります。

翻訳フェーズでは、`ADDX`/`SUBX`（op3が 0x8 / 0xc）のような命令を見たとき、`dc->cc_op == CC_OP_DYNAMIC` ならフラグ計算をヘルパー関数に委譲するコードが生成されます。実行時にこのヘルパーが `icc_table[CC_OP_DYNAMIC].compute_c`、つまり **NULLを呼び出す** 、という流れです。

対照的に、同じQEMU内のi386・ARM・RISC-V・MIPS・TriCoreターゲットでは、`realizefn`（CPU実体化処理）の末尾で明示的に`cpu_reset(cs)`を呼び、CPU状態をリセットしました。SPARCターゲットだけ、その呼び出しが`cpu_exec_realizefn()`までで止まっており、各アーキテクチャ固有のリセットへたどり着いていなかった、というのがバグの正体です。

チェックとして、他のターゲットの`cpu_reset(cs)`呼び出しの存在も確認しました（`qemu/target/i386/cpu.c` の4913行目、`qemu/target/arm/cpu.c` の1175行目）。

## 修正：2行の`cpu_reset(cs)`を足す

対応としては、SPARCの`realizefn`に他ターゲットと同じリセット呼び出しを足すのが最小です。

```c
    cpu_exec_realizefn(cs);

    cpu_reset(cs);
}
```

この2行追加のみで、修正コミット `a469af7` は `qemu/target/sparc/cpu.c` への2行追加だけ（GitHub APIで取得したPR #2429のfiles、+2/-0）です。SPARC固有のリセット処理 `sparc_cpu_reset()` は `cc->reset` に登録済みなので、共通リセットと合わせて、`cc_op` が安全な `CC_OP_FLAGS` に初期化され、`cwp`・`wim`・`regwptr`・`pc`・`npc`・MMU状態なども整います。`cpu_reset()` は `cpu.h` 経由ですでにこの翻訳単位から見えているため、追加のincludeは不要でした。

## 検証

修正コミット `a469af7` に対し、手元でビルドして検証しました。ビルドには `cmake -DUNICORN_ARCH=sparc`（SPARCターゲットのみ）を用いています。

結果は次のとおりです。ゲストコードは同一の4バイト、ドライバは上記Cコード、差分は修正の有無のみです。

- **修正前**：SIGSEGVでクラッシュ（手元の実行で終了コード139を再現、Issue報告では `RIP=0` のNULL関数ポインタ呼び出し）。SPARC32とSPARC64の両方で同じく落ちる。
- **修正後**：`uc_emu_start()`が`UC_ERR_OK`で正常終了（ドライバの出力は`[*] returned 0 (OK (UC_ERR_OK))`、プロセス終了コード0）。
- **対照実験**：ゲストコードを64回のNOP（`0x01000000`を64回）に変えると、修正前後どちらも`UC_ERR_INSN_INVALID`（終了コード0）で動作に差がない。

この対照実験は、クラッシュがゲストコードの内容、具体的には **キャリーを扱うADDXで、未初期化のcc_opが原因** であることを示します。NOPだけでは呼ばれないNULLテーブル経路が、ADDX 1命令で顕在化する、という構図です。

ユニットテストは`tests/unit/test_sparc`（`SUCCESS: All unit tests have passed.`）、`ctest`の`test_sparc`をパスしました。ローカルビルドで`test_mem`・`test_ctl`が失敗しましたが、**修正なしのベースラインビルドでも同じ失敗** だったため、PR本文には「本修正と無関係な既知の環境依存」と明記しました。誤って無関係な失敗をPRの検証証拠として見せることを避けるためです。

SPARC64（`UC_MODE_SPARC64`）でも同じ4バイト payload を使って同様にSIGSEGV→`UC_ERR_OK`の改善を再現しました。

## 学び

1. **「決して来ないはずの値」は、未初期化と結びついて一番先に来る**
   `icc_table` の `CC_OP_DYNAMIC` は「来ない前提」でNULLのまま残されています。しかし`CC_OP_DYNAMIC` は列挙値0であり、CPU状態を明示的にリセットしない実装ではゼロクリアメモリがそのままその値になります。「実行時不変条件」を数値のゼロと重ねる設計は、初期化漏れと結びつくと NULL 間接呼び出しに化けます。同種のテーブルとデフォルト値の設計を見たら、「列挙値0に相当する未初期化状態」を疑うのは良い視点です。

2. **ホストプロセスごと落ちるバグは、最小ゲストコードへ切り詰めて初めて手がかる**
   Issue報告はUBSanビルドやWSL2/i686ホストなど、環境情報が混ざった形で来ます。今回は4バイトのマシン語+数行のドライバへ切り詰めたことで、因果が「どのゲスト命令がテーブルを辿るか」（NOPでは再現せず、ADDXでは再現する）まで特定できました。エミュレータ系のクラッシュは、ゲストコードを最小化して命令の種類を絞るのが最短ルートです。

3. **同じQEMUフォークでもターゲットごとに初期化パスの整備状況が違う**
   i386・ARMなどでは再現しない問題が、ターゲット固有の`realizefn`の整備漏れとして残ります。マルチアーキテクチャ対応エミュレータでは、特定のアーキテクチャだけ初期化手順が古い・不十分なことがあるため、「他のターゲットはどうしているか」の横断比較が有効です。

## 参考資料

- [Issue #2425: Segfault (RIP=0, NULL function pointer call) in sparc/sparc64 emulation](https://github.com/unicorn-engine/unicorn/issues/2425)
- [PR #2429: sparc: reset CPU state during realize to avoid NULL compute_c call](https://github.com/unicorn-engine/unicorn/pull/2429)
- Unicorn Engine 公式サイト: https://www.unicorn-engine.org
- 修正対象の実装（`qemu/target/sparc/cpu.c` の `sparc_cpu_realizefn()`、`qemu/target/sparc/cc_helper.c` の `icc_table`）は、検証時のコミット `7c5db94`（修正前）および `a469af7`（修正後）時点のものです。
