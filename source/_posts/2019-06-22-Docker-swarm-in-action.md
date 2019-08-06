---
title: 用Docker swarm实现容器服务高可用
date: 2019-06-22
tags:
    - Docker
    - Docker swarm
    - 架构
categories:     Docker
---

## 背景与技术选择

根据我之前的几篇「Django 系列」文章，后端架构中我使用了 Django + Celery + RabbitMQ 三个框架/服务。现在有几个问题：

1. 如何用容器快速部署这三个应用？
2. 如何提高性能？
3. 如何保障后端可用性？

### Docker Compose vs Swarm vs K8s

在我以往的实践中，容器的编排使用了 `docker-compose` 实现，问题一就已经解决。但 `docker-compose` 也只是用于编排，可以各启动三个服务的一个容器，性能与高可用性就可能不能满足要求。

对于性能与高可用，如果是大型项目，目前不二的选择就是 `Kubernetes(K8s)` ，但是我的项目不足以称之为「大型项目」，因此我考虑的是，如何在单宿主机上提高性能与高可用。

<!--more-->

`Docker Swarm` 是官方集成至 Docker CLI 中的集群服务（虽然被 `K8s` 后来居上，但是其设计理念与功能也是十分成熟完善，架构上与 `K8s` 有很多共通点）。`Swarm` 可以连接多个宿主机作为集群节点，当然也是支持单机容器集群的部署。`Swarm` 的集群部署需要创建 `services`, `secret` 等，也是较为繁琐，因此有 `Stack` 工具能够解析 `compose` 文件，让创建各种服务的过程描述于 `YAML` 脚本中，更方便管理。

当然，`Stack` 解析的 `compose` 文件语法与 `docker-compose` 所支持的略微有不同，但大多是通用的，具体后面提到。

综上可以总结一下几个工具的关系与区别：

> 优势/劣势仅针对我的后端架构而言

|工具/服务|优势|劣势|是否满足|
|---|---|---|---|
|docker-compose|容器编排|不支持高可用|否|
|Kubernetes|容器编排，高可用，适合大型项目|无|是|
|Swarm|容器编排，高可用|成熟度略差于K8s|是|
|Stack|属于Swarm的命令，用于应用Compose文件|N/A|N/A|

表格中特意加上了 `Stack`，因为最开始我把 `Stack` 和 `Swarm` 视为同一级了。`Swarm` 更多是提供集群环境，协调各种服务组件，而`Stack` 更像是一个命令行工具，调用 `Swarm` 的各种命令启动不同服务等。

## Swarm 架构简介

![(图片来源: Docker文档)](/img/in-post/article-docker-swarm-in-action/docker-swam.png)

`Swarm` 可以是一个 Docker 集群，在单宿主机上，Swarm 集群中运行着各种应用（`App`），每个应用由一个或多个服务（`Service`）组成，例如前端、后端、数据库三个服务可以组成一个应用。

每个服务由任务（`Task`）的复制集组成，一个任务就是**一个**容器（`Container`）。例如一个后端服务，我们可以通过启动 3 个复制任务来提高性能。服务是程序的出入口，虽然启动了 3 个任务，我们访问这个服务时候还是用服务名即可，请求会在几个任务之间按 RR(轮询) 机制被处理。

## 实现

介绍了 Docker Swarm 的几个概念之后，我们来实际实现标题「用 Docker Swarm 实现容器服务高可用」吧！

### Swarm 集群初始化

> 因为我在单机上部署我的应用，扩展 Swarm 节点的操作就不赘述了。

首先初始一个Swarm集群：

```bash
docker swarm init
```

集群初始化成功后，`docker network ls` 会看到新创建了两个网络：

- 一个名为 `ingress` 的 overlay 网络，用于处理与 swarm 服务相关的控制命令和数据交互。
- 一个名为 `docker_gwbridge` 的网桥，用于打通 swarm 集群中的每个独立的容器之间的网络。

#### 异常解决

在调试 swarm 部署应用时，我在 3 台机器执行了一样的操作，奇怪的是其中一台服务器在初始化 swarm 集群后，只创建了 `ingress`，导致应用部署后任务无法启动。

通过执行：

```bash
docker service ps --no-trunc {serviceName}
```

