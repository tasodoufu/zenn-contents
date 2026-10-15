---
title: "Rust製touchでタイムゾーン省略時のUTC読み替えを直す"
emoji: "🕒"
type: "tech"
topics: [rust, timezone, jiff, opensource, linux]
published: true
---

## はじめに

GNU coreutilsの`touch -d "2026-08-01 12:00:00"`は、タイムゾーンを書かなければ**ローカル時刻**として扱われます。一方、Rust製実装uutils/coreutilsでは同じ入力が**UTC**として解釈されるバグがあり、ローカルのUTCオフセット分だけタイムスタンプがずっていました。

この記事は、この差異をuutils/coreutilsの[issue #15291](https://github.com/uutils/coreutils/issues/15291)で再現し、[PR #15353](https://github.com/uutils/coreutils/pull/15353)として修正した記録です。「タイムゾーンを省略した文字列をどう解釈するか」は日時処理で頻出する落とし穴で、Jiffでの正しい扱い方のメモとしても読めます。

## 症状

`touch -d`にオフセット付きの`+HHMM`など明示的なタイムゾーンが現れる形式は正しく解釈されていました。ところが、タイムゾーンを書かない次の形式がUTCとして解釈されていました:

```
$ TZ=Asia/Tokyo touch -d "2026-08-01 12:00:00" f
```

このとき期待されるファイルのmtimeは「日本時間の12:00」＝UTC 03:00ですが、バグのある実装では「UTCの12:00」がそのまま使われ、表示上は日本時間の21:00になります。Europe/Rome（夏期UTC+2）なら14:00に、Pacific/AucklandのようなUTC+12の地域では12時間のずれになります。つまりローカルタイムゾーンによる補正が一切入っていませんでした。

## 原因

`src/uu/touch/src/touch.rs`の`parse_date`を追うと、対応するstrftime形式ごとに分岐した解析になっています:

- `YYYYMMDDHHMMS`（`"%Y-%m-%d %H:%M:%S"`）— タイムゾーンなし
- `YYYYMMDDHHMMSS`（`"%Y-%m-%d %H:%M:%S.%f"`）
- `YYYYMMDDHHMM_OFFSET` — 末尾に`%z`オフセットが付く形式

修正前は、タイムゾーンなし入力を受け持つ`YYYYMMDDHHMMS`の経路が、他の形式と同じくおそらく共通実装由来の`TimeZone::UTC.to_zoned(dt)`で解決していました。これが原因で、オフセットなし入力（ローカル時刻とみなすべき）が問答無用でUTCとみなされます。

「文字列の形式」が意味するタイムゾーンの有無で分岐し、省略時の既定（GNUだとローカル）を実装が用意してあげる必要がありますが、この分岐がUTC固定になっていたわけです。

## Jiffでの正しい手順

Jiffでは、タイムゾーンのない日付+時刻の解決は次の段階で行われます:

1. `strtime::parse`でcivil datetime（タイムゾーンなし）までパースする
2. `TimeZone::system()`（環境変数`TZ`やシステム設定に従うローカルタイムゾーン）で解決する
3. この変換はDST切り替え時には曖昧になるため、`to_ambiguous_zoned`が`AmbiguousZoned`を返す
4. `.unambiguous()`でライブラリ既定のポリシー（曖昧 instant はearlier offset）で確定する

修正後の`parse_date`はこうなります（PR #15353、コミット cbe15da）:

```rust
// A date and time without an explicit timezone is interpreted in the
// local zone (GNU behavior), not as UTC (issue #15291). Formats that
// carry a trailing %z offset are still resolved via that offset, so they
// keep an explicit UTC branch.
if let Ok(filetime) = strtime::parse(format::YYYYMMDDHHMMS, s)
    .and_then(|tm| tm.to_datetime())
    .and_then(|dt| {
        TimeZone::system()
            .to_ambiguous_zoned(dt)
            .unambiguous()
    })
    .map(|zdt| timestamp_to_filetime(zdt.timestamp()))
{
    return Ok(filetime);
}
for fmt in [
    format::YYYYMMDDHHMMSS,
    format::YYYYMMDDHHMM_OFFSET,
] {
    // ... これらの形式は今まで通り（オフセット解決）のまま
}
```

ポイントは次の通りです:

- **タイムゾーン明示形式と非明示形式で分岐する**: 非明示の`"%Y-%m-%d %H:%M:%S"`だけをローカルへ、オフセット付きの2形式は従来のオフセット解決のまま。意味論は食わせた文字列が決めるものであり、実装が勝手に決めるものではない
- **`TimeZone::system()`はローカルタイムゾーンの取得**。環境変数`TZ`（例: `Asia/Tokyo`）やシステム設定に従う
- **`to_ambiguous_zoned`は「ローカル時刻→ゾーン付き時刻」の変換がDSTで曖昧になりうることを型で表現する**。Jiffは曖昧な解決ポリシーを勝手に選ばず、`AmbiguousZoned`を返してくる。`unambiguous()`でライブラリ既定ポリシー（ここではearlier offset＝GNU mktime相当）で確定できる

曖昧性のポリシーがGNU mktime相当（earlier offset）である点は、PRのコミットメッセージにも明記しています:

> ambiguous instants fall back to the earlier offset, like GNU mktime

## 回帰テスト

タイムゾーンバグの回帰テストは、テスト側で`TZ`を明示指定するのが定石です。テストランナーの既定は`TZ=UTC`であり、UTCでは「バグ挙動（UTC解釈）」と「正挙動（ローカル解釈）」が同一になってしまうため、明示的な指定なしにこのバグは検出できません。

PRでは次のテストを`tests/by-util/test_touch.rs`に追加しました。コメントもそのまま「ハーネスの既定TZ=UTCではバグと正解が区別できない」という趣旨です:

```rust
#[test]
fn test_touch_date_without_offset_is_local_time() {
    // Force TZ=Asia/Tokyo: the harness default TZ=UTC would make the buggy
    // "interpret as UTC" behavior indistinguishable from the correct one.
    let (at, mut ucmd) = at_and_ucmd!();
    let file = "touch_local_no_offset";
    at.touch(file);
    ucmd.env("TZ", "Asia/Tokyo")
        .args(&["-d", "2026-08-01 12:00:00", file])
        .succeeds()
        .no_output();
    let (atime, mtime) = get_file_times(&at, file);
    // 1_785_585_600 = 2026-08-01 12:00 UTC; Asia/Tokyo is UTC+9, so civil
    // 12:00 local is 03:00 UTC = 1_785_553_200.
    let expected = FileTime::from_unix_time(1_785_585_600 - 9 * 3600, 0);
    // ... atime/mtime が expected と一致することを検証
```

エポック秒の期待値は自分でも検算できます: `date -u -d "2026-08-01 12:00 UTC" +%s`は`1785585600`で、これから9時間（32400秒）を引いた`1785553200`が「日本時間2026-08-01 12:00」です。修正後はこの値がmtime/atimeとして記録されます。

PR本文とコミットメッセージには issue 番号（`Fixes #15291`）を含め、リグレッション検出の起点を明示しました。clippyは`-D warnings`でcleanです。

## 学び

- 「タイムゾーンを書かない文字列はローカルで解釈する」というGNUの意味論は、パース対象の形式ごとに明示的に分岐して初めて正しく守れる。単一のパーサーに`TimeZone::UTC`を埋め込むと、非明示形式が静かにUTCになってしまう
- Jiffで正しい手順は`TimeZone::system().to_ambiguous_zoned(dt).unambiguous()`。DST曖昧時刻も含めて、確定ポリシーをライブラリに任せられる
- タイムゾーンバグの回帰テストは、テスト側で`TZ`を明示指定するのが定石。CI実行環境（UTCが多い）に頼ると非決定的になるだけでなく、そもそもこの種のバグを検出できない

## 参考資料

- Issue: [uutils/coreutils#15291](https://github.com/uutils/coreutils/issues/15291) — touch -d interprets a date/time without timezone as UTC instead of local time
- PR: [uutils/coreutils#15353](https://github.com/uutils/coreutils/pull/15353)
- [Jiff (docs.rs)](https://docs.rs/jiff) — `jiff::tz::TimeZone`、`jiff::tz::AmbiguousZoned`
- [GNU Coreutils manual: touch invocation](https://www.gnu.org/software/coreutils/manual/html_node/touch-invocation.html) — `-d`オプション仕様
