---
title: "符号なし整数のラップで壊れたRAR5が「正常なEOF」になるバグを直した"
emoji: "📦"
type: "tech"
topics: ["c", "libarchive", "debug", "oss"]
published: true
---

壊れた入力を扱うパーサーでは、クラッシュだけが危険な失敗ではありません。エラーを正常終了として返すと、呼び出し側はデータが欠けたことに気づけないからです。

libarchiveのRAR5パーサーで、まさにその状態になる不具合を調査・修正しました。本記事では、符号なし整数のラップがどのように`ARCHIVE_EOF`へ化けたのか、回帰テストをどう設計し、どこに境界検査を入れたのかをまとめます。

対象IssueとPull Requestはこちらです。

- [libarchive/libarchive#3483](https://github.com/libarchive/libarchive/issues/3483)
- [libarchive/libarchive#3513](https://github.com/libarchive/libarchive/pull/3513)

## 症状：壊れたアーカイブなのに正常終了する

再現用RAR5には、サイズが0と宣言された未知のextra fieldがあり、その直後にfield IDが1バイトあります。

期待する結果は、形式不正を表す`ARCHIVE_FATAL`です。しかし未修正のコードでは、次のヘッダーを読むAPIが`ARCHIVE_EOF`を返しました。

```c
archive_read_next_header(a, &entry); // ARCHIVE_EOF
```

`ARCHIVE_EOF`はアーカイブ末尾への正常到達として扱われます。そのため、後続エントリがあっても読み取りが静かに終了します。クラッシュもSanitizerの警告もなく、利用側からは「短いが正常なアーカイブ」に見えるのが厄介です。

## 原因：残量計算が2段階で破綻した

RAR5のextra fieldは、可変長整数で記録されたfield全体のサイズとfield IDを読み、その後にpayloadを処理します。問題の中心を単純化すると次の形です。

```c
uint64_t field_size;
uint64_t id_bytes;
int64_t remaining;

field_size -= id_bytes;
remaining -= field_size;
consume(field_size);
```

不正入力では`field_size == 0`、`id_bytes == 1`でした。したがって最初の減算は数学的には`0 - 1`ですが、`field_size`は`uint64_t`です。Cの符号なし整数演算は法`2^N`で行われるため、値は`UINT64_MAX`へラップします。

その巨大な値を符号付きの残量`remaining`から引くと、通常の「残りバイトが足りない」という状態ではなくなります。さらに読み取り位置を進める処理へ不正な値が渡り、最終的に`archive_read_next_header()`が`ARCHIVE_EOF`を返していました。

ここで重要なのは、Sanitizerで異常が出なかったことです。符号なし整数のラップ自体はCで定義された動作なので、未定義動作の検出だけでは見つからない場合があります。APIの返り値まで確認する回帰テストが必要です。

## 回帰テストを先に置く

修正の受け入れ条件を「クラッシュしない」ではなく「壊れた形式を正常終了にしない」としました。

```c
DEFINE_TEST(test_read_format_rar5_extra_field_underflow)
{
    struct archive *a;
    struct archive_entry *entry;

    extract_reference_file(
        "test_read_format_rar5_extra_field_underflow.rar");
    a = archive_read_new();
    assert(a != NULL);
    assertEqualIntA(a, ARCHIVE_OK,
        archive_read_support_format_rar5(a));
    assertEqualIntA(a, ARCHIVE_OK,
        archive_read_open_filename(a,
            "test_read_format_rar5_extra_field_underflow.rar", 10240));
    assertEqualIntA(a, ARCHIVE_FATAL,
        archive_read_next_header(a, &entry));
    assertEqualInt(ARCHIVE_OK, archive_read_free(a));
}
```

未修正のベースラインでは、最後の`ARCHIVE_FATAL`の検査が`ARCHIVE_EOF`との差で失敗しました。これにより、修正対象を観測可能なAPI契約として固定できます。

libarchiveのテストでは参照アーカイブをuuencodeした`.uu`ファイルとして管理します。テスト本体だけでなく、CMakeとAutomakeのテスト一覧にも登録しました。

## 修正：減算する前に境界を検査する

未知のextra fieldを読み飛ばす直前で、payloadサイズがextra領域の残量を超えていないか確認します。

```c
if (extra_field_size > (uint64_t)extra_data_size) {
    archive_set_error(&a->archive,
        ARCHIVE_ERRNO_FILE_FORMAT,
        "RAR5 extra field size is too small");
    return ARCHIVE_FATAL;
}

extra_data_size -= extra_field_size;
if (ARCHIVE_OK != consume(a, extra_field_size)) {
    return ARCHIVE_EOF;
}
```

順序がポイントです。

1. 信頼できない長さを残量と比較する
2. 不正なら形式エラーとして終了する
3. 検査を通った場合だけ減算・読み飛ばしを行う

ラップした値を後段で補正するより、値を使う直前に不変条件を明示する方が、影響範囲が小さく意図も読み取りやすくなります。

## 検証

実施した検証は次の通りです。

- 未修正ベースラインで新しい回帰テストが失敗し、`ARCHIVE_EOF`を再現
- 修正後の対象テスト4件がすべて成功
- RAR5関連テスト74件がすべて成功（環境依存の既存テスト1件はCTestによるskip）
- ASan/UBSanビルドで、新しい回帰テストと近接する切り詰め入力テストの2件が成功
- `git diff --check`が成功

Sanitizer付き検証には、次のような設定を使いました。

```bash
cmake -S . -B build-asan \
  -DENABLE_WERROR=OFF \
  -DBUILD_SHARED_LIBS=OFF \
  -DENABLE_TEST=ON \
  -DCMAKE_C_FLAGS='-fsanitize=address,undefined -fno-omit-frame-pointer' \
  -DCMAKE_EXE_LINKER_FLAGS='-fsanitize=address,undefined'
cmake --build build-asan --target libarchive_test -j2
ASAN_OPTIONS=detect_leaks=0 UBSAN_OPTIONS=halt_on_error=1 \
  ctest --test-dir build-asan --output-on-failure \
  -R 'extra_field_underflow|truncated_htime'
```

結果は2件中2件成功でした。

## 学び

### 長さフィールドは減算前に検査する

`remaining -= declared_size`の後で異常を探すのでは遅すぎます。特に符号付き・符号なしが混在すると、整数変換によって条件が直感と異なる動作になります。

パーサーでは次の順序をテンプレート化すると安全です。

```c
if (declared_size > remaining)
    return FORMAT_ERROR;
remaining -= declared_size;
consume(declared_size);
```

### 「エラーがない」ことと「正しい」ことは別

ASan/UBSanが無反応でも、戻り値が誤っていれば堅牢性の問題は残ります。壊れた入力に対して、クラッシュの有無だけでなく、エラー種別と後続データの扱いを検証する必要があります。

### 回帰テストは外部から見える契約を固定する

内部の変数値だけを検査するより、公開APIが`ARCHIVE_FATAL`を返すことを固定すると、実装が変わっても「不正入力を正常EOFにしない」という要件を守れます。

## 参考資料

- [libarchive Issue #3483](https://github.com/libarchive/libarchive/issues/3483)
- [libarchive Pull Request #3513](https://github.com/libarchive/libarchive/pull/3513)
- [libarchive: How to add tests](https://github.com/libarchive/libarchive/wiki/LibarchiveAddingTest)
- [ISO/IEC 9899:201x Committee Draft N1570](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf) — 6.2.5で符号なし整数演算の法`2^N`を規定
- [archive_read(3)](https://man.archlinux.org/man/archive_read.3.en)
