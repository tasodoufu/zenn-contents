---
title: "Agent Skillsの設定ミスをJSON Schemaと契約テストで早期発見する"
emoji: "🧰"
type: tech
topics: [ai, githubactions, python, yaml, jsonschema]
published: true
---

Agent Skillsのように、MarkdownとYAMLの組み合わせで動作を定義する仕組みは、構文が正しくても安全上重要な文言が抜けることがあります。

たとえば、デプロイ用スキルから「リモート状態を変更する前に明示的な承認を得る」という手順が消えても、Markdownとしては正常です。そこで今回は、JSON Schemaによる設定ファイルの編集支援と、実際のスキル本文を検査する決定的な契約テストを組み合わせました。

## この記事で扱うもの

検証に使ったのは、Agent Skills向けのオフライン検査ツール [`agent-skill-contracts`](https://github.com/eun7661010/agent-skill-contracts) です。LLMを呼び出さず、契約ファイルに書いたルールを再現可能に検査できます。

このツールは次のような内容を確認します。

- 必須・禁止テキストまたは正規表現
- 必須ファイルと、SKILL.mdからの参照
- frontmatterの必須フィールドとツール宣言
- 個人環境固有の絶対パス
- スキルディレクトリ外へ出るシンボリックリンク

## JSON Schemaをエディタに接続する

契約ファイル自体の入力ミスは、リポジトリに同梱されたJSON SchemaをYAML拡張へ関連付けて減らせます。VS Codeのワークスペース設定（`.vscode/settings.json`）に次を追加します。

```json
{
  "yaml.schemas": {
    "./schema/skill-contract.schema.json": [
      "skill-contract.yaml",
      "skill-contract.yml"
    ]
  }
}
```

YAML拡張が `yaml.schemas` に対応していれば、契約ファイルを開いたときに補完とスキーマ検証が使えます。たとえば `version: 2` や `{ "regex": 42 }` のような値は、契約の実行前に編集画面で気づけます。

JetBrains IDEでは、Settingsの **Languages & Frameworks → Schemas and DTDs → JSON Schema Mappings** から、同じ `schema/skill-contract.schema.json` をプロジェクト内の `skill-contract.yaml` / `skill-contract.yml` に割り当てます。

## 契約で「残すべき挙動」を宣言する

安全なデプロイ用のサンプルでは、概念的に次のような契約を置いています。

```yaml
version: 1
skill: .
rules:
  - id: approval-before-remote-write
    require:
      any:
        - text: explicit approval before changing remote state
        - regex: ask\s+for\s+explicit\s+approval
    forbid:
      - text: skip confirmation
frontmatter:
  required_fields: [name, description]
  required_tools: [Read, Bash]
references:
  required:
    - references/release-safety.md
portability:
  forbid_personal_paths: true
```

ここで重要なのは、一般的なMarkdown文法ではなく、**そのスキルの運用上必要な条件**を検査対象にしていることです。

## 実行と失敗例

リポジトリを取得し、Python 3.10以上の環境でインストールした後、次のように実行します。

```bash
git clone https://github.com/eun7661010/agent-skill-contracts.git
cd agent-skill-contracts
python -m pip install .
skill-contract check examples/safe-deploy --format text
```

確認できた正常系の出力は次のとおりでした。

```text
PASS skill-contract.yaml (skill: .)
Summary: 1 contract(s), 1 passed, 0 failed, 0 finding(s), 0 config issue(s)
```

意図的に壊したサンプルでは、承認文の欠落、`Read` ツール宣言の欠落、`git push --force`、個人用絶対パスが検出され、終了コードは `1` になりました。

```text
FAIL skill-contract.yaml (skill: .)
Summary: 1 contract(s), 0 passed, 1 failed, 4 finding(s), 0 config issue(s)
```

つまり、CIでは「検査結果が失敗したらマージを止める」という単純なゲートとして利用できます。GitHub Actionsでは、公開リポジトリのComposite Actionとして使う方法と、PythonパッケージをインストールしてCLIを実行する方法があります。

## JSON Schemaと契約テストの役割を分ける

両者は似ていますが、検査対象が違います。

- JSON Schema: 契約ファイルの型、値、構造を編集時・実行時に検証する
- 契約テスト: SKILL.mdに必要な承認境界や禁止事項が残っているかを検証する

JSON Schemaだけでは、Markdown本文から安全手順が消えたことまでは分かりません。逆に契約テストだけでは、契約ファイルのスペルミスや型ミスを編集時に見つけにくくなります。設定の構造と本文の意味を別のゲートで検査することで、検出範囲を補えます。

## 導入時の注意点

契約は厳しくしすぎると、表現の変更だけで壊れます。最初から文章全体を固定するのではなく、次のような安全上の不変条件から始めるのが扱いやすいです。

- リモート変更前の承認
- 強制pushや秘密情報の出力の禁止
- 復旧手順へのリンク
- 必要なツール宣言
- ユーザー固有パスの持ち込み禁止

また、契約はランタイムの強制機構ではありません。実行ホストの権限、サンドボックス、承認UI、監査ログと併用し、契約テストを「安全条件の回帰検出」として使うのが現実的です。

## まとめ

Markdownベースのエージェント設定は読みやすい一方、重要な手順が消えても構文エラーにならないことがあります。

- 契約ファイルにはJSON Schemaを関連付ける
- スキル本文の安全条件は決定的な契約テストで検査する
- 正常系と意図的な失敗系をCIで確認する
- テストをランタイムの安全機構と混同しない

この分離により、「設定ファイルは正しいが、安全な運用条件が抜けている」という変更を、レビューや本番実行の前に検出できます。

## 参考資料

- [agent-skill-contracts](https://github.com/eun7661010/agent-skill-contracts)
- [JSON Schema: Combining schemas](https://json-schema.org/understanding-json-schema/reference/combining)
- [Red Hat YAML extension for VS Code](https://github.com/redhat-developer/vscode-yaml)
