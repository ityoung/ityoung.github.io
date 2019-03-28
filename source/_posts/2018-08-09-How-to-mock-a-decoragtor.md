---
title: 单元测试中Mock一个Python装饰器
date: 2018-08-09
tags:
    - Python
    - unittest
    - mock
    - 单元测试
    - 测试开发
categories:     单元测试
---

## 场景

测试一些方法时, 可能会遇到该方法需要鉴权的情况(如`login_required`之类), 需要想办法绕过装饰器.

## 解决方案

1. 为装饰器加开关. 在装饰器中通过读取某个配置判断是否开启该装饰器;
2. mock掉装饰器.

前者操作起来方便, 但是在遇到不同装饰器都需要mock时, 为每个装饰器添加开关有些工作量;
而后者也有弊端, 即此次测试中, 所有测试都基于该装饰器不生效的情况下测试. 因此对于如权限等的测试, 需要添加其他测试手段(如接口测试)来完成.

本文讲述的是第二个解决方案的具体实现.

<!--more-->

## 技术点

Mock装饰器与普通方法的不同之处在于, 装饰器在方法声明时就已经产生影响, 而不像普通方法被调用前mock即可, 因此需要在测试执行的最开始, 还没初始化服务端代码时就mock装饰器.

## 写一个自己的装饰器

```
from functools import wraps
def mock_decorator(*args, **kwargs):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            return f(*args, **kwargs)
        return decorated_function
    return decorator
```

## mock原来装饰器

在测试模块的`__init__.py`下执行Mock操作, 即可在运行测试前先替换原来的装饰器.

```
from unittest import mock

mock.patch('decoretor.path.to.mock',mock_decorator).start()
```

## 参考

[1] [Can I patch a Python decorator before it wraps a function?](https://stackoverflow.com/questions/7667567/can-i-patch-a-python-decorator-before-it-wraps-a-function), Stack Overflow
