---
title: 浏览器兼容性测试设计与原理剖析
date: 2019-03-28
tags:
    - Selenium
    - UI自动化测试
    - 自动化测试
categories:     自动化测试
---

## 概述

浏览器兼容性是目前前端项目迭代中常常遇到的问题.

每次迭代, 回归测试需要消耗大量人力进行手动操作, 去覆盖不同浏览器下不同业务场景的前端展示情况.

本文的目的即探讨解决这类问题的一个方案: 利用分布式的UI自动化测试框架Selenium Grid解决跨浏览器的兼容性问题.

## 技术方案

- Python + Selenium3, 用于驱动不同的浏览器执行验证操作
- Selenium Grid, 用于实现分布式执行用例
- VMware, 安装不同种类浏览器, 作为执行节点

<!--more-->

### Selenium原理

通过Selenium控制浏览器模拟人工操作的过程如下:

<center>
![Selenium工作原理](/img/in-post/article-Cross-browser-testing-with-selenium/Selenium.png)
<small>图1.1-Selenium原理</small>
</center>

WebDriver充当一个HTTP Server的角色, 它接收Selenium客户端发来的请求, 同时根据不同的请求内容对浏览器进行不同的操作.

Selenium封装了WebDriver所能接收的各类请求路由(URL)和内容(Body)，Python代码在调用Selenium库时其实是在操作Selenium库发送这些HTTP请求。

在代码执行之初，我们需要初始化一个webdriver对象，初始化的过程中，实际上是Selenium库启动了一个子进程，将WebDriver程序运行在后台中，监听接下来Selenium客户端发来的请求。

Selenium客户端，WebDriver与浏览器之间通过初始化浏览器时生成的SessionID进行浏览器对象绑定，使得Python代码能够准确的控制某一浏览器窗口。

### Selenium Grid原理

Selenium Grid 允许我们在不同机器/浏览器上, 同时执行测试用例.

一个Selenium Grid结构中包括一个主节点(hub)和多个从节点(node). 主从节点都是通过运行`selenium-server-standalone.jar`启动.

Selenium 客户端(如本项目)运行在hub上, 通过Selenium Grid将操作浏览器的指令分发到各个node中, 即实现了分布式执行测试用例.

主从结构的优势在于, 无需在所有从节点上配置执行环境(如Python和Selenium库), 即可直接执行主节点发来的执行指令.

<center>
![Selenium Grid架构图](/img/in-post/article-Cross-browser-testing-with-selenium/Selenium.png)
<small>图2.1-测试环境架构图</small>
</center>

## 技术实现

### 自动化测试代码

我们可以使用各种语言编写Selenium客户端执行程序，因为从Selenium原理上可知，WebDriver本身是一个HTTP Server，只要程序语言支持发送HTTP请求都能实现自动化测试的过程。本文将使用Python编码。

#### 自动化执行过程

##### 引用Selenium库

Selenium的Python库已由官方实现，直接安装然后调用即可。

安装：

```shell
pip install selenium==3.141.0
```

引入：

```python
from selenium import webdriver
```

##### 初始化webdriver

以Chrome浏览器为例：

```python
driver = webdriver.Chrome()
```

我们可以打开`Chrome()`这个类查看`__init__`方法，可以看到主要过程就是设置`capabilities`参数用来指定需要操作的对象是Chrome，然后在后台启动了一个子进程(`selenium/webdriver/common/service.py`中`Service`类的`start`方法)，这个过程与在终端中启动`chromedriver`的效果类似，都是启动了一个HTTP server，用于接收接下来的Selenium客户端的请求。

##### 打开浏览器

从这步开始，代码其实是在不断的给WebDriver这个server发送HTTP请求了。

```python
driver.get(url)
```

执行成功，将会打开一个浏览器页面。

##### 查找元素

查找元素的方法有很多，这里以CSS为例，查找一个按钮的元素ID：

```python
element = driver.find_element_by_css_selector(submit_btn)
```

