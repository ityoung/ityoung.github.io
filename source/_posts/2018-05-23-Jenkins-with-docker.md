---
title:        使用Docker部署Jenkins环境
date:         2018-05-23
tags:
    - Jenkins
    - 持续集成
categories:     持续集成
---

## 拉取镜像（以下二选一）

参考https://hub.docker.com/_/jenkins/，拉取Jenkins官方镜像

执行
```
docker pull jenkins
```

参考https://hub.docker.com/r/jenkinsci/blueocean/，拉取Jenkins官方提供的集成blueocean插件的镜像

执行
```
docker pull jenkinsci/blueocean
```

## 创建数据卷

创建一个数据卷，用于持久化Jenkins容器的数据到宿主机

```
docker volume create jenkins_data
```

<!--more-->

## 启动（以下二选一）

启动Jenkins

```
docker run -p 8080:8080 -p 50000:50000 -v jenkins_data:/var/jenkins_home jenkins
```

启动BlueOcean

```
docker run -p 8080:8080 -p 50000:50000 -v jenkins_data:/var/jenkins_home jenkinsci/blueocean
```

## 完成

打开网页`IP:8080`，复制粘贴终端中的密钥，点击下一步，安装插件后创建用户，Jenkins环境即搭建完成。
