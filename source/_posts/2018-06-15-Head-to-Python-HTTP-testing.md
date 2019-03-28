---
title:        Python - 接口自动化测试入门
date:         2018-06-15
tags:
    - 自动化测试
    - 接口测试
    - Python
categories:     接口测试
---

> 所谓师父领进门修行在个人，本文旨在介绍接口测试的学习路线，私下里，各位对每个工具都应更深入地了解一遍。同样的，在学习过程中可能会遇到一些问题，希望先自行搜索解决，要有解决问题的能力。

## 正文

HTTP接口测试主要工作可分为两部分：

- 发送HTTP请求（自动化部分）

- 判断响应内容是否符合预期（测试部分）

一般测试工作就是完成请求体的构造，然后根据开发给的接口文档，将构造好的请求体发送给服务端，再判断服务端返回的结果是否符合预期。然后通常会用一些测试框架，例如Python自带的`unittest`测试框架将零散的测试用例集中运行处理。

本文简单介绍的是第一部分，即如何用Python实现发送HTTP请求。以后可能会讲到`unittest`的用法，但**建议先自行了解与使用**。

### Chrome调试工具

打开Chrome浏览器，按F12就可以打开开发人员调试工具（后面简称DT）。

<!--more-->

对于接口测试，我们关注`Network`标签即可。

![点击切换到Network标签](https://upload-images.jianshu.io/upload_images/3520043-88b09c2cc8343b87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

访问一个网站，比如我访问的是知乎的某个问题，可以看到DT中刷新出了很多请求，有请求接口的也有请求图片的。找到请求接口的某条请求，点开可以看到请求的具体内容。

![需要关注的点](https://upload-images.jianshu.io/upload_images/3520043-886d7a41d734810f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于 HTTP 报文，我们重点关注的几个部分：

- **request**

  - header

  - request URL

  - Method

  - Data 就是发送给服务端的消息体

- **response**

  - header

  - body 就是服务器返回的消息体，可以通过Preview/Response查看

因此在DT中的接口请求，我们重点关注`Headers`部分和`Preview`/`Response`部分。

`Headers`部分包含了requests的headers和body，以及response的headers；`Response`包含了response的body；`Preview`则是`Response`美化后的显示。

### Postman 调试工具

下载地址：https://www.getpostman.com/apps

Postman是一款优秀的接口调试工具，当然，他的功能很多，但是从可控性上不如直接写测试代码好。自写代码更好扩展，不用考虑工具的限制，所以我将其视为“接口调试工具”。

关于Postman，学习基础即可，例如发送请求。接口测试新手在使用Postman调试一个接口成功之后，再将请求转换为Python代码实现一遍，能够避免少走一些弯路。

### Python 实现

试一个用Python实现豆瓣登陆，并测试登录是否成功。

#### Requests 库

 Python中，最常用的http请求库非requests莫属。首先在电脑安装‵requests`库：

```
pip3 install requests
```

先在浏览器试一遍登陆过程：

打开浏览器，进入豆瓣登录页面：https://www.douban.com/login

打开Chrome开发者工具，然后输入帐号密码，点击登录，再查看DT，可以看到已经发送了一个登录请求：

![豆瓣登录抓包](https://upload-images.jianshu.io/upload_images/3520043-f42dbe4a69388c97.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`Request URL`，`Request Method`，`Form Data`这些参数就是豆瓣登录请求需要用到的。

打开Python编辑器，构建需要发送的消息体，并用`requests`库发送请求:

```
import requests

 # Request URL
url = 'https://accounts.douban.com/login'
# Request Data
data = dict(
    source=None,
    redir='https://www.douban.com',
    form_email='test',
    form_password='test123'
)
# 使用Requests发送Post请求
response = requests.post(url, data)
print(response)
```

运行后可以看到，Response 200，表示**请求发送成功**。

注意，**响应码200仅代表请求的状态，而不代表接口返回给你的实际结果**。我们真正需要的是response的内容（content），将上面的print(response)改为：

```
print(response.content)
```

输出的内容其实就是服务端返回给浏览器的内容，没登陆成功是因为使用了错误的密码。
