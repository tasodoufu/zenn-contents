---
title: "GitHub PagesからnetcatできるCTF基盤をどう作ったか"
emoji: "🦙"
type: "tech"
topics: [ctf, docker, cloudflare, tailscale]
published: false
---

こんにちは、豆腐たそです。

AIアシスタントの**メギ**と一緒に、pwn専用の学習サイト「TofuCTF」を作りました。

https://tasodoufu.github.io/TofuCTF/

フロントエンドはGitHub Pagesです。しかし、pwnではHTTPページを表示するだけでなく、次のような体験が必要です。

```bash
nc tofuctf.tailc3d416.ts.net 10000
```

この記事では、静的サイト、flag判定、進捗同期、脆弱なバイナリの実行環境をどのように分離したかをまとめます。

## 全体構成

現在の構成は次のとおりです。

```text
GitHub Pages
  ├─ 問題一覧・カレンダー
  ├─ 問題バイナリ配布
  └─ Submit UI
          │ HTTPS
          ▼
Cloudflare Worker ── Cloudflare D1
  ├─ Google ID tokenの検証
  ├─ flag hashとの照合
  └─ アカウント別Solved履歴

Internet
  │ raw TCP :10000
  ▼
Tailscale Funnel
  │
  ▼
127.0.0.1:31337
  │
  ▼
専用OSユーザーのrootless Docker
  └─ pwn challenge + /flag
```

役割を分けた理由は、GitHub Pagesへ秘密情報やサーバー処理を持たせないためです。

## GitHub Pagesだけではflagを守れない

静的JavaScriptに正解flagを置くと、ソースやDevToolsから取得できます。ハッシュだけ置いた場合も、flagの強度や実装によってはオフラインで解析されます。

最初はflagの形式だけを確認する自己申告型でしたが、最終的にはCloudflare Workerへ判定を移しました。

WorkerはD1の`challenge_flags`テーブルから問題ごとのflag hashを取得し、送信されたflagのSHA-256と比較します。公開リポジトリに保存するのはランダムな128-bit flagのハッシュだけです。

概念的には次の処理です。

```js
const challenge = await env.DB
  .prepare("SELECT flag_hash FROM challenge_flags WHERE challenge_id = ?")
  .bind(challengeId)
  .first();

const submittedHash = await sha256Hex(flag.trim());
const correct = safeEqual(submittedHash, challenge.flag_hash);
```

flag本体はpwnサーバーの専用領域にだけ置きます。

## ログインを必須にしない

最初はGoogleアカウントごとに異なるflagを生成していました。この方式では起動トークンが必要になり、問題を始めるまでの手順が増えました。

学習サイトでは、不正防止より参加しやすさを優先したいと考え、問題ごとの固定flagへ変更しました。

- 未ログイン：正誤判定できる。Solvedは`localStorage`へ保存
- Googleログイン済み：正解時にD1へSolvedを保存
- 同じGoogleアカウント：別端末でも履歴を同期

ログインは進捗同期のためのオプションであり、問題を解くための入口にはしていません。

## Cloudflareだけでraw TCPは難しかった

Cloudflare WorkersにはTCP Socket APIがありますが、これはWorkerから外部へ接続するためのものです。通常のWorkerを`nc host port`の待受サーバーにはできません。

Cloudflare Containersも候補でしたが、Worker経由のHTTPを中心にした構成です。任意TCPを公開するCloudflare Spectrumは、カスタムTCPの場合Enterprise向けです。

そこで、サーバーで動くpwnサービスをTailscale Funnelのraw TCP転送で公開しました。

```bash
tailscale funnel --bg --tcp=10000 tcp://127.0.0.1:31337
```

FunnelはTailscale Serveと異なり、Tailnet外の利用者も接続できます。利用者側にTailscaleは不要です。

## OpenClawと脆弱コンテナを分離する

pwnサービスを動かすLinuxホストには、個人用AIアシスタントのOpenClawも動いています。意図的に脆弱なプログラムとの同居は慎重に扱う必要があります。

完全な分離が必要なら別VMや別マシンが第一候補です。今回は小規模な個人用環境として、次の防御を重ねました。

- `tofuctf`専用OSユーザー
- rootless Docker
- ホスト側は`127.0.0.1`だけで待受
- ホストディレクトリを問題コンテナへ渡さない
- flagファイルだけをread-only mount
- root filesystemをread-only化
- Linux capabilitiesをすべて削除
- `no-new-privileges`
- メモリ、CPU、PID、ファイルディスクリプタを制限
- コンテナ内は非rootユーザー

実行オプションの中心部分は次のようになっています。

```bash
docker run --detach \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=16m \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --memory 128m \
  --memory-swap 128m \
  --cpus 0.5 \
  --pids-limit 64 \
  --ulimit nofile=128:128 \
  --volume "$secret:/flag:ro" \
  --publish 127.0.0.1:31337:31337 \
  "$image"
```

rootlessコンテナも完全な安全境界ではありません。カーネルやコンテナランタイムの脆弱性は影響します。本格的な公開CTFや不特定多数を対象にする場合は、使い捨てVM、gVisor、Kata Containers、別ホストなど、より強い分離が必要です。

## 固定flagを公開Gitへ置かない

公開tarにDockerfileとflagを入れる構成では、当然ながら答えが見えます。また、ローカルDocker形式では利用者がコンテナ管理者なので、内部のflagを意図的に読むことを防げません。

現在は公開物を次だけに絞りました。

```text
warm-tofu/
├─ README.txt
└─ warm-tofu
```

サーバー側では、初回配備時にflagを生成し、専用ユーザーからしか辿れないディレクトリへ保存します。Workerへ登録するのはSHA-256だけです。

## 日次問題を安全に自動化する

日次ジョブは、単に問題を生成してpushするだけでは危険です。サイトだけ更新され、サーバー配備に失敗すると遊べない問題が公開されます。

そのため、次の順序を設計しました。

1. 前問を読み、復習＋新要素1つの問題を生成
2. バイナリをコンパイル
3. 作者用exploitで意図した解法を検証
4. 秘密情報とtar内容を検査
5. rootless環境へ配備
6. localhostとFunnelの両方で疎通確認
7. exploitで公開サービスからflagを取得
8. Workerの`/api/submit`で正解を確認
9. すべて成功した場合だけGitHub Pagesを更新

配備に失敗した場合はpushしません。自動化では「成功時に進む」だけでなく、「失敗時に不完全な状態を公開しない」ことが重要です。

## 現在の制約

Tailscale Funnelで利用できる公開TCPポートには制約があります。現在は10000番を「今日の問題」に割り当て、過去問はカレンダーとダウンロードに残す設計です。

複数の過去問を同時にリモート公開するには、次の選択肢があります。

- 1ポートのTCP dispatcherで問題を選択
- 問題ごとの短命インスタンスを生成
- CTF専用VMと独自TCP proxyを用意
- KubernetesとSpawnerを導入

利用者が少ない段階では、今日の1問だけを安全に公開するほうが運用しやすいと判断しました。

## おわりに

今回の一番の学びは、CTFサイトとCTFサーバーは別の問題だということでした。

カレンダーやUIはGitHub Pagesで作れます。しかし、pwnらしい`nc`体験、秘密flag、ユーザー進捗、脆弱サービスの隔離には、それぞれ別の仕組みが必要です。

小さく始める場合でも、公開経路と秘密情報と隔離境界を分けて考えると、後から構成を育てやすくなりました。

