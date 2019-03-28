---
title:        Jmeter遍历MongoDB大量数据
date:         2018-06-28
tags:
    - Jmeter
    - 性能测试
categories:     性能测试
---

## 测试场景

在MongoDB中存有上万个ID，某个接口需要带上ID参数，且每个ID只能使用一次，测试该接口的性能。

## 设计思路

*思路一：一个线程组中完成数据库读ID操作，用**ForEach逻辑控制器**来遍历ID并发送请求。*

这个过程听起来似乎能够完成这个场景，但是这需要一个很长的等待测试**串行**执行的时间：这可是上万个ID。

*思路二：多线程并行执行。*

并行执行测试用例的速度肯定要大于串行执行，并且这才符合用Jmeter的初衷。

更细化一些，我需要再建一个**setUp线程组**，在这个线程组中读取数据库中的ID信息并存为变量；然后，通过启动大于ID数量的线程数去执行这些用例。

为了覆盖到每一个ID，我在读取数据库后存为参数时，需要为每个ID赋上唯一的Key值，同时各个线程应该读到不同的ID，所以需要一个计数器**Counter**。

<!--more-->

## 具体实现

### Jmeter 3 开启MongoDB插件

由于MongoDB插件存在性能上的问题，Jmeter 3中已经将其删除（其实只是Deprated，标记未来不可用，但是还是能够开启），我们需要先解除限制。

进入Jmeter目录，修改`bin/user.properties`文件，在文件末尾添加：

```
not_in_menu=
```

重启Jmeter即可看到相关的插件。

### 操作MongoDB

首先需要添加一个**MongoDB Source Config**, 然后配置MongoDB的地址，并设置别名：

![MongoDB Source Config](https://upload-images.jianshu.io/upload_images/3520043-7b2e0c564b0d5713.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后添加**MongoDB Script**, 配置数据库与MongoDB源的别名：

![配置MongoDB Script](https://upload-images.jianshu.io/upload_images/3520043-e887e1bd08bd71a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

编写数据库查询语句：

![编写MongoDB Script](https://upload-images.jianshu.io/upload_images/3520043-ca111795a090fe11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

要注意，数据库查询语句后面要跟上`toArray()`将结果转换为JSON，否则查询的结果会显示为MongoDB服务器的状态而非数据库中的数据。

### 将ID保存为变量

在**MongoDB Script**下添加一个**BeanShell PostProcessor**, 编写代码将读取到的ID信息以“ID_number”的形式存为变量。

*BeanShell 脚本:*

```
import com.alibaba.fastjson.*;

String res = prev.getResponseDataAsString();
JSONArray captchaList = JSON.parseArray(res);
int length = captchaList.size();
for(int i=0; i<length; i++){
    String tmp = captchaList.get(i).getString("greeting");
    props.put("ID_"+i, tmp);
}
```

![BeanShell中操作JSON](https://upload-images.jianshu.io/upload_images/3520043-7afc3c09fb2e3918.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到我将ID存入的是`props`(properties)而非`vars`(variables)，原因在于前者可以在线程组间共享。如果存入vars，在执行测试的线程组中就无法读取到ID信息了。

### 计数器

为了能让每个线程读取到的ID不同，需要为每个线程配置唯一的变量与ID参数对应。使用计数器**Counter**就能完成，他会为每个线程配置从“Starting value”开始的唯一数值，正好与ID对应。

*Starting value*：表示从那个数值开始递增，我们从0开始即可；

*Increment*：表示增量，取1；

*Reference Name*：用来获取**Counter**在当前线程中的值，设置为`count`，在线程中调用`${count}`就能获取到真实的counter值：

![配置Counter](https://upload-images.jianshu.io/upload_images/3520043-4b2cce2c3676529e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 测试执行线程组

新建一个线程组，添加**Debug Sampler**和**BeanShell Sampler**，以及**结果察看树**。

> 为了方便查看结果，以及演示，就不发HTTP请求，而改为用BeanShell直接输出读取到的数据。

BeanShell中，使用`${count}`获取计数器的值，再通过`props.get`获取ID的值，然后打印出来：

![获取ID值](https://upload-images.jianshu.io/upload_images/3520043-4d7803f0e931e0c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 最后Test Plan树展示

![Test Plan Tree](https://upload-images.jianshu.io/upload_images/3520043-668a5a34b2927158.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 参考

1. [How to Load Test MongoDB with JMeter]([https://www.blazemeter.com/blog/how-load-test-mongodb-jmeter](https://www.blazemeter.com/blog/how-load-test-mongodb-jmeter)), Dmitri Tikhanski, Apr. 28th, 2015

## 推荐阅读

1. [What is different between props and vars object in JMeter](https://stackoverflow.com/questions/38845168/what-is-different-between-props-and-vars-object-in-jmeter), Stack Overflow