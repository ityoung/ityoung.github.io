---
title:        Python - 用协程并发执行测试用例
date:         2018-02-02
tags:
    - Python
    - 测试开发
    - 协程
categories:     Python
---

## 背景

最近在工作中，遇到需要执行大量测试用例的情况。大概2500个测试用例，每个用例有数条HTTP请求以及一些特殊处理，线性执行下来的结果是我花了2个半小时才执行完成！

为了解决这个问题，提高工作效率，实现测试用例并发执行是我所需要的。

## 几个想法

1. 多线程执行测试用例

2. 协程

## 可能遇到的问题:

### 多线程

值得说明的是，多线程执行测试用例是有前辈实现过的。具体实现过程没有看过，依稀记得他的实现过程会生成多份测试报告，然后用BeautifulSoup去解析、合并测试报告。

### 协程

1. 使用协程，可能需要修改PyUnit源码
2. 使用协程，可能需要修改当前测试用例为协程方法，改动可能很大

<!--more-->

## 实现过程

由于正在学习Python协程，我还是决定先用协程试试能否实现。当然，最终结果证明改动不大，仅修改了源码的一小小部分，测试用例部分则只修改了单元测试模块名

首先是获取python3的源码。

`git clone -b 3.5 https://github.com/python/cpython.git`

unittest源码在`Lib/unittest`中。

给已有的测试用例打断点，调试几波，理清unittest模块对测试用例的调用执行顺序。

猜测主要过程为：

![原实现流程](/img/in-post/article-fastunit/origin-process.png)

找到TestCase真正执行入口：

`unittest/case.py` 中的 `TestCase` 类中的 `run` 方法的 `testMethod()`。（Line 605）

![测试用例执行入口](/img/in-post/article-fastunit/entry.png)

上层调用在TestSuite类中，修改上层入口，把对TestCase的顺序执行改为调用协程并发执行：

![提取循环部分代码，改写为协程](/img/in-post/article-fastunit/code-changes.png)

## 最终结果

修改后，执行流程如下：

![现在实现流程](/img/in-post/article-fastunit/now-process.png)

经过上述修改，原来2600多个测试用例需要的执行时间，从两个半小时压缩到8分半钟（如图），提高效率不赘述。

![测试报告](/img/in-post/article-fastunit/test-report.png)

## 存在的问题

~~测试报告的Log获取不准确，应该是IO处理速度不足导致，后续需要想办法解决。（又有事做了:P）~~

~~已解决上述问题，原因在于使用HTMLTestRunner时，输出流代码未生效，导致原先重定向至stderr的结果被写入测试报告。在添加一秒钟睡眠时间后解决。（代码一行，debug一天，哭）~~

![修复报告问题](/img/in-post/article-fastunit/fix-HTMLTestRunner-error.png)

**18-04 更新：**

该问题的根本原因在于`HTMLTestRunner`本身不支持并发处理，因此在使用unittest的`TextTestRunner`时不会存在该问题！

## Github

https://github.com/ityoung/python3-fastunit

## 参考

https://stackoverflow.com/questions/30172821/python-asyncio-task-got-bad-yield
