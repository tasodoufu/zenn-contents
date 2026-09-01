---
title: "CI更新で露呈した設定ファイル相対パスの落とし穴"
emoji: "👀"
type: "tech"
topics: ["githubactions", "go", "ci", "oss"]
published: true
---

依存ライセンス検査ツールを更新したところ、CIが突然 `go.mod: no such file or directory` で失敗しました。

調べると、更新版の回帰ではなく、以前からあった設定ミスが旧版の偶然の挙動で隠れていました。本記事では Apache SkyWalking Eyes（License-Eye）を例に、設定ファイル内の相対パスを調査するときの切り分け方を説明します。

## 背景

GitHub Actionsから、リポジトリ直下の `go.mod` に含まれる依存関係のライセンスを検査していました。設定ファイルはリポジトリ直下ではなく `.github/` に置かれています。

```text
repository/
├── go.mod
└── .github/
    └── licenserc.yml
```

設定は次のようになっていました。

```yaml
dependency:
  files:
    - go.mod
```

SkyWalking Eyesを0.8.0から0.9.0へ更新すると、依存ライセンス検査が次の形で失敗します。

```text
INFO  Loading configuration from file: .github/licenserc.yml
ERROR open .../.github/go.mod: no such file or directory
ERROR one or more errors occurred checking license compatibility
```

エラーが更新直後に出たため、最初は0.9.0の回帰にも見えます。しかし、ログには期待したリポジトリ直下の `go.mod` ではなく、`.github/go.mod` が現れています。

## 再現条件

問題のある設定を `.github/licenserc.yml` に置き、0.9.0のCLIで検査します。

```bash
go install github.com/apache/skywalking-eyes/cmd/license-eye@v0.9.0
license-eye -v info -c .github/licenserc.yml dependency check
```

手元では終了コード1となり、存在しない `.github/go.mod` を開こうとして失敗しました。

ここで重要なのは、シェルのカレントディレクトリだけを見て「`go.mod` は存在する」と判断しないことです。設定を読み込んだ後のパスが、どの場所を基準に解決されるかを確認する必要があります。

## 原因1: `dependency.files` は設定ファイル基準

0.9.0の `ConfigDeps.Finalize()` は、絶対パスでない各ファイルを次のように処理します。

```go
config.Files[i] = filepath.Join(filepath.Dir(configFileAbsPath), file)
```

つまり、`.github/licenserc.yml` 内の `go.mod` はリポジトリ直下ではなく、次のパスになります。

```text
.github/go.mod
```

GitHub Actionsの `working-directory` や、コマンドを実行したディレクトリを変更しても、この規則は変わりません。基準は設定ファイル自身のディレクトリです。

## 原因2: 旧版では設定ミスが偶然隠れていた

では、なぜ0.8.0では通っていたのでしょうか。

0.8.0のGoモジュール解決処理は、指定された `go.mod` の親ディレクトリへ移動してから `go mod download` を実行していました。

```go
if err := os.Chdir(filepath.Dir(goModFile)); err != nil {
    return err
}

goModDownload := exec.Command("go", "mod", "download")
```

`.github/` 自体は存在するため、ディレクトリ移動は成功します。その後に実行されるGoコマンドは、親ディレクトリを探索してリポジトリ直下の `go.mod` を見つけられます。

```bash
cd .github
go env GOMOD
```

このため、存在しない `.github/go.mod` を設定していたにもかかわらず、実際には親にあるモジュールが検査されていました。

一方、0.9.0では処理の先頭に `validateGoModFile()` が追加され、指定パスを `os.ReadFile()` で直接検証します。

```go
func validateGoModFile(file string) error {
    content, err := os.ReadFile(file)
    if err != nil {
        return err
    }
    // moduleディレクティブの検証
}
```

その結果、以前から存在した設定ミスが、曖昧な探索に進む前に検出されるようになりました。更新版が壊したのではなく、更新版が不正な前提を可視化した形です。

## 解決

設定ファイルの場所から見た相対パスを指定します。

```diff
 dependency:
   files:
-    - go.mod
+    - ../go.mod
```

`filepath.Join()` により `.github/../go.mod` はリポジトリ直下の `go.mod` に正規化されます。

設定ファイルをリポジトリ直下へ移す方法もありますが、CI側の参照先まで変更する必要があります。今回の問題には1行の修正で十分です。

なお、同じ設定ファイル内にあるヘッダー検査用のglobまで機械的に `../` 付きへ変えてはいけません。設定項目ごとにパス解決の実装が異なる可能性があるため、失敗した項目の仕様とコードを個別に確認します。

## 検証

修正前後の設定を0.9.0で実行し、終了コードを比較しました。

```bash
license-eye -v info -c .github/licenserc-before.yml dependency check
echo $?  # 1

license-eye -v info -c .github/licenserc.yml dependency check
echo $?  # 0
```

修正前は `.github/go.mod` の不在で失敗し、修正後は同じバージョンのCLIで成功しました。

アプリケーションコードや `go.mod` 自体は変更していません。検査対象を、旧版が偶然見つけていたものと同じリポジトリ直下のファイルへ明示的に向けただけです。

## 学び

ツール更新後のCI失敗を調べるときは、次の順序が有効でした。

1. エラーに出た完全なパスと、期待するパスを比較する
2. 相対パスの基準が、カレントディレクトリ・設定ファイル・ワークスペースのどれかを実装で確認する
3. 旧版と新版の処理順を比較する
4. 修正前後を同じ新版で実行し、終了コードと対象パスを確認する
5. 「新版の回帰」と「新版が既存設定ミスを検出した」を分けて考える

特に、旧版で通っていたことは設定が正しかった証拠にはなりません。親ディレクトリ探索、暗黙のデフォルト、エラーの握りつぶしなどが、誤設定を長期間隠すことがあります。

## 参考資料

- [Apache SkyWalking Eyes](https://github.com/apache/skywalking-eyes)
- [ConfigDeps.Finalize（v0.9.0）](https://github.com/apache/skywalking-eyes/blob/v0.9.0/pkg/deps/config.go#L55-L68)
- [GoModResolver（v0.8.0）](https://github.com/apache/skywalking-eyes/blob/v0.8.0/pkg/deps/golang.go)
- [GoModResolver（v0.9.0）](https://github.com/apache/skywalking-eyes/blob/v0.9.0/pkg/deps/golang.go)
- [ORAS issue #1359](https://github.com/oras-project/oras-go/issues/1359)
