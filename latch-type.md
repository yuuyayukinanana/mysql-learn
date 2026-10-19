# InnoDB ラッチ／Mutex リファレンス（MySQL 8.0.46 / 8.4.11）

SEMAPHORES 出力や `performance_schema` の待ち情報を読むための対応表。

- 行番号は 8.0.46 のもの
- 8.0.46 → 8.4.11 で RW ラッチの追加・削除はなし（行番号が動いたのは 20 箇所中 9 箇所）
- Mutex 名は 8.0 / 8.4 で同じ（8.4 で `LOG_GOVERNOR_MUTEX` が追加）

---

## 1. RW ラッチ：created in file → 正体（本番ビルドで出る全 16 種）

`performance_schema` 名は `wait/synch/sxlock/innodb/…` 配下の名前。

| created in file | 行 | 正体 | performance_schema 名 |
|---|---|---|---|
| `dict0dict.cc` | 2555 | インデックスツリーのラッチ `index->lock`（インデックスごと） | `index_tree_rw_lock` |
| `dict0dict.cc` | 1027 | `dict_operation_lock`（データディクショナリ全体、1 個） | `dict_operation_lock` |
| `dict0dict.cc` | 326 | テーブル統計ラッチ（テーブルごと、初回使用時に生成） | `dict_table_stats` |
| `buf0buf.cc` | 856 | バッファプールのページラッチ `block->lock`（ページごと） | 計測対象外（ページ単位は除外） |
| `btr0sea.cc` | 211 | アダプティブハッシュインデックスのラッチ（8 分割） | `btr_search_latch` |
| `hash0hash.cc` | 117 | バッファプール `page_hash` のロック（インスタンスごとに 16 本） | `hash_table_locks` |
| `fil0fil.cc` | 3294 | テーブルスペースのラッチ（領域割り当て） | `fil_space_latch` |
| `trx0purge.cc` | 230 | パージのラッチ（purge view、1 個） | `trx_purge_latch` |
| `trx0purge.cc` | 2623 | undo テーブルスペース一覧のラッチ | `undo_spaces_lock` |
| `trx0rseg.cc` | 750 | ロールバックセグメント配列のラッチ | `rsegs_lock` |
| `trx0i_s.cc` | 953 | `I_S.INNODB_TRX` などのキャッシュのラッチ | `trx_i_s_cache_lock` |
| `fts0fts.cc` | 543 | FULLTEXT インデックスキャッシュのラッチ | `fts_cache_rw_lock` |
| `fts0fts.cc` | 545 | FULLTEXT キャッシュ初期化のラッチ | `fts_cache_init_rw_lock` |
| `ibuf0ibuf.cc` | 523 | チェンジバッファの B-tree の `index->lock` | `index_tree_rw_lock`（通常のインデックスと共用） |
| `sync0sharded_rw.h` | 82 | 行ロックシステムのグローバルラッチ（64 分割） | `lock_sys_global_rw_lock` |
| `gis0sea.cc` | 854 | 空間インデックス（R-tree）検索用の一時ブロックラッチ | 計測対象外 |

---

## 2. Mutex 名の一覧（本番で現れ得るもの、サブシステム別）

### トランザクション

| 名前 | created | 守るもの |
|---|---|---|
| `TRX` | `trx0trx.cc:283` | 個々のトランザクションの状態とロック |
| `TRX_UNDO` | `trx0trx.cc:284` | 個々のトランザクションの undo |
| `TRX_SYS` | `trx0sys.cc:566` | `trx_sys` の大半のフィールド |
| `TRX_SYS_SERIALISATION` | `trx0sys.cc:567` | コミット順序リスト |
| `TRX_SYS_SHARD` | `ut0mutex.h:255` | アクティブ RW トランザクションの表（256 分割） |
| `TRX_POOL` / `TRX_POOL_MANAGER` | `trx0trx.cc:386` / `406` | トランザクションオブジェクトのプール |
| `UNDO_DDL` | `trx0purge.cc:2625` | undo テーブルスペース DDL の直列化 |

### ロック

| 名前 | created | 守るもの |
|---|---|---|
| `LOCK_SYS_PAGE` | `lock0latches.cc:104` | レコードロックのページ単位シャード（512） |
| `LOCK_SYS_TABLE` | `lock0latches.cc:116` | テーブルロックのシャード（512） |
| `LOCK_SYS_WAIT` | `lock0lock.cc:322` | ロック待ちスロット |

### バッファプール

