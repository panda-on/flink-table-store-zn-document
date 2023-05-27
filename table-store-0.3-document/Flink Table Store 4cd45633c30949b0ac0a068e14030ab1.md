# Flink Table Store

# 1. Table Store概述

## 1.1 架构图

![Untitled](Flink%20Table%20Store%204cd45633c30949b0ac0a068e14030ab1/Untitled.png)

### 读写

读取数据: 支持历史快照读取,从最近偏移量开始流式读取和混合方式读取增量快照

写入数据: 支持流式同步数据库的变更记录(CDC)或者离线批量写入

### 生态

可以支持Hive,Spark,Trino读取Table Store的数据

### 内核架构

在table store的外壳下,table store采用混合存储架构，使用lake format存储历史数据，使用队列系统存储存储增量数据。lake format存储列式文件,这些文件存储在文件系统或者对象存储上,使用的是LSM树结构.这样的架构支持大数据量的更新和高性能查询.队列系统使用Kafka实时的抓取数据

### 补充资料

LSM-Tree原理

[深入浅出分析LSM树（日志结构合并树）](https://zhuanlan.zhihu.com/p/415799237)

## 1.2 统一存储

Flink SQL中的三种类型的链接器:

1. 消息队列, 比如Kafka, 在数据源和中间阶段都有用到以保证秒级别的延迟
2. OLAP系统, 比如Clickhouse, Doris, 这些数据库流式地写入数据并且服务用户的即席查询
3. 批存储系统, 比如Hive

Flink Table Store提供了表的抽象, 它的使用方法和传统的数据库无异, 在批模式可以像Hive一样使用, 在Flink流执行模式执行查询像是在一个历史数据永不过期的消息队列中查询一个流变化记录.

## 1.3 基本概念

Snapshot(快照): 截取一个表在某个时间节点的状态, 用户可以指定快照的版本进行访问

Partition(区): Table Store应用了和Hive相同的分区概念用于分割数据, 分区的操作是可选的. 一个表可以选择一个或者多个分区字段. 使用分区可以高效地操作表的一个切片.

如果表定义了主键, 那么分区键一定是主键的子集

Bucket(桶):  桶是读写的最小存储单元. 未分区的表或者分区表的某个分区可以进一步分桶以提供更加有效的查询. 根据bucket-key所指定的字段的hash值进行分桶, 如果没有指定bucket-key则按照主键字段hash值进行分桶. 分桶数限制了最大并行处理度, 分桶数不应太多. 推荐的桶大小为1GB.

Consistency Guarantees(一致性保证): Table Store的写出器使用两阶段提交协议自动提交一个批次的记录到表中. 每次提交至多在提交时产生两个快照. 如果有两个写出同时修改一张表, 只要它们不同时修改相同的桶(Bucket)就只能保证快照隔离. 最终修改完的表为两个提交的混合版本, 但是不会有记录丢失.

## 1.4 文件结构

一个表的所有文件存储在一个基准目录下. Table Store的文件是分层存储的. 存储层次如下图: 

![Untitled](Flink%20Table%20Store%204cd45633c30949b0ac0a068e14030ab1/Untitled%201.png)

### Snapshot Files(快照文件)

所有snapshot files都是存储在snapshot这个目录下.  一个snapshot文件是一个JSON文件, 文件中包含这个snapshot的信息. schema和manifest list, 其中manifest list包含snapshot的所有变化.

### Manifest Files(清单文件)

所有的manifest lists和manifest files都存储在manifest目录. 一个manifest list是manifest 文件的列表.一个manifest文件包含LSM数据的变化和变化日志(Changelog)文件的变动(创建, 更新, 删除等)

### Data Files

数据文件被分成不同的分区和桶, 每个桶的目录包含一个LSM Tree和它相应的变化日志文件.

### LSM Trees

Table Store应用LSM Tree作为文件存储的数据结构. LSM Tree的两个重要概念为Sorted Runs和Compaction.

Sorted Runs(排序运行), LSM Tree把数据文件组织分成几个sorted runs. 每个sorted runs包含一个或多个数据文件, 每个数据文件仅属于一个sorted run. 一个数据文件中的数据根据主键排序. 一个sorted run中的记录的主键值不重复, 而不同的sorted run中的记录可能主键值可能重复, 在查询的时候搜友sorted runs都会被合并, 所有相同主键的记录都会根据用户定义的合并引擎和时间戳进行合并.

随着数据的增多, sorted runs也会增多，太多的sorted runs会导致性能下降也可能会导致OOM。为了限制sorted runs数量，需要将几个sorted runs合并为一个大的sorted run，这个过程被称为compaction（合并）。太多的compaction会导致写入性能下降，因此必须在写入和查询之间取得性能的平衡。

Table Store采用的compaction策略和Rocksdb的策略相似。Rocksdb的Universal Compaction合并策略如下。

[Universal Compaction](https://github.com/facebook/rocksdb/wiki/Universal-Compaction)

# 2. 引擎

Table Store不仅原生支持Flink SQL的写入和查询，它还支持基于其他的流行的引擎进行查询，例如Hive、Spark，兼容性关系如下：

| Engine | Version | Feature | Read Pushdown |
| --- | --- | --- | --- |
| Flink | 1.16/1.15/1.14 | batch/streaming read, batch/streaming write, create/drop table, create/drop database | Projection, Filter |
| Hive | 3.1/2.3/2.2/2.1/2.1 CDH 6.3 | batch read | Projection, Filter |
| Spark | 3.3/3.2/3.1 | batch read, batch write, create/drop table, create/drop database | Projection, Filter |
| Spark | 2.4 | batch read | Projection, Filter |
| Trino | 388/358 | batch read | Projection, Filter |

## 2.1 基于Flink的Table Store的使用

Step1: 下载Flink，解压安装

```bash
tar -xzf flink-*.tgz
```

Step2: 拷贝Table Store的.jar包到Flink的lib目录下

```bash
cp flink-table-store-flink-*.jar <FLINK_HOME>/lib/
```

Step3: 拷贝Flink与Hadoop捆绑jar包到Flink的lib目录下

```bash
cp flink-shaded-hadoop-2-uber-*.jar <FLINK_HOME>/lib/
```

Step4: 开启一个Flink本地集群（Local Cluster）开启一个SQL Client

```bash
<FLINK_HOME>/bin/start-cluster.sh
<FLINK_HOME>/bin/sql-client.sh
```

Step5: 开发示例

```sql
-- 在分布式系统中使用table store必须要指定warehouse的路径以存储catalog
CREATE CATALOG my_catalog WITH (
    'type'='table-store',
    'warehouse'='file:/tmp/table_store'
);

-- 切换catalog
USE CATALOG my_catalog;

-- 在my_catalog下创建一个表
CREATE TABLE word_count (
    word STRING PRIMARY KEY NOT ENFORCED,
    cnt BIGINT
);

--- 创建数据生成临时表
CREATE TEMPORARY TABLE word_table (
    word STRING
) WITH (
    'connector' = 'datagen',
    'fields.word.length' = '1'
);

-- 在流模式中需要指定checkpoint的间隔
SET 'execution.checkpointing.interval' = '10 s';

-- 将流式数据插入到动态表中 
INSERT INTO word_count SELECT word, COUNT(*) FROM word_table GROUP BY word;
```

Step6：退出客户端，关闭集群

```bash
-- 退出SQL-Client
Exit 
-- 退出集群
./bin/stop-cluster.sh
```

## 2.2 Flink支持的数据类型

[https://nightlies.apache.org/flink/flink-docs-release-1.16/docs/dev/table/types/](https://nightlies.apache.org/flink/flink-docs-release-1.16/docs/dev/table/types/)

# 3. 文件系统

Flink Table Store使用的是和Flink相同的可插拔的文件系统。

Flink Table Store支持的文件系统如下表所示。

| FileSystem | URI Scheme | Pluggable | Description |
| --- | --- | --- | --- |
| Local File System | file:// | N | Built-in Support |
| HDFS | hdfs:// | N | Built-in Support, ensure that the cluster is in the hadoop environment |
| Aliyun OSS | oss:// | Y |  |
| S3 | s3:// | Y |  |

# 4. 如何基于Flink Table Store进行开发

## 4.1 创建Catalog

Table Store支持两种类型的metastore，分别是filesystem metastore和hive metastore。filesystem metastore是默认的元数据存储模式，在这种情况下元数据和表的相关文件都存储到文件系统中。对于hive metastore模式，元数据存储在Hive metastore中，用户可以直接从Hive访问这些表。 

关于Catalog可选的配置见如下链接

[https://nightlies.apache.org/flink/flink-table-store-docs-master/docs/maintenance/configurations/#catalogoptions](https://nightlies.apache.org/flink/flink-table-store-docs-master/docs/maintenance/configurations/#catalogoptions)

### 4.1.1 创建基于文件系统存储元数据的Catalog

```sql
CREATE CATALOG my_catalog WITH (
    'type' = 'table-store',
    'warehouse' = 'hdfs://path/to/warehouse'
);

USE CATALOG my_catalog;
```

### 4.1.2 创建基于Hive Metastore服务存储元数据的Catalog

Step1：准备Table Store Hive Catalog的Jar包，并将该Jar包拷贝到Flink安装目录的lib目录下，这一步需要在启动集群前完成。如果在SQL Client中使用，则需要在启动命令中添加如下内容：

- -- jar /path/to/flink-table-store-hive-catalog-0.4-SNAPSHOT.jar

Step2：创建基于Hive Metastore的Catalog

```sql
CREATE CATALOG my_hive WITH (
    'type' = 'table-store',
    'metastore' = 'hive',
    'uri' = 'thrift://<hive-metastore-host-name>:<port>',
    'warehouse' = 'hdfs://path/to/warehouse'
);

USE CATALOG my_hive;
```

## 4.2 创建表

Table Store的catalog中创建的表由catalog进行管理。当在一个catalog中删除一个表时，这个表相应的文件也会被删除。这个特点区别于Hive的外部表。

### 4.2.1 建表语句

**注意！：在创建分区表时，分区键必须是主键的子集**

```sql
-- 在选择的catalog下的default数据创建MyTable表
CREATE TABLE MyTable (
    user_id BIGINT,
    item_id BIGINT,
    behavior STRING,
    dt STRING,
    hh STRING,
    PRIMARY KEY (dt, hh, user_id) NOT ENFORCED
);
```

### 4.2.2 建分区表语句

创建的分区表可以使用partition.expiration-time配置设置分区有效时间，当时间过去了这个配置指定的时间分区数据会被自动删除。创建分区表的示例如下：

**注意！：在创建分区表时分区键必须是主键的子集；**

```sql
-- 创建分区表，主键为dt，hh，user_id，分区字段为dt，hh
CREATE TABLE MyTable (
    user_id BIGINT,
    item_id BIGINT,
    behavior STRING,
    dt STRING,
    hh STRING,
    PRIMARY KEY (dt, hh, user_id) NOT ENFORCED
) PARTITIONED BY (dt, hh);
```

### 4.2.3 Create Table As语句

使用create table as语句可以同时创建一个表并向这个表插入数据。具体的语法如下：

```sql
-- 示例一：创建表并插入数据
CREATE TABLE MyTable (
    user_id BIGINT,
    item_id BIGINT
);
CREATE TABLE MyTableAs AS SELECT * FROM MyTable;

-- 示例二：创建分区表
-- 使用create table as 创建分区表
CREATE TABLE MyTablePartition (
     user_id BIGINT,
     item_id BIGINT,
     behavior STRING,
     dt STRING,
     hh STRING
) PARTITIONED BY (dt, hh);
-- 在AS关键词前使用WITH指定分区字段
CREATE TABLE MyTablePartitionAs WITH ('partition' = 'dt') AS SELECT * FROM MyTablePartition;

-- 示例三：修改选项
CREATE TABLE MyTableOptions (
       user_id BIGINT,
       item_id BIGINT
) WITH ('file.format' = 'orc');
CREATE TABLE MyTableOptionsAs WITH ('file.format' = 'parquet') AS SELECT * FROM MyTableOptions;

-- 示例四：指定主键
CREATE TABLE MyTablePk (
      user_id BIGINT,
      item_id BIGINT,
      behavior STRING,
      dt STRING,
      hh STRING,
      PRIMARY KEY (dt, hh, user_id) NOT ENFORCED
) ;
CREATE TABLE MyTablePkAs WITH ('primary-key' = 'dt,hh') AS SELECT * FROM MyTablePk;
```

### 4.2.4 Create Table Like语句

使用Create Table Like语句创建一个与另一个表的模式，分区和表属性一样的新表。语句示例如下：

```sql
CREATE TABLE MyTable (
    user_id BIGINT,
    item_id BIGINT,
    behavior STRING,
    dt STRING,
    hh STRING,
    PRIMARY KEY (dt, hh, user_id) NOT ENFORCED
) ;

CREATE TABLE MyTableLike LIKE MyTable;
```

## 4.3 改变表

### 4.3.1 改变和修改表的属性

```sql
-- 修改表的写入缓存大小为256M
ALTER TABLE my_table SET (
    'write-buffer-size' = '256 MB'
);
```

### 4.3.2 重命名表

注意！：在S3或者OSS等对象存储系统中这个语句不是原子的，当语句运行失败时可能会导致只有部分数据被移动。

```sql
ALTER TABLE my_table RENAME TO my_table_new;
```

### 4.3.3 添加新的列

```sql
ALTER TABLE my_table ADD COLUMNS (
    c1 INT,
    c2 STRING
);
```

## 4.4 写入表

Flink SQL中可以使用INSERT语句插入新的行到一个表中或者覆写表中已经存在的数据。插入的行可以由值表达式或者一个查询指定。

### 4.4.1 写入语法

**注意：**

- 如果指定要插入的列，那么值表达式或者查询结果的列的个数和数据类型应该能与要插入的列个数和数据类型相匹配。另外，要插入数据的列必须存在于要插入数据的表中。
- 对于值表达式，需要显式的指定某个具体的值或者NULL，值之间需要使用逗号隔开，一次可以插入多条数据。
- 目前Flink不支持直接插入NULL，所以应该将NULL转化为指定的数据类型，比如：CAST(NULL AS data_type)。
- 流式读取数据将会忽略掉有由INSERT OVERWRITE默认产生的COMMIT，如果要读取这部分数据，需要配置“streaming-read-overwrite”
- 在Sink阶段，Table Store支持以桶（Bucket）为单位打散（shuffle）数据，为了避免数据倾斜，Table Store也支持以分区字段打散数据。要实现这个效果需要添加配置“sink.partition-shuffle”到表中

```sql
INSERT { INTO | OVERWRITE } table_identifier [ part_spec ] [ column_list ] { value_expr | query }
INSERT { INTO | OVERWRITE } 表名 [ 指定分区（多个分区使用键值对以逗号隔开） ] [ 指定的列 ] { 值表达式 | 查询语句 }
```

### 4.4.2 覆写整张表

对于没有分区的表，Table Store支持覆写整张表

```sql
INSERT OVERWRITE MyTable SELECT ...
```

### 4.4.3 覆写分区的数据

```sql
INSERT OVERWRITE MyTable PARTITION (key1 = value1, key2 = value2, ...) SELECT ...
```

### 4.4.4 使用INSERT OVERWRITE清除数据

```sql
-- 清除整个表的数据
INSERT OVERWRITE MyTable SELECT * FROM MyTable WHERE false
-- 清除分区的数据
INSERT OVERWRITE MyTable PARTITION (key1 = value1, key2 = value2, ...) SELECT selectSpec FROM MyTable WHERE false
```

## 4.5 从表中删除数据(Since version 0.4)

目前Flink Table Store支持删除表中的行，要达到这个目的需要使用flink run命令提交”delete“任务。示例入下：

```bash
<FLINK_HOME>/bin/flink run \
    -c org.apache.flink.table.store.connector.action.FlinkActions \
    /path/to/flink-table-store-flink-**-0.4-SNAPSHOT.jar \
    delete \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table <table-name>
    --where <filter_spec>
 
# filter_spec is equal to the 'WHERE' clause in SQL DELETE statement. Examples:
#    age >= 18 AND age <= 60
#    animal <> 'cat'
#    id > (SELECT count(*) FROM employee)
```

## 4.6 合并表(Since version 0.4)

Table Store支持通过flink run命令运行“MERGE INTO”任务，”MERGE INTO“任务可以把来源表的数据合并到目标表中。合并使用的是“upsert“语义而不是”update”语义，这意味着在目标表中如果存在符合匹配条件的一行数据，那么合并操作会更新这一行；如果这一行数据在目标表中本来不存在，那么就会插入这条数据到目标表。

> “upsert”和“update”的区别在于，对于一个没有主键的表，你可以更新任意的列；但是在不用”upsert“的情况下，对于有主键的表更新主键就必须插入一条新的包含修改后主键的数据。
> 

**注意！：**

1. **只有有主键的表支持这个特性；**
2. **这语句不会生成UPDATE_BEFORE，所以不推荐使用配置‘changelog-producer’ = ‘input’**

```sql
MERGE INTO target-table
  USING source-table | source-expr AS source-alias
  ON merge-condition
  WHEN MATCHED [AND matched-condition]
    THEN UPDATE SET xxx
  WHEN MATCHED [AND matched-condition]
    THEN DELETE
  WHEN NOT MATCHED [AND not-matched-condition]
    THEN INSERT VALUES (xxx)
  WHEN NOT MATCHED BY SOURCE [AND not-matched-by-source-condition]
    THEN UPDATE SET xxx
  WHEN NOT MATCHED BY SOURCE [AND not-matched-by-source-condition]
    THEN DELETE
```

```bash
<FLINK_HOME>/bin/flink run \
    -c org.apache.flink.table.store.connector.action.FlinkActions \
    /path/to/flink-table-store-flink-**-0.4-SNAPSHOT.jar \
    merge-into \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table <target-table> \
    [--target-as <target-table-alias>] \
    --source-table <source-table> \
    [--source-as <source-table-alias>] \
    --on <merge-condition> \
    --merge-actions <matched-upsert,matched-delete,not-matched-insert,not-matched-by-source-upsert,not-matched-by-source-delete> \
    --matched-upsert-condition <matched-condition> \
    --matched-upsert-set <upsert-changes> \
    --matched-delete-condition <matched-condition> \
    --not-matched-insert-condition <not-matched-condition> \
    --not-matched-insert-values <insert-values> \
    --not-matched-by-source-upsert-condition <not-matched-by-source-condition> \
    --not-matched-by-source-upsert-set <not-matched-upsert-changes> \
    --not-matched-by-source-delete-condition <not-matched-by-source-condition>
    
# Alternatively, you can use '--source-sql <sql> [, --source-sql <sql> ...]' to create a new table as source table at runtime.
    
-- Examples:
-- Find all orders mentioned in the source table, then mark as important if the price is above 100 
-- or delete if the price is under 10.
./flink run \
    -c org.apache.flink.table.store.connector.action.FlinkActions \
    /path/to/flink-table-store-flink-**-0.4-SNAPSHOT.jar \
    merge-into \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table T \
    --source-table S \
    --on "T.id = S.order_id" \
    --merge-actions \
    matched-upsert,matched-delete \
    --matched-upsert-condition "T.price > 100" \
    --matched-upsert-set "mark = 'important'" \
    --matched-delete-condition "T.price < 10" 
    
-- For matched order rows, increase the price, and if there is no match, insert the order from the 
-- source table:
./flink run \
    -c org.apache.flink.table.store.connector.action.FlinkActions \
    /path/to/flink-table-store-flink-**-0.4-SNAPSHOT.jar \
    merge-into \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table T \
    --source-table S \
    --on "T.id = S.order_id" \
    --merge-actions \
    matched-upsert,not-matched-insert \
    --matched-upsert-set "price = T.price + 20" \
    --not-matched-insert-values * 

-- For not matched by source order rows (which are in the target table and does not match any row in the
-- source table based on the merge-condition), decrease the price or if the mark is 'trivial', delete them:
./flink run \
    -c org.apache.flink.table.store.connector.action.FlinkActions \
    /path/to/flink-table-store-flink-**-0.4-SNAPSHOT.jar \
    merge-into \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table T \
    --source-table S \
    --on "T.id = S.order_id" \
    --merge-actions \
    not-matched-by-source-upsert,not-matched-by-source-delete \
    --not-matched-by-source-upsert-condition "T.mark <> 'trivial'" \
    --not-matched-by-source-upsert-set "price = T.price - 20" \
    --not-matched-by-source-delete-condition "T.mark = 'trivial'"
    
-- An source-sql example: 
-- Create a temporary view S in new catalog and use it as source table
./flink run \
    -c org.apache.flink.table.store.connector.action.FlinkActions \
    /path/to/flink-table-store-flink-**-0.4-SNAPSHOT.jar \
    merge-into \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table T \
    --source-sql "CREATE CATALOG test WITH (...)" \
    --source-sql "USE CATALOG test" \
    --source-sql "USE DATABASE default" \
    --source-sql "CREATE TEMPORARY VIEW S AS SELECT order_id, price, 'important' FROM important_order" \
    --source-as test.default.S \
    --on "T.id = S.order_id" \
    --merge-actions not-matched-insert\
    --not-matched-insert-values *
```

## 4.7 查询数据

和其他所有的表一样，Flink Table Store可以使用SELECT语句对表进行查询。不同的是，在Flink Table Store中，用户可以指定不同的“scan.mode”配置以决定Table Store提供查询结果的方式。

### 4.7.1 扫描模式

| 扫描模式 | 批模式数据源的查询 | 流模式数据源的查询 |
| --- | --- | --- |
| default | 在默认模式下，“scan.mode”由其他的表属性决定，如果配置-"scan.timestamp-millis"被设定了的话，那么此时的扫描模式是“from-timestamp”，如果“scan.snapshot-id”指定了的话，那么实际的模式将会是“from-snapshot”。在其他情况下都是“latest-full”模式。 | 与批模式下的”default”扫描模式相同。 |
| latest-full | 返回要查询的表的最近的一次快照。 | 在任务启动的时候查询表的最近快照，随后持续地读取最新的变化数据。 |
| compacted-full | 返回最近的一次合并后的一次快照。 | 启动时返回最近的一次合并后的一次快照，随后持续地读取最新的变化数据。 |
| latest | 和“latest-full”相同 | 持续的读取最新的变化，但是在启动时不返回最近的快照 |
| from-timestamp | 返回早于或者等于指定时间戳的快照，时间戳由“scan.timestamp-millis”配置指定。 | 持续地从指定的时间戳读取变化的数据，在任务启动时不会返回表的快照。时间戳由配置“scan.timestamp-millis”指定。 |
| from-snapshot | 返回由配置"scan.snapshot-id"指定编号的快照。 | 持续地从指定的编号的快照读取变化的数据，在任务启动时不会返回表的指定编号的快照。快照的编号由"scan.snapshot-id"指定。 |

> 流式数据源也可以有界限，通过指定配置“scan.bounded.watermark”定义流模式下的边界条件，当比指定的边界条件水印更大的快照出现时数据的读取会停止。
> 

### 4.7.2 系统表

系统表包含每张表的元数据和信息，例如快照的信息和配置选项信息。用户可以使用批查询访问系统表。目前只有Flink，Spark和Trino支持查询系统表。在查询这些表的是偶，要把系统表用反引号括起来以防止语法解析冲突，如下的查询使用了三层访问模式。

**注意！：查询系统表不支持三层表示法，请显示地切换目录（Catalog）和数据库**

```sql
SELECT * FROM my_catalog.my_db.`MyTable$snapshots`;
```

Flink Table Store提供了以下的系统表

- Snapshots Table中包含快照历史信息，这些信息包含快照中的记录条数、数据的提交和过期信息等等。

```sql
SELECT * FROM MyTable$snapshots;

/*
+--------------+------------+-----------------+-------------------+--------------+-------------------------+---------------------+---------------------+-------------------------+
|  snapshot_id |  schema_id |     commit_user | commit_identifier |  commit_kind |             commit_time |  total_record_count |  delta_record_count |  changelog_record_count |
+--------------+------------+-----------------+-------------------+--------------+-------------------------+---------------------+---------------------+-------------------------+
|            2 |          0 | 7ca4cd28-98e... |                 2 |       APPEND | 2022-10-26 11:44:15.600 |                   2 |                   2 |                       0 |
|            1 |          0 | 870062aa-3e9... |                 1 |       APPEND | 2022-10-26 11:44:15.148 |                   1 |                   1 |                       0 |
+--------------+------------+-----------------+-------------------+--------------+-------------------------+---------------------+---------------------+-------------------------+
2 rows in set
*/
```

- Schemas Table中包含了表的历史模式（Schemas）

```sql
SELECT * FROM MyTable$schemas;

/*
+-----------+--------------------------------+----------------+--------------+---------+---------+
| schema_id |                         fields | partition_keys | primary_keys | options | comment |
+-----------+--------------------------------+----------------+--------------+---------+---------+
|         0 | [{"id":0,"name":"word","typ... |             [] |     ["word"] |      {} |         |
|         1 | [{"id":0,"name":"word","typ... |             [] |     ["word"] |      {} |         |
|         2 | [{"id":0,"name":"word","typ... |             [] |     ["word"] |      {} |         |
+-----------+--------------------------------+----------------+--------------+---------+---------+
3 rows in set
*/
```

可以将snapshot表和schema表关联起来以得到指定快照的字段信息，示例代码如下：

```sql
SELECT s.snapshot_id, t.schema_id, t.fields 
    FROM MyTable$snapshots s JOIN MyTable$schemas t 
    ON s.schema_id=t.schema_id where s.snapshot_id=100;
```

- 选项信息表

在Flink Table Store中可以查询表的DDL语句中设置的配置选项及其设置的值，没有查询到的选项为默认值。

```sql
SELECT * FROM MyTable$options;

/*
+------------------------+--------------------+
|         key            |        value       |
+------------------------+--------------------+
| snapshot.time-retained |         5 h        |
+------------------------+--------------------+
1 rows in set
*/
```

- 审计明细表

如果想查看表中的每条数据的变化和更新情况，那么可以使用audit_log系统表。在audit_log表中有一个叫做rowkind列，这一列有几种不同的值，每个值有不同的含义，具体含义如下：

- +I：插入数据操作
- -U：更新数据操作，更新之前的内容
- +U：更新数据操作，更新行之后的内容
- -D：删除数据操作

```sql
SELECT * FROM MyTable$audit_log;

/*
+------------------+-----------------+-----------------+
|     rowkind      |     column_0    |     column_1    |
+------------------+-----------------+-----------------+
|        +I        |      ...        |      ...        |
+------------------+-----------------+-----------------+
|        -U        |      ...        |      ...        |
+------------------+-----------------+-----------------+
|        +U        |      ...        |      ...        |
+------------------+-----------------+-----------------+
3 rows in set
*/
```

# 5. 特性

## 5.1 表的类型

Table Store支持许多类型的表，用户可以在建表的时候指定“write-mode”这个表属性。主要有以下类型的表。

### 5.1.1 有主键的变化日志表（Changelog Tables with Primary Keys）

Changelog Table是建表时的默认表类型。用户可以插入、更新或者删除表中的记录。

主键是一组列的集合，每一条记录的主键都不相同。Table Store对数据进行了排序，这意味着系统将会在每个桶（Bucket）内根据主键排序。由于这个特性，用户在往主键上添加过滤条件时会有很好地性能。通过定义主键，用户可以使用以下的特性。

- 合并引擎（Merge Engine）

当Table Store写入相同主键的两条或者多条记录时，Table Store会把这些记录合并为一条记录以保证主键唯一。用户通过指定”merge-engine“的表属性来选择记录是怎么样合并到一起的。

> ”merge-engine“=”deduplicate“为默认的配置，在这个设置下，Table Store只会保留相同主键的最新的一条记录并删除非最新的记录。特别地，如果最新的一条数据时删除记录，那么这些主键的值相同的所有记录都会被删除。
> 

> ”merge-engine“=”partial-update“，用户可以在多次更新中设置一条记录的列，最后得到一条完整的记录。具体来说，在同一主键下，值字段被逐一更新为最新数据，但空值不会被覆盖。
> 

> ”merge-engine“=”aggregate“，有时候用户只关注聚合的结果，聚合引擎把相同主键的最新的各条记录的各个字段的值根据聚合函数聚合形成一条新的记录。在这种引擎的工作机制中，所有的非主键字段必须指定聚合函数。用户需要通过“fields.<field-name>.aggregate-function”配置指定每个字段的聚合函数。以下是示例代码。
> 

> For streaming queries, `aggregation` merge engine must be used together with `full-compaction` [changelog producer](https://nightlies.apache.org/flink/flink-table-store-docs-release-0.3/docs/features/table-types/#changelog-producers).
> 

```sql
CREATE TABLE MyTable (
    product_id BIGINT,
    price DOUBLE,
    sales BIGINT,
    PRIMARY KEY (product_id) NOT ENFORCED
) WITH (
    'merge-engine' = 'aggregation',
    'fields.price.aggregate-function' = 'max',
    'fields.sales.aggregate-function' = 'sum'
);
```

- 
    
    

# 6. 维护

## 6.1 写入性能

### 6.1.1 并行度

### 6.1.2 合并（compaction）

## 6.2 内存

# 7. 问题

1. 性能是否足够？
2. 是否支持断点续传？
3. 与外部系统交互是否方便？