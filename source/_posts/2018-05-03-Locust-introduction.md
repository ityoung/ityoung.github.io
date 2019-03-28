---
title:        Locust简单使用 - Python性能测试工具
date:         2018-05-03
tags:
    - Python
    - Locust
    - 性能测试
categories:     性能测试
---

## 简介

Locust 是一个开源压力测试工具.

与其他类似的压测工具相比, locust有几大优势: 

- 使用代码定义用户行为, 无需在GUI中使用鼠标点选, 或者将用户行为写成难以理解的XML文件;

- 支持分布式执行测试代码, 因此能够模拟百万级别的用户量;

- 使用协程执行请求, 较进程与线程而言, 在单机上支持更高的并发量;

- 由于测试代码使用Python编写, 对于用Python编写接口测试代码的用户来说,可以只维护一套代码.

<!--more-->

## 安装

```
pip install locustio
```

> 注：locustio==0.8.1，locust==0.8.0，建议安装0.8.1，有更多特性

安装完成, 运行`locust -h`验证

## 运行

### 单节点（standalone）

```
locust --host=http://host.you.test
```

### 多节点：

#### master：

```
locust --master --host=http://host.you.test
```

#### slave:

```
locust --slave --master-host=your.master.host.ip
```
