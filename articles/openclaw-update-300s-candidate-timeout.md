---
title: "OpenClawの更新が300秒で失敗する件：候補検証タイムアウトの内部固定と二次障害の切り分け"
emoji: "⏱️"
type: "tech"
topics: [openclaw, troubleshooting, debugging, migration]
published: true
---

`openclaw update` で 2026.9.3 から 2026.9.5 へ更新しようとしたところ、`--timeout` を延ばしても必ず約300秒で失敗する事象に遭遇しました。ログ上ではGatewayが `ready` に到達しているのに失敗するため、最初は原因の見当がつきにくい状態でした。

本記事では、この失敗の原因となった**更新候補検証のタイムアウト上限が実装上300秒に固定されている問題**と、失敗後に出る「schema 17を読めない」エラーが**二次障害であり本番DBは無事**という切り分けを、確認できた範囲でまとめます。

## 症状

- `openclaw update`（2026.9.3 → 2026.9.5）が毎回ほぼ同じ時点（約298秒）で失敗する
- `--timeout 3600` を指定しても失敗時刻が変わらない
- 失敗直前のログでは候補Gatewayが `ready` を出力しており、「起動に失敗した」ようには見えない
- 失敗後に旧バージョンへロールバックすると、今度は「schema 17を読めない」系のエラーが出る場合がある

再現条件は明確で、**エージェントDBが大きい（確認時は約215MB）環境で更新候補の検証を実行する**ことです。DBが小さければ300秒以内に収まるため、この問題は表面化しません。

## 調査：まずタイムアウト値が効いているのか確認する

`--timeout 3600` が効かないなら、どこかで別の上限に丸められているはずです。インストール済みの OpenClaw 2026.9.3 の配布ソース（`dist/` 以下）を静的に確認しました。

その結果、更新候補の検証（canary validation）を起動する箇所で、次のようなコードを確認できました。

```js
timeoutMs: Math.min(params.updateStepTimeoutMs, 3e5)
```

`3e5` は300,000ミリ秒、つまり300秒です。**ユーザーが指定したタイムアウトと300秒の小さい方**が常に使われるため、`--timeout 3600` を指定しても上限は延びません。

さらに検証パイプライン全体が単一の予算（deadline）で管理されていることも確認しました。大まかな流れは次のとおりです。

1. 更新候補の migration rehearsal（`doctor --fix` を候補側で実行）
2. `doctor --lint`
3. `config validate`
4. migration continuation（移行継続処理）
5. 候補バージョンのGatewayを canary モードで起動し、`/startupz` → `/readyz` をポーリングして起動を確認

これらすべてが `deadline = 開始時刻 + 300秒` の中で完結する必要があります。途中でdeadlineを超過すると「Candidate validation deadline exceeded」として検証全体が失敗扱いになります。

## 原因：300秒の予算を大きいDBの移行リハーサルが食い潰す

自分の環境での観測では、予算の内訳はおおよそ次のようでした。

- 約215MBのエージェントDBに対する migration rehearsal: 約145秒
- doctor lint: 約40秒
- 残りでGateway起動確認（`/startupz` 完了待ち）を始めるが、予算がほぼ尽きた状態

つまり候補Gateway自体は正常に起動してログに `ready` と出力しているのに、**検証側のdeadlineが先に切れて失敗**となります。Gatewayの起動が遅いのではなく、その前の検証ステップで予算を使い切ったことが原因です。

この挙動は上流のバグ報告とも一致します。

- [openclaw/openclaw#144858](https://github.com/openclaw/openclaw/issues/144858)「Candidate rehearsal is capped at 300 seconds despite a larger update step timeout」— この記事執筆時点でCLOSED（修正が取り込まれた）
- [openclaw/openclaw#144739](https://github.com/openclaw/openclaw/issues/144739)「2026.9.3 → 2026.9.4 npm update runs 2026.9.3 against schema-17 candidate state」— 執筆時点でOPEN

#144858 が修正済みなので、新しいバージョンへ更新できる環境では、この問題そのものは解消しているはずです。問題は**更新に失敗するからこそ新しいバージョンへ行けない**という鶏と卵の状態です。

## 二次障害：「schema 17を読めない」エラーの正体

更新失敗後、旧バージョンへロールバックすると「schema 17を2026.9.3が読めない」系のエラーが出ることがあります。一見すると本番DBが壊れたように見えますが、そうではありません。

観測と上流issueから整合する説明は次のとおりです。

1. 2026.9.5候補が検証中に**一時的なcanary DB**をschema 17へ移行する
2. 検証がタイムアウトで失敗し、旧バージョンへロールバックする
3. ロールバックした旧版（2026.9.3）が、その一時DB（schema 17）を読もうとして失敗する

つまり失敗しているのは**検証用に作られた一時DBを読む処理**であり、本番DBはschema 16のまま保護されていました。実際、自分の環境でもロールバック後にGateway・RPC・`/startupz`・`/readyz` はすべて正常で、日常利用には影響しませんでした。

切り分けのポイントは次の2点です。

- **本番のstateディレクトリとは別の一時canary DB**が失敗の対象になっていないか確認する
- 失敗後もGatewayの正常系エンドポイント（`/startupz`、`/readyz`）が応答するか確認する

また、更新失敗後に「usable inference route」が表示されモデル設定の問題のように見えることがありますが、これも上記の失敗経路に由来する誤診で、LLMプロバイダー設定自体の障害ではありませんでした。**更新機構の失敗直後に出る診断表示は、別の障害と混同しない**ことが重要です。

## 対処

- 修正が取り込まれたバージョンへ「更新によって」到達できない場合、サポートされている更新経路の修復を待つのが安全です。今回は強制操作（手動でのDB書き換え、スキーマ手動変更など）は行わず、2026.9.3のまま維持することにしました
- 本番DBがschema 16のままで健全なこと（`/readyz`、Gateway動作）を確認してから判断する
- 上流issueの状態を追い、修正版がリリースされたら改めて更新を試す

大きなDBを持つ環境では、更新前に**DBサイズと検証予算の関係**を意識しておくと、この手の「必ず300秒で失敗する」挙動に早く気付けます。

## 学び

- タイムアウトが「指定値を無視して必ず同じ時刻で失敗する」なら、実装内で上限がクランプされている可能性が高い。`Math.min(x, 定数)` のパターンを探すと早い
- 更新系ツールの失敗ログで「起動は成功しているのに失敗」の場合、起動確認そのものではなく、**その前の検証ステップが予算を消費した**可能性を疑う
- 更新失敗後のエラーは一次障害とは限らない。canary・一時ファイル・リハーサル経由の**二次障害**かどうかを、本番データの状態と分けて確認する
- 失敗直後の診断表示（今回は推論経路の警告）は、更新機構の失敗に引きずられた誤診のことがある

## 参考資料

- [openclaw/openclaw#144858: Candidate rehearsal is capped at 300 seconds despite a larger update step timeout](https://github.com/openclaw/openclaw/issues/144858)
- [openclaw/openclaw#144739: 2026.9.3 → 2026.9.4 npm update runs 2026.9.3 against schema-17 candidate state](https://github.com/openclaw/openclaw/issues/144739)
- OpenClaw公式ドキュメント: [Update / CLI](https://docs.openclaw.ai/cli/update)
