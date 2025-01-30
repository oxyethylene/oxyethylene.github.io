+++
title = '使用Docker部署SkyWakling'
date = 2025-01-25T15:18:58+08:00
draft = false
+++

目标：基于docker和docker compose，部署一个单点的ES + OAP server + UI，接入java agent

- 开启鉴权(Token)
- 使用TLS(对于gRPC)

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

- ElasticSearch 镜像: elasticsearch:7.17.23 (sha256:37ac55d4ec9db03a1be37a1b0c394d513f5c85a0c80576891dcffdd9803bec1c)

- Skywalking-oap-server镜像: apache/skywalking-oap-server:10.1.0-java17 (sha256:1323eb935f8295ef9a99c0159581a63ea1096fcc0aeb5edd3f939fb4124ef66c)

- Skywalking-ui 镜像: apache/skywalking-ui:10.1.0-java17 (sha256:1fbe25c8c110fd7708a720dc8313d94cc1250909e99f2ea8f014821584d6a1f9)

- openssl: OpenSSL 3.4.0 22 Oct 2024 (Library: OpenSSL 3.4.0 22 Oct 2024)



## 生成证书

复制如下脚本并执行
```bash
#!/usr/bin/env bash
set -e

ES_SSL_PASSWORD=DT8vBxEs9kMd4BqGRtRoEF6HT9FQwNyb

mkdir -p certs/ca
cd certs

echo Generate CA key:
openssl genrsa -out ca.key 4096

echo Generate CA cert:
openssl req -x509 -days 1095 -key ca.key -out ca.crt -subj "/CN=MyCustomCA"

generate_cert() {
    local name=$1
    echo Generate "$name" key:
    openssl genrsa -out "$name".key 4096

    echo Generate "$name" CSR:
    openssl req -new -key "$name".key -out "$name".csr -subj "/CN=$name" -addext "subjectAltName = DNS:$name, DNS:localhost, IP:127.0.0.1"

    echo Sign "$name" certificate:
    openssl x509 -req -days 365 -in "$name".csr -copy_extensions copy -CA ca.crt -CAkey ca.key -CAcreateserial -out "$name".crt

    echo Generate "$name" PEM:
    openssl pkcs8 -topk8 -nocrypt -in "$name".key -out "$name".pem

    mkdir -p "$name"
    mv "$name".* "$name"/
}

# Generate certificates for elasticsearch, skywalking-oap
generate_cert elasticsearch
generate_cert skywalking-oap

mv ca.* ca/

echo Generate truststore for elasticsearch:
docker run --rm -it \
    -v ./:/mnt/certs \
    elasticsearch:7.17.27 \
    /usr/share/elasticsearch/jdk/bin/keytool \
    -import \
    -alias my-root-ca \
    -file /mnt/certs/ca/ca.crt \
    -keystore /mnt/certs/skywalking-oap/es-ssl.jks \
    -storepass "$ES_SSL_PASSWORD" \
    -noprompt

```
会在当前目录下创建certs文件夹，结构如下
```text
certs
├── ca
│   ├── ca.crt
│   ├── ca.key
│   └── ca.srl
├── elasticsearch
│   ├── elasticsearch.crt
│   ├── elasticsearch.csr
│   ├── elasticsearch.key
│   └── elasticsearch.pem
└── skywalking-oap
    ├── es-ssl.jks
    ├── skywalking-oap.crt
    ├── skywalking-oap.csr
    ├── skywalking-oap.key
    └── skywalking-oap.pem
```

## 部署单点ES并开启https和http basic auth

