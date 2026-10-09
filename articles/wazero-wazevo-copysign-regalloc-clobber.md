---
title: "wazeroのwazevoコンパイラがcopysignを誤コンパイルするバグを直した"
emoji: "🔢"
type: "tech"
topics: ["wasm", "go", "compiler", "regalloc", "oss"]
published: true
---

## はじめに

Go製WebAssemblyランタイム [wazero](https://wazero.io) の最適化コンパイラ（wazevo）が、特定の条件で `f32.copysign` / `f64.copysign` を誤コンパイルするバグを[Issue #2555](https://github.com/wazero/wazero/issues/2555)で見つけ、修正PRを[#2562](https://github.com/wazero/wazero/pull/2562)として出しました。本記事はその原因究明と修正、検証の記録です。

**修正の現状（2026-10-10時点）: PR #2562はopen、マージは未です。** 内容として本記事は検証済みのコミット `c42203eb` に基づきます。

## バグの現象

まず、`f32.copysign` を呼び出すモジュールを考えます。

```wat
(module
  (func $g (param f32) (result f32) local.get 0)
  (func (export "f") (param $p f32) (result f32)
    local.get $p call $g local.set $p   ;; p := g(p)   (a call result)
    local.get $p                        ;; magnitude operand
    local.get $p call $g                ;; sign operand: g(p) ...
    f32.const 0 i32.const 1 select      ;; ... passed through a select
    f32.copysign))                      ;; copysign(p, p)
```

この関数に `1.5` を渡すと、セマンティクス上は `copysign(1.5, 1.5) = 1.5` を返すべきです。しかし、次のGoプログラムで compiler / interpreter の2モードを実行すると：

```go
package main

import (
	"context"
	"fmt"
	"math"
	"os"

	"github.com/tetratelabs/wazero"
	"github.com/tetratelabs/wazero/api"
)

func main() {
	wasm, _ := os.ReadFile("min.wasm")
	ctx := context.Background()
	for _, c := range []struct {
		name string
		cfg  wazero.RuntimeConfig
	}{
		{"compiler", wazero.NewRuntimeConfigCompiler()},
		{"interpreter", wazero.NewRuntimeConfigInterpreter()},
	} {
		r := wazero.NewRuntimeWithConfig(ctx, c.cfg)
		mod, err := r.Instantiate(ctx, wasm)
		if err != nil {
			panic(err)
		}
		res, err := mod.ExportedFunction("f").Call(ctx, api.EncodeF32(1.5))
		if err != nil {
			panic(err)
		}
		fmt.Printf("%-11s f(1.5) = %v\n", c.name, math.Float32frombits(uint32(res[0])))
		r.Close(ctx)
	}
}
```

結果はこのようになります。

```
compiler    f(1.5) = -1.5
interpreter f(1.5) = 1.5
```

つまり **compilerモードだけ符号が反転します**。interpreterは正しく、他のランタイム（wasmtime / V8 / SpiderMonkey / JSC）も正しいため、wazeroのcompilerバックエンド固有の誤コンパイルと分かります。

## 再現条件の抽出

Issue報告者の記述どおり、このバグが顔を出すには全要素が必要です：

- `sign` 側のオペランドが **call結果** であること
- そのcall結果が **`local.set` でローカルに退避** されていること
- `sign` 側が **`select` を通って** `copysign` に渡ること

どれか1つでも欠けると誤コンパイルは顕在化しません。この「全要素が揃ったときだけ起きる」という性質が、発見を難しくしています。

## 原因：下流レイヤーの定数下げ遅延とレジスタ再利用の相互作用

原因は `internal/engine/wazevo/backend/isa/amd64/machine.go` の `lowerFcopysign` にあります。

`copysign` はSSE命令で次のように実装されます（省略形）:

```go
signBitReg := m.c.AllocateVReg(x.Type())
m.lowerFconst(signBitReg, signMask, _64)          // 符号bitマスク
nonSignBitReg := m.c.AllocateVReg(x.Type())
m.lowerFconst(nonSignBitReg, ^signMask, _64)      // 符号以外ゼロマスク

and := m.allocateInstr().asXmmRmR(opAnd, rn, signBitReg)
m.insert(and)   // rn（sign側）から符号bitを抽出
xor := m.allocateInstr().asXmmRmR(opAnd, rm, nonSignBitReg)
m.insert(xor)   // rm（magnitude側）から符号bitを消す
or := ... // ORで合成
```

問題の起点は [#1983](https://github.com/wazero/wazero/pull/1983) の変更で `lowerConstAfterRegalloc` 相当のパスが使われるようになったことです。**符号マスク用の一時レジスタ `signBitReg` / `nonSignBitReg` は、定数マテリアライズのタイミングがレジスタアロケーション後まで遅延します。**

一方で `copysign` のオペランド `rm` / `rn` はレジスタ割り付けが先に行われます。ここで `rm` / `rn` が **call結果が入るxmm0** に割り当てられていると、レジスタアロケータは後から定義されるマスク一時レジスタの実体として **同じxmm0を選べてしまう** のです。

結果として命令列は：

```
movq gp→xmm0                           ; マスク実体がオペランドと同じxmm0に書かれる
andps xmm0, ...                        ; オペランドxmm0が破壊される
```

という具合に、**マスクの実体書き込みがオペランドそのものを上書きします**。マスクは符号bitだけが1なので、これが符号bitの反転として現れ、`1.5` が `-1.5` に化けます。

本日、私の手元でこのチェーンを再確認しました（方法の詳細は「検証」節）：

- 修正前のコミット `e234f6fe` に対し、同じテスト用 `.wasm` を使う検証プログラムで compiler / interpreter それぞれ2回ずつ（f32/f64含め計8ケース）実行 → **compilerは f32/f64 ともに常に `-1.5` / `-2.5` を返し、interpreterはすべて正値**（f32のbitsは `3217031168` = `0xBFC00000`、f64は `-4610560118520545280` と符号bitが立った値）。

## 修正方法：マスク定義前にオペランドを一時レジスタに避難

最も直接的で、wazevoのレジスタ割り付けに手を入れる必要のない修正は、 **マスクレジスタの実体が書かれる前にオペランドを新規のtempにコピーしてエイリアスを切る** ことです。

PRのdiff（12行追加）:

```go
// The mask registers (signBitReg/nonSignBitReg) delay their definition to
// constant-materialization time after register allocation (see
// lowerConstAfterRegalloc), which happens later than the regalloc
// assignment of rm/rn. That allows xmm0 (the call result register) to be
// assigned as the machine register backing the mask registers, silently
// clobbering the copysign operands via the xmm instructions below and
// negating the result. Break that aliasing by first copying rm/rn to
// freshly allocated temporaries. See #2555.
tmpRM := m.copyToTmp(rm.reg())
tmpRN := m.copyToTmp(rn.reg())
rm, rn = newOperandReg(tmpRM), newOperandReg(tmpRN)
```

変更点は **`lowerFcopysign` 冒頭に3行 + コメント9行の合計12行** で、**IRやレジスタアロケータ自体には触れません**。

具体的には：

1. `rm.reg()` / `rn.reg()` を**新しく確保したVReg**に `copyToTmp` でコピー
2. `rm` / `rn` をそのtempを指すオペランドに差し替える

これで、マスクの実体がどのレジスタにマテリアライズされても、 **マスクの書き込み先とオペランドの実体が重ならない** ことが保証されます。

## 検証

追加した統合テスト `internal/integration_test/engine/copysign_localset_test.go`（108行）は、f32/f64 × compiler/interpreter の全組み合わせで動作を検証します。テストは組み込みの `.wasm`/`.wat` テストデータを利用し、`select_case` と `negative_sign` の2つのパターンを確認します。

本日すべての検証を再実行しました：

- **パッチ適用済みコミット `c42203eb` で** 同じ検証プログラムが全8ケース期待どおり（`select_case(1.5) = 1.5`、`negative_sign(1.5) = -1.5`、f64も同様）。compiler / interpreter で結果が一致
- **ビルド** `go build ./...` はエラーなし
- **回帰テスト** `go test ./internal/integration_test/engine/ -run TestCopysignWithSelectAfterCallResultLocalSet -count=1` → `ok`（0.072s）

## 学び

1. **コンパイラのバグは「どのレイヤーが何を後から定義するか」から疑うとうまくいく**
   ソース上は `signBitReg` は「新しく確保したレジスタ」に見えますが、実体のマシンレジスタはレジスタアロケータが**後で**決めます。この「VRegは予約だけ、実体は後で決まる」設計は、定数マテリアライズが後段に遅延すると干渉しやすい、という一般化した理解を得ました。

2. **誤コンパイルの検証はbits表現まで数値で確認する**
   「返り値が違う」だけでは、ほかの符号反転バグと区別できません。今回は **bits表現（f32の `3217031168` = `0xBFC00000` など）まで揃える** ことで、「単に符号が反る」オペランド依存の誤コンパイルと正確に特定できました。

3. **最小修正はレジスタのlivenessを切ること**
   レジスタ割り付け自体に手を入れる代わりに、**オペランドをtempにコピーしてエイリアスを断つ** アプローチは、既存のregallocの挙動を一切変えないため、回帰リスクを最小化できます。

## 参考資料

- [Issue #2555: Compiler miscompiles copysign with select operand](https://github.com/wazero/wazero/issues/2555)
- [PR #2562: wazevo(amd64): fix copysign operand clobbering after regalloc](https://github.com/wazero/wazero/pull/2562)
- [PR #1983（`lowerFcopysign` 周辺の定数 lowering がレジスタ割り当て後まで遅延するようになった経緯）](https://github.com/wazero/wazero/pull/1983)
- wazeroのamd64バックエンド実装 `internal/engine/wazevo/backend/isa/amd64/machine.go`（コミット `c42203eb` 時点）
