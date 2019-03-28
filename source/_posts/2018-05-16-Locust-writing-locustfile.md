---
title:        Locust - 编写locustfile
date:         2018-06-01
tags:
    - Python
    - Locust
    - 性能测试
    - 翻译
categories:     性能测试
---

> 本文是目前最完整的locustfile编写指南中文版！
> 关注微信公众号: devintest 加入微信群，共同学习进步

locustfile就是一份Python代码文件. locustfile的唯一要求是至少要包含一个`Locust`类（或其派生类）, 例如测试HTTP请求时, 至少要包含一个`HttpLocust`类.

## Locust 类

一个`Locust`类代表一类模拟用户。

一个`Locust`类 **必须** 定义一个 `task_set` 属性，用于指向一个`TaskSet`类。`TaskSet`类定义了一系列用户行为，具体内容在后面会介绍到。

另外还有一对常用属性就是 `min_wait`/`max_wait`，分别定义虚拟用户执行两个task之间的最小/最大等待时间，默认值1000，所以不声明这两个属性时，每个task之间默认会有1秒等待时间。

<!--more-->

可以参考下面的例子与注释理解一下Locust类：

```Python
from locust import Locust, TaskSet, task

class MyTaskSet(TaskSet):
    @task
    def my_task(self):
        print("executing my_task")

class MyLocust(Locust):
    task_set = MyTaskSet    # task_set属性指向一个TaskSet类
    min_wait = 5000         # 每个task执行间隔最少5秒
    max_wait = 15000        # 每个task执行间隔最多15秒
```

>更多`Locust`类中的属性请参考官方文档。

## TaskSet 类

`TaskSet`类，从字面上理解，就是一系列task的集合。task定义了用户某些行为，例如“请求某个页面”，“搜索某个产品”等等。`TaskSet`类相当于定义了`Locust`类定义的用户需要执行哪些测试行为。

当一个压力测试启动，每个`Locust`类的实例会执行属性`task_set`指向的`TaskSet`测试集，然后每个`TaskSet`类会调用执行一个task，然后在上节提到的`min_wait`和`max_wait`之间取一个随机等待时间，然后再执行下一个，如此往复。

你需要在`TaskSet`类中编写一些方法用于实现用户行为，声明task方法需要将这些方法使用`task`装饰器装饰。

`task`装饰器最简单的用法可以参考上一节中的代码。除了直接装饰，我们还可以添加一个权重值，task之间执行的次数比值会与他们的权重比值相同：

```
class MyTestSet(TestSet):
    @task(1)
    def a:
        do_something

    @task(3)
    def b:
        do_something
```

上述例子中，task b的权重是a的三倍，因此a执行一次时，b会执行3次。

### setup/teardown

`setup`和`teardown`用于初始化/清理一个测试类所需要/所产生的环境.

它们是怎么被执行,又会被执行多少次呢? 先看locust源码:

_locust/core.py, Line: 282_
```python
if hasattr(self, "setup") and self._setup_has_run is False:
    self._set_setup_flag()
    self.setup()
if hasattr(self, "teardown") and self._teardown_is_set is False:
    self._set_teardown_flag()
    events.quitting += self.teardown
```

从源码中可以看出, `setup`和`teardown`在执行之后会改变标志, 且执行时需要判断标志, 所以`setup`只会在`task`执行前执行一次, 而`teardown`只会在该类中所有`task`执行完毕后执行一次.

> `Locust`类中也有`setup`和`teardown`方法，用法与`TaskSet`类中的基本一致，因此不做赘述。

### on_start/on_stop

一个`TaskSet`类也可以声明一个`on_start`或者`on_stop`方法。`on_start`方法在locust启动一个虚拟用户执行 `TaskSet`类时被调用, 相反的，`on_stop`在`TaskSet`停止时被调用。

### 执行顺序

由于`Locust`类和`TaskSet`类有多种`setup`/`teardown`相互依赖，下面是这些类的执行顺序，方便理解：

1. Locust setup
2. TaskSet setup
3. TaskSet on_start
4. TaskSet tasks…
5. TaskSet on_stop
6. TaskSet teardown
7. Locust teardown

