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

1. 创建`instances.yaml`:
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
1. 


但是把生成的certs挂在到了宿主机上而不是volume里。

在无网络环境，可以注释掉command里下载unzip的部分，在宿主机解压

