---
title: "Rust製sortの外部ソートで一時ファイル書き込みエラーをpanicにしない"
emoji: "🧯"
type: "tech"
topics: [rust, testing, debugging, opensource, linux]
published: true
---

外部ソートは、入力がメモリに収まらないとソート済みの一時ファイル（spill file）を書き出し、最後にそれらをマージします。この一時ファイルへの書き込みが失敗したとき、コマンドはエラー終了すべきであり、`unwrap()` によるpanicで異常終了すべきではありません。

uutils/coreutils の `sort` でこの問題を再現・修正する機会があったため、原因の切り分け、回帰テスト、Rustでのエラー伝播をまとめます。

## 症状：一時領域の失敗がpanicになる

対象のIssueでは、`-T` で指定した一時ディレクトリが容量制限に達したとき、外部ソート中に `sort` がpanicしていました。出力先を`/dev/null`にしても発生するため、出力書き込みではなく、一時ファイルへのspillが失敗していることを切り分けられます。

問題の実装は、各行と区切り文字の書き込み結果を`unwrap()`で処理していました。

```rust
fn write_lines<T: Write>(lines: &[Line], writer: &mut T, separator: u8) {
    for line in lines {
        writer.write_all(line.line).unwrap();
        writer.write_all(&[separator]).unwrap();
    }
}
```

容量不足などの`io::Error`は回復不能なバグとは限りません。CLIとしてはエラーを上位へ返し、利用者に失敗を通知して、panicではない終了経路に進めるべきです。

- Issue: [uutils/coreutils#14377](https://github.com/uutils/coreutils/issues/14377)
- 修正PR: [uutils/coreutils#14388](https://github.com/uutils/coreutils/pull/14388)

## 調査で分けるべき2つの書き込み経路

`sort`の書き込み失敗には、少なくとも次の2経路があります。

1. マージ結果を最終出力へ書く経路
2. 外部ソートの途中で一時ファイルへspillする経路

先行するIssueと修正は1の経路を対象にしていました。しかし、それで2の経路まで安全になるわけではありません。エラーを「書き込み処理全体」で検索するだけでなく、外部ソートの状態遷移（chunk生成、spill、merge）ごとに確認する必要があります。

## 回帰テストを先に作る

実際の容量不足を模擬する統合テストでは、テスト用ディレクトリを小さなtmpfsとしてマウントし、入力を外部ソートへ追い込みます。ただし、マウントには環境によって権限が必要です。今回のテストは、権限がなければスキップし、権限がある環境では次の性質を確認します。

- 一時領域を使うほど大きい入力を与える
- `--buffer-size=1`でspillを強制する
- `--temporary-directory`で対象領域を明示する
- 終了コード1で失敗する
- stderrに容量不足を示すエラーが出る
- `panicked`を含まない

```rust
let lines = (0..200_000)
    .map(|line| format!("{line:06}\n"))
    .collect::<String>();
scene.fixtures.write("input.txt", &lines);

scene
    .ucmd()
    .args(&[
        "--buffer-size=1",
        "--temporary-directory",
        tmp_dir,
        "input.txt",
    ])
    .fails_with_code(1)
    .stderr_contains("No space left on device")
    .stderr_does_not_contain("panicked");
```

統合テストだけでは、権限や実行環境によって常に実行できるとは限りません。そこで、書き込みエラーを返す小さな`Write`実装を使ったユニットテストも追加します。

```rust
struct FailingWriter(io::ErrorKind);

impl Write for FailingWriter {
    fn write(&mut self, _buf: &[u8]) -> io::Result<usize> {
        Err(io::Error::from(self.0))
    }

    fn flush(&mut self) -> io::Result<()> {
        Ok(())
    }
}
```

これにより、実際にディスクを満杯にしなくても、`write_lines`が`StorageFull`を握りつぶさず返すことを確認できます。

## 修正：`io::Result`を上位へ伝播する

書き込み関数の戻り値を`io::Result<()>`に変更し、各`write_all`で`?`を使います。呼び出し側も同じく`?`で上位へ渡します。

```rust
fn write_lines<T: Write>(
    lines: &[Line],
    writer: &mut T,
    separator: u8,
) -> std::io::Result<()> {
    for line in lines {
        writer.write_all(line.line)?;
        writer.write_all(&[separator])?;
    }
    Ok(())
}

fn write<I: WriteableTmpFile>(/* ... */) -> UResult<I::Closed> {
    let mut tmp_file = I::create(file, compress_prog)?;
    write_lines(chunk.lines(), tmp_file.as_write(), separator)?;
    tmp_file.finished_writing()
}
```

重要なのは、低レベル関数だけを直して終わりにしないことです。`write_lines`がエラーを返しても、呼び出し側が`unwrap()`や無視で捨てれば、利用者には届きません。エラーを意味のあるCLIエラーへ変換する既存の上位経路まで戻して確認します。

## 検証結果と注意点

ローカルでは、書き込みエラーを模擬するユニットテスト、フォーマット、対象パッケージのテストを実行しました。tmpfsを使う統合テストは、実行環境のマウント権限不足により実際のマウントまでは行えませんでした。そのため、容量不足を実環境で再現できたとは主張せず、権限依存のテストはスキップ可能な設計にしています。

また、修正PRの履歴整理では、テストを先に置き、その後に実装を変更する線形履歴へ整理しました。レビューやCIで問題を追いやすくするには、再現条件、回帰テスト、最小修正を分離して示すのが有効です。

## 学び

- `unwrap()`は、入力やI/Oが外部要因で失敗する箇所ではpanicへの変換になり得る。
- 既に修正された書き込み経路があっても、spillやmergeなど別経路は別に監査する。
- 容量不足のような環境依存エラーは、失敗する`Write`実装によるユニットテストで安定して検証できる。
- 特権が必要な統合テストは、実行できない環境で無理に成功扱いにせず、前提条件を明示してスキップする。
- 「panicしない」だけでなく、非ゼロ終了、エラーメッセージ、エラー種別まで検証すると回帰テストが具体的になる。

## 参考資料

- [uutils/coreutils Issue #14377](https://github.com/uutils/coreutils/issues/14377)
- [uutils/coreutils Pull Request #14388](https://github.com/uutils/coreutils/pull/14388)
- [Rust `std::io::Write::write_all`](https://doc.rust-lang.org/std/io/trait.Write.html#method.write_all)
