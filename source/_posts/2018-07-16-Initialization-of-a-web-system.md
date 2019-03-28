---
title: Python-web系统初始化的设计与实现
date: 2018-07-16
tags:
    - Python
    - Tornado
    - MongoDB
    - 测试开发
categories:     Python-web
---

## 简介
---
>本文针对后端服务初始化时，用户表的创建流程与具体实现做介绍。

Motor是一个异步非阻塞的MongoDB驱动程序，很好地支持了Tornado和asyncio的应用。
基于Tornado开发的后端server，为了提供优秀的系统性能通常会大量使用异步编程，因此Motor是此类系统操作MongoDB时的不二之选。
下面基于特定的场景，介绍系统初始化时涉及到的Motor操作MongoDB的方法：

## 设计
---
对于一个系统初始化，我设计为在首次访问系统时会自动创建一个admin用户用于后面的登录。

*流程图：*
![首次访问系统的流程](http://upload-images.jianshu.io/upload_images/3520043-a1bc36327f504cf0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1080/q/50)

<!--more-->

这里面涉及到：

1. 连接数据库
2. 查询数据库是否有Users表
3. 插入一条数据（插入数据时会自动创建表）


## 实现
---

下面分点介绍实现过程：

### 连接数据库

首先写一个异步连接数据库方法，使用Motor库的`AsyncIOMotorClient`类初始化连接：

```
from motor.motor_asyncio import AsyncIOMotorClient as MotorClient

async def conn_mongo():
    conn = MotorClient('localhost', 27017)
    db = conn.qaadmin
    return db
```

注意：

- 连接数据库的配置以你本人的环境为准；
- 需要提前创建一个`qaadmin`数据库。

### 查询数据库是否有Users表

使用`MotorDatabase`类的`list_collection_names`方法来获取数据库中的所有collection名的列表，再判断`Users`是否在列表中来判断是否创建过用户：

```
async def user_check(self):
    collections = await conn_mongo.list_collection_names()
    if 'Users' in collections:
        return True
    else:
        return False
```

### 插入用户数据

对于用户密码，在数据库中需要加密后存储。原始密码加盐后，使用md5加密：

```
import hashlib

def encrypt_password(origin: str, salt: int) -> str:
    """
    md5随机数加密用户密码
    :param origin: 原始密码
    :param salt:
    :return:
    """
    string = origin + str(salt)
    return hashlib.md5(string.encode(encoding='utf-8')).hexdigest()
```

使用MotorColletion类的`insert_one`方法实现数据的插入，整个系统初始化部分对数据库的操作代码如下：

```
from libs.crypto import encrypt_password
import random

async def user_check(self):
    collections = await conn_mongo.list_collection_names()
    if 'Users' in collections:
        return
    else:
        salt = random.randint(10, 99)
        new_user = {
            'salt': salt,
            'username': 'admin',
            "password": encrypt_password('admin', salt)}
        result = await conn_mongo.Users.insert_one(new_user)
        print('Create user: admin, inserted id: %s' % repr(result.inserted_id))
```

## 参考
---

[1] [MotorDatabase.list_collection_names](https://motor.readthedocs.io/en/stable/api-tornado/motor_database.html#motor.motor_tornado.MotorDatabase.list_collection_names)
[2] [inserting-a-document](https://motor.readthedocs.io/en/stable/tutorial-asyncio.html#inserting-a-document)