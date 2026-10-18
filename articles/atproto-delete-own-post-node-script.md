---
title: "Blueskyの投稿をAT Protocolで削除する最小Node.jsスクリプトを作った"
emoji: "🧹"
type: "tech"
topics: ["bluesky", "atprotocol", "nodejs", "javascript"]
published: true
---

こんにちは、豆腐たそです。Bluesky（AT Protocol）で自分の過去の投稿をコマンド1発で削除する、依存ライブラリゼロのNode.jsスクリプトを作ったので、要点をまとめます。投稿ボットを運用していると「テスト投稿を誤って本番アカウントに流してしまった」という事故が起きがちで、実際にその事故から復旧するために書いたものです。そのまま使える再現手順と、書き換え系APIの要点を解説します。

## 背景：投稿ボット運用での「誤爆」問題

Blueskyで自動投稿（cronでの定期投稿など）を運用していると、開発中のテスト投稿がうっかり本命アカウントに流れてしまうことがあります。筆者環境でも、診断作業中のテスト投稿5件が本番アカウントに流れる事故が実際に発生しました。Web UIからも1件ずつ削除できますが、以下の理由でスクリプト化が有効です。

- 複数件を確実に（人力ポチポチより漏れなく）削除したい
- 今後のために「削除スクリプトは投稿スクリプトとセットで先に用意する」体制を作りたい
- 依存を増やさず、Node.js標準モジュールだけで動かしたい

## 前提：AT Protocolの削除はレコード削除である

Blueskyの投稿は、PDS（Personal Data Server）上のリポジトリに記録されたレコードです。削除に使うのは次の2つのXRPCエンドポイントです。

1. `com.atproto.server.createSession` — アカウントパスワードでアクセストークンを取得
2. `com.atproto.repo.deleteRecord` — コレクションとレコードキーを指定してレコード削除

投稿の削除に必要なのはリポジトリ（自分のDID）と、`app.bsky.feed.post` コレクション内の **rkey（レコードキー）** です。投稿URLの末尾部分がそのままrkeyになります。

```
https://bsky.app/profile/<did または handle>/post/3mxxxxxxxxxxxxxxx
                                                    ^^^^^^^^^^^^^ これがrkey（22文字）
```

## コード全体

標準モジュールのみで約60行です。HTTPはcurlのラッパー関数で行います（後述）。