查看到部分报错信息如下：

```text
ERROR: could not find an available, non-overlapping IPv4 address pool among the defaults to assign to the network
```

发现是 docker 网络出了问题，通过对比几台服务器的网络找到是 `docker_gwbridge` 的缺失。

手动创建该网桥解决问题：

```bash
docker network create \
--subnet 172.20.0.0/20 \
--gateway 172.20.0.1 \
-o com.docker.network.bridge.enable_icc=false \
-o com.docker.network.bridge.name=docker_gwbridge \
docker_gwbridge
```

### 应用部署

应用可以通过手动一个个创建服务的方式部署，通过引用同一个 docker 网络来保证服务间的通信可达。

但是 docker 提供了更方便的方式来部署、扩展一个应用———— `docker-compose.yml` 配置文件

#### Compose 文件

先按我的服务架构，写一份 `docker-compose.yml` 文件：

```yaml
version: '3'

services:
  rabbit:
    image: rabbitmq:3
    ports:
      - "5672:5672"
    networks:
      - webnet

  web:
    image: myweb:latest
    command: python manage.py runserver 0.0.0.0:8000
    environment:
      - DJANGO_SETTINGS_MODULE=bd_annotation_proj.settings.staging
    deploy:
      replicas: 3
    depends_on:
      - rabbit
      - worker
    ports:
      - "8000:8000"
    networks:
      - webnet

  celery-worker:
    image: myweb:latest
    command: celery -A bd_annotation_proj worker -l info
    environment:
      - DJANGO_SETTINGS_MODULE=bd_annotation_proj.settings.staging
    deploy:
      replicas: 2
    depends_on:
      - rabbit
    networks:
      - webnet

networks:
  webnet:
```

重要字段含义如下：

- `version`: docker-compose 文件的版本，docker stack 只支持版本 "3"
- `service`: 服务列表
- `image`: 指定运行的镜像
- `command`: 容器启动的运行命令
- `environment`: 容器内的环境变量，此处配置了 Django 的配置文件指向 `staging`
- `deploy`: 指定部署限制，例如可以限制复制集（`replicas`）大小、CPU 上限等
- `depends_on`: 指明服务间的依赖关系，应用部署时会以依赖关系按顺序启动服务
- `network`: 指明应用中的各服务所接入的网络，指定后服务间通过服务名进行通信

#### 创建应用与服务

有了 `docker-compose.yml` 文件，创建应用以及服务就简单了。假设应用名为 `myapp` ，直接执行：

```bash
docker stack deploy -c docker-compose.yml myapp
```

#### 服务查看

```bash
# 查看所有服务
docker service ls
# 查看与 myapp 相关的服务
docker stack services myapp
```

服务中运行的任务可以直接通过 `docker ps` 查看容器 ID 等信息。

### 应用扩容

修改 `docker-compose.yml` 中的 `replicas` 配置来扩展复制集：

```yaml
...
delopy:
  replicas: 5
```

修改后应用新配置：

```bash
docker stack deploy -c docker-compose.yml myapp
```

### 镜像更新

如果镜像发生了更新，此时需要针对性地更新服务：

```bash
docker service update myapp_web --image myweb:latest --force
```

会有进度条显示当前服务中任务的更新进度。

### 结束应用

通过执行：

```bash
docker stack rm myapp
```

结束应用的生命周期，应用相关的任务、服务与网络都会被删除。

## 总结

通过 Docker Swarm 部署完成我的应用后，我通过删除容器查看重启情况实际测试了一下高可用性，完全符合我的预期。Swarm 中的 `ingress` 类似 Nginx，已经替我们完成了负载均衡，我们只要享受这种便利就好。

性能上，由于启动了 3 个容器，相比之前在一个容器中使用 `uWSGI` 的方式运行服务，部分请求响应时间**提升 10 倍**，相当满意。

容器是开发者的福音，无论是前后端还是测试运维都有必要学习与应用它。虽然容器编排的战役可以说是 K8s 拿下，但是 Swarm 在很多场景也是能够胜任。还是那句话：技术没有对错，只有合适不合适！

## 参考

1. https://docs.docker.com/compose/compose-file
2. https://docs.docker.com/get-started/part3/
3. https://blog.51cto.com/wutengfei/2342901
