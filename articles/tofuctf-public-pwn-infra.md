---
title: "GitHub Pagesで配る、ローカルDocker型pwn環境の作り方"
emoji: "🍜"
type: "tech"
topics: [ctf, pwn, docker, github]
published: true
---

こんにちは、豆腐たそです。

AIアシスタントの**メギ**と一緒に、毎日ひと口ずつpwnを学ぶサイト「TofuCTF」を作りました。

https://tasodoufu.github.io/TofuCTF/

フロントエンドはGitHub Pagesです。一方、pwn問題には脆弱なバイナリを実行し、`nc`で接続できる環境が必要です。

以前は自宅ホスト上のサービスをraw TCPで公開していましたが、現在は方針を変えました。問題ごとのDocker環境を配布し、利用者自身のPCでlocalhost起動します。

```bash
./run.sh
nc 127.0.0.1 31337
```

この記事では、静的サイトとローカルpwn環境をどう組み合わせたか、flagをどこまで隠せるのか、日次公開をどう自動化したかをまとめます。

## 全体構成

現在の役割分担は次のとおりです。

```text
GitHub Pages
  ├─ 問題一覧・カレンダー
  ├─ challenge.tar.gz の配布
  ├─ flag hashのローカル照合
  └─ ブラウザ内のSolved記録

利用者のPC
  └─ run.sh
      └─ Docker container
          ├─ pwn challenge
          └─ /flag ← Docker volume

Googleログイン時のみ
  └─ Cloudflare Worker ── D1
      └─ アカウント別Solved履歴
```

GitHub Pagesは問題の案内と配布を担当します。脆弱なプログラムはGitHub上では動かさず、ダウンロード後に各利用者のDockerで実行します。

## 配布物は4ファイル

各問題のtarには、次の4ファイルだけを入れています。

```text
challenge-name/
├─ challenge-name
├─ Dockerfile
├─ README.txt
└─ run.sh
```

操作用シェルは`run.sh`一本です。

```bash
./run.sh       # buildして起動
./run.sh stop  # containerとflag volumeを削除
```

以前は`entrypoint.sh`と`stop.sh`も分けていました。しかし、小さな教材で操作ファイルが増えると、読む側にも生成する側にも負担になります。socatの起動設定はDockerfileの`ENTRYPOINT`へ移し、停止処理は`run.sh stop`へ統合しました。

## localhostだけで待ち受ける

起動時はDockerの公開ポートを`127.0.0.1`へ固定します。

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=16m \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --memory 128m \
  --memory-swap 128m \
  --cpus 0.5 \
  --pids-limit 64 \
  --ulimit nofile=128:128 \
  -p "127.0.0.1:31337:31337" \
  "$image"
```

これでLANやインターネットへ意図せず公開することを避けられます。capability削除、read-only root filesystem、非rootユーザー、リソース制限も重ねています。

ただし、コンテナは完全な仮想マシンではありません。信頼できない第三者バイナリを安全に実行できるという意味ではなく、ここで扱うのは自作した学習用バイナリだけです。

## ローカル配布でflagを完全には隠せない

利用者はDockerホストの管理者です。したがって、volumeやcontainerを意図的に調べる人からflagを完全に隠すことはできません。

TofuCTFでは「競技としての厳密な不正防止」より、「うっかり答えを見ないこと」を目標にしました。

flagのリテラル値は`run.sh`へ書かず、問題IDと配布バイナリのSHA-256から決定的に導出します。

```bash
binary_hash="$(sha256sum "./$slug" | awk '{print $1}')"
flag_hex="$(printf '%s' "tofuctf-local-v1:$slug:$binary_hash" \
  | sha256sum | cut -c1-32)"
flag="TofuCTF{$flag_hex}"
```

生成したflagはDocker volumeへ書き込み、challenge containerにはread-onlyでmountします。これなら`run.sh`を開いただけでflagそのものは見えません。ただし、生成式は公開されているため、意図的に計算すれば答えは分かります。この限界は隠さず、ローカル教材として割り切っています。

## flag判定もローカル優先

`challenges.json`にはflag本体ではなくSHA-256を載せています。Submit時はWeb Crypto APIで入力値をhash化し、ブラウザ内で比較します。

```js
const submittedHash = await sha256Hex(flag.trim());
const correct = submittedHash === challenge.flagHash;
```

未ログインならSolved状態を`localStorage`へ保存します。Googleログイン済みの場合だけ、Cloudflare Workerを通してD1へ進捗を同期します。

この設計では、オフラインでも問題起動とflag判定ができます。Cloudflare側に障害があっても、ローカルのSolved記録は失われません。

## 日次公開をcronで自動化する

日次ジョブは毎朝4時15分に起動し、5時までの公開を目標にしています。

1. `git pull --ff-only`と未コミット変更を確認
2. 前問の復習に新要素を1つ加えた問題を生成
3. コンパイルと作者用exploitを検証
4. Dockerfile、run.sh、README、配布tarを作成
5. shell、JavaScript、JSON、tar内容、flag hashを検査
6. D1へ同期用hashを登録できる場合は登録
7. 関連ファイルだけをcommitしてGitHubへpush
8. GitHub Pagesのdeploy成功を確認

途中の検証に失敗した場合はpushしません。D1同期だけが失敗した場合は、ローカル判定できるサイト公開を止めず、同期未適用として報告します。

問題名は豆腐だけに固定せず、食べ物とpwn用語を掛け合わせます。たとえば`ROP Roll`や`Heap Hotpot`のように、学習テーマが少し伝わり、毎日の問題を覚えやすくする狙いです。

## GitHub Pagesとnetcatの関係

「GitHub Pagesなのにnetcat」という表現は、以前のリモート公開方式ではそのまま当てはまりました。現在は少し違います。

- GitHub Pages：問題を探す、ダウンロードする、flagを判定する
- Docker：脆弱なサービスをローカルで動かす
- netcat：localhostの問題サービスへ接続する

静的サイトがTCPサーバーになるわけではありません。Pagesは入口で、実行環境は利用者のPCです。この分離により、公開サーバーで脆弱なサービスを常時運用する必要がなくなりました。

## おわりに

この構成は、本格的な競技CTFの不正防止には向きません。その代わり、サーバー費用や公開TCPの運用を抱えず、毎日のpwn学習を始めやすくできます。

TofuCTFで優先したのは、強固な採点基盤よりも「ダウンロードして、`./run.sh`を実行し、localhostへ`nc`する」という短い導線です。

小さな個人学習環境なら、秘密を完全に守ろうとするより、どこまで守れて何を割り切るかを明確にするほうが、仕組みを長く育てやすいと感じました。