```javascript
#!/usr/bin/env node
// Delete an own post by rkey. Usage: node bluesky-delete.mjs <rkey>
import fs from 'node:fs';
import crypto from 'node:crypto';
import { execFileSync } from 'node:child_process';

const IDENTIFIER = process.env.BSKY_IDENTIFIER || 'yourhandle.bsky.social';
const PASSWORD_FILE = process.env.BSKY_PASSWORD_FILE || '/path/to/pas.txt';
const PDS = process.env.BSKY_PDS || 'https://bsky.social';

function loadPassword() {
  let raw = fs.readFileSync(PASSWORD_FILE, 'utf8');
  const m = raw.match(/bsky-symmetric-v1:([A-Za-z0-9+/=]+)/);
  if (m) {
    const iv = Buffer.from(m[1].slice(0, 32), 'hex');
    const payload = Buffer.from(m[1].slice(32), 'base64');
    const dk = crypto.scryptSync('bluesky-local', '', 32);
    const d = crypto.createDecipheriv('aes-256-cbc', dk, iv);
    raw = Buffer.concat([d.update(payload), d.final()]).toString('utf8');
  }
  return raw.trim();
}

function fetchSync(url, opts = {}) {
  const args = ['-sS', '-w', '\n%{http_code}', url];
  if (opts.method) args.push('-X', opts.method);
  for (const [k, v] of Object.entries(opts.headers || {})) args.push('-H', `${k}: ${v}`);
  if (opts.body) args.push('--data-binary', opts.body);
  const out = execFileSync('curl', args, { encoding: 'utf8', maxBuffer: 10 * 1024 * 1024 });
  const idx = out.lastIndexOf('\n');
  return { status: parseInt(out.slice(idx + 1), 10), text: out.slice(0, idx) };
}

const session = JSON.parse(fetchSync(`${PDS}/xrpc/com.atproto.server.createSession`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ identifier: IDENTIFIER, password: loadPassword() }),
}).text);

const rkey = process.argv[2];
if (!rkey) { console.log('Usage: node bluesky-delete.mjs <rkey>'); process.exit(1); }
const res = fetchSync(`${PDS}/xrpc/com.atproto.repo.deleteRecord`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${session.accessJwt}` },
  body: JSON.stringify({ repo: session.did, collection: 'app.bsky.feed.post', rkey }),
});
console.log('HTTP', res.status, res.text.slice(0, 200));
```

`loadPassword()` の暗号化解除ブロックは環境固有の保存形式に対応する部分なので、パスワードを平文ファイルに置けるなら中身は不要です（ファイル読んで `trim()` するだけで動きます）。実際の運用では認証情報はファイル経由で渡し、コマンドライン引数やシェル履歴に載せないようにしています。

## 使い方

```bash
node bluesky-delete.mjs 3mxxxxxxxxxxxxxxx
# => HTTP 200 {}
```

削除したい投稿のURLからrkeyをコピーして渡すだけです。連続削除する場合はrkeyを列挙してループを回します。

## 実装の要点

### 1. createSessionでアクセストークンを取る

`com.atproto.server.createSession` にハンドル（またはDID）と**アプリパスワード**をPOSTすると、`accessJwt` とDIDを含むセッションJSONが返ります。アプリパスワード（`xxxx-xxxx-xxxx-xxxx`形式）はアカウント設定から発行でき、通常パスワードの代わりにこれを使うのが安全です。

### 2. deleteRecordに必要なのはrepo / collection / rkeyの3つ

`com.atproto.repo.deleteRecord` は、公式のLexicon定義で `repo`・`collection`・`rkey` の3つが必須のJSONをPOSTします（公式リポジトリの `deleteRecord.json` でrequired確認済み）。

```json
{
  "repo": "<自分のDID>",
  "collection": "app.bsky.feed.post",
  "rkey": "<削除対象のrkey>"
}
```

成功時はHTTP 200で空オブジェクト `{}` が返ります。`repo` はセッションレスポンスの `did` をそのまま使い、他人のレポジトリを指定しない限り、自分のレコードしか消えません。

### 3. Node.jsから同期的にHTTPを叩く

今回のスクリプトはcronやワンショット実行を想定しているため、`fetch` のPromiseチェーンを書かず、curlを `execFileSync` で包んで同期的に扱う構成にしました。ポイントは、curlの `-w '\n%{http_code}'` でレスポンス末尾にステータスコードを付け加え、最終行と本体を分割してパースする部分です。これで本文がJSONでも改行を含まない限り安全に分離できます。

```javascript
const out = execFileSync('curl', args, { encoding: 'utf8', maxBuffer: 10 * 1024 * 1024 });
const idx = out.lastIndexOf('\n');
return { status: parseInt(out.slice(idx + 1), 10), text: out.slice(0, idx) };
```

## 検証

実際のアカウントで次を確認しました。

- 誤爆で流したテスト投稿5件をrkey指定で順次削除し、全件 `HTTP 200` で完了。アプリとWebからも消失を確認。
- 存在しないrkeyを渡すと400系エラーが返ることも確認し、冪等に見える再実行で「削除済み」が判別できることを確認。

なお、削除はレコード削除であり、リプライや引用で既に拡散された内容の完全な撤回にはなりません。公開範囲の取り扱いは従来どおり慎重に。

## 学び

- AT Protocolの投稿削除は「`app.bsky.feed.post` コレクションのレコード削除」。`com.atproto.repo.deleteRecord` にDID・コレクション・rkeyを渡すだけのシンプルな構造
- 投稿URLの末尾セグメントがそのままrkeyになるので、URLから機械的に変換できる
- 投稿スクリプトを運用するなら、**削除スクリプトを先に用意しておく**のが事故対応の定石。ステージングのつもりの操作が本番を向いていないか、仕組み側で分かるようにするのが次の一手
- Node.js標準モジュール＋curlラッパーで、依存ゼロのXRPCクライアントが十分実用的に書ける

## 参考資料

- [atprotoリポジトリ: com.atproto.repo.deleteRecord のLexicon定義](https://github.com/bluesky-social/atproto/blob/main/lexicons/com/atproto/repo/deleteRecord.json)
- [atprotoリポジトリ: com.atproto.server.createSession のLexicon定義](https://github.com/bluesky-social/atproto/blob/main/lexicons/com/atproto/server/createSession.json)
- [AT Protocol - HTTP API (XRPC) 仕様](https://atproto.com/specs/xrpc)
- [Bluesky公式ドキュメント](https://docs.bsky.app/)
