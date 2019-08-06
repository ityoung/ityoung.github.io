---
title: Django - 两周从入门到熟练工
date: 2019-06-08
tags:
    - Django
    - Python
    - 后端开发
categories:     Python
---

## 初识 Django

之前 Python 后端开发框架中，对 Tornado 和 Flask 接触比较多，前者适合作为服务框架，后者由于轻量常用来构建简单的后台或服务。

Django 之于上面两个 Web 框架，其自身实现了很多工具类库，显得更为笨重，但上手之后，很多功能不需要再自己实现，比较方便。

由于已经集成诸多功能，Djang 也更常被用来作为后台开发框架。

最近新接手一个后台开发任务也是基于 Django 开发，因此本文对最近这两周的所学做个自我总结。

<!--more-->

## Django 几个模块用法

在具体讲代码之前，我先讲讲整个系统的架构。

刚接手任务时，数据库使用 MySQL5.5，对应的 Django 版本为 2.0（支持MySQL5.5的最高版本）。

考虑到系统对接了不同产品线，一些数据格式存在不确定性，我将数据库改为支持 JSON 格式的 MySQL5.7，Django 也对应升级到最新版 2.2。当然，使用其他数据库也是可以的，例如 PostgreSQL 或者 MongoDB，为了不影响原来开发人员的操作习惯，我还是继续使用 MySQL。

同时考虑到平台存在上传、解析 Excel 文件这种耗时操作，我引入 Celery + RabbitMQ 实现这类操作的异步执行，优化用户体验。

Django 还有很多功能我暂时未使用到，其本身功能之强大着实减少了我造轮子的代码量，但由于初学，也确实花了很大功夫琢磨每个功能。

下面讲讲这两周接触到的 Django 的几个模块以及基础用法，更详细的功能请查看官方文档。

### manage.py

在正式讲功能模块之前，必须得先介绍一下 Django 自带的强大的项目管理工具： `django-admin` 。

通常我们创建一个 Django 项目的过程是这样：

```bash
# 安装 Django
pip install django
# 创建一个 Django 项目
django-admin startproject myproject
```

在创建好 Django 项目后，项目根路径会生成一个 `manage.py` 文件。该文件实际执行的就是 `django-admin` 的功能，只不过已经配置好一些系统变量。

例如在项目内创建一个名为 `mysite` 的 App ，那么可以：

```bash
# 创建一个 App
python manage.py startapp mysite
```

> 记住这个 mysite ，这是你的应用名而不仅仅是文件夹名称，后面的开发代码都会准确引用这个名称。

用 `manage.py` 管理工具，可以方便地实现服务启动、数据库迁移等操作，特别是数据库迁移是我最常用也最爱用的功能，后面会提到。

### Project 与 App

Django 中，有 `Project`(项目) 与 `App`(应用) 的概念，其区别如下：

`Project` 是整个项目的最高层级，可以在 `Project` 中对项目所需的依赖、环境等进行配置；

`App` 依托于项目，一个项目可以有多个应用。

一般来说，一个独立的管理后台项目**配置一个应用**即可，将不同的用户、项目、任务等功能都同属于一个 `App`。而如果需要开发一个内部管理后台，集成 2 个不同的系统，这两个系统中各有项目、任务等模块，那么我们就创建 2 个 `App`，因为这两个应用中的功能互不相关。

### Model

Django 可用 ORM(对象关系映射) 模式操作数据库，举个例子：

```python
# 获取当前用户的用户名
operator = request.user
return operator.username
```

简单的两行代码，底层已经由 Django 实现了连接数据库(连接池)、查询用户表、返回 username 字段值等功能。

> 犹记得我在使用 Tornado 写服务时，需要自己维护一个数据库单例，关心服务启动时数据库实例的初始化顺序等等...

使用 ORM 就需要你对每张表创建一个 `Model` ，内含你对表字段的定义，例如字段类型、默认值、外键约束等。

举个例子，一个 `Task` 表的 Model ：

_models.py_
```python
class Task(models.Model):
    status = models.IntegerField(default=0)
    assignee = models.ForeignKey(settings.AUTH_USER_MODEL,
                                 related_name='+',
                                 null=True,
                                 on_delete=models.SET_NULL)
    create_time = models.DateTimeField(auto_now_add=True)
    update_time = models.DateTimeField(auto_now=True)
```

这里也有要注意的地方：如果不特殊配置，Model 类名会被作为数据库表名的一部分，完整为：<APP_NAME>_<MODEL_NAME>，例如本例为 `mysite_task` 。

### Migration

创建数据库表的 Model 之后，我们就可以用 Migrations 功能创建表的初始化脚本，以及真正在数据库中建表：

```bash
# 初始化迁移脚本
python manage.py makemigrations [<APP_NAME>]
# 执行迁移操作，将数据库脚本在对应的数据库中执行一遍
python manage.py migrate
```

Migration的功能强大在于，当你需要更新某个字段（例如将为某字段设置默认值）时，你可以直接修改 Model 中的字段配置即可，然后再次执行 `makemigrations` ，Django 会为你新建一个数据库脚本记录此次更新，你再执行 `migrate` 便能将此次更新写入到数据库中。

不仅如此，我们还可以新建一个空的 migrations 脚本，编写数据插入操作。有这个脚本，我们可以在系统迁移时都保持一部分测试数据：

