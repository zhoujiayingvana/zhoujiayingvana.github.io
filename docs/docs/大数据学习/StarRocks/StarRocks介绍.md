# 背景介绍
Cassandra的qoe存储成本过高

业务的定时任务随着网络数增加处理时长过长，定时任务中的count等聚合操作存在瓶颈

qoe成本优化专项，使用大数据流处理框架+MPP数据库改造qoe链路，降低Cassandra存储成本，提高数据处理速度

# StarRocks概述
参考文档：[https://docs.mirrorship.cn/zh/docs/introduction/Features/](https://docs.mirrorship.cn/zh/docs/introduction/Features/)

## 定位介绍
<font style="color:rgb(28, 30, 33);">极速全场景 MPP (Massively Parallel Processing)</font>**<font style="color:rgb(28, 30, 33);"> 数据库</font>**<font style="color:rgb(28, 30, 33);">。</font>

> 在 MPP 执行框架中，一条查询请求会被拆分成多个物理计算单元，在多机并行执行。每个执行节点拥有独享的资源（CPU、内存）。MPP 执行框架能够使得单个查询请求可以充分利用所有执行节点的资源，所以单个查询的性能可以随着集群的水平扩展而不断提升。
>

![](../../../images/14534329e9ca33a57c181cc47753d9e5.png)

## 适用场景
+ **OLAP分析**：财务报表、自助数据查询平台、业务报表
+ **实时数仓**：实时指标计算、qoe计算
+ **准实时计算**：小时级批处理流程
+ **高并发查询**：面向用户侧的查询
+ 多源数据分析：统一查询数据湖和数据仓库

## <font style="color:rgb(28, 30, 33);">特点</font>
### 优势
1. 使用MPP框架：查询速度快，资源利用更充分
2. 列式存储：数据压缩率高，聚合查询快

> 数据压缩率是Cassandra的3倍
>

3. 分布式架构：高可用，扩容方便
4. 兼容MySQL协议：开发友好
5. 支持高并发查询：支持主键表，灵活的索引，物化视图特性，查询性能良好
6. 支持实时数据导入：秒级同步mysql数据，导入flink数据

### 劣势
1. 列式存储，不适合一次查询过多列

![](../../../images/19d90ebf01d68e138c5c9b6d5003d4c5.png)

2. 事务支持较弱，不适合频繁插入数据

> 生产环境禁用insert into方式，采用分批插入。两次插入数据间隔推荐>5s
>

## 架构
参考文档：[https://docs.mirrorship.cn/zh/docs/introduction/Architecture/](https://docs.mirrorship.cn/zh/docs/introduction/Architecture/)

![](../../../images/d3133bb9550f04a0b91e94ef3b943cc9.png)

StarRocks 由两种类型的节点组成：FE 和 BE。

+ FE 负责元数据管理和构建执行计划。
+ BE 执行查询计划并存储数据。BE 利用本地存储加速查询，并使用多副本机制确保高数据可用性。

# 数据分布
参考文档：[https://docs.mirrorship.cn/zh/docs/table_design/StarRocks_table_design/](https://docs.mirrorship.cn/zh/docs/table_design/StarRocks_table_design/)

![](../../../images/37bc71ad9a609e1dffcd239dde81445b.png)![](../../../images/79f2782c6cc075963b0655d5ca701aec.png)



StarRocks 采用分区+分桶的两级数据分布策略，将数据均匀分布各个 BE 节点。查询时能够有效裁剪数据扫描量，最大限度地利用集群的并发性能，从而提升查询性能。

## 分区
starrocks数据结构第一层级为分区。表中数据可以根据分区列（通常是时间和日期）分成一个个更小的数据管理单元。查询时，通过分区裁剪，可以减少扫描的数据量，显著优化查询性能。

> 分区的主要作用是将一张表按照分区键拆分成不同的管理单元，多用于数据管理，例如分区过期策略，设置不同分桶数等，也可以通过分区剪裁，起到加速查询的作用。
>

StarRocks 提供简单易用的分区方式，即表达式分区。此外还提供较灵活的分区方式，即 Range 分区和 List 分区。三种分区方式中，**只有表达式分区支持根据新数据自动创建新分区**，较为实用。

### 表达式分区
参考文档：[https://docs.mirrorship.cn/zh/docs/3.3/table_design/data_distribution/expression_partitioning/](https://docs.mirrorship.cn/zh/docs/3.3/table_design/data_distribution/expression_partitioning/)

优点：仅需要在建表时设置分区表达式。在数据导入时，StarRocks 会根据数据和分区表达式的定义规则**自动创建分区**

缺点：支持的分区键类型有限，稳定版本不支持通过函数转换分区键

#### 时间表达式分区
**适用场景**：按照日期来管理数据。sr会根据表达式规则自动将新数据匹配到对应分区。如果分区未创建，则会自动创建对应分区。

**语法**：

```sql
PARTITION BY 
{ date_trunc ( <time_unit> , <partition_column> ) |
  time_slice ( <partition_column> , INTERVAL <N> <time_unit> [ , boundary ] ) }
[ PROPERTIES( 'partition_live_number' = 'xxx' ) ]

```

**参数解释**：

+ 通过partition by来声明分区，可以支持date_trunc或time_slice语法

> date_trunc：根据指定的精度，将一个日期截断。例如 date_trunc("day", "2020-11-04 11:12:13")，返回"2020-11-04";
>
> time_slice：根据指定的时间粒度周期，将给定的时间转化为其所在的时间粒度周期的起始或结束时刻。例如：time_slice(1691-12-23 04:01:09, interval 5 second)，返回"1691-12-23 04:01:05"
>



**示例**：假设您经常按天查询数据，则建表时可以使用分区表达式 `date_trunc()` ，并且设置分区列为 `event_day` ，分区粒度为 `day`，实现导入数据时自动按照数据所属日期划分分区。将同一天的数据存储在一个分区中，利用分区裁剪可以显著提高查询效率。如果希望自动删除历史分区，可以使用 `partition_live_number` 设置只保留最近多少数量的分区。

```sql
CREATE TABLE site_access1 (
    event_day DATETIME NOT NULL,
    site_id INT DEFAULT '10',
    city_code VARCHAR(100),
    user_name VARCHAR(32) DEFAULT '',
    pv BIGINT DEFAULT '0'
)
DUPLICATE KEY(event_day, site_id, city_code, user_name)
PARTITION BY date_trunc('day', event_day) -- 分区声明
PROPERTIES(
    "partition_live_number" = "3" -- 只保留最近 3 个分区
);
DISTRIBUTED BY HASH(event_day, site_id);
```

<font style="color:rgb(28, 30, 33);">导入如下</font>两行数据，则 StarRocks 会根据导入数据的日期范围自动创建两个分区 `p20230226`、`p20230227`，范围分别为 [2023-02-26 00:00:00,2023-02-27 00:00:00)、[2023-02-27 00:00:00,2023-02-28 00:00:00)。如果后续导入数据的日期属于这两个范围，则都会自动划分至对应分区。

```sql
-- 导入两行数据
INSERT INTO site_access1 
    VALUES ("2023-02-26 20:12:04",002,"New York","Sam Smith",1),
           ("2023-02-27 21:06:54",001,"Los Angeles","Taylor Swift",1);

-- 查询分区
mysql > SHOW PARTITIONS FROM site_access1;
+-------------+---------------+----------------+---------------------+--------------------+--------+--------------+------------------------------------------------------------------------------------------------------+--------------------+---------+----------------+---------------+---------------------+--------------------------+----------+------------+----------+
| PartitionId | PartitionName | VisibleVersion | VisibleVersionTime  | VisibleVersionHash | State  | PartitionKey | Range                                                                                                | DistributionKey    | Buckets | ReplicationNum | StorageMedium | CooldownTime        | LastConsistencyCheckTime | DataSize | IsInMemory | RowCount |
+-------------+---------------+----------------+---------------------+--------------------+--------+--------------+------------------------------------------------------------------------------------------------------+--------------------+---------+----------------+---------------+---------------------+--------------------------+----------+------------+----------+
| 17138       | p20230226     | 2              | 2023-07-19 17:53:59 | 0                  | NORMAL | event_day    | [types: [DATETIME]; keys: [2023-02-26 00:00:00]; ..types: [DATETIME]; keys: [2023-02-27 00:00:00]; ) | event_day, site_id | 6       | 3              | HDD           | 9999-12-31 23:59:59 | NULL                     | 0B       | false      | 0        |
| 17113       | p20230227     | 2              | 2023-07-19 17:53:59 | 0                  | NORMAL | event_day    | [types: [DATETIME]; keys: [2023-02-27 00:00:00]; ..types: [DATETIME]; keys: [2023-02-28 00:00:00]; ) | event_day, site_id | 6       | 3              | HDD           | 9999-12-31 23:59:59 | NULL                     | 0B       | false      | 0        |
+-------------+---------------+----------------+---------------------+--------------------+--------+--------------+------------------------------------------------------------------------------------------------------+--------------------+---------+----------------+---------------+---------------------+--------------------------+----------+------------+----------+
2 rows in set (0.00 sec)
```



#### 列表达式分区
**适用场景**：按照枚举值来管理数据。建表时指定分区列，StarRocks 会根据导入的数据的分区列值，来自动划分并创建分区。每个（组合）枚举值都会生成一个分区

**语法**：

```sql
PARTITION BY (column1,[column2...])
```

**参数解释**：

+ 通过partition by对应的列名来声明分区，可以使用组合列名



**示例**：假设经常按日期范围和特定城市查询机房收费明细，则建表时可以使用分区表达式指定分区列为日期 `dt` 和城市 `city`。这样属于相同日期和城市的数据分组到同一个分区中，利用分区裁剪可以显著提高查询效率。

```sql
CREATE TABLE t_recharge_detail1 (
    id bigint,
    user_id bigint,
    recharge_money decimal(32,2), 
    city varchar(20) not null,
    dt varchar(20) not null
)
DUPLICATE KEY(id)
PARTITION BY (dt,city) -- 分区声明
DISTRIBUTED BY HASH(`id`);
```

导入一条数据，StarRocks 根据导入数据的分区列值自动创建一个分区 `p20220401_Houston` ，如果后续导入数据的分区列 `dt` 和 `city` 的值是 `2022-04-01`和 `Houston`，则都会被划分至该分区。

```sql
INSERT INTO t_recharge_detail1 
    VALUES (1, 1, 1, 'Houston', '2022-04-01');
```

## 分桶
starrocks数据结构第二层级为分桶。同一个分区中的数据通过分桶，划分成更小的数据管理单元。并且分桶以多副本形式（默认为3）**均匀分布**在 BE 节点上，保证数据的**高可用**。

<font style="color:rgb(28, 30, 33);">Starrocks主键表使用</font>**<font style="color:rgb(28, 30, 33);">哈希分桶</font>**<font style="color:rgb(28, 30, 33);">。对每个分区的数据，StarRocks 会根据分桶键</font>和分桶数量进行哈希分桶。在哈希分桶中，使用特定的列值作为输入，通过哈希函数计算出一个哈希值，然后将数据根据该哈希值分配到相应的桶中。以下分桶的介绍均指”哈希分桶“。

### <font style="color:rgb(28, 30, 33);">作用</font>
+ <font style="color:rgb(28, 30, 33);">提高查询性能。相同分桶键值的行会被分配到一个分桶中，在查询时能减少扫描数据量。</font>
+ <font style="color:rgb(28, 30, 33);">均匀分布数据。通过选取较高基数（唯一值的数量较多）的列作为分桶键，能更均匀的分布数据到每一个分桶中。</font>

### <font style="color:rgb(28, 30, 33);">语法</font>
```sql
DISTRIBUTED BY HASH(column1,[column2...]) [BUCKETS N];
```

参数解释：

+ 通过distributed by hash声明分桶，可以使用组合列作为分桶
+ 可以手动声明桶数量，也可以由starrocks自动判断桶数量。

### 示例
```sql
CREATE TABLE site_access (
    site_id INT DEFAULT '10',
    city_code SMALLINT,
    user_name VARCHAR(32) DEFAULT '',
    event_day DATE,
    pv BIGINT SUM DEFAULT '0')
AGGREGATE KEY(site_id, city_code, user_name,event_day)
PARTITION BY date_trunc('day', event_day)
DISTRIBUTED BY HASH(site_id,city_code); -- 分桶声明
```

### <font style="color:rgb(28, 30, 33);">如何选择分桶键</font>
<font style="color:rgb(28, 30, 33);">假设某列同时满足</font>**<font style="color:rgb(28, 30, 33);">高基数和经常作为查询条件</font>**<font style="color:rgb(28, 30, 33);">，则建议选择其为分桶键，进行哈希分桶。 如果不存在这些同时满足两个条件的列，则需要根据查询进行判断。</font>

+ <font style="color:rgb(28, 30, 33);">如果查询比较复杂，则建议选择高基数的列为分桶键，保证数据在各个分桶中尽量均衡，提高集群资源利用率。</font>
+ **<font style="color:rgb(28, 30, 33);">如果查询比较简单，则建议选择经常作为查询条件的列为分桶键，提高查询效率。</font>**

<font style="color:rgb(28, 30, 33);">并且，如果数据倾斜情况严重，您还可以使用多个列作为数据的分桶键，但是建议不超过 3 个列。</font>

### <font style="color:rgb(28, 30, 33);">如何确定桶数量</font>
<font style="color:rgb(28, 30, 33);">通用公式：分桶数量 = BE节点数量* CPU核数 / 2</font>

<font style="color:rgb(28, 30, 33);">估算公式：单个桶大小 = 表大小 / 副本数量 / 分区数量 / 分桶数量</font>

<font style="color:rgb(28, 30, 33);">每个桶大小建议<1GB</font>

<font style="color:rgb(28, 30, 33);">分桶数量过多会影响fe节点的调度性能</font>

# 主键表的设计与优化
参考文档：

[https://docs.mirrorship.cn/zh/docs/3.3/table_design/table_types/primary_key_table/](https://docs.mirrorship.cn/zh/docs/3.3/table_design/table_types/primary_key_table/)

[https://docs.mirrorship.cn/zh/docs/3.3/table_design/indexes/Prefix_index_sort_key/](https://docs.mirrorship.cn/zh/docs/3.3/table_design/indexes/Prefix_index_sort_key/)

## 主键表介绍
定义：拥有非空约束主键的表，相同主键的数据视作同一行。如果新数据的主键值与表中原数据的主键值相同，新数据会替代原数据。

特点：除了建表语法，查询语法其主要优势在于支撑实时数据更新的同时，也能保证高效的复杂即席查询性能。

### 建表语句
通过指定 primary key 声明是主键表

```sql
CREATE TABLE tauc.ap_wifi_quality
(
    network_id                       bigint,
    mac                              string,
    collect_time                     datetime,
    active                           int,
    alert_count                      int,
  ...
)
PRIMARY KEY (network_id, mac, collect_time) -- 声明主键
partition by date_trunc('day', collect_time) -- 声明分区键
distributed by hash(network_id) -- 声明哈希分桶键
order by (network_id,mac,collect_time) -- 声明排序键（前缀索引）
```

### 查询语句
参考文档：[https://docs.mirrorship.cn/zh/docs/sql-reference/sql-statements/table_bucket_part_index/SELECT/](https://docs.mirrorship.cn/zh/docs/sql-reference/sql-statements/table_bucket_part_index/SELECT/#union)

select 语法基本都兼容 MySQL语法，部分函数语法略有不同

示例：

```sql
  select network_id from tauc.ap_wifi_quality_full where network_id = 1 and mac = 'mac_1';
  select network_id from tauc.ap_wifi_quality_full where collect_time <'2025-06-26' and collect_time>'2025-06-25';
```

mysql和StarRocks语法对比

| 特性 | MySQL 支持 | StarRocks 支持 | StarRocks 优势说明 |
| --- | --- | --- | --- |
| SELECT 基础语法 | √ | √ | 大部分兼容 |
| JOIN | √ | √ | StarRocks 支持分布式 Join、更快 |
| 子查询 | √（有限） | √（更优化） | 更好地执行计划 |
| 聚合查询 | √ | √ | StarRocks 借助物化视图/列存更快 |
| 窗口函数 | √（8.0） | √ | StarRocks 支持 + 执行性能更强 |
| 物化视图 | X | √ | 自动重写、聚合查询大幅提速 |
| BITMAP/HLL 函数 | X | √ | 精准/估算去重，适合用户画像场景 |
| 复杂类型（ARRAY、MAP、STRUCT） | X | √ | 原生支持多维数据模型 |


StarRock不支持的语法

| MySQL语法 | StarRocks说明 |
| --- | --- |
| SELECT ... FOR UPDATE | 不支持，StarRocks 不支持事务锁机制 |
| SELECT SQL_CALC_FOUND_ROWS | 不支持 |
| `PROCEDURE`, `FUNCTION` 查询   |  不支持存储过程和函数调用   |
| `USER VARIABLES` (`@var`)   |  不支持   |
|  SELECT INTO OUTFILE   |  不支持   |


### <font style="color:rgb(28, 30, 33);">主键</font>
<font style="color:rgb(28, 30, 33);">主键用于唯一标识表中的每一行数据，组成主键的一个或多个列在</font><font style="color:rgb(28, 30, 33);"> </font>`<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">PRIMARY KEY</font>`<font style="color:rgb(28, 30, 33);"> </font><font style="color:rgb(28, 30, 33);">中定义，具有非空唯一性约束。其注意事项如下：</font>

+ <font style="color:rgb(28, 30, 33);">在建表语句中，</font>**<font style="color:rgb(28, 30, 33);">主键列必须定义在其他列之前</font>**<font style="color:rgb(28, 30, 33);">。</font>
+ **<font style="color:rgb(28, 30, 33);">主键必须包含分区列和分桶列</font>**<font style="color:rgb(28, 30, 33);">。</font>
+ <font style="color:rgb(28, 30, 33);">主键列支持以下数据类型：数值（包括整型和布尔）、日期和字符串。</font>
+ <font style="color:rgb(28, 30, 33);">默认设置下，单条主键值编码后的最大长度为 128 字节。</font>
+ **<font style="color:rgb(28, 30, 33);">建表后不支持修改主键</font>**<font style="color:rgb(28, 30, 33);">。</font>
+ <font style="color:rgb(28, 30, 33);">主键列的值不能更新，避免破坏数据一致性。</font>

### <font style="color:rgb(28, 30, 33);">主键索引</font>
<font style="color:rgb(28, 30, 33);">主键索引来保存主键值和其标识的数据行所在位置的映射。</font>

### 前缀索引
主键表可以声明排序键。排序键由 `ORDER BY` 定义的排序列组成。导入数据时数据按照排序键排序后存储，并且排序键还用于构建前缀索引，能够加速查询。

+ 如果指定了排序键，就根据排序键构建前缀索引；如果没指定排序键，就根据主键构建前缀索引。
+ 建表后支持通过 `ALTER TABLE ... ORDER BY ...` 修改排序键。不支持删除排序键，不支持修改排序列的数据类型。
+ 修改排序键是异步操作，会重新对数据进行排序存储，当表数据量较大时，修改排序键速度较慢，通常需要十几分钟。
+ 前缀索引项的最大长度为 36 字节，前缀索引遇到varchar、string类型会自动截断。因此字符串类型只能出现一次且需要放在前缀索引最后。

> 某表的建表语句为order by(uid, name, ctime)，前缀索引项为 uid (4 字节) + name (只取前 32 字节)，前缀字段为 `uid` 和 `name`，ctime无法作为索引<font style="color:rgb(28, 30, 33);">。</font>
>

# 主键表性能优化
## 建表优化
1. 选择合适的分区和分桶策略
    1. 单分区大小<100GB
    2. 单分桶大小1~10G，推荐<1GB
2. 前缀索引遇到varchar、string类型会自动截断。前缀索引遇到范围查询也会截断。
    1. 如果经常查询的字段中，有string类型，不能放在最后
        1. 选择非string类型创建索引，然后为string类型创建别的索引，例如bitmap索引，bloom filter索引
        2. 选择string类型创建索引，考虑分区分桶剪裁是否覆盖查询语句

## 查询优化
假设建表语句为：

```sql
CREATE TABLE tauc.ap_wifi_quality
(
    network_id                       bigint,
    mac                              string,
    collect_time                     datetime,
    active                           int,
    alert_count                      int,
  ...
)
PRIMARY KEY (network_id, mac, collect_time) -- 声明主键
partition by date_trunc('day', collect_time) -- 声明分区键
distributed by hash(network_id) -- 声明哈希分桶键
order by (network_id,mac,collect_time) -- 声明排序键（前缀索引）
```

### 分区剪裁
在查询的sql中添加分区字段的条件，会触发分区剪裁，加速查询

示例：按天分区，只查询某一天数据

```sql
explian select network_id from tauc.ap_wifi_quality_full where collect_time <'2025-06-26' and collect_time>'2025-06-25' limit 10;
```

![](../../../images/c497afb5d20e0be57acba60c2d01e79b.png)

explain执行计划中只查询1个分区的数据

### 分桶剪裁
在查询的sql中，添加**所有分桶键**，才会触发分桶剪裁。分桶剪裁也会加速查询

示例：每个分区下6个分桶，按network_id分区，查询某个network_id下的7天数据

```sql
select network_id from tauc.ap_wifi_quality_full where network_id = 1 limit 10;
```

![](../../../images/c16304782587ddf740bc7663aa9d9361.png)

explain执行计划中只查询了1/6的桶

### 查询列数量的影响
由于starrocks是列式存储，查询的列数量几乎与查询qps成正比，因此对于列数很多的表，不建议select *查询所有字段。

![](../../../images/19d90ebf01d68e138c5c9b6d5003d4c5.png)

### network health 业务主要查询场景讲解
假设建表语句为：

```sql
CREATE TABLE tauc.ap_wifi_quality
(
    network_id                       bigint,
    mac                              string,
    collect_time                     datetime,
    active                           int,
    alert_count                      int,
  ...
)
PRIMARY KEY (network_id, mac, collect_time) -- 声明主键
partition by date_trunc('day', collect_time) -- 声明分区键
distributed by hash(network_id) -- 声明哈希分桶键
order by (network_id,mac,collect_time) -- 声明排序键（前缀索引）
```

场景一：查过去7天数据：指定network_id，mac，collect_time范围（分桶剪裁，前缀索引mac）

场景二：按时间查最新一次数据：指定network_id，mac，collect_time倒序，limit1（分桶剪裁，前缀索引mac）

> 建表语句中collect_time无法作为索引，但starrocks对limit1做了优化，仅扫描每个桶的top1，查询速度不会很慢
>

场景三：只用network_id和collect_time查过去x天数据：指定network_id，collect_time范围（分桶剪裁，分区剪裁）

> 需要将原有cassandra查询中，先查一个network所有mac，再根据mac多次查询合并为只查一次network_id下所有数据
>

# 如何从Cassandra迁移到StarRocks
## 微服务中配置
StarRocks兼容MySQL协议，与MySQL配置基本一致

## 类型映射
| **字段含义** | **StarRocks 类型** | **MySQL 类型** | **Cassandra 类型** | **Java 类型（JDBC/ORM）** | **备注说明** |
| --- | --- | --- | --- | --- | --- |
| 主键 ID | `BIGINT` | `BIGINT` | `BIGINT` | `Long` | 三者兼容，Java 使用 Long |
| 整数 | `INT` | `INT` | `INT` | `Integer` | 完全兼容 |
| 小整数 | `TINYINT`<br/> / `SMALLINT` | `TINYINT`<br/> / `SMALLINT` | `TINYINT`<br/> / `SMALLINT` | `Byte`<br/> / `Short` | 类型长度相近 |
| 浮点数 | `FLOAT` | `FLOAT` | `FLOAT` | `Float` | |
| 双精度数 | `DOUBLE` | `DOUBLE` | `DOUBLE` | `Double` |  |
| 高精度金额/计量 | `DECIMAL(p, s)` | `DECIMAL(p, s)` | `DECIMAL` | `BigDecimal` |  |
| 字符串 | `VARCHAR(n)`/`STRING` | `VARCHAR(n)`<br/> / `TEXT` | `TEXT` | `String` | StarRocks的STRING实际是varchar(65535)，相同长度的字符串，varchar和string在存储大小，查询性能上没有区别 |
| 长文本 | `TEXT` | `TEXT` | `TEXT` | `String` | 不建议用于主键表，适合宽表分析场景 |
| 布尔值 | `BOOLEAN`<br/>（即 `TINYINT(1)`<br/>) | `BOOLEAN`<br/> / `TINYINT(1)` | `BOOLEAN` | `Boolean`<br/> / `boolean` | StarRocks 实际上底层为 `TINYINT`，0 代表 false，1 代表 true |
| 日期 | `DATE` | `DATE` | `DATE` | `java.sql.Date` | 均支持 `yyyy-MM-dd`<br/> 格式 |
| 时间戳 | `DATETIME` | `DATETIME`<br/> / `TIMESTAMP` | `TIMESTAMP` | `LocalDateTime`<br/> / `Timestamp` | StarRocks `DATETIME`<br/> 是毫秒精度，与 Cassandra `TIMESTAMP`<br/> 对应 |
| UUID / 唯一ID | `CHAR(36)` | `CHAR(36)`<br/> / `VARCHAR(36)` | `UUID` | `String`<br/> / `UUID` | StarRocks 没有 UUID 类型，使用 `CHAR(36)`<br/> 存储 |
| 数组（分析场景） | `ARRAY<type>` | ❌（无原生支持） | `LIST<type>` | `List<T>` |  |
| 位图（分析场景） | `BITMAP` | ❌ | ❌ | `String`<br/> / 自定义对象 | 用于高性能去重分析 |
| HLL（分析场景） | `HLL` | ❌ | ❌ | `String`<br/> / 自定义对象 | 用于近似去重 |




# FAQ
## 是否有高可用能力？节点宕机如何保证数据不丢失？
fe节点通过raft分布式协议同步元数据，可以允许（n-1）/2=1个节点故障

be节点通过3副本存储，可以允许（n-1）/2=1个节点故障

## 是否有负载均衡？
starrocks本身不提供负载均衡，但是每个fe节点都可以接受请求，可以添加负载均衡器将请求均匀打到多个fe节点，实现负载均衡。

此外，负载均衡仅能针对读操作，follower fe节点无法写数据，写操作和管理集群等必须路由到leader fe节点上处理，写操作对leader节点的负担更大。



