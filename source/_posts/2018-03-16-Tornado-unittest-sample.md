---
title:        Tornado - 一个简单的单元测试实例
date:         2018-03-16
tags:
    - Python
    - Tornado
    - unittest
    - 单元测试
categories:     单元测试
---

创建一个简单的 tornado 服务程序 `server.py`

```
#!/usr/bin/env python
# coding=utf-8
from tornado.web import RequestHandler, Application
import tornado.ioloop
class MainHandler(RequestHandler):
    def get(self):
        self.write("Hello, world")
application = Application([
    (r"/", MainHandler),
])
if __name__ == "__main__":
    application.listen(8888)
    tornado.ioloop.IOLoop.current().start()
```

启动后，浏览器访问 `localhost:8888` 可以看到输出 `Hello, world`

<!--more-->

> 运行单元测试时，服务器可以关闭开启，不影响

创建一个 tornado 单元测试文件 `test_main.py`

```
#!/usr/bin/env python
# coding=utf-8
import unittest
from tornado.testing import AsyncHTTPTestCase
from server import application
class TestMain(AsyncHTTPTestCase):
    def get_app(self):
        return application
    def test_main(self):
        response = self.fetch('/')
        self.assertEqual(response.code, 200)
        self.assertEqual(response.body, b'Hello, world')
if __name__ == '__main__':
    unittest.main()
```

执行，可以看到结果为通过。
