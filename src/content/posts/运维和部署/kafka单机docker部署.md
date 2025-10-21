---
title: Kafka单机Docker部署踩坑与解决方法
published: 2025-10-17
description: 我的Kafka单机Docker部署
tags: [Kafka, 部署]
category: 运维和部署
draft: false
---

# Kafka 单机 Docker 部署

## 简介

Kafka 是一个高吞吐量的分布式发布-订阅消息系统，常用于构建实时数据管道和流处理应用。本部署指南将介绍如何使用 Docker 在单机上部署 Kafka，采用 KRaft 模式（无 Zookeeper）。

## 踩坑

- 官方 Bitnami Kafka 镜像默认采用 Zookeeper 模式，不支持 KRaft 模式。因此，我们需要使用旧版镜像`bitnamilegacy/kafka:4.0.0`，该镜像支持 KRaft 模式。
- 使用类似 1panel 等容器管理工具特别是在他的应用商店直接搜然后下载那个 kafka 的应用然后部署的时候就会出现：使用正常的客户端在同一局域网下尝试连接时，是正常连接的，但是一旦尝试创建主题或者看别的信息的时候就会显示`KafkaJSConnectionError: Connection error: getaddrinfo ENOTFOUND 1Panel-kafka-NOai`这种类型的错误。

### 错误的 docker-compose 配置

```yaml
networks:
  1panel-network:
    external: true
services:
  kafka:
    container_name: ${CONTAINER_NAME}
    deploy:
      resources:
        limits:
          cpus: ${CPUS}
          memory: ${MEMORY_LIMIT}
    environment:
      ALLOW_PLAINTEXT_LISTENER: "yes"
      KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://${CONTAINER_NAME}:${PANEL_APP_PORT_HTTP}
      KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 0@127.0.0.1:9093
      KAFKA_CFG_FETCH_MESSAGE_MAX_BYTES: 524288000
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_CFG_LOG_RETENTION_MS: 60000
      KAFKA_CFG_MAX_REQUEST_SIZE: 524288000
      KAFKA_CFG_MESSAGE_MAX_BYTES: 524288000
      KAFKA_CFG_NODE_ID: 0
      KAFKA_CFG_PARTITION_FETCH_BYTES: 524288000
      KAFKA_CFG_PROCESS_ROLES: controller,broker
      KAFKA_CFG_REPLICA_FETCH_MAX_BYTES: 524288000
      KAFKA_HEAP_OPTS: -Xmx512m -Xms256m
    image: bitnamilegacy/kafka:4.0.0
    labels:
      createdBy: Apps
    networks:
      - 1panel-network
    ports: #怀疑是这里映射不对，暂时没有深究。
      - ${HOST_IP}:${PANEL_APP_PORT_HTTP}:9092
    restart: always
```

### 这个错误非常典型：

```plaintext
KafkaJSConnectionError: Connection error: getaddrinfo ENOTFOUND 1Panel-kafka-NOai
```

意思是：
Kafka 客户端无法解析主机名 1Panel-kafka-NOai，即 DNS 解析失败。

### 问题来源

docker-compose 配置里定义了：

```yaml
KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://${CONTAINER_NAME}:${PANEL_APP_PORT_HTTP}
```

这行会告诉 Kafka：
“我自己是 ${CONTAINER_NAME} 这个主机名。”
Kafka 客户端（KafkaJS）在连接时会去解析这个主机名。如果这个主机名不是容器网络里可以解析的名字，就会报：
`getaddrinfo ENOTFOUND 1Panel-kafka-NOai`

### 原因总结

| 可能原因                                                           | 说明                                          |
| ------------------------------------------------------------------ | --------------------------------------------- |
| ❌ `${CONTAINER_NAME}` 在 Kafka 客户端容器或主机上无法解析         | 例如 `1Panel-kafka-NOai` 不是一个实际的主机名 |
| ❌ Kafka 容器的 `advertised.listeners` 没有配置成可访问的主机名/IP | 客户端拿到的地址无法访问                      |
| ❌ 网络 `1panel-network` 没有连接 Kafka 客户端容器                 | 导致容器之间无法通过服务名互相访问            |

## 正确部署的 docker-compose 配置

```yaml
version: "3.8"

services:
  kafka:
    image: bitnamilegacy/kafka:4.0.0 # 官方Bitnami旧版镜像，支持KRaft
    container_name: kafka
    restart: always
    environment:
      # 启用KRaft模式（无Zookeeper）
      KAFKA_CFG_PROCESS_ROLES: broker,controller
      KAFKA_CFG_NODE_ID: 0
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 0@localhost:9093
      KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER

      # Kafka监听配置：内部、外部访问
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      # advertised.listeners 告诉客户端用哪个地址访问（改成你的机器IP）
      KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://192.168.1.166:9092
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      ALLOW_PLAINTEXT_LISTENER: "yes"

      # 资源与性能参数（可选）
      KAFKA_HEAP_OPTS: "-Xmx512m -Xms256m"
      KAFKA_CFG_LOG_RETENTION_HOURS: 24
      KAFKA_CFG_MESSAGE_MAX_BYTES: 524288000
      KAFKA_CFG_REPLICA_FETCH_MAX_BYTES: 524288000
      KAFKA_CFG_FETCH_MESSAGE_MAX_BYTES: 524288000
      KAFKA_CFG_MAX_REQUEST_SIZE: 524288000

    ports:
      - "9092:9092" # 外部访问Kafka
      - "9093:9093" # 控制器端口（一般内部使用）

    networks:
      - kafka-net

networks:
  kafka-net:
    driver: bridge
```
