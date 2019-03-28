---
title:        tornado SSL 证书获取与服务器配置
date:         2018-01-16
tags:
    - Python
    - 测试开发
    - 后端
    - Tornado
categories:     Python-web
---

## 服务端生成证书

### 进入 openssl 目录

` $ cd /usr/lib/ssl `

### 生成私钥

` $ sudo openssl genrsa -des3 -out server.key 1024 `

### 生成 CSR 文件

` $ sudo openssl req -new -key server.key -out server.csr -config openssl.cnf `

<!--more-->

其中必填项有: 

* Country Name (2 letter code) [AU]:

* Common Name (e.g. server FQDN or YOUR name) []:

### 生成 CA (用于自签名)

* 新建demoCA, demoCA/certs, demoCA/newcerts
` $ sudo mkdir demoCA demoCA/certs demoCA/newcerts `

* 在 demoCAm 目录下新建空文件 index.txt

* 在 demoCAm 目录下新建文件 serial, 内容是一个合法的 16 进制数字, 例如 0000

* 返回 /usr/lib/ssl 目录, 执行如下命令生成 CA
` $ sudo openssl req -new -x509 -keyout ca.key -out ca.crt -config openssl.cnf `

其中必填项有: 

* Country Name (2 letter code) [AU]:

* Common Name (e.g. server FQDN or YOUR name) []:

### 签名

` $ sudo openssl ca -in server.csr -out server.crt -cert ca.crt -keyfile ca.key -config openssl.cnf `

---

# tornado 启动时加上 SSL 选项

* 复制证书文件到 tornado server 目录下(可选)

* 修改测试服务器代码 test.py
    ```
    import tornado.ioloop
    import tornado.web
    import os
    
    class TestGetHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("Hello, World!")
    
    def make_app():
        return tornado.web.Application([
            (r"/", TestGetHandler),
        ])
    
    if __name__ == "__main__":
        application = make_app()
        http_server = tornado.httpserver.HTTPServer(application, ssl_options={
               "certfile": os.path.join(os.path.abspath("."), "server.crt"),
               "keyfile": os.path.join(os.path.abspath("."), "server.key"),
        })
        http_server.listen(443)
        tornado.ioloop.IOLoop.instance().start()
    
    ```

* 由于端口号小于1000, 因此需要使用 su 权限的用户运行脚本
    ```
    $ sudo python test.py
    ```

---

# 客户端访问

### 浏览器

输入`https://localhost`直接访问

### CURL

添加 `-k` 选项忽略 SSL 验证, 如下:

```
$ curl -k https://localhost
```

### requests

添加 `verify=False` 选项, 如下:

```Python
requests.get(URL, verify=False)
```

---

# 参考

[1] [使用Tornado搭建HTTPS网站](http://www.yeolar.com/note/2015/04/30/tornado-ssl-https/), yeolar

[2] [curl - 为什么不能识别自签名的SSL证书？](https://code.i-harness.com/zh-CN/q/10c8411)