| 名前 | created | 守るもの |
|---|---|---|
| `BUF_BLOCK_MUTEX` | `buf0buf.cc:0`（固定） | ページ 1 枚の状態（io_fix、fix count） |
| `BUF_POOL_ZIP` | `buf0buf.cc:0`（固定） | 圧縮のみページ |
| `BUF_POOL_CHUNKS` | `buf0buf.cc:1284` | チャンクの割り当て（リサイズ） |
| `BUF_POOL_LRU_LIST` | `buf0buf.cc:1285` | LRU リスト |
| `BUF_POOL_FREE_LIST` | `buf0buf.cc:1286` | free / withdraw リスト |
| `BUF_POOL_ZIP_FREE` / `BUF_POOL_ZIP_HASH` | `buf0buf.cc:1287` / `1288` | 圧縮ページ用アロケータ／ハッシュ |
| `BUF_POOL_FLUSH_STATE` | `buf0buf.cc:1290` | フラッシュの状態 |
| `FLUSH_LIST` | `buf0buf.cc:1381` | フラッシュリスト（ダーティページ） |
| `PAGE_CLEANER` | `buf0flu.cc:2829` | ページクリーナーの協調 |
| `DBLWR` | `buf0dblwr.cc:1296` | ダブルライトバッファ |
| `AHI_ENABLED` | `btr0sea.cc:192` | AHI の有効／無効切り替え |

### REDO ログ・リカバリ

| 名前 | created | 守るもの |
|---|---|---|
| `LOG_CHECKPOINTER` / `LOG_CLOSER` / `LOG_WRITER` / `LOG_FLUSHER` | `log0log.cc:635〜638` | 各ログスレッドの一時停止用 |
| `LOG_WRITE_NOTIFIER` / `LOG_FLUSH_NOTIFIER` | `log0log.cc:639` / `640` | 書き込み・フラッシュ完了通知 |
| `LOG_LIMITS` | `log0log.cc:641` | チェックポイント関連 LSN |
| `LOG_FILES` | `log0log.cc:642` | REDO ファイル集合 |
| `LOG_SN_MUTEX` | `log0log.cc:643` | ログバッファの排他（sn） |
| `RECV_SYS` / `RECV_WRITER` | `log0recv.cc:342` / `343` | クラッシュリカバリ中のみ |
| `LOG_ARCH` / `PAGE_ARCH*` / `REDO_LOG_ARCHIVE_*` | `arch0*.cc` / `log0meb.cc` | REDO・ページアーカイブ（クローン、MEB） |

### ファイル・I/O・クローン

| 名前 | created | 守るもの |
|---|---|---|
| `FIL_SHARD` | `fil0fil.cc:2014` | テーブルスペース／オープンファイル管理のシャード |
| `MASTER_KEY_ID_MUTEX` | `srv0srv.cc:1117` | 暗号化マスターキー ID |
| `CLONE_SYS` / `CLONE_TASK` / `CLONE_SNAPSHOT` | `clone0*.cc` | CLONE |

### ディクショナリ・DDL・統計

| 名前 | created | 守るもの |
|---|---|---|
| `DICT_SYS` | `dict0dict.cc:1019` | データディクショナリキャッシュ |
| `DICT_TABLE` | `dict0dict.ic:1197` | 個々のテーブル定義 |
| `AUTOINC` / `PERSIST_AUTOINC` | `dict0dict.cc:713` / `718` | AUTO_INCREMENT カウンタ／永続化値 |
| `DDL_AUTOINC` | `ddl0ctx.cc:116` | DDL 中の AUTO_INCREMENT |
| `DICT_PERSIST_DIRTY_TABLES` | `dict0dict.cc:3915` | 永続メタデータのダーティテーブル一覧 |
| `RECALC_POOL` | `dict0stats_bg.cc:231` | 統計再計算キュー |
| `ALTER_STAGE` | `ut0mutex.h:255` | ALTER の進捗 |
| `INDEX_ONLINE_LOG` | `row0log.cc:3116` | オンライン DDL のログ |
| `ROW_DROP_LIST` | `row0mysql.cc:4753` | バックグラウンド DROP リスト |
| `PARALLEL_READ` | `row0pread.cc:195` | 並列読み取り |
| `WORK_QUEUE` / `PARSER` / `ZIP_PAD_MUTEX` / `PAGE_ZIP_STAT_PER_INDEX` | 各所 | ワークキュー／内部 SQL パーサ／圧縮統計 |

### パージ・UNDO

| 名前 | created | 守るもの |
|---|---|---|
| `PURGE_SYS_PQ` | `trx0purge.cc:232` | パージ対象の優先度キュー |
| `TRX_SYS_RSEG` / `UNDO_SPACE_RSEG` / `TEMP_SPACE_RSEG` | `trx0rseg.cc:234〜238`, `380〜384` | ロールバックセグメント（置かれているテーブルスペース種別で名前が変わる） |

### チェンジバッファ・全文検索・空間索引

| 名前 | created | 守るもの |
|---|---|---|
| `IBUF` / `IBUF_BITMAP` / `IBUF_PESSIMISTIC_INSERT` | `ibuf0ibuf.cc:476` / `478` / `480` | チェンジバッファ |
| `FTS_BG_THREADS` / `FTS_DELETE` / `FTS_OPTIMIZE` / `FTS_DOC_ID` | `fts0fts.cc:5444` / `548` / `550` / `552` | FULLTEXT の背景スレッド／削除 ID／OPTIMIZE／Doc ID |
| `RTR_SSN_MUTEX` / `RTR_ACTIVE_MUTEX` / `RTR_MATCH_MUTEX` / `RTR_PATH_MUTEX` | `mem.cc` / `gis0sea.cc` | 空間インデックス（R-tree） |