总的来说，`setup`和`teardown`就是互补的关系，因此执行顺序也是先执行`setup`/`on_start`再执行`teardown`/`on_stop`。

## HTTP请求的压力测试

在前面几篇文章中，只介绍了如何创建Locust类、TaskSet类去模拟用户，本文介绍如何对提供HTTP接口的系统进行压力测试。Locust已经有`HttpLocust`类，每个`HttpLocust`类的实例都有一个`client`属性用于构造HTTP请求。

### HttpLocust类

一个HttpLocust类的实例可以表示一组用于HTTP压力测试的“用户”，该用户的行为由`task_set`属性定义，这些与父类`Locust`类都一致。

HttpLocust类有个额外的`client`属性，用于建立与保持HTTP请求会话。熟悉Python的朋友一般都知道`requests`库，HttpLocust类的client正是封装了该库，用法基本一致。

当你的Locust类继承自HttpLocust，那你指向的TaskSet可以直接使用client属性发起HTTP请求，下面是一个例子，可以用来对**/**和**/about**两个URL进行压力测试：

```
from locust import HttpLocust, TaskSet, task

class MyTaskSet(TaskSet):
    @task(2)
    def index(self):
        self.client.get("/")

    @task(1)
    def about(self):
        self.client.get("/about/")

class MyLocust(HttpLocust):
    task_set = MyTaskSet
    min_wait = 5000
    max_wait = 15000
```

上面的示例代码中，每个模拟用户在5-15秒的等待间隔会发起一次请求，task的权重显示了对URL **/** 的请求会是 **/about** 的两倍。

有心的读者会发现，我们可以在TaskSet中直接使用*self.client*调用HttpSession，而不是*self.locust.client*。这是因为为了用户方便，`TaskSet`类中的*self.client*已经直接指向*self.locust.client*。

### HTTP client基本用法

在*client*属性中,每个HttpLocust实例都有一个`HttpSession`实例。 HttpSession类实际上是`requests.Session`的一个子类，可用于发出`get`，`post`，`put`，`delete`，`head`，`patch`和`options`等HTTP请求并报告给Locust的用于统计。 HttpSession实例会在请求之间保存cookie，以便它可以用来登录网站、保持请求会话。 

下面是一个简单的例子，它向 */about* 发出GET请求（假设self是TaskSet或HttpLocust类的实例：

```
response = self.client.get("/about")
print("Response status code:", response.status_code)
print("Response content:", response.text)
```

下面是POST请求的例子：

```
response = self.client.post("/login", {"username":"testuser", "password":"secret"})
```

### 人为标记成功失败

默认情况下，Locust将HTTP响应码为OK(2xx)的请求标记为成功，其他的就标记为失败。可能大多数时候这种处理方式就是我们想要的，但是有的时候就是想测试返回404的情况呢？或者server在返回错误时，错误信息在响应的消息体中而不体现在响应码，这些情况的处理就需要手动控制成功/失败。

使用catch_response参数和with，可以获取response的内容并标记失败：

```
with client.get("/", catch_response=True) as response:
    if response.content != b"Success":
        response.failure("Got wrong response")
```

正如可以将请求OK响码标记为失败一样，用catch_response参数与with语句也可以将请求不为2xx的响应码标记为成功：

```
with client.get("/does_not_exist/", catch_response=True) as response:
    if response.status_code == 404:
        response.success()
```

### 设置检查点

Locust中设置检查点是很简单的，可以直接用assert设置检查点，对response的内容检查，并输出错误信息：

```
response = self.client.get('/test_assert')
assert '‘success' in response.content, "Respense error: " + response
```

### 动态参数请求分组

网站的实参(Query String)经常是动态的，比如*/blog?id=diff_id*，一次查询一个新的id，如果不指定分组，在Locust的测试结果中和会看到默认以"/blog?id=diff_id"的很多分组，而这些请求都是在测试获取某个id的blog，其实可以被分为一组。

通过给client传入*name*参数能够实现分组：

```
# 这些id不同的请求都会被分组到 /blog/?id=[id] 中
for i in range(10):
    client.get("/blog?id=%i" % i, name="/blog?id=[id]")
```

## 翻译自

[locust document - writing a locustfile](https://docs.locust.io/en/latest/writing-a-locustfile.html)

