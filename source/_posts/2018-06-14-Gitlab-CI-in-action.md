---
title:        Python+Gitlab 持续集成完整实践
date:         2018-06-14
tags:
    - Docker
    - 持续集成
    - 静态代码分析
    - Sonar
    - Python
    - Gitlab-CI
categories:     持续集成
---

借着公司代码库迁移到私有Gitlab的契机，我接下持续集成的工作，实现了对**Python**服务端代码的单元测试、静态代码分析和接口测试的持续集成。总体架构如下：

![架构图](https://upload-images.jianshu.io/upload_images/3520043-037c6419d9b7834e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

执行过程：

1. 开发提交代码后，自动触发gitlab-runner拉取executor镜像执行单元测试，单元测试代码中包含上传测试结果到x-utest测试平台；

2. 单元测试通过后，gitlab-runner拉取sonar-scanner镜像执行静态代码分析，分析结果评论在commit中或保存于sonarqube；

<!--more-->

3. 静态代码分析结束，执行分发操作，将代码分发至灰度测试服务器，并运行；

4. 执行接口测试，执行完成后上传测试结果到x-utest测试平台。

四个任务相互依赖，只在前一个任务状态为成功情况下才会执行。

> 本文主要描述一些技术实现，会适当贴出代码，可能能为相关从业者提供灵感与解决方案，不保证每个细节都深入讲解。

## Gitlab CI 基本配置

针对某个需要做CI/CD的项目，需要将代码库的该设置打开，并为其配置 gitlab-runner。

### gitlab runner

gitlab-runner不仅可以运行在物理机，还可以运行在容器中。考虑到gitlab-runner消耗的资源少，使用容器更合适。

拉取gitlab-runner Docker 镜像：

```
sudo docker pull gitlab/gitlab-runner
```

启动容器：

```
sudo docker run -d --name gitlab-runner --restart always \
    -v /srv/gitlab-runner/config:/etc/gitlab-runner \
    -v /var/run/docker.sock:/var/run/docker.sock \
    gitlab/gitlab-runner:latest
```

在容器中执行register操作，将gitlab上的项目注册到gitlab-runner中：

```
sudo docker exec -it gitlab-runner gitlab-ci-multi-runner register
```

>输入上述命令后会有一系列的配置需要输入，当然也可以设置完后进行更改
按照提示输入即可，前两项可以在指定项目设置中CI/CD选项里的Runners settings选项中的Specific Runners里看到，tags是gitlab-ci.yml文件中所要用到的，executor选择docker
配置成功后，我们可以在设置中CI/CD选项里的Runners settings选项中的Specific Runners里看到runner信息 

### 本地executor镜像

为了部署与测试，需要一个镜像用于执行。当选用**本地镜像**时，会发现如下报错

![拉取镜像失败](https://upload-images.jianshu.io/upload_images/3520043-ef053efa262510a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

报错的原因在于，gitlab-runner尝试去官方的docker hub仓库拉取镜像。通过修改gitlab-runner中的配置，设置只拉取本地镜像：

修改 */etc/gitlab-runner/config.toml* ，在 `[runners.docker]` 下，添加：

```
pull_policy = never  # 该配置默认always，即只在线上拉取镜像
```

**如果有需要**添加一些hosts映射，仍然在 `[runners.docker]` 下，添加：

```
extra_hosts = ["hostname:ip"]
```

另外为了加快单元测试执行速度，将服务端代码的依赖提前安装至executor镜像中：

```
COPY requirement.txt .
RUN pip install -r requirement.txt
```

## 编写 .gitlab-ci.yaml 

### 单元测试部分

#### 用nose执行测试

对于Python，nosetest工具可以嗅探与执行你写的所有测试用例，并输出结果。在执行测试前，使用nose需要使用pip安装

```
pip install nose
```

安装完成后，使用 `nosetests` 执行。

```
nosetests
```

#### 自写测试入口

另一个执行测试的选择，是自写测试入口，不依赖nose。好处是能够将测试结果上传至x-utest。
对测试结果做判断，如果全部用例通过（即wasSuccessful为True），则sys.exit(0)，否则sys.exit(1)

#### redis与mongo服务化

对于redis与mongo这种外部服务，有两种解决方式，一是mock对数据库的读写，二是使用服务化的redis与mongo，保证外部环境的一致性。（更推荐第一种方式，此处介绍第二种。）

由于设置了不从docker hub拉取镜像，因此需要先拉取redis与mongo服务镜像到本地
```
docker pull redis:2.8
docker pull mongo:3.2
```

在gitlab-ci.yaml中添加services:
```
services:
  - redis:2.8
  - mongo:3.2
```

修改代码的local_config配置文件中的mongo与redis连接URL，指向“mongo”与“redis”

### 静态代码分析

#### sonarqube搭建

制做了一个docker-compose项目可以[一键部署SonarQube平台](https://github.com/ityoung/sonarqube-docker) **<==欢迎fork/start**，使用postgres作为后端数据库，并将数据持久化在宿主机本地。

```
git clone https://github.com/ityoung/sonarqube-docker.git
pip install docker-compose
cd sonarqube-docker
docker-compose up
```

#### sonar scanner配置

同时我也针对Python开源了[sonar-scanner镜像的Dockerfile](https://github.com/ityoung/sonar-scanner-docker) **<==欢迎fork/start**，该镜像已经安装pylint，方便做Python的静态代码分析。

```
git clone https://github.com/ityoung/sonar-scanner-docker.git
cd sonar-scanner-docker
docker build -t sonar-scanner .
```

#### YAML添加执行命令

启动SonarQube后，进入IP:9000到SonarQube管理页面，登录admin/admin，新建一个项目，按步骤执行完成

![创建一个project](https://upload-images.jianshu.io/upload_images/3520043-a8a8597adc85200b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

创建完成后，获取到执行代码，复制这段代码，添加到yaml中，能够实现分析结果上传到SonarQube。

![获取sonar-scanner执行脚本](https://upload-images.jianshu.io/upload_images/3520043-db6f17925c82c844.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 注意：如果yaml中用到了两个镜像，尽量不要有before_script，否则可能两个镜像，触发错误。

#### Sonar分析后评论

对于develop分支，可以不保存分析结果，而改为将分析结果评论在当次commit下。

在yaml脚本中添加如下参数：

```
    - sonar-scanner
      -Dsonar.analysis.mode=preview
      -Dsonar.gitlab.commit_sha=$CI_BUILD_REF
      -Dsonar.gitlab.ref_name=$CI_BUILD_REF_NAME
      -Dsonar.gitlab.project_id=$CI_PROJECT_ID
```

> 注意：无新issue时默认不会评论，需要在SonarQube修改gitlab配置才会每次都评论。

### 持续交付

这部分交由对服务端部署更熟悉的运维操作，因此不做赘述。

### 接口测试

接口测试代码在另一个仓库，这就涉及到从另一个仓库clone测试代码时的权限问题。

给仓库URL添加token能够实现跨仓库clone代码：

```
git clone https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.xxx.com/yx/apitest.git
```

## 附一：完整的.gitlab-ci.yml

```yaml
image: pro1_executor

stages:
  - unittest
  - analyze
  - deploy
  - apitest

variables:
  SONAR_HOST: "http://192.168.0.29:9000"
  SONAR_PROJ: "pro_1"
  SONAR_LOGIN: "b3135dd602b61ce7ff5f4202a3ec2ec0865fa7f5"

services:
  - redis:3
  - mongo:3.2

UnitTest:
  stage: unittest
  script:
  - pip install nose
  - python -m runtest xtest

Sonar_Preview:
  image: sonar-scanner
  stage: analyze
  script:
    - sonar-scanner
      -Dsonar.analysis.mode=preview
      -Dsonar.gitlab.commit_sha=$CI_BUILD_REF
      -Dsonar.gitlab.ref_name=$CI_BUILD_REF_NAME
      -Dsonar.gitlab.project_id=$CI_PROJECT_ID
      -Dsonar.projectKey=$SONAR_PROJ
      -Dsonar.sources=.
      -Dsonar.host.url=$SONAR_HOST
      -Dsonar.login=$SONAR_LOGIN
  except:
    - master

Sonar_Analyze:
  image: sonar-scanner
  stage: analyze
  script:
    - sonar-scanner
      -Dsonar.projectKey=$SONAR_PROJ
      -Dsonar.sources=.
      -Dsonar.host.url=$SONAR_HOST
      -Dsonar.login=$SONAR_LOGIN
  only:
    - master

Deploy_TestServer:
  stage: deploy
  script: echo "deploy"

API_Test:
  stage: apitest
  script:
    - cd /
    - git clone https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.xxx.com/yx/apitest.git
    - cd apitest
    - pip install -r requirements.txt
    - python runtest.py
```

## 参考

[1] [how to access multiple repositories in CI build?](https://stackoverflow.com/a/50163888/6354733), Stack Overflow
