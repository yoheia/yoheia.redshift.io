
# Redshift - VACUUM



## VACCUM を高速化する

#### BOOST オプション

VACUUM で使用するメモリやストレージなどを使うリソースを増やす。他のクエリへの性能影響が問題なければ、BOOST をつける。

```
vacuum full schema1.table1 boost;
```

>Runs the VACUUM command with additional resources, such as memory and disk space, as they're available. With the BOOST option, VACUUM operates in one window and blocks concurrent deletes and updates for the duration of the VACUUM operation. Running with the BOOST option contends for system resources, which might affect query performance. Run the VACUUM BOOST when the load on the system is light, such as during maintenance operations.
>
>Consider the following when using the BOOST option:
>
>- When BOOST is specified, the table_name value is required.
>- BOOST isn't supported with REINDEX.
>- BOOST is ignored with DELETE ONLY.
> <cite>[VACUUM](https://docs.aws.amazon.com/ja_jp/redshift/latest/dg/r_VACUUM_command.html)

#### CTAS

ただし、CTAS では元テーブルと作成先のテーブルの圧縮エンコーディングが変わる。例えば、元テーブルでは encoding=zstd だったカラムが lzo になったりする。CREATE TABLE で元テーブルと同定義で空テーブルを作り、INSERT SELECT すると同じ定義にすることができる。

```
CREATE TABLE schema1.table2 
DISTSTYLE EVEN 
SORTKEY ( col1, col2 )
AS 
SELECT * FROM schema1.table1;
```

#### INSERT SELECT
日付列があるようなテーブルは insert ... select で部分的にソートして分割コミットすることができる

## 運用

- 運用は AUTO VACUUM に任せる（AUTO VACUUM の無効化はできない）
- VACUUM は中断しても開始前の状態には戻らない。途中まで VACUUM が進んだ状態になる。
- テーブル名を指定しないとクラスターの全テーブルを VACUUM することができる。

>すべてのテーブルで、デフォルトのバキュームしきい値 95 パーセントに基づいて、容量とデータベースの再利用と行の再ソートを実行します。
>
>```
>vacuum;
>```

### VACUMM が必要なテーブルを確認する

- [`SVV_TABLE_INFO`](https://docs.aws.amazon.com/ja_jp/redshift/latest/dg/r_SVV_TABLE_INFO.html) の `unsorted` が 0 より大きいテーブルは VACUUM の余地がある。
- ソートキーが指定されていないテーブルの `unsorted`、`vacuum_sort_benefit ` は `null` になる。

### VACUMM の実行状況を確認する

- `svv_vacuum_progress` の `time_remaining_estimate` で残り時間の予測を確認できる。

> | Column name |  Data type  | Description |
> | ---- | ---- | ---- |
> |  status  |  text |  Description of the current activity being done as part of the vacuum operation: <br>-Initialize<br>-Sort<br>-Merge<br>-Delete<br>-Select<br>-Failed<br>-Complete<br>-Skipped<br>-Building INTERLEAVED SORTKEY order |
> | `time_remaining_estimate`  |  text  |  Estimated time left for the current vacuum operation to complete, in minutes and seconds: 5m 10s, for example. An estimated time is not returned until the vacuum completes its first sort operation. If no vacuum is in progress, the last vacuum that was performed is displayed with Completed in the STATUS column and an empty `TIME_REMAINING_ESTIMATE` column. The estimate typically becomes more accurate as the vacuum progresses.  |
>
>
>The following queries, run a few minutes apart, show that a large table named SALESNEW is being vacuumed.
>
> ```
> select * from svv_vacuum_progress;
> table_name    |            status             | time_remaining_estimate
> --------------+-------------------------------+-------------------------
> salesnew      |  Vacuum: initialize salesnew  |
> (1 row)
> ...
> select * from svv_vacuum_progress;
> 
> table_name   |         status         | time_remaining_estimate
> -------------+------------------------+-------------------------
> salesnew     |  Vacuum salesnew sort  | 33m 21s ★ 残り時間の予測
> (1 row)
> ```
>The following query shows that no vacuum operation is currently in progress. The last table to be vacuumed was the SALES table.
>
>```
>select * from svv_vacuum_progress;
>
>table_name   |  status  | time_remaining_estimate
>-------------+----------+-------------------------
>  sales      | Complete |
> (1 row)
```

- VACUUM の実行状況を確認できるシステムテーブル・ビュー
	- [`STL_VACUUM`](https://docs.aws.amazon.com/ja_jp/redshift/latest/dg/r_STL_VACUUM.html)
	- [`SVV_VACUUM_SUMMARY`](https://docs.aws.amazon.com/ja_jp/redshift/latest/dg/r_SVV_VACUUM_SUMMARY.html)
	- [`SVV_VACUUM_PROGRESS`](https://docs.aws.amazon.com/ja_jp/redshift/latest/dg/r_SVV_VACUUM_PROGRESS.html)

## 注意事項

#### 複数テーブルの VACUUM を同時に実行できない。

>一度にクラスターで実行できる VACUUM コマンドは 1 つだけです。複数のバキューム動作を同時に実行しようとすると、Amazon Redshift はエラーを返します。
-- <cite>[VACUUM - 使用に関する注意事項](https://docs.aws.amazon.com/ja_jp/redshift/latest/dg/r_VACUUM_command.html)</cite>