查找到元素后，WebDriver会返回元素的ID等信息，Selenium客户端会创建一个`WebElement`对象存储这个元素信息。

##### 执行点击操作

`WebElement`类中包含封装好的click，submit等多种操作，许多操作过程实际是以JavaScript代码执行。

有了这些封装好的方法，执行点击某元素的操作就很简单了：

```python
element.click()
```

#### 测试结果判断

有了自动化执行的过程，作为自动化测试，少不了测试的过程。

##### 测试框架

测试框架可以使用Python自带的`unittest`即可，在`setUp`方法中初始化浏览器对象，在每个测试用例中使用封装好的断言方法对操作的响应进行比较并返回结果。

##### 更多可能性

在机器学习应用日趋广泛的情况下，提取预期结果的特征，训练出符合自己业务的机器学习模型，将每次执行操作的结果截图作为模型输入，可能能够解决UI自动化测试需要主观判断测试结果的难题。

### Selenium Grid分布式执行用例

#### 环境准备

对于hub, 需要安装:
- JDK8环境, 用于提供selenium-server-standalone.jar包的运行环境
- selenium-server-standalone.jar, 用于建立与node的连接
- Python3环境, 用于执行UI测试代码

对于node, 需要安装:
- JDK8环境, 用于提供selenium-server-standalone.jar包的运行环境
- selenium-server-standalone.jar, 用于建立与hub的连接
- WebDriver, 用于控制浏览器执行操作指令

##### 主节点

1. 以hub角色启动Selenium-server:

```shell
java -jar selenium-server-standalone-3.141.59.jar -role hub
```

待node节点连接成功后, 在hub节点上执行本项目代码, 即可看到node节点上的浏览器启动并执行相应操作.

##### 从节点

1. 以node角色启动Selenium-server:

```shell
java -jar selenium-server-standalone-3.141.59.jar -role node 
  -hub http://<IP>:4444/grid/register \
  -browser "browswerName=<BrowserName>,\
    version=<Version>,\
    platform=<PLATFORM>"
```

参数说明:
- <IP>: hub节点的IP, 注意hub节点与node节点需要互通, 若是虚拟机, 需要将网络配置为桥接(Bridge)模式
- <BrowserName>: 浏览器名, 如: internet explorer, chrome, firefox等
- <Version>: 浏览器版本号: 以IE9为例, version=9
- <PLATFORM>: 系统类型, 如: WINDOWS, LINUX等

在连接成功后可以看到控制台中的一段信息:

```json
Capabilities are: {
    "browswerName": "internet explorer",
    "platform": "WINDOWS",
    "version":""
}
```

这段浏览器配置信息是用于Selenium客户端连接所用:

```python
browser = webdriver.Remote(
    command_executor = 'http://localhost:4444/wd/hub',
    desired_capabilities={
        "browswerName": "internet explorer",
        "platform": "WINDOWS",
        "version":""
    }
)
```

#### 执行过程

##### 修改WebDriver入口

由于所有的浏览器入口都统一成Hub节点的server，因此需要将WebDriver初始化入口代码修改。以IE为例：

```python
driver = webdriver.Remote(
    command_executor = 'http://localhost:4444/wd/hub',
    desired_capabilities=DesiredCapabilities.INTERNETEXPLORER
)
```

##### 运行

除了WebDriver入口的修改，其他代码可以基本保持不动。
和单机的Selenium环境一样，在Hub节点上直接运行测试代码即可，运行后，Node中可用的某个IE浏览器将被启动并执行接下来的操作。

## 参考文档

1. [Wire Protocol](https://github.com/SeleniumHQ/selenium/wiki/JsonWireProtocol), Selenium, Github Wiki
2. [Selenium Grid](https://www.seleniumhq.org/docs/07_selenium_grid.jsp), Selenium, Github Wiki
3. [Selenium with Python](https://selenium-python.readthedocs.io/), Selenium, Readthedocs
4. [InternetExplorerDriver](https://github.com/SeleniumHQ/selenium/wiki/InternetExplorerDriver), Selenium, Github Wiki