```python
from django.db import migrations

# 插入方法
def insert_task(apps, schema_editor):
    Task = apps.get_model('mysite', "Task")
    db_alias = schema_editor.connection.alias
    Task.objects.using(db_alias).bulk_create([
        Task(name='task1'),
    ])

# 回滚方法
def rollback_task(apps, schema_editor):
    Task = apps.get_model('mysite', "Task")
    db_alias = schema_editor.connection.alias
    Task.objects.using(db_alias).filter(name="task1").delete()


class Migration(migrations.Migration):
    dependencies = [
        ('mysite', '0001_initial'),
    ]

    operations = [
        migrations.RunPython(insert_task, rollback_task),
    ]

```

每次迁移，Django都会存有记录，能够判断当前已经执行的操作，待执行的操作，存在风险的操作等。可以使用 `showmigrations` 查看当前各个应用的数据库迁移脚本执行情况：

```bash
python manage.py showmigrations
```

### View

Django 是 MVT 模式，View 相当于 MVC 设计模式中的 Controller(控制器) 而非 View(视图)。真正的视图由 Template(模版)或者静态资源提供。

既然作为控制器，那么 View 的功能就相对清晰了：对接收到的请求数据（Request）做相应处理，并返回处理结果数据（Response）。

View 可以是类，也可以是函数+View装饰器的模式，直接上官方文档的代码吧：

基于类的 View:

```python
from django.http import HttpResponse
from django.views.generic import ListView
from books.models import Book

class BookListView(ListView):
    model = Book

    def head(self, *args, **kwargs):
        last_book = self.get_queryset().latest('publication_date')
        response = HttpResponse('')
        # RFC 1123 date format
        response['Last-Modified'] = last_book.publication_date.strftime('%a, %d %b %Y %H:%M:%S GMT')
        return response
```

基于装饰器的 View:

```python
from django.views.decorators.http import require_http_methods

@require_http_methods(["GET", "POST"])
def my_view(request):
    # I can assume now that only GET or POST requests make it this far
    # ...
    pass
```

这部分更多地涉及到业务逻辑，不够通用因此不多赘述了。

### URL

URL 模块定义了 View 的路由地址：

_mysite/urls.py_
```python
from django.urls import path
from books.views import BookListView

urlpatterns = [
    path('books/', BookListView.as_view()),
]
```

可以配置多层路由，然后在最上层 URL 使用 `includes` 为子路由添加 Prefix:

_myproject/urls.py_
```python
path('api/v1/', include('mysite.urls')),
```

### Test

作为测试开发工程师，对单元测试模块还是需要了解下的（可惜目前正处于开发前期，测试用例可能需要在项目基本稳定之后再补充了）。

和大部分 Web 框架一样，Django 的测试模块也是提供了一个 HTTP Client，使用类似接口测试的模式进行“单元测试”。

我在编写 Tornado 单元测试基础框架时候，是将 Tornado 与数据库的连接 Mock 掉，实际并未操作数据库。但在 Django 中，由于数据库的连接并非由我自己实现，也因为使用了 ORM 模式，所以想真正与其他服务解耦进行单测有些困难，目前就先保持这种测试模式了。

还是以官方示例代码为例吧，编写一个测试套件类：

```python
import unittest
from django.test import Client

class SimpleTest(unittest.TestCase):
    def setUp(self):
        # Every test needs a client.
        self.client = Client()

    def test_details(self):
        # Issue a GET request.
        response = self.client.get('/customer/details/')

        # Check that the response is 200 OK.
        self.assertEqual(response.status_code, 200)

        # Check that the rendered context contains 5 customers.
        self.assertEqual(len(response.context['customers']), 5)
```

运行单元测试：

```bash
python manage.py test
```

### 代码目录结构

在我接手该项目时候，大部分的代码都糅合在一个 Python 文件中，让我有代码洁癖的人看着着实难受。

因为我自己也是才接触 Django，为了数理目录结构我也踩了不少坑，包括将各个模块拆分成不同的应用等骚操作...

我就直接分享我目前正在使用的大致代码结构吧：

```
myapp/          # 存放应用具体实现的代码等
    app_mod1/
        models.py
        views.py
    app_mod2/
        models.py
        views.py
    admin.py
    app.py
    models.py
    views.py
    urls.py
myproject/      # 存放项目配置文件
    settings.py
    urls.py
dockerfiles/    # 存放前后端Dockerfile
docs/           # 存放项目相关的文档
tests/          # 存放测试套件
manage.py
README.md
```

## 总结

在这个项目中，我还引入了 Django-REST-Framework(DRF) 框架的序列化和分页模块，Django-MySQL 对 JSON/List 格式数据的支持，以及 Celery 实现异步与定时任务等操作，整体来说两周时间对 Django 以及扩展模块的学习算是满意，不过背后也是我每天下班继续工作到11点，周末在家查文档学习的辛苦。

Django 本身功能强大，是一个非常成熟的 Web 开发框架，但是上手有一定难度，需要下功夫好好消化官方文档阅读源码，并且应用在自己的项目中。

> 后期会继续分享本项目开发过程中的一些经验技巧，欢迎持续关注！可私聊加群。

## 参考

1. [Django 官方文档](https://www.djangoproject.com/)