### サーバー・同期基盤

| 名前 | created | 守るもの |
|---|---|---|
| `SRV_SYS` / `SRV_SYS_TASKS` | `srv0srv.cc:1156` / `1158` | スレッドスロット／タスクキュー |
| `SRV_INNODB_MONITOR` | `srv0srv.cc:1120` | INNODB STATUS 出力自体の直列化 |
| `SRV_MONITOR_FILE` / `SRV_MISC_TMPFILE` | `srv0start.cc` | 一時ファイル |
| `SYNC_ARRAY_MUTEX` | `sync0arr.cc:131` | 待機配列そのもの |
| `RW_LOCK_LIST` | `sync0debug.cc:1679` | RW ラッチ全体のリスト |

### 登録はあるが実際には現れない名前

| 名前 | 理由 |
|---|---|
| `OS_AIO_READ` / `OS_AIO_WRITE` / `OS_AIO_IBUF_MUTEX` | OS の mutex 実装で待機配列を使わない |
| `EVENT_MANAGER`, `EVENT_MUTEX`, `SYNC_THREAD`, `TEST_MUTEX`, `HASH_TABLE_MUTEX`, `FILE_OPEN` | 現れない |
| `FTS_PLL_TOKENIZE` | 実体が作られない |
| `RW_LOCK_DEBUG` | デバッグビルドのみ |

---

## 3. 待ち場所（`has waited at …`）のファイル名 → 何の処理か

| ファイル名 | 処理 | よく待つもの |
|---|---|---|
| `btr0cur.cc` / `btr0btr.cc` / `btr0pcur.cc` | B-tree の検索・挿入・分割・結合 | `index->lock`、ページラッチ |
| `row0ins.cc` / `row0upd.cc` / `row0sel.cc` / `row0purge.cc` / `row0umod.cc` | 行の INSERT / UPDATE / SELECT / パージ / ロールバック | ページラッチ（数段上の位置で出る）、`index->lock` |
| `buf0buf.cc` / `buf0lru.cc` / `buf0flu.cc` / `buf0rea.cc` | バッファプールの取得・追い出し・フラッシュ・先読み | `BUF_POOL_LRU_LIST`、`FLUSH_LIST`、`page_hash`、ページラッチ |
| `dict0dict.cc` / `dict0stats.cc` / `dict0dd.cc` | ディクショナリの参照・統計 | `DICT_SYS`、`dict_operation_lock`、統計ラッチ |
| `trx0trx.cc` / `trx0sys.h` / `trx0undo.cc` / `trx0rseg.cc` | トランザクションの開始・終了、undo | `TRX_SYS`（常に `trx0sys.h` line 599）、rseg 系 |
| `lock0lock.cc` / `lock0wait.cc` | 行ロックの取得・待ち・デッドロック検出 | `LOCK_SYS_*`、グローバルラッチ |
| `log0*.cc` | REDO ログの書き込み・チェックポイント | `LOG_*` |
| `fil0fil.cc` / `fsp0fsp.cc` | ファイル・テーブルスペース・領域割り当て | `FIL_SHARD`、テーブルスペースラッチ |
| `trx0purge.cc` / `srv0srv.cc` | パージ、バックグラウンドスレッド | パージラッチ、`PURGE_SYS_PQ`、`SRV_SYS` |
| `ibuf0ibuf.cc` / `fts0*.cc` / `gis0*.cc` | チェンジバッファ／全文検索／空間インデックス | それぞれの Mutex とラッチ |
| `ha_innodb.cc` / `handler0alter.cc` | SQL 層からの入口、ALTER TABLE | `dict_operation_lock`、`DICT_SYS` |

### 具体例（8.0.46）

| 場所 | 関数 | 待っているもの |
|---|---|---|
| `btr0cur.cc` line 823 / 830 / 832 | `btr_cur_search_to_nth_level`（ツリー変更） | `index->lock` を X / X / SX |
| `btr0cur.cc` line 865 | 同（通常の検索） | `index->lock` を S |
| `btr0cur.cc` line 299 / 359 | `btr_cur_latch_leaves` | 隣接リーフページのラッチを S |
| `row0ins.cc` line 2458 | `row_ins_clust_index_entry_low` | クラスタインデックスのページラッチ |
| `dict0dict.cc` line 1076 | `dict_table_open_on_name` | `DICT_SYS` |
| `trx0sys.h` line 599 | `trx_sys_mutex_enter`（inline） | `TRX_SYS` |
| `buf0flu.cc` line 512 | `buf_flush_insert_into_flush_list` | `FLUSH_LIST` |
| `buf0buf.cc` line 1950 | `buf_pool_withdraw_blocks`（リサイズ） | `BUF_POOL_LRU_LIST` |
