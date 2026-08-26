---
title: "Pythonのraw_unicode_escapeで正しいUnicodeが\\uXXXX化する落とし穴"
emoji: "🔤"
type: "tech"
topics: ["python", "unicode", "encoding", "debugging"]
published: true
---

Pythonで文字化けを修復する処理が、逆に正しいUnicode文字列を壊してしまうバグを調査しました。

原因は、`str` を `raw_unicode_escape` でいったん `bytes` に戻してからUTF-8としてデコードする処理です。この方法はLatin-1として誤ってデコードされたUTF-8を直せる場合がある一方、U+00FFを超える正しい文字をリテラルな `\uXXXX` に変えてしまいます。

本記事では、最小コードで現象を再現し、なぜ一部の言語だけで壊れるように見えるのか、既存の修復経路を保ったままどう防いだかを説明します。

## 背景

対象の関数は、データベースから得た文字列に対して次のような修復をしていました。

```python
def repair(value: str) -> str:
    bytes_value = value.encode("raw_unicode_escape")
    try:
        return bytes_value.decode("utf-8")
    except UnicodeDecodeError:
        return value
```

この処理が想定している入力は、UTF-8のバイト列をLatin-1として誤って文字列化した「mojibake」です。

たとえば、本来 `Łukasz Jabłoński` だったUTF-8を誤ってLatin-1として扱うと、次のような文字列になります。

```python
mojibake = "Å\x81ukasz JabÅ\x82oÅ\x84ski"
```

このケースでは、既存処理で元の文字列を復元できます。

```pycon
>>> mojibake.encode("raw_unicode_escape").decode("utf-8")
'Łukasz Jabłoński'
```

ところが、入力が最初から正しいUnicode文字列の場合に問題が起きます。

## 再現

以下はPython標準機能だけで再現できます。

```pycon
>>> value = "Łukasz Jabłoński"
>>> value.encode("raw_unicode_escape")
b'\\u0141ukasz Jab\\u0142o\\u0144ski'
>>> value.encode("raw_unicode_escape").decode("utf-8")
'\\u0141ukasz Jab\\u0142o\\u0144ski'
```

画面上では `Ł`、`ł`、`ń` ではなく、文字としての `\u0141`、`\u0142`、`\u0144` が表示されます。

同じ現象はキリル文字やCJK文字でも起きます。

```pycon
>>> "Иван Петров".encode("raw_unicode_escape").decode("utf-8")
'\\u0418\\u0432\\u0430\\u043d \\u041f\\u0435\\u0442\\u0440\\u043e\\u0432'
>>> "山田太郎".encode("raw_unicode_escape").decode("utf-8")
'\\u5c71\\u7530\\u592a\\u90ce'
```

一方、Latin-1の範囲に収まる文字だけなら挙動が異なります。

```pycon
>>> "Jürgen Müller".encode("raw_unicode_escape")
b'J\xfcrgen M\xfcller'
>>> "Jürgen Müller".encode("raw_unicode_escape").decode("utf-8")
Traceback (most recent call last):
  ...
UnicodeDecodeError: ...
```

元の関数は `UnicodeDecodeError` を捕捉して入力値を返すため、このケースは見かけ上壊れません。つまり、英語や西欧言語のテストだけでは見逃しやすく、U+00FFを超える文字を含むテストが必要です。

## 原因

Python 3の `str` はUnicodeコードポイントの列です。`bytes.decode()` はバイト列を文字列に変換する操作であり、すでに正しくデコードされた `str` に対して再度「バイト列へ戻してデコードする」なら、その変換が本当に逆変換になっているかを確認しなければなりません。

`raw_unicode_escape` は、U+00FFまでの文字を1バイトとして表現する一方、それより大きいコードポイントをバックスラッシュエスケープとして表現します。

```python
ord("ü")  # 0x00FC
ord("Ł")  # 0x0141
```

そのため `Ł` はASCIIだけからなる `b"\\u0141"` になります。ASCIIは有効なUTF-8でもあるので、その後の `.decode("utf-8")` は例外を出さず、壊れた文字列を正常な結果として返してしまいます。

ここで重要なのは、例外が出ないことが変換の正しさを保証しない点です。

## 修正

対象アプリケーションでは、修復対象のmojibakeはLatin-1として誤デコードされた文字列です。そのような文字列に含まれる各コードポイントはU+00FF以下になります。

そこで、U+00FFを超えるコードポイントがすでに存在する場合は、正しいUnicode文字列としてそのまま返すガードを追加しました。

```python
def repair(value: str) -> str:
    if not value:
        return ""

    if any(ord(char) > 0xFF for char in value):
        return value

    bytes_value = value.encode("raw_unicode_escape")
    try:
        return bytes_value.decode("utf-8")
    except UnicodeDecodeError:
        return value
```

この判定は汎用的な文字化け検出器ではありません。「修復対象はLatin-1として誤デコードされたUTF-8」という入力条件があるから使えるガードです。入力経路が異なる場合は、元のバイト列と明示された文字コードからデコードし直す設計を優先すべきです。

## 回帰テスト

修正では、少なくとも次の2方向を同時に固定します。

```python
def test_valid_unicode_is_unchanged():
    value = "Łukasz Jabłoński"
    assert repair(value) == value


def test_latin1_mojibake_is_still_repaired():
    value = "Å\x81ukasz JabÅ\x82oÅ\x84ski"
    assert repair(value) == "Łukasz Jabłoński"
```

1本目だけでは「常に入力を返す」実装でも通ってしまいます。2本目を残すことで、既存の修復機能を壊していないことも確認できます。

実際の修正PRでは、空文字、正しいUnicode、Latin-1 mojibake、正常な4バイトUTF-8相当のケース、途中で切れたUTF-8相当の既存ケースを確認しました。CIはPython 3.10〜3.14とPostgreSQL/MySQLの組み合わせですべて成功し、変更行のカバレッジも満たしました。

## 学び

- Python 3では、`str` と `bytes` の境界を先に特定する。文字化け修復は可能なら元の `bytes` に対して行う。
- `encode(...).decode(...)` が例外なく完了しても、元の文字列を保つとは限らない。
- 文字コード処理のテストには、ASCII、Latin-1範囲内、U+00FF超、CJKなど異なる範囲の文字を入れる。
- 修復処理には「何として誤デコードされた入力を直すのか」という前提を明文化する。
- 回帰テストは、壊れていた入力と、以前から直せていた入力の両方を固定する。

## 参考資料

- [Python公式ドキュメント: codecs — Codec registry and base classes](https://docs.python.org/3/library/codecs.html)
- [Python公式ドキュメント: Unicode HOWTO](https://docs.python.org/3/howto/unicode.html)
- [modoboa/modoboa Issue #4142](https://github.com/modoboa/modoboa/issues/4142)
- [modoboa/modoboa Pull Request #4148](https://github.com/modoboa/modoboa/pull/4148)
