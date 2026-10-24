---
title: "OpenClawの自動化がモデル呼び出し前に止まる：inference account未記録の直し方"
emoji: "🔐"
type: "tech"
topics: [openclaw, automation, troubleshooting]
published: true
---

OpenClawの自動化でモデルを指定しているのに、実行するとモデル呼び出し前に失敗することがあります。エラーが `Automation has no recorded inference account` なら、まず疑うのはモデルIDやAPIの障害ではありません。既存ジョブに、どの保存済み認証アカウントで推論するかが記録されていない状態です。

この記事では、モデル選択と認証アカウントの紐付けを分けて調べ、対象ジョブだけに意図したアカウントを明示する手順をまとめます。

## モデル指定とアカウント指定は別

`provider/model` は、どのモデルを使うかを指定します。一方、`provider/model@profile` は、そのモデルをどの保存済み認証プロファイルで使うかを明示します。

そのため、次の2つは同じ意味ではありません。

```text
openai/gpt-4.1
openai/gpt-4.1@work
```

前者はモデルだけの指定です。後者はモデルとプロファイルを組み合わせた指定です。対話セッションで認証できることや、モデルがカタログに存在することだけでは、古い自動化ジョブに認証アカウントが記録済みとは限りません。

OpenClawのトラブルシューティング資料では、`Automation has no recorded inference account` を、保存済みアカウントをまだ採用していない古いジョブのエラーとして説明しています。実行履歴に残った以前のモデル名から、過去に使ったアカウントを推定することもしません。

## まず失敗の種類を見分ける

対象のジョブIDを使い、状態・設定・実行履歴を確認します。

```bash
openclaw automations show <job-id>
openclaw automations runs <job-id> --limit 20
```

エラーが `Automation has no recorded inference account` なら、モデルAPIに届く前のアカウント紐付けを確認します。これは、たとえば `Unknown model` のようなモデル解決エラーや、providerから返る認証エラーとは切り分けて考えます。

また、ジョブの `--account` はチャットへの配信アカウントを指定するものです。推論に使うアカウントとは別なので、配信先アカウントを設定してもこのエラーの修正にはなりません。

## 意図したプロファイルを対象ジョブに明示する

最初に、選ぶモデルが利用可能であること、対象providerの保存済みプロファイルが存在すること、そしてそのプロファイルをこのジョブで使う許可があることを確認します。複数の候補があるとき、名前が `default` だからという理由だけで選ばず、認証先や課金先を決めてよい人に確認してください。

選択が確定したら、対象ジョブだけを編集します。

```bash
openclaw automations edit <job-id> \
  --model "openai/gpt-4.1@work"
```

ここで `openai/gpt-4.1` と `work` は例です。自分の環境で許可された `provider/model@profileId` に置き換えます。OpenClaw公式のCLI資料でも、明示的なアカウント採用にはこの形式でモデルを選択する方法が案内されています。

モデルの切り替えだけでは、既存ジョブのアカウント所有者が自動で別アカウントへ移るとは限りません。profile指定は「このジョブはこの認証を使う」という意図を明確にする操作です。fallbackの追加や別ジョブの一括変更を混ぜず、変更対象を絞ると意図しない認証先への切り替えを防ぎやすくなります。

## 設定反映と実行成功を別々に検証する

編集後はジョブの設定を再表示し、モデル指定が保存されたことを確認します。

```bash
openclaw automations show <job-id>
```

その後、実行履歴で次回の結果を確認します。

```bash
openclaw automations runs <job-id> --limit 5
```

設定のreadbackは「意図した値が保存された」ことの確認であって、「推論が成功した」ことの証明ではありません。外部投稿やメールなど副作用のあるジョブを手動実行する前には、その影響を確認してください。安全なら、手動実行または次の定期実行後に、アカウント未記録エラーが消え、モデル応答まで完了したかを確認します。

## まとめ

- モデルが登録済みでも、自動化に推論アカウントが記録されているとは限らない。
- `Automation has no recorded inference account` は、モデルAPIへの到達前にアカウント紐付けを確認するサイン。
- 配信先の `--account` と推論用の `provider/model@profile` は別の設定。
- 認証・課金先を確認してから、対象ジョブだけに正しいプロファイルを明示する。
- 設定readbackと、次回実行の成功確認は分けて扱う。

## 参考資料

- [OpenClaw公式: Automation troubleshooting](https://docs.openclaw.ai/automation/cron-jobs/troubleshooting)
- [OpenClaw公式: Automation payloads — Inference account continuity](https://docs.openclaw.ai/automation/cron-jobs/payloads#inference-account-continuity)
- [OpenClaw公式: Automations CLI](https://docs.openclaw.ai/cli/cron)
