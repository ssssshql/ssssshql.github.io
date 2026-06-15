---
title: TDengine Kafka Connect 配置踩坑记录
date: 2026-06-15 16:30:00
tags: [TDengine, Kafka, Kafka Connect, 数据库]
categories: 数据库
description: 记录使用 TDengine Kafka Connect 插件将数据从 Kafka 发送到 TDengine 的过程，包括几个常见的坑和解决方案
---

最近在做一个物联网数据采集项目，需要将 Kafka 中的数据实时写入 TDengine。在使用官方的 TDengine Kafka Connect 插件时遇到了几个问题，特此记录。

## 简介

TDengine 提供了官方的 Kafka Connector 插件，可以方便地将数据从 Kafka 发送到 TDengine，支持多种数据格式（InfluxDB Line Protocol、OpenTSDB JSON、OpenTSDB Telnet）。

官方文档：[https://docs.taosdata.com/3.3.6/third-party/collection/kafka/](https://docs.taosdata.com/3.3.6/third-party/collection/kafka/)

## 环境准备

### Docker Compose 配置

我使用 Docker Compose 来部署整个环境，包括：

- Kafka 服务
- Kafka Connect 服务
- Kafbat UI（Kafka 管理界面）

```yaml
services:
  kafka:
    image: apache/kafka:4.3.0
    container_name: kafka
    restart: always
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://192.168.10.4:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_LOG_DIRS: /var/lib/kafka/data
    volumes:
      - ./data/kafka:/var/lib/kafka/data
    networks:
      - kafka-net

  kafka-connect:
    image: confluentinc/cp-kafka-connect:8.1.3
    container_name: kafka-connect
    restart: always
    ports:
      - "8083:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect
      CONNECT_GROUP_ID: "connect-cluster"
      CONNECT_CONFIG_STORAGE_TOPIC: "connect-configs"
      CONNECT_OFFSET_STORAGE_TOPIC: "connect-offsets"
      CONNECT_STATUS_STORAGE_TOPIC: "connect-status"
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_KEY_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"
      CONNECT_VALUE_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"
      CONNECT_INTERNAL_KEY_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"
      CONNECT_INTERNAL_VALUE_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"
      CONNECT_PLUGIN_PATH: "/usr/share/java,/opt/kafka/plugins"
    volumes:
      - ./data/kafka-connect/plugins:/opt/kafka/plugins
      - ./data/kafka-connect/sdk:/opt/sdk
    networks:
      - kafka-net
    depends_on:
      - kafka

  kafbat-ui:
    image: kafbat/kafka-ui:latest
    container_name: kafbat-ui
    restart: always
    ports:
      - "8081:8080"
    environment:
      DYNAMIC_CONFIG_ENABLED: 'true'
    networks:
      - kafka-net
    depends_on:
      - kafka

networks:
  kafka-net:
    driver: bridge
```

## 踩坑记录

### 坑 1：插件下载方式

**问题**：官方文档建议从源码构建插件，但这样比较麻烦。

**解决方案**：可以直接从 GitHub Releases 下载编译好的插件包。

下载地址：[https://github.com/taosdata/kafka-connect-tdengine/releases](https://github.com/taosdata/kafka-connect-tdengine/releases)

下载后解压到 `data/kafka-connect/plugins/` 目录即可。

### 坑 2：必须安装 TDengine Client

**问题**：TDengine Connector 插件依赖 TDengine Client 的本地库（.so 文件），如果没有安装 Client，会报错找不到相关依赖。

**解决方案**：需要在 Kafka Connect 容器中安装 TDengine Client。

```bash
# 下载 TDengine Client
wget https://www.taosdata.com/assets-download/3.0/TDengine-client-3.3.6.13-Linux-x64.tar.gz

# 解压到指定目录
tar -xzf TDengine-client-3.3.6.13-Linux-x64.tar.gz
```

将解压后的目录挂载到容器的 `/opt/sdk` 目录（参考上面的 docker-compose 配置）。

### 坑 3：必须使用 confluentinc/cp-kafka-connect 镜像

**问题**：最初尝试使用官方的 `apache/kafka` 镜像作为 Kafka Connect 基础镜像，但安装 TDengine Client 时报错，提示找不到 C 文件等依赖。这是因为官方镜像是精简版，缺少很多系统依赖。

**解决方案**：**必须使用 `confluentinc/cp-kafka-connect` 镜像**，这个镜像包含了所有必要的依赖。

这是我踩过的最大的坑！浪费了很多时间才发现这个问题。

## 配置 Sink Connector

创建 `sink-config.json` 文件：

```json
{
  "name": "TDengineSinkConnector",
  "config": {
    "connector.class": "com.taosdata.kafka.connect.sink.TDengineSinkConnector",
    "tasks.max": "1",
    "topics": "meters",
    "connection.url": "jdbc:TAOS://192.168.10.4:6030",
    "connection.user": "root",
    "connection.password": "taosdata",
    "connection.database": "power",
    "db.schemaless": "line",
    "data.precision": "ns",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.storage.StringConverter",
    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "dead_letter_topic",
    "errors.deadletterqueue.topic.replication.factor": 1
  }
}
```

**关键配置说明**：

| 参数 | 说明 |
|------|------|
| `connector.class` | 必须是 `com.taosdata.kafka.connect.sink.TDengineSinkConnector` |
| `topics` | Kafka 中的源 topic |
| `connection.url` | TDengine 的 JDBC 连接地址 |
| `connection.database` | 目标数据库，如果不存在会自动创建（纳秒精度） |
| `db.schemaless` | 数据格式，推荐使用 `line`（InfluxDB Line Protocol） |
| `data.precision` | 时间戳精度，必须与你的数据匹配 |

## 注册 Connector

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @sink-config.json
```

## 验证连接

```bash
# 查看 connector 状态
curl -s http://localhost:8083/connectors/TDengineSinkConnector/status

# 查看所有 connectors
curl -s http://localhost:8083/connectors
```

## 测试数据发送

向 Kafka 发送测试数据：

```bash
# 进入 Kafka 容器
docker exec -it kafka bash

# 使用 kafka-console-producer 发送数据
kafka-console-producer --broker-list localhost:9092 --topic meters

# 输入测试数据（InfluxDB Line Protocol 格式）
meters,location=California.LosAngeles,groupid=2 current=11.8,voltage=221,phase=0.28 1648432611249
```

## 总结

使用 TDengine Kafka Connect 时需要注意以下几点：

1. **插件下载**：直接从 GitHub Releases 下载即可，无需自己构建
2. **Client 安装**：必须安装 TDengine Client，并将其挂载到容器中
3. **镜像选择**：必须使用 `confluentinc/cp-kafka-connect` 镜像，否则会遇到依赖问题
4. **数据格式**：推荐使用 InfluxDB Line Protocol 格式，配置 `db.schemaless: line`
5. **时间精度**：`data.precision` 必须与实际数据的时间戳精度匹配

## 参考链接

- [TDengine 官方 Kafka Connect 文档](https://docs.taosdata.com/3.3.6/third-party/collection/kafka/)
- [kafka-connect-tdengine GitHub](https://github.com/taosdata/kafka-connect-tdengine)
- [TDengine Client 下载](https://www.taosdata.com/assets-download/3.0/TDengine-client-3.3.6.13-Linux-x64.tar.gz)
