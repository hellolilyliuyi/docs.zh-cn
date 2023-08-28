# Range 分区

**动态分区**

建表时[配置动态分区属性](./dynamic_partitioning.md)，StarRocks 会⾃动提前创建新的分区，删除过期分区，从而确保数据的时效性，实现对分区的⽣命周期管理（Time to Life，简称 “TTL”）。

区别于表达式分区中自动创建分区功能，动态创建分区只是根据您配置的动态分区属性，定期提前创建一些分区。如果导入的新数据不属于这些提前创建的分区，则导入任务会报错。而表达式分区中自动创建分区功能会根据导入数据创建对应的新分区。

**手动创建分区**

选择合理的分区键可以有效的裁剪扫描的数据量。**目前仅支持分区键的数据类型为日期和整数类型**。在实际业务场景中，一般从数据管理的角度选择分区键，常见的分区键为时间或者区域。

```SQL
CREATE TABLE site_access(
    event_day DATE,
    site_id INT DEFAULT '10',
    city_code VARCHAR(100),
    user_name VARCHAR(32) DEFAULT '',
    pv BIGINT SUM DEFAULT '0'
)
AGGREGATE KEY(event_day, site_id, city_code, user_name)
PARTITION BY RANGE(event_day)(
    PARTITION p1 VALUES LESS THAN ("2020-01-31"),
    PARTITION p2 VALUES LESS THAN ("2020-02-29"),
    PARTITION p3 VALUES LESS THAN ("2020-03-31")
)
DISTRIBUTED BY HASH(site_id);
```

**批量创建分区**

建表时和建表后，支持批量创建分区，通过 START、END 指定批量分区的开始和结束，EVERY 子句指定分区增量值。其中，批量分区包含 START 的值，但是不包含 END 的值。分区的命名规则同动态分区一样。

- **建表时批量创建日期分区**

    当分区键为日期类型时，建表时通过 START、END 指定批量分区的开始日期和结束日期，EVERY 子句指定分区增量值。并且 EVERY 子句中用 INTERVAL 关键字表示日期间隔，目前支持日期间隔的单位为 HOUR（自 3.0 版本起）、DAY、WEEK、MONTH、YEAR。

    如下示例中，批量分区的开始日期为 `2021-01-01` 和结束日期为 `2021-01-04`，增量值为一天：

    ```SQL
    CREATE TABLE site_access (
        datekey DATE,
        site_id INT,
        city_code SMALLINT,
        user_name VARCHAR(32),
        pv BIGINT DEFAULT '0'
    )
    ENGINE=olap
    DUPLICATE KEY(datekey, site_id, city_code, user_name)
    PARTITION BY RANGE (datekey) (
        START ("2021-01-01") END ("2021-01-04") EVERY (INTERVAL 1 DAY)
    )
    DISTRIBUTED BY HASH(site_id)
    PROPERTIES (
        "replication_num" = "3" 
    );
    ```

    则相当于在建表语句中使用如下 PARTITION BY 子句：

    ```SQL
    PARTITION BY RANGE (datekey) (
        PARTITION p20210101 VALUES [('2021-01-01'), ('2021-01-02')),
        PARTITION p20210102 VALUES [('2021-01-02'), ('2021-01-03')),
        PARTITION p20210103 VALUES [('2021-01-03'), ('2021-01-04'))
    )
    ```

- **建表时批量创建不同日期间隔的日期分区**

    建表时批量创建日期分区时，支持针对不同的日期分区区间（日期分区区间不能相重合），使用不同的 EVERY 子句指定日期间隔。一个日期分区区间，按照对应 EVERY 子句定义的日期间隔，批量创建分区，例如：

    ```SQL
    CREATE TABLE site_access (
        datekey DATE,
        site_id INT,
        city_code SMALLINT,
        user_name VARCHAR(32),
        pv BIGINT DEFAULT '0'
    )
    ENGINE=olap
    DUPLICATE KEY(datekey, site_id, city_code, user_name)
    PARTITION BY RANGE (datekey) (
        START ("2019-01-01") END ("2021-01-01") EVERY (INTERVAL 1 YEAR),
        START ("2021-01-01") END ("2021-05-01") EVERY (INTERVAL 1 MONTH),
        START ("2021-05-01") END ("2021-05-04") EVERY (INTERVAL 1 DAY)
    )
    DISTRIBUTED BY HASH(site_id)
    PROPERTIES (
        "replication_num" = "3"
    );
    ```

    则相当于在建表语句中使用如下 PARTITION BY 子句：

    ```SQL
    PARTITION BY RANGE (datekey) (
        PARTITION p2019 VALUES [('2019-01-01'), ('2020-01-01')),
        PARTITION p2020 VALUES [('2020-01-01'), ('2021-01-01')),
        PARTITION p202101 VALUES [('2021-01-01'), ('2021-02-01')),
        PARTITION p202102 VALUES [('2021-02-01'), ('2021-03-01')),
        PARTITION p202103 VALUES [('2021-03-01'), ('2021-04-01')),
        PARTITION p202104 VALUES [('2021-04-01'), ('2021-05-01')),
        PARTITION p20210501 VALUES [('2021-05-01'), ('2021-05-02')),
        PARTITION p20210502 VALUES [('2021-05-02'), ('2021-05-03')),
        PARTITION p20210503 VALUES [('2021-05-03'), ('2021-05-04'))
    )
    ```

- **建表时批量创建数字分区**

    当分区键为整数类型时，建表时通过 START、END 指定批量分区的开始值和结束值，EVERY 子句指定分区增量值。

    > **说明**
    >
    > START、END 所指定的分区列的值需要使用英文引号包裹，而 EVERY 子句中的分区增量值不用英文引号包裹。

    如下示例中，批量分区的开始值为 `1` 和结束值为 `5`，分区增量值为 `1`：

    ```SQL
    CREATE TABLE site_access (
        datekey INT,
        site_id INT,
        city_code SMALLINT,
        user_name VARCHAR(32),
        pv BIGINT DEFAULT '0'
    )
    ENGINE=olap
    DUPLICATE KEY(datekey, site_id, city_code, user_name)
    PARTITION BY RANGE (datekey) (
        START ("1") END ("5") EVERY (1)
    )
    DISTRIBUTED BY HASH(site_id)
    PROPERTIES (
        "replication_num" = "3"
    );
    ```

    则相当于在建表语句中使用如下 PARTITION BY 子句：

    ```SQL
    PARTITION BY RANGE (datekey) (
        PARTITION p1 VALUES [("1"), ("2")),
        PARTITION p2 VALUES [("2"), ("3")),
        PARTITION p3 VALUES [("3"), ("4")),
        PARTITION p4 VALUES [("4"), ("5"))
    )
    ```

- **建表后批量创建分区**

    建表后，支持通过ALTER TABLE 语句批量创建分区。相关语法与建表时批量创建分区类似，通过指定 ADD PARTITIONS 关键字，以及 START、END 以及 EVERY 子句来批量创建分区。示例如下：

    ```SQL
    ALTER TABLE site_access 
    ADD PARTITIONS START ("2021-01-04") END ("2021-01-06") EVERY (INTERVAL 1 DAY);
    ```