---
title: "OpenClawプラグイン導入失敗後に設定だけ残ったときの安全な復旧"
emoji: "🧯"
type: "tech"
topics: ["openclaw", "troubleshooting", "security", "plugin"]
published: true
---

OpenClawへコミュニティ製プラグインを導入したところ、インストール処理は失敗してプラグイン本体がロールバックされた一方、プラグインが追加したモデル設定だけが残る事象に遭遇しました。

この状態でGatewayを再起動すると、存在しないローカルプロバイダーが既定モデルとして選ばれ、新しい会話まで失敗する可能性があります。本記事では、**再インストールを繰り返す前に「ファイル・登録・設定」を分けて確認し、変更されたキーだけを戻す**手順をまとめます。

## 検証環境と症状

確認した環境はOpenClaw `2026.9.3`です。あるコミュニティ製モデルルータープラグインの導入中に、次のエラーで処理が止まりました。

```text
config changed since last load
```

一見すると「導入はほぼ完了し、再起動すればよい」ようにも見えます。しかし、実際には次の食い違いがありました。

- プラグインの配置先ディレクトリは空
- `openclaw plugins info <plugin-id>` は `Plugin not found`
- 既定モデルはプラグイン提供のモデルへ変更済み
- そのモデルのプロバイダー設定も残存

つまり、状態は次のようになっていました。

```text
プラグイン本体: なし
プラグイン登録: なし
既定モデル設定: あり
プロバイダー設定: あり
```

ここで重要なのは、**インストールコマンドの終了状態だけでは、実行時に使われる設定の整合性まで証明できない**ことです。

## 最初に再起動しない

OpenClaw公式ドキュメントでは、プラグインのインストールや更新後にGatewayの再起動が必要になる場合があると案内されています。しかし、導入がエラーで終わった場合は、その前に次の3層を確認したほうが安全です。

1. 配布ファイルが存在するか
2. OpenClawのプラグイン登録に存在するか
3. プラグインが参照する設定だけが残っていないか

今回のように既定モデルだけが変更されていると、再起動後の新規セッションが到達不能なローカルURLへ接続し続ける可能性があります。先に設定を確認し、少なくとも既定モデルが正常なプロバイダーを向いていることを確かめます。

## 1. プラグイン本体と登録を確認する

まず、CLIがプラグインを認識しているか確認します。

```bash
openclaw plugins info <plugin-id>
openclaw plugins list --verbose
```

正常に導入されているだけではなく、実行中Gatewayへ読み込まれたことまで確かめる場合は、公式ドキュメントが案内するruntime検査を使います。

```bash
openclaw plugins inspect <plugin-id> --runtime --json
```

今回の環境では`plugins info`が`Plugin not found`となり、管理対象の拡張ディレクトリにも本体がありませんでした。この2点から、少なくとも「本体は導入済みだが無効」という状態ではないと判断できました。

## 2. 関連する設定を個別に確認する

次に、既定モデルと対象プロバイダーを別々に確認します。

```bash
openclaw config get agents.defaults.model
openclaw config get models.providers.<provider-id>
openclaw config get plugins.entries.<plugin-id>
```

今回、プラグイン登録は存在しないのに、概念的には次のような設定が残っていました。

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "orphan-provider/auto",
        "fallbacks": []
      }
    }
  },
  "models": {
    "providers": {
      "orphan-provider": {
        "baseUrl": "http://127.0.0.1:<port>/v1"
      }
    }
  }
}
```

`127.0.0.1`だから安全、とは限りません。プラグイン本体がなくサーバーが起動しないなら単純に接続不能になります。また、固定ポートを別プロセスが先に利用している場合、意図しないプロセスへ会話内容を送る設計になっていないかも、ソース監査の対象です。

## 3. バックアップとの差分を「値を漏らさず」確認する

設定ファイル全体をログや記事へ貼るのは危険です。APIキー、トークン、Webhook URLなどが含まれる可能性があるためです。

今回は、導入直前のバックアップと現在の設定を比較し、秘密情報らしいキーの値を伏せる小さなスクリプトを使いました。以下は汎用化した例です。

```python
import json
import re
from pathlib import Path

old = json.loads(Path("before.json").read_text())
new = json.loads(Path("after.json").read_text())
sensitive = re.compile(r"secret|token|password|api.?key|credential", re.I)


