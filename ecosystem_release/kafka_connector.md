# Kafka connector 版本发布

## 发布说明

**使用手册参见：** 使用 kafka connector 导入数据

**压缩包命名规则：**`starrocks-kafka-connector-${connector_version}.tar.gz`

**压缩包下载地址：**[starrocks-kafka-connector-1.0.0.tar.gz](https://releases.starrocks.io/starrocks/starrocks-kafka-connector-1.0.0.tar.gz)

**版本要求：**

| Kafka Connector | Kafka | StarRocks | Java |
| --------------- | ----- | --------- | ---- |
| 1               |       |           | 8    |

## 发布记录

### 1.0

发布日期：2023 年 6 月 25 日

新增功能

- 支持导入 CSV、JSON、Avro 和 Protobuf 格式的数据
- 支持从 自建 Apach Kafka 集群 或 Confluent 平台导入数据。
