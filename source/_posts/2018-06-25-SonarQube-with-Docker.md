---
title:        使用Docker快速搭建静态代码分析平台
date:         2018-06-25
tags:
    - Docker
    - 静态代码分析
    - Sonar
categories:     Docker
---

## 简介

本文分享两个Docker镜像，能够：
1. 快速部署SonarQube静态代码分析平台；
2. 完整的SonarScanner **Python**环境用于快速使用。

根据这两个镜像，你可以设计合理的持续集成流程，让你的代码在提交时自动执行代码分析。

## 环境

- Docker
- Git
- Docker-compose

## 使用

1. 下载 [SonarQube Docker镜像](https://github.com/ityoung/sonarqube-docker) 项目

<!--more-->

2. 进入sonarqube-docker项目根目录，执行`docker-compose up`初始化并启动SonarQube
3. 下载 [SonarScanner Docker镜像](https://github.com/ityoung/sonar-scanner-docker) 项目
4. 进入sonar-scanner-docker项目根目录，修改`sonar-scanner.properties`中的`sonar.host.url`为你的docker宿主机 **IP:端口**
5. 执行`docker build -t sonar-scanner .`构建sonar-scanner镜像（注意小数点）
6. 将你要测试的项目代码clone至镜像中进行测试即可（可通过基于该镜像写一个Dockerfile，或者启动一个容器后将代码复制到容器中）

## Github链接

1. [SonarQube Docker镜像](https://github.com/ityoung/sonarqube-docker)
2. [SonarScanner Docker镜像](https://github.com/ityoung/sonar-scanner-docker)

## 加群

**测试开发交流群**，添加个人微信进群：nxyx520