def walk(a, b, path=""):
    if isinstance(a, dict) and isinstance(b, dict):
        for key in sorted(set(a) | set(b)):
            child = f"{path}.{key}" if path else key
            if key not in a:
                print("+", child, "<redacted>" if sensitive.search(child) else repr(b[key]))
            elif key not in b:
                print("-", child, "<redacted>" if sensitive.search(child) else repr(a[key]))
            else:
                walk(a[key], b[key], child)
    elif a != b:
        value = "<redacted>" if sensitive.search(path) else f"{a!r} -> {b!r}"
        print("~", path, value)


walk(old, new)
```

この比較で、別用途のローカルモデル設定はインストール以前から必要だった一方、問題のプラグインが追加した差分は次の2種類に絞れると分かりました。

- `agents.defaults.model`の変更
- `models.providers.<provider-id>`の追加

設定ファイル全体を古いバックアップへ戻すと、その間に行った正当な設定変更まで失います。**全体復元ではなく、原因となったキーだけを戻す**ほうが安全です。

## 4. 変更されたキーだけを復旧する

復旧前に、バックアップから元の既定モデルを確認します。そのうえで、対象キーだけをCLIで変更しました。

```bash
openclaw config set agents.defaults.model '"provider/model"' --strict-json
openclaw config unset models.providers.<provider-id>
```

環境によって`agents.defaults.model`が文字列ではなく`primary`と`fallbacks`を持つオブジェクトの場合があります。上の値をそのままコピーせず、**自分のバックアップにあった型と値を復元**してください。

復旧後は再度確認します。

```bash
openclaw config get agents.defaults.model
openclaw config get models.providers.<provider-id>
openclaw plugins info <plugin-id>
```

今回の検証では、次を確認できました。

- 既定モデルが導入前の値へ戻った
- 孤立したプロバイダー設定が消えた
- プラグインは未導入のまま
- 別用途のローカルモデル設定は維持された

## 原因の切り分け

対象プラグインの配布物とOpenClaw側の実装を静的に確認すると、次の競合と整合する挙動でした。

1. OpenClawが設定の基準状態を読み込む
2. プラグイン配布物をステージングする
3. プラグインの初期化処理がOpenClawの設定を直接変更する
4. OpenClawが基準状態から変わったことを検出する
5. `config changed since last load`としてインストールを中断する
6. 配布物はロールバックされるが、先に行われた設定変更が残る

これは今回調べたバージョンとプラグインの組み合わせについての説明であり、すべてのインストール失敗で同じことが起きるとは限りません。ただし、**プラグインの初期化中にホスト設定を直接書き換える設計**は、インストーラー側のトランザクションと競合し得ます。

また、プラグインがアクティブな`session`情報や会話ログの保存ファイルを直接書き換える場合も注意が必要です。OpenClawの現行SDK移行資料では、アクティブセッションの`JSONL`や旧セッションストアを直接操作せず、セッションIDとGateway／SDKのruntime APIを使うよう案内されています。

## 再発防止のチェックリスト

コミュニティ製プラグインを導入するときは、次の順序にすると被害を限定しやすくなります。

1. バージョンまたはコミットを固定する
2. 設定のバックアップを取る
3. manifest、依存関係、起動時処理を読む
4. 設定ファイル、既定モデル、セッションへ書き込む箇所を探す
5. localhostサーバーを起動するなら、認証、ポート競合、fail-closed動作を確認する
6. インストール後に「ファイル・登録・設定・runtime」の4層を検証する
7. エラー時は再インストールや再起動を急がず、孤立した設定を先に確認する

公式ドキュメントも、プラグインのインストールをコード実行と同様に扱い、再現可能な環境では固定バージョンを優先するよう案内しています。

## まとめ

今回のポイントは、ロールバックを「完全に元へ戻った」と決めつけないことです。

- インストール結果だけでなく、本体・登録・設定・runtimeを分けて確認する
- 既定モデルなど起動経路に関わる設定を、再起動前に確認する
- バックアップ全体ではなく、検証できた差分だけを戻す
- 設定差分を共有するときは秘密情報を伏せる
- コミュニティ製プラグインは、localhost通信やセッション操作も含めてコードとして監査する

導入失敗そのものより、失敗後に残った半端な状態のほうが見つけにくいことがあります。復旧では「何がないか」と「何だけ残ったか」を別々に確認するのが有効でした。

## 参考資料

- [OpenClaw公式ドキュメント: プラグイン](https://docs.openclaw.ai/ja-JP/tools/plugin)
- [OpenClaw公式ドキュメント: Plugins CLI](https://docs.openclaw.ai/ja-JP/cli/plugins)
- [OpenClaw公式リポジトリ: Plugin SDK migration](https://github.com/openclaw/openclaw/blob/main/docs/plugins/sdk-migration.md)