1. 创建`.env`文件
    ```ini
    COMPOSE_PROJECT_NAME=skywalking
    CERTS_DIR=/usr/share/elasticsearch/config/certificates
    ELASTIC_PASSWORD=ChangeMe
    ```
    `COMPOSE_PROJECT_NAME`是`docker compose`预定义的一个环境变量，用于定义项目的名字，详见[COMPOSE_PROJECT_NAME](https://docs.docker.com/compose/how-tos/environment-variables/envvars/#compose_project_name)。之后所有`docker compose`创建的volume和network等都会使用`skywaling_`作为前缀

    `ELASTIC_PASSWORD`是第一次启动ES时，`elastic`用户的初始密码。

1. 创建`docker-compose.yaml`
    ```yaml
    services:
    elasticsearch:
        image: elasticsearch:7.17.27
        container_name: elasticsearch
        environment:
        - node.name=es
        - discovery.type=single-node
        - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
        - ELASTIC_PASSWORD=$ELASTIC_PASSWORD
        - xpack.license.self_generated.type=basic
        - xpack.security.enabled=true
        - xpack.security.http.ssl.enabled=true
        - xpack.security.http.ssl.key=$CERTS_DIR/elasticsearch/elasticsearch.key
        - xpack.security.http.ssl.certificate_authorities=$CERTS_DIR/ca/ca.crt
        - xpack.security.http.ssl.certificate=$CERTS_DIR/elasticsearch/elasticsearch.crt
        - xpack.security.transport.ssl.enabled=true
        - xpack.security.transport.ssl.verification_mode=certificate
        - xpack.security.transport.ssl.certificate_authorities=$CERTS_DIR/ca/ca.crt
        - xpack.security.transport.ssl.certificate=$CERTS_DIR/elasticsearch/elasticsearch.crt
        - xpack.security.transport.ssl.key=$CERTS_DIR/elasticsearch/elasticsearch.key
        ulimits:
        memlock:
            soft: 65536
            hard: 65536
        volumes:
        - ./elasticsearch/data:/usr/share/elasticsearch/data
        - ./certs:$CERTS_DIR
        healthcheck:
        test: curl --cacert $CERTS_DIR/ca/ca.crt -s https://localhost:9200 >/dev/null; if [[ $$? == 52 ]]; then echo 0; else echo 1; fi
        interval: 30s
        timeout: 10s
        retries: 5
    ```

1. 使用`docker compose -f docker-compose.yaml up -d`启动ES。

    可以使用`docker ps | grep elasticsearch`来查看ES的运行情况
    ```
    docker ps -a | grep elasticsearch
    eb91b9ca102d   elasticsearch:7.17.27            "/bin/tini -- /usr/l…"   10 minutes ago   Up 10 minutes (healthy)    9200/tcp, 9300 tcp                          elasticsearch
    ```

    用容器内的curl来验证TLS和basic auth已经生效。
    ```
    docker exec elasticsearch curl -sS --cacert /usr/share/elasticsearch/config/certificates/ca/ca.crt -u elastic:ChangeMe https://127.0.0.1:9200/
    {
    "name" : "es",
    "cluster_name" : "docker-cluster",
    "cluster_uuid" : "GS8EiBn0QIKf4EbS02kWYw",
    "version" : {
        "number" : "7.17.27",
        "build_flavor" : "default",
        "build_type" : "docker",
        "build_hash" : "0f88dde84795b30ca0d2c0c4796643ec5938aeb5",
        "build_date" : "2025-01-09T14:09:01.578835424Z",
        "build_snapshot" : false,
        "lucene_version" : "8.11.3",
        "minimum_wire_compatibility_version" : "6.8.0",
        "minimum_index_compatibility_version" : "6.0.0-beta1"
    },
    "tagline" : "You Know, for Search"
    }
    ```

### 修改ES用户密码

此时ES的`elastic`用户的密码是`.env`文件里通过`ELASTIC_PASSWORD`配置的初始密码，ES提供了`elasticsearch-setup-passwords`命令来给内置用户设置密码。
可以给所有用户设置随机的密码
```text
docker exec elasticsearch /bin/bash -c "bin/elasticsearch-setup-passwords auto --batch --url https://localhost:9200"
Changed password for user apm_system
PASSWORD apm_system = esnCEjhQJ6ostLOaaiEw

Changed password for user kibana_system
PASSWORD kibana_system = WBLDQsBEKSCHEUvdaJ6s

Changed password for user logstash_system
PASSWORD logstash_system = A4La55YFLrybDVWdT8d4

Changed password for user beats_system
PASSWORD beats_system = gCYVxIUJRsnWVZBiwxp5

Changed password for user remote_monitoring_user
PASSWORD remote_monitoring_user = PvOFYEzdykgzCwXHBrla

Changed password for user elastic
PASSWORD elastic = asU8F9EJYcHD0sTf1UPK
```
也可以手动设置每个用户的密码，注意这个要加`-it`，因为要交互
```text
docker exec -it elasticsearch /bin/bash -c "bin/elasticsearch-setup-passwords interactive --url https://localhost:9200"
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,kibana_system,logstash_system,beats_system,remote_monitoring_user.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N]y


Enter password for [elastic]:
Reenter password for [elastic]:
Enter password for [apm_system]:
Reenter password for [apm_system]:
Enter password for [kibana_system]:
Reenter password for [kibana_system]:
Enter password for [logstash_system]:
Reenter password for [logstash_system]:
Enter password for [beats_system]:
Reenter password for [beats_system]:
Enter password for [remote_monitoring_user]:
Reenter password for [remote_monitoring_user]:
Changed password for user [apm_system]
Changed password for user [kibana_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]
```

## 部署SkyWalking OAP Server

修改`docker-compose.yaml`，添加skywalking-oap的配置
```yaml {hl_lines="33-62"}
services:
  elasticsearch:
    image: elasticsearch:7.17.27
    container_name: elasticsearch
    environment:
      - node.name=es
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - ELASTIC_PASSWORD=$ELASTIC_PASSWORD
      - xpack.license.self_generated.type=basic
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=$CERTS_DIR/elasticsearch/elasticsearch.key
      - xpack.security.http.ssl.certificate_authorities=$CERTS_DIR/ca/ca.crt
      - xpack.security.http.ssl.certificate=$CERTS_DIR/elasticsearch/elasticsearch.crt
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.certificate_authorities=$CERTS_DIR/ca/ca.crt
      - xpack.security.transport.ssl.certificate=$CERTS_DIR/elasticsearch/elasticsearch.crt
      - xpack.security.transport.ssl.key=$CERTS_DIR/elasticsearch/elasticsearch.key
    ulimits:
      memlock:
        soft: 65536
        hard: 65536
    volumes:
      - ${DATA_DIR}/skywalking/elasticsearch/data:/usr/share/elasticsearch/data
      - ./certs:$CERTS_DIR
    healthcheck:
      test: curl --cacert $CERTS_DIR/ca/ca.crt -s https://localhost:9200 >/dev/null; if [[ $$? == 52 ]]; then echo 0; else echo 1; fi
      interval: 30s
      timeout: 10s
      retries: 5
  skywalking-oap:
    image: apache/skywalking-oap-server:10.1.0-java17
    container_name: skywalking-oap
    volumes:
      - ./certs/skywalking-oap:${SW_CERTS_DIR}/skywalking-oap
      - ./certs/ca:${SW_CERTS_DIR}/ca
    environment:
      - SW_STORAGE=elasticsearch
      - SW_STORAGE_ES_CLUSTER_NODES=elasticsearch:9200
      - SW_STORAGE_ES_HTTP_PROTOCOL=https
      - SW_ES_USER=elastic
      - SW_ES_PASSWORD=asU8F9EJYcHD0sTf1UPK
      - SW_STORAGE_ES_SSL_JKS_PATH=${SW_CERTS_DIR}/skywalking-oap/es-ssl.jks
      - SW_STORAGE_ES_SSL_JKS_PASS=DT8vBxEs9kMd4BqGRtRoEF6HT9FQwNyb
      - SW_CORE_GRPC_SSL_ENABLED=true
      - SW_CORE_GRPC_SSL_KEY_PATH=${SW_CERTS_DIR}/skywalking-oap/skywalking-oap.pem
      - SW_CORE_GRPC_SSL_CERT_CHAIN_PATH=${SW_CERTS_DIR}/skywalking-oap/skywalking-oap.crt
      - SW_CORE_GRPC_SSL_TRUSTED_CA_PATH=${SW_CERTS_DIR}/ca/ca.crt
      - SW_HEALTH_CHECKER=default
      - SW_HEALTH_CHECKER_INTERVAL_SECONDS=30
      - SW_TELEMETRY=prometheus
    depends_on:
      elasticsearch:
        condition: service_healthy
    healthcheck:
      test: [ "CMD-SHELL", "curl http://localhost:12800/healthcheck" ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
```

## 添加SkyWalking UI的配置

```yaml {hl_lines="63-73"}
services:
  elasticsearch:
    image: elasticsearch:7.17.27
    container_name: elasticsearch
    environment:
      - node.name=es
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - ELASTIC_PASSWORD=$ELASTIC_PASSWORD
      - xpack.license.self_generated.type=basic
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=$CERTS_DIR/elasticsearch/elasticsearch.key
      - xpack.security.http.ssl.certificate_authorities=$CERTS_DIR/ca/ca.crt
      - xpack.security.http.ssl.certificate=$CERTS_DIR/elasticsearch/elasticsearch.crt
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.certificate_authorities=$CERTS_DIR/ca/ca.crt
      - xpack.security.transport.ssl.certificate=$CERTS_DIR/elasticsearch/elasticsearch.crt
      - xpack.security.transport.ssl.key=$CERTS_DIR/elasticsearch/elasticsearch.key
    ulimits:
      memlock:
        soft: 65536
        hard: 65536
    volumes:
      - ${DATA_DIR}/skywalking/elasticsearch/data:/usr/share/elasticsearch/data
      - ./certs:$CERTS_DIR
    healthcheck:
      test: curl --cacert $CERTS_DIR/ca/ca.crt -s https://localhost:9200 >/dev/null; if [[ $$? == 52 ]]; then echo 0; else echo 1; fi
      interval: 30s
      timeout: 10s
      retries: 5
  skywalking-oap:
    image: apache/skywalking-oap-server:10.1.0-java17
    container_name: skywalking-oap
    volumes:
      - ./certs/skywalking-oap:${SW_CERTS_DIR}/skywalking-oap
      - ./certs/ca:${SW_CERTS_DIR}/ca
    environment:
      - SW_STORAGE=elasticsearch
      - SW_STORAGE_ES_CLUSTER_NODES=elasticsearch:9200
      - SW_STORAGE_ES_HTTP_PROTOCOL=https
      - SW_ES_USER=elastic
      - SW_ES_PASSWORD=asU8F9EJYcHD0sTf1UPK
      - SW_STORAGE_ES_SSL_JKS_PATH=${SW_CERTS_DIR}/skywalking-oap/es-ssl.jks
      - SW_STORAGE_ES_SSL_JKS_PASS=DT8vBxEs9kMd4BqGRtRoEF6HT9FQwNyb
      - SW_CORE_GRPC_SSL_ENABLED=true
      - SW_CORE_GRPC_SSL_KEY_PATH=${SW_CERTS_DIR}/skywalking-oap/skywalking-oap.pem
      - SW_CORE_GRPC_SSL_CERT_CHAIN_PATH=${SW_CERTS_DIR}/skywalking-oap/skywalking-oap.crt
      - SW_CORE_GRPC_SSL_TRUSTED_CA_PATH=${SW_CERTS_DIR}/ca/ca.crt
      - SW_HEALTH_CHECKER=default
      - SW_HEALTH_CHECKER_INTERVAL_SECONDS=30
      - SW_TELEMETRY=prometheus
    depends_on:
      elasticsearch:
        condition: service_healthy
    healthcheck:
      test: [ "CMD-SHELL", "curl http://localhost:12800/healthcheck" ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
  skywalking-ui:
    image: apache/skywalking-ui:10.1.0-java17
    container_name: skywalking-ui
    ports:
      - "18080:8080"
    environment:
      - SW_OAP_ADDRESS=https://skywalking-oap:12800
      - SW_ZIPKIN_ADDRESS=https://skywalking-oap:9412
    depends_on:
      skywalking-oap:
        condition: service_healthy
```

打开浏览器，访问`http://localhost:18080/`就可以访问skywalking的web ui了

## 参考资料

[Encrypting communications in an Elasticsearch Docker Container](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/configuring-tls-docker.html)

[SkyWalking - Security(SSL/TLS/mTLS)](https://skywalking.apache.org/docs/main/v10.1.0/en/setup/backend/grpc-security/)

[SkyWalking - Token Authentication](https://skywalking.apache.org/docs/skywalking-java/v9.3.0/en/setup/service-agent/java-agent/token-auth/)