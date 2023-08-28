# 表达式分区（推荐）

自 v3.0 起，StarRocks 支持表达式分区（原称自动创建分区），更加灵活易用，适用于大多数场景，比如按照连续日期范围或者枚举值来查询和管理数据。

您仅需要在建表时设置分区表达式（时间函数表达式或列表达式）。在数据导入时，StarRocks 会根据数据和分区表达式的定义规则自动创建分区，您无需在建表时预先手动/批量创建大量分区，或者配置动态分区属性。 

## 时间函数表达式分区

如果您经常按照连续日期范围来查询和管理数据，则只需要在时间函数分区表达式中，指定一个日期类型（DATE 或者 DATETIME ）的分区列，以及指定分区粒度（年、月、日或小时）。StarRocks 会根据导入的数据和分区表达式，自动创建分区并且设置分区的起止时间。

不过在一些特殊场景下，比如历史数据按月划分分区、最近数据按天划分分区，则需要采用 [Range 分区](./Data_distribution.md#range-分区)创建分区。 

### 使用方式<!--建表后是否支持-->

**示例一**: 假设您经常**按天**查询数据，则建表时可以使用分区表达式 `date_trunc()` ，并且设置分区列为 `event_day` ，分区粒度为 `day`，实现导入数据时自动按照数据所属日期划分分区。将同一天的数据存储在一个分区中，利用分区裁剪可以显著提高查询效率。

```SQL
CREATE TABLE site_access1 (
    event_day DATETIME NOT NULL,
    site_id INT DEFAULT '10',
    city_code VARCHAR(100),
    user_name VARCHAR(32) DEFAULT '',
    pv BIGINT DEFAULT '0'
)
DUPLICATE KEY(event_day, site_id, city_code, user_name)
PARTITION BY date_trunc('day', event_day)
DISTRIBUTED BY HASH(event_day, site_id);
```

导入如下两行数据。

```SQL
INSERT INTO site_access1 
    VALUES ("2023-02-26 20:12:04",002,"New York","Sam Smith",1),
           ("2023-02-27 21:06:54",001,"Los Angeles","Taylor Swift",1);
```

查看具体分区。

```SQL
mysql > SHOW PARTITIONS FROM site_access1;
+-------------+---------------+----------------+---------------------+--------------------+--------+--------------+------------------------------------------------------------------------------------------------------+--------------------+---------+----------------+---------------+---------------------+--------------------------+----------+------------+----------+
| PartitionId | PartitionName | VisibleVersion | VisibleVersionTime  | VisibleVersionHash | State  | PartitionKey | Range                                                                                                | DistributionKey    | Buckets | ReplicationNum | StorageMedium | CooldownTime        | LastConsistencyCheckTime | DataSize | IsInMemory | RowCount |
+-------------+---------------+----------------+---------------------+--------------------+--------+--------------+------------------------------------------------------------------------------------------------------+--------------------+---------+----------------+---------------+---------------------+--------------------------+----------+------------+----------+
| 17138       | p20230226     | 2              | 2023-07-19 17:53:59 | 0                  | NORMAL | event_day    | [types: [DATETIME]; keys: [2023-02-26 00:00:00]; ..types: [DATETIME]; keys: [2023-02-27 00:00:00]; ) | event_day, site_id | 6       | 3              | HDD           | 9999-12-31 23:59:59 | NULL                     | 0B       | false      | 0        |
| 17113       | p20230227     | 2              | 2023-07-19 17:53:59 | 0                  | NORMAL | event_day    | [types: [DATETIME]; keys: [2023-02-27 00:00:00]; ..types: [DATETIME]; keys: [2023-02-28 00:00:00]; ) | event_day, site_id | 6       | 3              | HDD           | 9999-12-31 23:59:59 | NULL                     | 0B       | false      | 0        |
+-------------+---------------+----------------+---------------------+--------------------+--------+--------------+------------------------------------------------------------------------------------------------------+--------------------+---------+----------------+---------------+---------------------+--------------------------+----------+------------+----------+
2 rows in set (0.00 sec)
```

返回结果显示，StarRocks 根据导入数据的日期范围自动创建两个分区 `p20230226`、`p20230227`，范围分别为 [2023-02-26 00:00:00,2023-02-27 00:00:00)、[2023-02-27 00:00:00,2023-02-28 00:00:00)。如果后续导入数据的日期属于这两个范围，则都会自动划分至对应分区。

<!--这个示例是不是可以放到sql reference-->
<!--
示例二：如果您希望引入分区生命周期管理，即仅保留最近一段时间的分区，删除历史分区，则可以使用 `partition_live_number` 设置只保留最近多少数量的分区。

```SQL
CREATE TABLE site_access2 (
    event_day DATETIME NOT NULL,
    site_id INT DEFAULT '10',
    city_code VARCHAR(100),
    user_name VARCHAR(32) DEFAULT '',
    pv BIGINT DEFAULT '0'
) 
DUPLICATE KEY(event_day, site_id, city_code, user_name)
PARTITION BY date_trunc('month', event_day)
DISTRIBUTED BY HASH(event_day, site_id)
PROPERTIES(
    "partition_live_number" = "3" -- 只保留最近 3 个分区
);
```
-->

**示例二**：假设您经常**按周**查询数据，则建表时可以使用分区表达式 `time_slice()`，设置分区列为 `event_day`，分区粒度为七天。将一周的数据存储在一个分区中，利用分区裁剪可以显著提高查询效率。
> **说明**
>
> `time_slice()` 相比于 `date_trunc()`，支持更加灵活的分区粒度。

```SQL
CREATE TABLE site_access3 (
    event_day DATETIME NOT NULL,
    site_id INT DEFAULT '10',
    city_code VARCHAR(100),
    user_name VARCHAR(32) DEFAULT '',
    pv BIGINT DEFAULT '0'
)
DUPLICATE KEY(event_day, site_id, city_code, user_name)
PARTITION BY time_slice(event_day, INTERVAL 7 day)
DISTRIBUTED BY HASH(event_day, site_id);
```

### 使用说明

- 在导入的过程中 StarRocks 根据导入数据已经自动创建了一些分区，但是由于某些原因导入作业最终失败，则在当前版本中，已经自动创建的分区并不会由于导入失败而自动删除。
- StarRocks 自动创建分区数量上限默认为 4096，由 FE 配置参数 `max_automatic_partition_number` 决定。该参数可以防止您由于误操作而创建大量分区。
- 分区命名规则与动态分区的命名规则一致。

## 列表达式分区（自 v3.1）

如果您经常按照枚举值来查询和管理数据，则您只需要指定表示类型的列为分区列，StarRocks 会根据导入的数据的分区列值，来自动划分并创建分区。

不过在一些特殊场景下，比如表中包含表示城市的列，您经常按照国家和城市来查询和管理数据，希望将同属于一个国家的多个城市的数据存储在一个分区中，则需要使用 [List 分区](./list_partitioning.md)。

### 使用方式

示例一：假设经常按特定日期和特定城市查询机房收费明细，则建表时可以使用分区表达式指定分区列为日期 `dt` 和城市 `city`。这样属于相同日期和城市的数据分组到同一个分区中，利用分区裁剪可以显著提高查询效率。

```SQL
CREATE TABLE t_recharge_detail1 (
    id bigint,
    user_id bigint,
    recharge_money decimal(32,2), 
    city varchar(20) not null,
    dt varchar(20) not null
)
DUPLICATE KEY(id)
PARTITION BY (dt,city)
DISTRIBUTED BY HASH(`id`);
```

导入一条数据。

```SQL
INSERT INTO t_recharge_detail1 
    VALUES (1, 1, 1, 'Houston', '2022-04-01');
```

查看具体分区。

> **说明**
>
> 分区中只能包含各分区列的一个值，如果需要一个分区中包含各分区列的多值，请使用 [List 分区](./list_partitioning.md)。

```SQL
MySQL > SHOW PARTITIONS from t_recharge_detail1\G
*************************** 1. row ***************************
             PartitionId: 16890
           PartitionName: p20220401_Houston
          VisibleVersion: 2
      VisibleVersionTime: 2023-07-19 17:24:53
      VisibleVersionHash: 0
                   State: NORMAL
            PartitionKey: dt, city
                    List: (('2022-04-01', 'Houston'))
         DistributionKey: id
                 Buckets: 6
          ReplicationNum: 3
           StorageMedium: HDD
            CooldownTime: 9999-12-31 23:59:59
LastConsistencyCheckTime: NULL
                DataSize: 2.5KB
              IsInMemory: false
                RowCount: 1
1 row in set (0.00 sec)
```

返回结果显示，StarRocks 根据导入数据的分区列值自动创建一个分区 `p20220401_Houston` ，如果后续导入数据的分区列 `dt` 和 `city` 的值是 `2022-04-01`和 `Houston`，则都会被划分至该分区。

<!--move-->
<!--
示例二：您也可以在建表时配置参数 `partition_live_number` 进行分区生命周期管理，例如指定该表只保留最近 3 个分区。

```SQL
CREATE TABLE t_recharge_detail2 (
    id bigint,
    user_id bigint,
    recharge_money decimal(32,2), 
    city varchar(20) not null,
    dt varchar(20) not null
)
DUPLICATE KEY(id)
PARTITION BY (dt,city)
DISTRIBUTED BY HASH(`id`) 
PROPERTIES(
    "partition_live_number" = "3" -- 只保留最近 3 个分区。
);
```
-->

### 使用说明

- 在导入的过程中 StarRocks 根据导入数据已经自动创建了一些分区，但是由于某些原因导入作业最终失败，则在当前版本中，已经自动创建的分区并不会由于导入失败而自动删除。
- StarRocks 自动创建分区数量上限默认为 4096，由 FE 配置参数 `max_automatic_partition_number` 决定。该参数可以防止您由于误操作而创建大量分区。
- 分区命名规则：如果存在多个分区列，则不同分区列的值以下划线（_）连接。例如：存在有两个分区列 `dt` 和 `city`，均为字符串类型，导入一条数据 `2022-04-01`, `beijing`，则自动创建的分区名称为 `p20220401_beijing`。

## 管理分区

### 导入数据至分区

导入数据时，StarRocks 会根据数据和分区表达式定义的分区规则，自动创建分区。

值得注意的是，如果您建表时使用表达式分区，并且需要使用 [INSERT OVERWRITE](../loading/InsertInto.md#通过-insert-overwrite-select-语句覆盖写入数据) 覆盖写入**指定分区**的数据，不论该分区是否已经创建，目前您都需要在 `PARTITION()` 提供明确的分区范围。而不是同 Range 分区或 List 分区一样，只需要在`PARTITION (partition_name)` 中提供分区名称。

如果建表时您使用时间函数表达式分区，则此时覆盖写入指定分区，您需要提供该分区的起始范围（分区粒度与建表时配置分区粒度一致）。如果该分区不存在，则导入数据时会自动创建该分区。

```SQL
INSERT OVERWRITE site_access1 PARTITION(event_day='2022-06-08 20:12:04')
    SELECT * FROM site_access2 PARTITION(p20220608);
```

如果建表时您使用列表达式分区，则覆盖写入指定分区时，您需要提供该分区包含的分区列值。如果该分区不存在，则导入数据时会自动创建该分区。

```SQL
INSERT OVERWRITE t_recharge_detail1 PARTITION(dt='2022-04-02',city='texas')
    SELECT * FROM t_recharge_detail2 PARTITION(p20220402_texas);
```

### 查看分区

查看自动创建的分区的具体信息时，您需要使用 `SHOW PARTITIONS FROM <table_name>` 语句。而`SHOW CREATE TABLE <table_name>` 语句返回的结果中仅包含建表时配置的表达式分区语法。

```SQL
MySQL > SHOW PARTITIONS FROM t_recharge_detail1;
+-------------+-------------------+----------------+---------------------+--------------------+--------+--------------+-----------------------------+-----------------+---------+----------------+---------------+---------------------+--------------------------+----------+------------+----------+
| PartitionId | PartitionName     | VisibleVersion | VisibleVersionTime  | VisibleVersionHash | State  | PartitionKey | List                        | DistributionKey | Buckets | ReplicationNum | StorageMedium | CooldownTime        | LastConsistencyCheckTime | DataSize | IsInMemory | RowCount |
+-------------+-------------------+----------------+---------------------+--------------------+--------+--------------+-----------------------------+-----------------+---------+----------------+---------------+---------------------+--------------------------+----------+------------+----------+
| 16890       | p20220401_Houston | 2              | 2023-07-19 17:24:53 | 0                  | NORMAL | dt, city     | (('2022-04-01', 'Houston')) | id              | 6       | 3              | HDD           | 9999-12-31 23:59:59 | NULL                     | 2.5KB    | false      | 1        |
| 17056       | p20220402_texas   | 2              | 2023-07-19 17:27:42 | 0                  | NORMAL | dt, city     | (('2022-04-02', 'texas'))   | id              | 6       | 3              | HDD           | 9999-12-31 23:59:59 | NULL                     | 2.5KB    | false      | 1        |
+-------------+-------------------+----------------+---------------------+--------------------+--------+--------------+-----------------------------+-----------------+---------+----------------+---------------+---------------------+--------------------------+----------+------------+----------+
2 rows in set (0.00 sec)
```

## 使用限制

- 自 v3.1 起，StarRocks 存算分离模式支持时间函数表达式分区，暂时不支持列表达式分区。
- 使用 CTAS 建表时暂时不支持表达式分区。
- 暂时不支持使用 Spark Load 导入数据至表达式分区的表。
- 使用 `ALTER TABLE <table_name> DROP PARTITION <partition_name>` 删除列表达式分区时，分区直接被删除并且不能被恢复。
- 列表达式分区暂时不支持[备份与恢复](../administration/Backup_and_restore.md)。
- 如果使用表达式分区，则仅支持回滚到 2.5.4 及以后的版本。
