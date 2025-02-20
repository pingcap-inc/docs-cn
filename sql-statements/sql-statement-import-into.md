---
title: IMPORT INTO
summary: TiDB 数据库中 IMPORT INTO 的使用概况。
---

# IMPORT INTO

`IMPORT INTO` 语句使用 TiDB Lightning 的[物理导入模式](/tidb-lightning/tidb-lightning-physical-import-mode.md)，用于将 `CSV`、`SQL`、`PARQUET` 等格式的数据导入到 TiDB 的一张空表中。

> **警告：**
>
> 目前该语句为实验特性，不建议在生产环境中使用。

`IMPORT INTO` 支持导入存储在 Amazon S3、GCS 和 TiDB 本地的数据文件。

- 对于存储在 S3 或 GCS 的数据文件，`IMPORT INTO` 支持通过[后端任务分布式框架](/tidb-distributed-execution-framework.md)运行。

    - 当此框架功能开启时（即 [tidb_enable_dist_task](/system-variables.md#tidb_enable_dist_task-从-v710-版本开始引入) 为 `ON`），`IMPORT INTO` 会将一个数据导入任务拆分成多个子任务并分配到各个 TiDB 节点上运行，以提高导入效率。
    - 当此框架功能关闭时，`IMPORT INTO` 仅支持在当前用户连接的 TiDB 节点上运行。

- 对于存储在 TiDB 本地的数据文件，`IMPORT INTO` 仅支持在当前用户连接的 TiDB 节点上运行，因此数据文件需要存放在当前用户连接的 TiDB 节点上。如果是通过 PROXY 或者 Load Balancer 访问 TiDB，则无法导入存储在 TiDB 本地的数据文件。

## 使用限制

- 目前该语句仅支持导入 1 TiB 以下的数据。
- 只支持导入数据到数据库中已有的空表。
- 不支持事务，也无法回滚。在显式事务 (`BEGIN`/`END`) 中执行会报错。
- 在导入完成前会阻塞当前连接，如果需要异步执行，可以添加 `DETACHED` 选项。
- 不支持和 [Backup & Restore](/br/backup-and-restore-overview.md)、[`FLASHBACK CLUSTER TO TIMESTAMP`](/sql-statements/sql-statement-flashback-to-timestamp.md)、[创建索引加速](/system-variables.md#tidb_ddl_enable_fast_reorg-从-v630-版本开始引入)、TiDB Lightning 导入、TiCDC 数据同步、[Point-in-time recovery (PITR)](/br/br-log-architecture.md) 等功能同时工作。
- 每个集群上同时只能有一个 `IMPORT INTO` 任务在运行。`IMPORT INTO` 会 precheck 是否存在运行中的任务，但并非硬限制，如果多个客户端同时执行 `IMPORT INTO` 仍有可能启动多个任务，请避免该情况，否则可能导致数据不一致或者任务失败的问题。
- 导入数据的过程中，请勿在目标表上执行 DDL 和 DML 操作，也不要在目标数据库上执行 [`FLASHBACK DATABASE`](/sql-statements/sql-statement-flashback-database.md)，否则会导致导入失败或数据不一致。导入期间也不建议进行读操作，因为读取的数据可能不一致。请在导入完成后再进行读写操作。
- 导入期间会占用大量系统资源，建议 TiDB 节点使用 32 核以上的 CPU 和 64 GiB 以上内存以获得更好的性能。导入期间会将排序好的数据写入到 TiDB [临时目录](/tidb-configuration-file.md#temp-dir-从-v630-版本开始引入)下，建议优先考虑配置闪存等高性能存储介质。详情请参考[物理导入使用限制](/tidb-lightning/tidb-lightning-physical-import-mode.md#必要条件及限制)。
- TiDB [临时目录](/tidb-configuration-file.md#temp-dir-从-v630-版本开始引入)至少需要有 90 GiB 的可用空间。建议预留大于等于所需导入数据的存储空间，以保证最佳导入性能。
- 一个导入任务只支持导入数据到一张目标表中。如需导入数据到多张目标表，需要在一张目标表导入完成后，再新建一个任务导入下一张目标表。

## 导入前准备

在使用 `IMPORT INTO` 开始导入数据前，请确保：

- 要导入的目标表在 TiDB 中已经创建，并且是空表。
- 当前集群有足够的剩余空间能容纳要导入的数据。
- 当前连接的 TiDB 节点的[临时目录](/tidb-configuration-file.md#temp-dir-从-v630-版本开始引入)至少有 90 GiB 的磁盘空间。如果开启了 [`tidb_enable_dist_task`](/system-variables.md#tidb_enable_dist_task-从-v710-版本开始引入)，需要确保集群中所有 TiDB 节点的临时目录都有足够的磁盘空间。

## 需要的权限

执行 `IMPORT INTO` 的用户需要有目标表的 `SELECT`、`UPDATE`、`INSERT`、`DELETE` 和 `ALTER` 权限。如果是导入存储在 TiDB 本地的文件，还需要有 `FILE` 权限。

## 语法图

```ebnf+diagram
ImportIntoStmt ::=
    'IMPORT' 'INTO' TableName ColumnNameOrUserVarList? SetClause? FROM fileLocation Format? WithOptions?

ColumnNameOrUserVarList ::=
    '(' ColumnNameOrUserVar (',' ColumnNameOrUserVar)* ')'

SetClause ::=
    'SET' SetItem (',' SetItem)*

SetItem ::=
    ColumnName '=' Expr

Format ::=
    'CSV' | 'SQL' | 'PARQUET'

WithOptions ::=
    'WITH' OptionItem (',' OptionItem)*

OptionItem ::=
    optionName '=' optionVal | optionName
```

## 参数说明

### ColumnNameOrUserVarList

用于指定数据文件中每行的各个字段如何对应到目标表列，也可以将字段对应到某个变量，用来跳过导入某些字段或者在 `SetClause` 中使用。

- 当不指定该参数时，数据文件中每行的字段数需要和目标表的列数一致，且各个字段会按顺序导入到对应的列。
- 当指定该参数时，指定的列或变量个数需要和数据文件中每行的字段数一致。

### SetClause

用于指定目标列值的计算方式。在 SET 表达式的右侧，可以引用在 `ColumnNameOrUserVarList` 中指定的变量。

SET 表达式左侧只能引用 `ColumnNameOrUserVarList` 中没有的列名。如果目标列名已经在 `ColumnNameOrUserVarList` 中，则该 SET 表达式无效。

### fileLocation

用于指定数据文件的存储位置，该位置可以是 S3 或 GCS URI 路径，也可以是 TiDB 本地文件路径。

- S3 或 GCS URI 路径：配置详见[外部存储](/br/backup-and-restore-storages.md#uri-格式)。
- TiDB 本地文件路径：必须为绝对路径，数据文件后缀必须为 `.csv`、`.sql` 或 `.parquet`。确保该路径对应的文件存储在当前用户连接的 TiDB 节点上，且当前连接的用户有 `FILE` 权限。

> **注意：**
>
> 如果目标集群开启了 [SEM](/system-variables.md#tidb_enable_enhanced_security)，则 fileLocation 不能指定为本地文件路径。

使用 fileLocation 可以指定单个文件，也可使用通配符 `*` 来匹配需要导入的多个文件。注意通配符只能用在文件名部分，不会匹配目录，也不会递归处理子目录下相关的文件。下面以数据存储在 S3 为例：

- 导入单个文件：`s3://<bucket-name>/path/to/data/foo.csv`
- 导入指定路径下的所有文件：`s3://<bucket-name>/path/to/data/*`
- 导入指定路径下的所有以 `.csv` 结尾的文件：`s3://<bucket-name>/path/to/data/*.csv`
- 导入指定路径下所有以 `foo` 为前缀的文件：`s3://<bucket-name>/path/to/data/foo*`
- 导入指定路径下以 `foo` 为前缀、以 `.csv` 结尾的文件：`s3://<bucket-name>/path/to/data/foo*.csv`

### Format

`IMPORT INTO` 支持 3 种数据文件格式，包括 `CSV`、`SQL` 和 `PARQUET`。当不指定该参数时，默认格式为 `CSV`。

### WithOptions

你可以通过 WithOptions 来指定导入选项，控制数据导入过程。例如，如需使导入任务在后台异步执行，你可以通过在 `IMPORT INTO` 语句中添加 `WITH DETACHED` 选项来开启导入任务的 `DETACHED` 模式。

目前支持的选项包括：

| 选项名 | 支持的数据格式 | 描述 |
|:---|:---|:---|
| `CHARACTER_SET='<string>'` | CSV | 指定数据文件的字符集，默认为 `utf8mb4`。目前支持的字符集包括 `binary`、`utf8`、`utf8mb4`、`gb18030`、`gbk`、`latin1` 和 `ascii`。 |
| `FIELDS_TERMINATED_BY='<string>'` | CSV | 指定字段分隔符，默认为 `,`。 |
| `FIELDS_ENCLOSED_BY='<char>'` | CSV | 指定字段的定界符，默认为 `"`。 |
| `FIELDS_ESCAPED_BY='<char>'` | CSV | 指定字段的转义符，默认为 `\`。 |
| `FIELDS_DEFINED_NULL_BY='<string>'` | CSV | 指定字段为何值时将会被解析为 NULL，默认为 `\N`。 |
| `LINES_TERMINATED_BY='<string>'` | CSV | 指定行分隔符，默认 `IMPORT INTO` 会自动识别分隔符为 `\n`、`\r` 或 `\r\n`，如果行分隔符为以上三种，无须显式指定该选项。 |
| `SKIP_ROWS=<number>` | CSV | 指定需要跳过的行数，默认为 `0`。可通过该参数跳过 CSV 中的 header，如果是通过通配符来指定所需导入的源文件，该参数会对 fileLocation 中通配符匹配的所有源文件生效。 |
| `DISK_QUOTA='<string>'` | 所有格式 | 指定数据排序期间可使用的磁盘空间阈值。默认值为 TiDB [临时目录](/tidb-configuration-file.md#temp-dir-从-v630-版本开始引入)所在磁盘空间的 80%。如果无法获取磁盘总大小，默认值为 50 GiB。当显式指定 DISK_QUOTA 时，该值同样不能超过 TiDB [临时目录](/tidb-configuration-file.md#temp-dir-从-v630-版本开始引入)所在磁盘空间的 80%。 |
| `DISABLE_TIKV_IMPORT_MODE` | 所有格式 | 指定是否禁止导入期间将 TiKV 切换到导入模式。默认不禁止。如果当前集群存在正在运行的读写业务，为避免导入过程对这部分业务造成影响，可开启该参数。 |
| `THREAD=<number>` | 所有格式 | 指定导入的并发度。默认值为 TiDB 节点的 CPU 核数的 50%，最小值为 1。可以显示指定该参数来控制对资源的占用，但最大值不能超过 CPU 核数。如需导入数据到一个空集群，建议可以适当调大该值，以提升导入性能。如果目标集群已经用于生产环境，请根据业务要求按需调整该参数值。 |
| `MAX_WRITE_SPEED='<string>'` | 所有格式 | 控制写入到单个 TiKV 的速度，默认无速度限制。例如设置为 `1MiB`，则限制写入速度为 1 MiB/s。|
| `CHECKSUM_TABLE='<string>'` | 所有格式 | 配置是否在导入完成后对目标表是否执行 CHECKSUM 检查来验证导入的完整性。可选的配置项为 `"required"`（默认）、`"optional"` 和 `"off"`。`"required"` 表示在导入完成后执行 CHECKSUM 检查，如果 CHECKSUM 检查失败，则会报错退出。`"optional"` 表示在导入完成后执行 CHECKSUM 检查，如果报错，会输出一条警告日志并忽略报错。`"off"` 表示导入结束后不执行 CHECKSUM 检查。 |
| `DETACHED` | 所有格式 | 该参数用于控制 `IMPORT INTO` 是否异步执行。开启该参数后，执行 `IMPORT INTO` 会立即返回该导入任务的 `Job_ID` 等信息，且该任务会在后台异步执行。 |

## 输出内容

当 `IMPORT INTO` 导入完成，或者开启了 `DETACHED` 模式时，`IMPORT INTO` 会返回当前任务的信息。以下为一些示例，字段的含义描述请参考 [`SHOW IMPORT JOB(s)`](/sql-statements/sql-statement-show-import-job.md)。

当 `IMPORT INTO` 导入完成时输出：

```sql
IMPORT INTO t FROM '/path/to/small.csv';
+--------+--------------------+--------------+----------+-------+----------+------------------+---------------+----------------+----------------------------+----------------------------+----------------------------+------------+
| Job_ID | Data_Source        | Target_Table | Table_ID | Phase | Status   | Source_File_Size | Imported_Rows | Result_Message | Create_Time                | Start_Time                 | End_Time                   | Created_By |
+--------+--------------------+--------------+----------+-------+----------+------------------+---------------+----------------+----------------------------+----------------------------+----------------------------+------------+
|  60002 | /path/to/small.csv | `test`.`t`   |      363 |       | finished | 16B              |             2 |                | 2023-06-08 16:01:22.095698 | 2023-06-08 16:01:22.394418 | 2023-06-08 16:01:26.531821 | root@%     |
+--------+--------------------+--------------+----------+-------+----------+------------------+---------------+----------------+----------------------------+----------------------------+----------------------------+------------+
```

开启了 `DETACHED` 模式时，执行 `IMPORT INTO` 语句会立即返回输出。从输出中，你可以看到该任务状态 `Status` 为 `pending`，表示等待执行。

```sql
IMPORT INTO t FROM '/path/to/small.csv' WITH DETACHED;
+--------+--------------------+--------------+----------+-------+---------+------------------+---------------+----------------+----------------------------+------------+----------+------------+
| Job_ID | Data_Source        | Target_Table | Table_ID | Phase | Status  | Source_File_Size | Imported_Rows | Result_Message | Create_Time                | Start_Time | End_Time | Created_By |
+--------+--------------------+--------------+----------+-------+---------+------------------+---------------+----------------+----------------------------+------------+----------+------------+
|  60001 | /path/to/small.csv | `test`.`t`   |      361 |       | pending | 16B              |          NULL |                | 2023-06-08 15:59:37.047703 | NULL       | NULL     | root@%     |
+--------+--------------------+--------------+----------+-------+---------+------------------+---------------+----------------+----------------------------+------------+----------+------------+
```

## 查看和控制导入任务

对于开启了 `DETACHED` 模式的任务，可通过 [`SHOW IMPORT`](/sql-statements/sql-statement-show-import-job.md) 来查看当前任务的执行进度。

任务启动后，可通过 [`CANCEL IMPORT JOB <job-id>`](/sql-statements/sql-statement-cancel-import-job.md) 来取消对应任务。

## 使用示例

### 导入带有 header 的 CSV 文件

```sql
IMPORT INTO t FROM '/path/to/file.csv' WITH skip_rows=1;
```

### 以 `DETACHED` 模式异步导入

```sql
IMPORT INTO t FROM '/path/to/file.csv' WITH DETACHED;
```

### 忽略数据文件中的特定字段

假设数据文件为以下 CSV 文件：

```
id,name,age
1,Tom,23
2,Jack,44
```

假设要导入的目标表结构为 `CREATE TABLE t(id int primary key, name varchar(100))`，则可通过以下方式来忽略导入文件中的 `age` 字段：

```sql
IMPORT INTO t(id, name, @1) FROM '/path/to/file.csv' WITH skip_rows=1;
```

### 使用通配符 `*` 导入多个数据文件

假设在 `/path/to/` 目录下有 `file-01.csv`、`file-02.csv` 和 `file-03.csv` 三个文件，如需通过 `IMPORT INTO` 将这三个文件导入到目标表 `t` 中，可使用如下 SQL 语句：

```sql
IMPORT INTO t FROM '/path/to/file-*.csv'
```

### 从 S3、GCS 导入数据

- 从 S3 导入数据

    ```sql
    IMPORT INTO t FROM 's3://bucket-name/test.csv?access-key=XXX&secret-access-key=XXX';
    ```

- 从 GCS 导入数据

    ```sql
    IMPORT INTO t FROM 'gs://bucket-name/test.csv';
    ```

S3 或 GCS 的 URI 路径配置详见[外部存储](/br/backup-and-restore-storages.md#uri-格式)。

### 通过 SetClause 语句计算列值

假设数据文件为以下 CSV 文件：

```
id,name,val
1,phone,230
2,book,440
```

要导入的目标表结构为 `CREATE TABLE t(id int primary key, name varchar(100), val int)`，并且希望在导入时将 `val` 列值扩大 100 倍，则可通过以下方式来导入：

```sql
IMPORT INTO t(id, name, @1) SET val=@1*100 FROM '/path/to/file.csv' WITH skip_rows=1;
```

### 导入 SQL 格式的数据文件

```sql
IMPORT INTO t FROM '/path/to/file.sql' FORMAT 'sql';
```

### 限制写入 TiKV 的速度

限制写入单个 TiKV 的速度为 10 MiB/s：

```sql
IMPORT INTO t FROM 's3://bucket/path/to/file.parquet?access-key=XXX&secret-access-key=XXX' FORMAT 'parquet' WITH MAX_WRITE_SPEED='10MiB';
```

## MySQL 兼容性

该语句是对 MySQL 语法的扩展。

## 另请参阅

* [`SHOW IMPORT JOB(s)`](/sql-statements/sql-statement-show-import-job.md)
* [`CANCEL IMPORT JOB`](/sql-statements/sql-statement-cancel-import-job.md)
