+++
title = '使用Docker部署带鉴权的kafka'
date = 2025-01-30T18:00:41+08:00
draft = true
+++

目标：基于docker和docker compose，部署一个单点的Kafka，开启SASL鉴权

## 使用版本

- 主机: Mac Studio (Apple M1 Max)

- 操作系统: MacOS 15.2 (24C101)

- OrbStack (提供容器环境): 1.9.4

- Docker:
    ```
    docker version
    Client:
    Version:           27.4.1
    API version:       1.47
    Go version:        go1.22.10
    Git commit:        b9d17ea
    Built:             Tue Dec 17 15:42:24 2024
    OS/Arch:           darwin/arm64
    Context:           orbstack

    Server: Docker Engine - Community
    Engine:
    Version:          27.4.1
    API version:      1.47 (minimum version 1.24)
    Go version:       go1.22.10
    Git commit:       c710b88
    Built:            Tue Dec 17 15:45:02 2024
    OS/Arch:          linux/arm64
    Experimental:     false
    containerd:
    Version:          v2.0.1
    GitCommit:        88aa2f531d6c2922003cc7929e51daf1c14caa0a
    runc:
    Version:          1.2.3
    GitCommit:        0d37cfd4b557771e555a184d5a78d0ed4bdb79a5
    docker-init:
    Version:          0.19.0
    GitCommit:        de40ad0
    ```

- docker compose:
    ```
    Docker Compose version 083f676
    ```

- kafka镜像: bitnami/kafka:3.9.0

## 创建`docker-compose.yaml`文件

```yaml
services:
  kafka:
    image: bitnami/kafka:3.9.0
    container_name: kafka
    ports:
      - '9092:9092'
    volumes:
      - ${DATA_DIR}/kafka/data:/bitnami/kafka
    environment:
      ########### Accessing Apache Kafka with internal and external clients
      - KAFKA_CFG_LISTENERS=SASL_PLAINTEXT://:9092,CONTROLLER://:9093
      - KAFKA_CFG_INTER_BROKER_LISTENER_NAME=SASL_PLAINTEXT
      - KAFKA_CFG_ADVERTISED_LISTENERS=SASL_PLAINTEXT://localhost:9092
      # Maps each listener with a Apache Kafka security protocol. If node is set with controller role, this setting is required in order to assign a security protocol for the CONTROLLER LISTENER.
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=SASL_PLAINTEXT:SASL_PLAINTEXT,CONTROLLER:SASL_PLAINTEXT

      ########### Apache Kafka KRaft mode configuration
      - KAFKA_CFG_PROCESS_ROLES=controller,broker
      - KAFKA_CFG_NODE_ID=0
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@localhost:9093
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER

      ########### Inter-Broker communications
      - KAFKA_CFG_SASL_MECHANISM_INTER_BROKER_PROTOCOL=SCRAM-SHA-512
      # Apache Kafka inter broker communication user
      - KAFKA_INTER_BROKER_USER=broker_user
      - KAFKA_INTER_BROKER_PASSWORD=88vsVcT2xTThN46bEj793xk89oCt776C

      ########### Control plane communications
      - KAFKA_CFG_SASL_MECHANISM_CONTROLLER_PROTOCOL=SCRAM-SHA-512
      # Apache Kafka controllers communication user
      - KAFKA_CONTROLLER_USER=controller_user
      - KAFKA_CONTROLLER_PASSWORD=8Gtynzs8EnEwoTvXwJbvfHkgJLrhm2FR

      ########### kafka auth
      # Kafka sasl.enabled.mechanisms configuration override
      - KAFKA_CFG_SASL_ENABLED_MECHANISMS=SCRAM-SHA-512
      # Name of the listener intended to be used by clients, if set, configures the producer/consumer accordingly
      - KAFKA_CLIENT_LISTENER_NAME=SASL_PLAINTEXT
      # List of additional users to KAFKA_CLIENT_USER that will be created into Zookeeper when using SASL_SCRAM for client communications
      - KAFKA_CLIENT_USERS=admin,writer,reader
      - KAFKA_CLIENT_PASSWORDS=FJAksTVkPvDhJKyaaUJXdTPX6domgNYR,n3sXFBBRFUNYF9btrbJPWwAfr4h3L6en,BPYJy29jDjGLGFxDQ9TW3BM4b9poWyUf

      ########### Uncategorized
      # Optional ID of the Kafka broker. If not set, a random ID will be automatically generated
      # - KAFKA_CFG_BROKER_ID=0
      ########### Additional config
      - KAFKA_CFG_LOG_RETENTION_BYTES=5368709120
      - KAFKA_CFG_ALLOW_EVERYONE_IF_NO_ACL_FOUND=false
      - KAFKA_CFG_AUTHORIZER_CLASS_NAME=org.apache.kafka.metadata.authorizer.StandardAuthorizer
      - KAFKA_CFG_SUPER_USERS=User:admin;User:controller_user;User:broker_user
```

`docker compose -f docker-compose.yaml up -d`即可启动