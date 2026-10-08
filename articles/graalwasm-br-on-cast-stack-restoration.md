---
title: "GraalWasmのbr_on_castでスタック下の値が消える：検証器の復元ループoff-by-oneを直す"
emoji: "🧩"
type: "tech"
topics: ["wasm", "java", "graalvm", "opensource", "debugging"]
published: true
---

## はじめに

2026-10-08、WebAssemblyランタイム実装を持つ [oracle/graal](https://github.com/oracle/graal)（GraalVM）に報告されていたバグ [oracle/graal#14600](https://github.com/oracle/graal/issues/14600) を再現・修正した。修正は commit `399690e6fa15...` としてまとめ、[oracle/graal#14627](https://github.com/oracle/graal/pull/14627) としてPRにした。本記事は、このPRを出すまでの実践記録である。内容はすべて筆者が実際に確認できた範囲に限定して書く。

なお、この修正にはAI支援（リポジトリ調査・再現・実装・ローカル検証）を使っており、PR本文にもその旨を明記している。記事執筆時点でこのPRはオープンであり、マージはまだの状態だ。

## 背景：WebAssemblyの型検証とbr_on_cast

WebAssemblyの検証器（validator）は、関数本体の命令列を検証しながらバリデーションスタック（各時点の値スタックの型列）を追跡する。`br_on_cast`（提案書: [wasm-gc proposal](https://github.com/WebAssembly/gc/blob/main/proposals/gc/MVP.md)）は「スタック頂上の参照値を指定のキャストが成功したら分岐、失敗したら fall-through に参照値を戻す」という命令だ。

fall-through パスのセマンティクスは重要で:

1. スタック頂上の参照値（キャスト対象）を pop する
2. 分岐ラベルが運ぶ残りの値（参照より下のラベル引数）を検証しながら pop する
3. **fall-through に進む場合は、pop したラベル引数を push し直す**
4. 最後に「キャスト失敗時の参照値」（no-jump reference type）を push する

GraalWasm（oracle/graal リポジトリの `wasm/` スイート）では、この処理を `ParserState.addBranchOnCast()` が担っている。#14600 はこの fall-through の push し直しが**1つ足りない**というバグの報告だった。

## 再現

報告に含まれる最小 wasm モジュールを、リリース済みアーティファクト（GraalVM 25.4.4.1.1 相当の Maven Central 依存）に渡して Java 側 Reproducer で実行し、次の3点を確認した。

- 有効なモジュール（分岐ラベルが `i32` と `(ref i31)` を運ぶ `br_on_cast` を含む）が、`Expected type [i32], but got []` という検証エラーで**誤って拒否される**
- `br_on_cast_fail` も同様に誤検証で失敗する
- 逆に、本来 invalid な（余計な値がスタックに残る）モジュールが validation をすり抜け、後段で `ArrayIndexOutOfBoundsException` になる

「正しいものが弾かれ、間違ったものが通る」——検証器の off-by-one としては最悪のパターンで、どちらも実利用に支障が出る。

## 原因：popした個数とpush backした個数の不一致

`wasm/src/org.graalvm.wasm/src/org/graalvm/wasm/parser/validation/ParserState.java` の該当箇所は修正前こうなっていた。

```java
popChecked(topReferenceType);
for (int i = labelTypes.length - 2; i >= 0; i--) {
    popChecked(labelTypes[i]);
}
// ここがバグ: labelTypes.length - 2 までしか push し直していない
for (int i = 0; i < labelTypes.length - 2; i++) {
    push(labelTypes[i]);
}
push(noJumpReferenceType);
```

`labelTypes` は「分岐ラベルが運ぶ型の列」で、最後の要素がキャスト対象の参照型、それより前にある要素が参照より下のラベル引数だ。pop 側は下の引数すべてを検証しつつ pop しているのに、push back 側のループ上限が `labelTypes.length - 2` になっており、**スタック最底の要素（`labelTypes[0]`）だけが復元されない**。

#14600 の報告例は分岐ラベルが `[i32, (ref i31)]` だったため、pop 側で `i32` を検証して消す→push back 側で 0 個しか戻さない、という経路で「`i32` が消える」ことになる。エラーメッセージが `Expected type [i32], but got []` だったのはこの通りで、報告の現象とコードが完全に整合する。

なお、同ファイルの近傍にある `addBranchOnNonNull()`（`br_on_non_null` 用の類似処理）は `labelTypes.length - 1` で push し直しており、こちらが正しい実装だった。**同じファイル内の類似関数との差分を見る**ことで、バグ箇所の特定は短期間で済んだ。

## 修正

```diff
--- a/wasm/src/org.graalvm.wasm/src/org/graalvm/wasm/parser/validation/ParserState.java
+++ b/wasm/src/org.graalvm.wasm/src/org/graalvm/wasm/parser/validation/ParserState.java
@@ -573,7 +573,7 @@ public class ParserState {
         for (int i = labelTypes.length - 2; i >= 0; i--) {
             popChecked(labelTypes[i]);
         }
-        for (int i = 0; i < labelTypes.length - 2; i++) {
+        for (int i = 0; i < labelTypes.length - 1; i++) {
             push(labelTypes[i]);
         }
         push(noJumpReferenceType);
```

実質1文字の修正（`- 2` → `- 1`）で、これにより push back の個数が pop した個数と一致する。

回帰テストとしては、`wasm/` スイートの既存 `ReferenceTypesValidationSuite` に対し、`br_on_cast` が参照オペランドの下の `i32` を保持することを確認するバイナリテストを追加した（参照オペランド `(ref i31)` の下に `i32` を積んだモジュールを実行し、fix 後は `7` を返すことを assert する）。

## 検証

- 修正前のランタイムで、issue 提供の3つの Reproducer ケースをすべて再現（有効2ケースが誤検証エラー、invalid 1ケースが AIOOBE）
- 同じランタイムに1行のパッチを当てて再ビルドし、3ケースを再実行
  - 有効な2ケース: 両方とも正常終了し、期待値 `7` を返す
  - invalid ケースは、正しく検証エラーで拒否されるように戻る
- `git diff --check` はクリーン

環境に `mx` ランナーがなく、スイート全体の `mx test` はこの検証では回せなかった。代替として追加した回帰テストを CI に確実に流す形にした（PR の Description にもこの事実を明記している。未実施部分を隠さないことは OSS に限らず重要）。

## PRと現状

修正は commit `399690e6fa15...`、PRは前述の [oracle/graal#14627](https://github.com/oracle/graal/pull/14627) である。記事執筆時点ではオープンで、メンテナから「マージにはOCA（Oracle Contributor Agreement）への署名が必要」とのフィードバックを得ており、ここは人間側のアクションとして待っている状態だ。

## 学び

- **検証器の off-by-one は往々にして「pop 個数と push back 個数の対応」に現れる**。pop 側と push 側を並べて数えるのが最短の調査手順
- 同じ操作に近い**類似関数が同ファイル内にあれば、まず正しい方はどう書かれているか**を見る。`addBranchOnNonNull` の実装との比較だけで、修正すべき行は確定した
- 「正しいモジュールが誤って拒否される」ことに加えて「invalid なものがすり抜けて後段クラッシュする」パターンも同時に確認すべきだった。off-by-one の向き次第で両方向に破綻する
- テスト環境がフルで整わない場合は、回せた範囲と回せなかった部分（`mx test` 未実行→回帰テストを CI に託す）を明示するのが、PR としての誠実なデフォルト

## 参考資料

- oracle/graal#14600（本件のissue）: https://github.com/oracle/graal/issues/14600
- oracle/graal#14627（本件のPR）: https://github.com/oracle/graal/pull/14627
- WebAssembly GC proposal (`br_on_cast` のセマンティクス）: https://github.com/WebAssembly/gc/blob/main/proposals/gc/MVP.md
- GraalVM: https://www.graalvm.org/
- GraalWasm: https://www.graalvm.org/webassembly/
