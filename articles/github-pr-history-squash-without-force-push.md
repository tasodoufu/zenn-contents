---
title: "OSS PRで「履歴をsquashして」と言われたら：tree同一の単一コミット再作成と検証手順"
emoji: "🧹"
type: "tech"
topics: [git, github, oss, opensource, rust]
published: true
---

OSSのPRを長期戦にしていると、レビュアーから「rebaseして履歴を整理してほしい」と言われ、さらにそれが「merge commitsを外してsquashしてほしい」に変わることがあります。著者は uutils/coreutils の `sort` 修正PR（[uutils/coreutils#14388](https://github.com/uutils/coreutils/pull/14388)）でこの局面に遭遇し、**tree が同一であることをSHAで証明しながら単一コミットへ書き直した**ので、その手順と検証方法をまとめます。

## 経緯

このPRで起きた実際のやり取りの流れは次の通りです。

1. 初回のレビューで「履歴を整理・rebaseしてほしい」という要求があった（2026-09-24のコメント）。
2. 対応時にmainの更新が重なり、PRブランチにmerge commitが溜まった。
3. 詰めの段階でrebaseを重ねる間に、著者は誤って別の新規PR（#14898）を作って旧PRをcloseしてしまった。
4. レビュアーから「**既にPRを出しているなら、rebaseが必要だからといって新しいPRを作らず、既存PRのブランチをrebaseせよ**」という指摘が入った（2026-09-28 07:45Z）。
5. 最終的に「**merge commitsをdropし、squashしてクリーンな履歴にしてほしい**」という要求がきた（2026-09-29 08:56Z）。

5番の対応として行ったのが、**PRブランチを「merge commitを含まない単一コミット」に書き直す**操作です。ヘッド更新を伴う履歴書き換えはpushにforce系の操作が必要になるため、「diffが全く変わっていないこと」を事前に確定させておくことが重要になります。

## 前提チェック：treeが同一ならdiffはゼロ

Gitのコミットには「ツリー」（その時点のファイル全体スナップショット）が紐づきます。`git diff`が空になるなら、コミットをどう組み直しても差分内容は1バイトも変わりません。書き換え前には必ずこれを確認します。

著者のケースでは、PRヘッドの単一コミット `4d8f05f` を、base側の `3f11648`（merge commitを含まない親）の直上に再作成しました。どちらのtreeも、PRの本体修正（`src/uu/sort/src/ext_sort/threaded.rs` と `tests/by-util/test_sort.rs`、74行追加・4行削除）を含む同一スナップショットです。

## 手順：tree同一の単一コミットを再作成する

親を確認します。`4d8f05f` の親がbase相当のコミット `3f11648` だと分かります。

```bash
git log -1 --format='%P' 4d8f05f
# → 3f116488698a57bde0497126a5073a9500892fa2
```

書き換え後のブランチには、現行main相当の親から新ブランチを切り、その上にツリーを持ち上げます。

```bash
# ベースから新ブランチを切る
git switch -c fix/squash-clean 3f11648

# ワークツリーをPRヘッドのツリーと一致させる
git checkout 4d8f05f -- .

# ツリーの確認（空なら同一）
git diff 4d8f05f

# 単一コミットとして作成
git commit -m "sort: propagate temporary file write errors"

# ツリー同一性をSHAで確定証明
git rev-parse HEAD^{tree}
git rev-parse 4d8f05f^{tree}
```

`git rev-parse HEAD^{tree}` が元ヘッドのtreeと一致していれば、**diffは確実に変わっていない**ことがSHAレベルで証明された状態です。

## pushとPR検証

forkのブランチへ `git push --force-with-lease` で更新します。ツリーが同一なので、レビュー側は diff の内容確認をやり直す必要がありません（リモートの「新しい単一コミット」）。

push後はGitHub APIでPRの状態を確認しました。

```bash
gh api repos/uutils/coreutils/pulls/14388 --jq '{commits: .commits, head_sha: .head.sha, mergeable_state: .mergeable_state}'
```

確認できた項目:

- `commits: 1`（merge commitが消えて単一コミットになった）
- `head_sha: 4d8f05fd...`（新しい単一コミットがヘッドへ反映された）
- `mergeable_state: blocked`（レビュー承認待ちであり、履歴の問題ではなく「レビュー待ち」の正常な状態）

PRコメントには、親・ツリーの同一性と検証コマンドの結果（`cargo test -p uu_sort --lib`（31 passed）、`cargo fmt --all -- --check`、`cargo clippy -p uu_sort --all-features --all-targets --tests`がクリーン）を添え、レビュアーの要求（merge commitの除去とsquash）を満たしたことを明示しました。

## 学び

- **履歴整理は「ファイル内容の変更」ではなく「コミットの再配置」**です。`git rev-parse HEAD^{tree}` を比較すれば、diffが変わっていないことをSHAレベルで証明できます。
- **merge commitはレビューの観点では「見にくさ」の問題**です。CI・レビューがmerge commitを嫌がるリポジトリでは、PR画面のコミット数を常に意識しましょう。
- **既存PRブランチの書き換えを避けるために新規PRを作るのはアンチパターン**になり得ます。今回も「置換PRを作った方が安全」と判断したものの、レビュアーには「rebaseは既存PRでやるべき」と指摘されました。force-pushの要否でPRを複製する前に、maintainerの運用方針を確認した方が回り道にならないことが実証されました。
- 全体で一貫しているのは**「マウント権限のない統合テストは、成功扱いにせず明示的にskipする」**という回帰テスト設計の原則です。tmpfsのマウントできない環境のCIでも有効でした。

## 参考資料

- [uutils/coreutils Pull Request #14388](https://github.com/uutils/coreutils/pull/14388)
- [uutils/coreutils Issue #14377](https://github.com/uutils/coreutils/issues/14377)
- [git rev-parse公式ドキュメント](https://git-scm.com/docs/git-rev-parse)
- [git diff公式ドキュメント](https://git-scm.com/docs/git-diff)
