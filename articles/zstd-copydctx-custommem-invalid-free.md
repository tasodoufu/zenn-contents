---
title: "zstdのZSTD_copyDCtxが宛先のアロケータを上書きするバグと修正"
emoji: "🧯"
type: "tech"
topics: [c, zstd, debugging, opensource, memory]
published: true
---

zstd は `ZSTD_customMem` でコンテキストごとに独自のアロケータを指定できます。この設計が絡むメモリ、具体的には「別々のアロケータで生成した2つのDCtxを `ZSTD_copyDCtx()` でコピーすると、解放時に他人のfree関数が呼ばれる」というバグの調査・修正の機会があったため、原因の切り分け、回帰テスト、負の証明の設計をまとめます。facebook/zstd には修正を [PR #4837](https://github.com/facebook/zstd/pull/4837) として出しました（2026-10-11時点で未マージ。以降の記載はPR時点のコードに基づきます）。

## 背景：ZSTD_customMemとcopyDCtx

zstdのコンテキストは `ZSTD_createDCtx_advanced(ZSTD_customMem)` で生成でき、structの先頭付近に `customMem`（アロケータ関数ポインタとopaqueデータ）を持っています。生成時に使ったアロケータで解放するのが契約です。

一方 `ZSTD_copyDCtx(dst, src)` は、辞書やパラメータ済みの「雛形」DCtxを別のコンテキストへコピーして初期化コストを省略する関数です。実装はこんな形でした（修正前、v1.6.0近傍のHEAD `49cf517` 時点）：

```c
void ZSTD_copyDCtx(ZSTD_DCtx* dstDCtx, const ZSTD_DCtx* srcDCtx)
{
    size_t const toCopy = (size_t)((char*)(&dstDCtx->inBuff) - (char*)dstDCtx);
    ZSTD_memcpy(dstDCtx, srcDCtx, toCopy);  /* no need to copy workspace */
}
```

つまり **customMem を含む領域を丸ごとコピーしている** のが一目瞭然です。

## 再現条件と現象

- `ZSTD_createDCtx_advanced(customMem)` で独自アロケータを持つ **src** を作る
- `ZSTD_createDCtx()` 既定アロケータで **dst** を作る
- `ZSTD_copyDCtx(dst, src)` を呼ぶ
- `ZSTD_freeDCtx(dst)` を呼ぶ

このとき dst の確保は既定アロケータで行われているのに、memcpy後の `dst->customMem` は **srcのそれ** に書き換わっているため、`ZSTD_freeDCtx(dst)` はsrcのcustomFreeを使ってdstのメモリを解放します。free単体で許されるのは「mallocに対応するfree」であり、カスタムfreeは割当時と同一のアロケータでしか呼べないため、glibc環境では `free(): invalid pointer` でabortします。さらに手順的に厄介なのは、誤解が「解放時」に発火する点で、コピーした直後では何も起きないことです。

## 負の証明で現象を固定する

allocator不整合をそのままreproすると実際にabortするため検証には使えますが、回帰テストとしては「クラッシュする」しか観測点がありません。そこでPRでは **テスト用の数え上げアロケータ** を用意しました。

```c
/* dst に決して呼ばれてはいけない free を検出する */
static void FUZ_countingWrongFree(void* counter, void* address)
{
    int* const flag = (int*)counter;
    *flag += 1;   /* このfreeは起きてはならない */
    free(address);
}
```

テストでは dst に既定アロケータ、src にこのカウンタ付きアロケータを与え、`ZSTD_copyDCtx(dst, src)` → `ZSTD_freeDCtx(dst)` の後にフラグが0のままであることを検証します（`tests/fuzzer.c` の `basicUnitTests` に test 76 として追加）。

負の証明設計の要点は、**修正をstashしたときだけテストがFAILし、復元したときにPASSする**ことを必ず確認することです。手元でも lib 側の修正を stash して test 76 だけがFAIL・他のテスト群は通ること、修正復元で `17 fuzzer tests completed` まで通ることを確認しました。FAILが既存テストの偶発では無いことをこの手順で確定できます。

## 独立reproでも確認

テストとは別に、libに対する単独のCフロントで同じ構図を再現しました。dstを既定アロケータ、srcをwrongMem（customFreeがフラグを立てる）で作り、`ZSTD_freeDCtx(dst)` 直後にフラグを表示します：

```c
ZSTD_DCtx* src = ZSTD_createDCtx_advanced(wrongMem);
ZSTD_DCtx* dst = ZSTD_createDCtx();
ZSTD_copyDCtx(dst, src);
ZSTD_freeDCtx(dst);          /* dstの確保に使われたアロケータで解放されるべき */
fprintf(stderr, "wrongFreeCalled=%d\n", wrongFreeCalled);
```

- 修正 **あり** のビルド：`wrongFreeCalled=0`（free(dst)後にsrcのfreeは一度も呼ばれていない）
- 修正 **無し** のビルド（PR対象と同じ `49cf517` ベースにzstd_decompress.cだけ前の状態へ戻したもの）：`wrongFreeCalled=1`（free(dst)の時点でsrcのcustomFreeが呼ばれた）

`ZSTD_freeDCtx(src)` はsrc自身のアロケータでの解放なので正当です。この構成では、「アロケ不一致でも固定の順序で並べればabortでなくともオブザーバブルに不一致を検証できる」ことが確認できます。

## 原因と修正

原因はシンプルで、`customMem` が「実体の属性」ではなく「確保に使われたアロケータの記録」だという点を memcpy が無視していたことです。宛先の確保と解放は必ず宛先自身のcustomMemでなければならないため、修正はこの1箇所です：

```c
void ZSTD_copyDCtx(ZSTD_DCtx* dstDCtx, const ZSTD_DCtx* srcDCtx)
{
    size_t const toCopy = (size_t)((char*)(&dstDCtx->inBuff) - (char*)dstDCtx);
    ZSTD_customMem const dstCustomMem = dstDCtx->customMem;
    ZSTD_memcpy(dstDCtx, srcDCtx, toCopy);  /* no need to copy workspace */
    /* keep the destination's allocator: the destination context was
     * allocated (and will be freed) with its own customMem. */
    dstDCtx->customMem = dstCustomMem;
}
```

memcpy前に宛先のcustomMemを退避し、コピー後に復元する（実質6行）だけの修正です。対になる `ZSTD_copyCCtx()`（圧縮側）にも同様の構造があり得ますが、workspace割当がmemcpy後のcustomMemで一貫して行われるため、構造体自体の分配不一致のみ残ります。ここは1件の明確なバグ修正の中に範囲を広げないため、PR本文に範囲外として明記しました。

## 影響範囲と注意

- `ZSTD_copyDCtx` / `ZSTD_copyCCtx` は [zstdのヘッダ](https://github.com/facebook/zstd/blob/dev/lib/zstd.h) で既にdeprecated扱いです（`ZSTD_DEPRECATED("This function will likely be removed in the next minor release. It is misleading and has very limited utility.")`）。新規コードではこのAPIを使うべきではありません。
- 一方で、内部実装の `ZSTDMT` やサードパーティの既存コードがまだこの構造に乗っている可能性があります（実際、このバグの基になったIssueも並列展開（multithreaded decompress）まわりの実装から発見されていました）。「deprecatedだから放置」ではなく、deprecated済みでも現用コードが参照しているうちはここを直す価値があります。
- Cのstructレベルでのmemcpy/逆参照は「アロケータのような状態を持つ構造体」で粗いミスが起きやすい好例です。memcpyの範囲に「コピーしてよいフィールドと、宛先の状態を表すためコピーしてはいけないフィールド」が混ざっていないかを最初に確認すると、この種のミスは防げます。

## 学び

1. **構造体memcpyで「宛先固有のまま維持すべきフィールド」を意識する**。customMemは生成時点で固定され、src側の値をdstへ引き継ぐと「他人のアロケータでdstを解放する」事態になる典型。
2. **回帰テストでは、修正があるべき場所を明示的にFAILさせる**（stashでlib修正を一時退避してテストだけが落ちることを確認）。
3. **abortでなくflagで観測する負の証明パターン**は、allocator系の不整合を安全に、しかもCI上で安定して検証するのに有効。
4. deprecatedなAPIも現用コードから参照されているうちはバグが隠れているので、調査対象から外さない方がよい。

## 参考資料

- [facebook/zstd PR #4837 - decompress: keep the destination's allocator in ZSTD_copyDCtx](https://github.com/facebook/zstd/pull/4837)
- [zstd lib/zstd.h（ZSTD_copyDCtxのdeprecation注記）](https://github.com/facebook/zstd/blob/dev/lib/zstd.h)
- facebook/zstd v1.6.0 近傍の `lib/decompress/zstd_decompress.c`・`tests/fuzzer.c`（PR時点のdiffとほぼ同一の内容を手元のcloneで確認済み）
