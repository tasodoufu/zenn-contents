---
title: "libeventのbufferevent解放後UAF：共有ロックの寿命ミスマッチを、trace lock profilerで検出して直す"
emoji: "🔒"
type: "tech"
topics: [c, libevent, debugging, opensource, asan]
published: true
---

`bufferevent_free()` を呼び、さらに生存している `evbuffer` をロックした瞬間に、AddressSanitizerが heap-use-after-free を報告する——しかも「解放したはずのlock」を指している。本記事は、libeventに報告されたこのバグ（[libevent#1842](https://github.com/libevent/libevent/issues/1842)）の再現・原因究明・修正・回帰テスト追加までの実践記録である。

## 背景：buffereventとevbufferはロックを共有する

libeventの `bufferevent` は、入出力バッファとして2つの `evbuffer`（`input` / `output`）を保持する。`BEV_OPT_THREADSAFE` を付けて生成すると、`bufferevent_init_common_()` 内で `evbuffer_enable_locking()` が呼ばれ、**bufferevent自身のロックを両evbufferと共有**する構造になる。

ここで要注意なのが、`evbuffer` は参照カウント（`refcnt`）で寿命管理されている点だ。`evbuffer_add_buffer_reference()` を使うと、別のevbufferから対象evbufferへの「参照」を張れる（multicastバッファなどで使われる機構で、内部的には `APPEND_CHAIN_MULTICAST` 経由で参照が増える）。参照が増えたevbufferは、元の所有者が `evbuffer_free()` を呼んでも、refcntが1以上なら破壊されず生き残る。

問題は、**bufferevent側の解放処理がその生存を考慮していなかった**ことにある。

## 再現：ASANで確定させるまで

報告者はDebian trixieコンテナ+ASANで再現コードを公開していたが、環境固有の要素はできるだけ自分の手で確認したい。そこで、まずはローカルのlibeventクローンにASANを効かせてビルドした上で、issue記載のシーケンス（`evbuffer_add_buffer_reference()` で参照を張ったまま `bufferevent_free()`、その後生存evbufferをロック）を再現コードとしてCで書いた。

その結果、再現コードのASAN出力に次のシーケンスが現れた（簡略化）。

```
==ERROR: AddressSanitizer: heap-use-after-free on address ...
WRITE of size 8 at ... thread T0
    #0 evbuffer_decref_and_unlock_ buffer.c
    ...
freed by thread T0 here:
    #0 free
    #1 event_mm_free_ event.c
    ...
```

`event_mm_free_` がfree元に現れるのがポイントで、イベントライブラリ内部の確保領域（この場合はlockオブジェクト）が先行してfreeされていることを示す。つまり「evbufferが生きているのに、共有ロックだけが先に死んだ」という仮説と一致する。

## 原因究明：finalize時に「refcnt > 1」を見ていない

`bufferevent_finalize_cb_()` を読むと、該当箇所にこんなコメントが元からあった。

```c
/* XXX what happens if refcnt for these buffers is > 1?
 * The buffers can share a lock with this bufferevent object,
 * but the lock might be destroyed below. */
```

つまり**上流のコード自体がこのリスクを認識していた**（XXXコメントで放置されていた）。しかしコードは以下のようになっていた。

```c
/* evbuffer will free the callbacks */
evbuffer_free(bufev->input);
evbuffer_free(bufev->output);
...
BEV_UNLOCK(bufev);

if (bufev_private->own_lock)
    EVTHREAD_FREE_LOCK(bufev_private->lock,
        EVTHREAD_LOCKTYPE_RECURSIVE);
```

`evbuffer_free()` はrefcntを減らすだけで、refcnt > 1のevbufferは生存する。しかし `EVTHREAD_FREE_LOCK` は条件なく走る。結果、生存evbufferが持つlockポインタがdanglingになる。

## 修正：refcnt > 1ならEVTHREAD_FREE_LOCKをスキップ

修正方針はシンプルに「finalize前に生存判定を入れ、生存するならロック解放をスキップ」。小さなone-time leakとUAFのトレードオフであり、明らかに後者がマシと判断した。

```c
/* The input/output evbuffers share this bufferevent's lock (see
 * evbuffer_enable_locking() in bufferevent_init_common_()). If a
 * reference to one of them is still held elsewhere (e.g. by
 * evbuffer_add_buffer_reference()), evbuffer_free() below only
 * drops that reference and the evbuffer survives this
 * finalization. Freeing the shared lock would then leave the
 * surviving evbuffer with a dangling lock pointer, so keep the
 * lock alive in that case. It is a small one-time leak, which is
 * preferable to a use-after-free on the next lock operation of
 * the surviving evbuffer. */
#ifndef EVENT__DISABLE_THREAD_SUPPORT
int evbuffer_outlives_lock =
    bufev->input->refcnt > 1 || bufev->output->refcnt > 1;
#endif

/* evbuffer will free the callbacks */
evbuffer_free(bufev->input);
evbuffer_free(bufev->output);
...
#ifndef EVENT__DISABLE_THREAD_SUPPORT
if (bufev_private->own_lock && !evbuffer_outlives_lock)
    EVTHREAD_FREE_LOCK(bufev_private->lock,
        EVTHREAD_LOCKTYPE_RECURSIVE);
#endif
```

ポイントは2つ。

- 判定を**evbuffer_free()より前に**行うこと。free後にrefcntを見ても手遅れ。
- `EVENT__DISABLE_THREAD_SUPPORT` のときはlock関連コード自体が存在しないので、判定変数ごとコンパイルアウトすること。非スレッドビルドに影響を出さない。

トレードオフについては、PR本文にも明記した。refcntが1を超えて生存したevbufferのロックは、evbuffer自身が破壊されるタイミングで解放される（evbufferは自身のロックは解放しないため、厳密にはライブラリ全体のshutdown時に解放されるまで残る。つまり「小さなone-time leak」）。UAFよりはるかに害が少ないと判断した。

## 回帰テスト：trace lock profilerで「解放済みロックへのlock」を検出する

ASANだけに頼らない回帰テストにしたかった。libeventのテストスイートには、`evthread_set_lock_callbacks()` で差し替え可能なロックAPIをフックして、**lockのalloc/free/lock/unlockを追跡する profiler** が `test/regress_bufferevent.c` に実装されている。各lockは状態（ALLOC/FREE）を持たされ、FREE状態のlockに `trace_lock_lock()` が呼ばれたら `"lock: lock error"` でテスト失敗にする。

この仕組みを利用した回帰テストが次の通り。

```c
static void test_bufferevent_output_survives_reference(void *arg)
{
	struct basic_test_data *data = arg;
	use_lock_unlock_profiler();
	{
		struct bufferevent *bev;
		struct evbuffer *out, *holder;

		bev = bufferevent_socket_new(NULL, -1, BEV_OPT_THREADSAFE);
		tt_assert(bev);
		out = bufferevent_get_output(bev);
		holder = evbuffer_new();
		tt_assert(holder);

		/* A non-empty chain is required so that
		 * evbuffer_add_buffer_reference() increfs `out` (see
		 * APPEND_CHAIN_MULTICAST). */
		tt_int_op(evbuffer_add(out, "x", 1), ==, 0);
		tt_int_op(evbuffer_add_buffer_reference(holder, out), ==, 0);

		bufferevent_free(bev);
		/* Finalization is deferred; run it. */
		event_loop(EVLOOP_NONBLOCK | EVLOOP_ONCE);

		/* `out` is still alive and shares its lock with the freed
		 * bufferevent; locking it used to hit a freed lock. */
		evbuffer_lock(out);
		evbuffer_unlock(out);

		evbuffer_free(holder);
	}
	free_lock_unlock_profiler(data);
end:
	;
}
```

このテストは**修正なしでは trace profiler が "lock: lock error" を出してFAIL**し、修正ありではPASSする。つまりASANビルドに依存せず、CIの通常テストでも「共有ロックが先に解放された」ことを検出できる。落とし穴として、`evbuffer_add_buffer_reference()` が参照を増やすのは**対象が非空チェーンのときだけ**（`APPEND_CHAIN_MULTICAST` の条件）なので、`evbuffer_add(out, "x", 1)` を先に呼んでから参照を張る必要がある。ここを飛ばすとrefcntが増えずテストがバグを検出できない（通るだけのテストになる）。

なお、この「テストが本来守るべき分岐を通っているか確認する」視点は、以前 [回帰テストの前提条件とミューテーション検証](https://zenn.dev/tasodoufu/articles/effective-regression-tests-preconditions) で書いた内容と同型の落とし穴である。

## 検証：各修正コミットをASANビルドで通す

公開時点のmaster（libevent 2.2系の開発線）をASAN (`-fsanitize=address -g -O1`) でビルドし、ff6f4f3を適用した状態で検証した。PRのCIチェックは、無人ジョブではリポジトリ側の承認が必要なためまだ未実行だ。ローカルで確認できた範囲は次の通り。

- 修正コミットff6f4f3の回帰テスト `bufferevent_output_survives_reference` を含むbufferevent系テスト群を実行: **39 tests ok (0 skipped)**。新規テストもPASS。なお LeakSanitizer は修正後も 72 バイトのリークを検出する。これは前述の「refcnt>1で生存したevbufferのロック放置」漏れそのものであり、UAFを防ぐために意図的に受け入れたリークである。
- 修正前との比較は、PR#1927で報告したASAN/テスト結果に加え、本記事の再現コード（GitHubPRリンク参照）で修正前は heap-use-after-free、修正後は crash しないことを確認した。
- 修正コミットは [libeventリポジトリのforkブランチ `bugfix/1842-bev-free-shared-lock-uaf`](https://github.com/tasodoufu/libevent/tree/bugfix/1842-bev-free-shared-lock-uaf) で公開している。

既存の `bufferevent_pair_release_lock` テストは類似シナリオ（pair bufferevent間でのロック共有）を扱うが、`evbuffer_add_buffer_reference()` 経由の参照生存ケースは扱っておらず、今回のテストで初めてカバーされる。

## 学び

- **「オブジェクトAの解放処理が、共有リソースの生存判定をしないまま解放する」** パターンは、参照カウント型の共有バッファと組み合わさるとUAFになる。修正は「生存判定して解放スキップ」か「参照者がロックの所有権を引き取る」のどちらかで、前者はone-time leak、後者は複雑さ増。小規模修正では前者が選ばれやすい。
- libeventのように**テスト用のlock profilerが用意されている**ライブラリでは、UAFをASANに頼らず「lock状態追跡」として検出できる。回帰テストの移植性・実行速度の点でASANより優れる場面もある。
- 既存コードの `XXX` コメントは「作者も気づいていたが手を出していなかった」領域の地図になる。issue を探すとき、`XXX` や `TODO` コメントをissue番号と突き合わせて読むのは有効な手がかりになる。

## 参考資料

- [libevent#1842: UAF via shared lock lifetime mismatch in `bufferevent_finalize_cb_`](https://github.com/libevent/libevent/issues/1842)
- [libevent#1927: 本修正のPR](https://github.com/libevent/libevent/pull/1927)
- [libevent bufferevent.c](https://github.com/libevent/libevent/blob/master/bufferevent.c)
- [libevent buffer.c (evbuffer refcount)](https://github.com/libevent/libevent/blob/master/buffer.c)
- [Zenn: 回帰テストの前提条件とミューテーション検証](https://zenn.dev/tasodoufu/articles/effective-regression-tests-preconditions)
