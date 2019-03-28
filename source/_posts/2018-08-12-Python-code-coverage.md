---
title: Python 测试覆盖率统计
date: 2018-08-12
tags:
    - Python
    - Coverage
    - 单元测试
    - 测试开发
categories:     单元测试
---

## 安装：

Python 的测试覆盖率使用 `Coverage` 模块， 需要先安装：

```
pip install coverage
```

## 执行：

假设你原来执行单元测试的命令为：

```
python runtest.py
```

<!--more-->

那么需要分析测试覆盖率时，只要将命令改为如下即可：

```
coverage run --source . --omit */tests* runtest.py
```

参数解释：

- `--source .`指定分析的路径为当前路径下的文件，不会分析引用其他库的覆盖率；
- `--omit */tests*`指定不分析的当前路径下的`tests`文件夹。

## 查看结果：

输出到控制台的简单统计结果：

```
coverage report
```

也可以转化成HTML，会在当前目录生成*covhtml*文件夹，打开html文件即可查看详细的覆盖率情况：

```
coverage html
```

## 集成至gitlab

yaml脚本添加如下两行：

```
coverage run --source . --omit */tests* runtest.py
coverage report
```

在gitlab的`CI/CD` -> `General pipelines settings`配置中，添加`Test coverage parsing`的正则：

```
\d+\%\s*$
```

运行后，单元测试的`Job`页面即可看到coverage

---EOF---