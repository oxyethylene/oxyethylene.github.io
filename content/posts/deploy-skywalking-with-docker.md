+++
title = 'Deploy Skywalking With Docker'
date = 2025-01-25T15:18:58+08:00
draft = true
+++

# 使用Docker部署Skywalking

目标：基于docker和docker compose，部署一个单点的ES + OAP server + UI，接入java agent

- 开启鉴权
- 使用TLS
- 使用nginx给UI添加http basic鉴权

# 使用版本

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

- ElasticSearch 镜像: elasticsearch:7.17.23

- Skywalking-oap-server镜像: apache/skywalking-oap-server:10.1.0-java17

- Skywalking-ui 镜像: apache/skywalking-ui:10.1.0-java17

# 部署单点ES并开启https和http basic auth

## 生成证书

TLS参考[Encrypting communications in an Elasticsearch Docker Container](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/configuring-tls-docker.html)。

1. 创建`instances.yml`:
    ```yaml
    instances:
      - name: elasticsearch
        dns:
        - elasticsearch
        - localhost
        ip:
        - 127.0.0.1
    ```
    因为使用了单点ES，所以根据实际情况修改了配置，instances部分只只有一个元素。dns部分除了`localhost`，还有一个`elasticsearch`，这是后面ES容器的名字，用于容器间直接用域名访问

1. 创建`.env`文件
    ```ini
    COMPOSE_PROJECT_NAME=skywalking
    CERTS_DIR=/usr/share/elasticsearch/config/certificates
    ```
    `COMPOSE_PROJECT_NAME`是`docker compose`预定义的一个环境变量，用于定义项目的名字，详见[COMPOSE_PROJECT_NAME](https://docs.docker.com/compose/how-tos/environment-variables/envvars/#compose_project_name)。之后所有`docker compose`创建的volume和network等都会使用`skywaling_`作为前缀

1. 创建`generate-certs.yaml`

    和官方的教程相比，把生成的certs挂载到了宿主机上而不是volume里。在无网络环境，可以注释掉command里下载unzip的部分，在宿主机解压生成的`bundle.zip`
    ```yaml
    services:
    create_certs:
        container_name: create_certs
        image: docker.elastic.co/elasticsearch/elasticsearch:7.17.27
        command: >
        bash -c '
            if [[ ! -f /certs/bundle.zip ]]; then
            bin/elasticsearch-certutil cert --silent --pem --in config/certificates/instances.yml -out /certs/bundle.zip;
            unzip /certs/bundle.zip -d /certs;
            fi;
            chown -R 1000:0 /certs
        '
        user: "0"
        working_dir: /usr/share/elasticsearch
        volumes:
        - ./certs:/certs
        - .:/usr/share/elasticsearch/config/certificates
    ```

1. 生成ca和es证书
    运行`docker compose -f generate-certs.yaml run --rm create_certs`，即可在挂载的目录下(`./certs`)看到如下的文件结构
    ```
    certs
    ├── bundle.zip
    ├── ca
    │   └── ca.crt
    └── elasticsearch
        ├── elasticsearch.crt
        └── elasticsearch.key
    ```
    此时所需的文件已经生成了，但是`docker compose`默认创建的network还存在，需要额外执行`docker compose -f generate-certs.yaml down`来删除network

## 启动Elasticsearch

1. 修改`.env`
    ```ini { hl_lines=3}
    COMPOSE_PROJECT_NAME=skywalking
    CERTS_DIR=/usr/share/elasticsearch/config/certificates
    ELASTIC_PASSWORD=ChangeMe
    ```
    添加`ELASTIC_PASSWORD`，这个是第一次启动ES时，`elastic`用户的初始密码。

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
    使用`docker compose -f docker-compose.yaml up -d`启动ES。

    可以使用`docker ps | grep elasticsearch`来查看ES的运行情况
    ```
    docker ps -a | grep elasticsearch
    eb91b9ca102d   elasticsearch:7.17.27            "/bin/tini -- /usr/l…"   10 minutes ago   Up 10 minutes (healthy)    9200/tcp, 9300 tcp                          elasticsearch
    ```

    用容器内的curl来验证TLS和basic auth已经生效。
    ```
    docker exec elasticsearch curl --cacert /usr/share/elasticsearch/config/certificates/ca/ca.crt -u elastic:ChangeMe https://127.0.0.1:9200/
    % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
    0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0{
    "name" : "es",
    "cluster_name" : "docker-cluster",
    "cluster_uuid" : "B8cPvx-IQPSgvI8JxsXm7g",
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
    100   537  100   537    0     0   5370      0 --:--:-- --:--:-- --:--:--  5370
    ```

## 修改ES用户密码

此时ES的`elastic`用户的密码是`.env`文件里配置的初始密码，ES提供了`elasticsearch-setup-passwords`命令来给内置用户设置密码。
可以用`docker exec elasticsearch /bin/bash -c "bin/elasticsearch-setup-passwords auto --batch --url https://localhost:9200"`给所有用户设置随机的密码，
也可以用`docker exec elasticsearch /bin/bash -c "bin/elasticsearch-setup-passwords interactive --url https://localhost:9200`手动设置每个用户的密码