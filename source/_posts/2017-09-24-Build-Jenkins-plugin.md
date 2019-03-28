---
title:        Jenkins插件编译
date:         2017-09-24
tags:
    - Jenkins
    - 持续集成
categories:     持续集成
---

## 环境说明
* 系统: Deepin 15.04
* Java环境: [JDK 8](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
_Java环境的搭建与配置请百度。_
* 其他软件: Git
---
## maven环境搭建
### maven安装与配置
* 下载maven: [apache-maven](http://maven.apache.org/download.cgi) 此处以我下载的[apache-maven-3.5.0-bin.tar.gz](http://mirrors.shuosc.org/apache/maven/maven-3/3.5.0/binaries/apache-maven-3.5.0-bin.tar.gz)为例；
* 解压maven到/usr/local/目录
```
$ sudo tar -zxvf apache-maven-3.5.0-bin.tar.gz -C /usr/local/
```
* 配置环境变量
编辑_~/.profile_，追加如下内容：
```
export M2_HOME=/usr/local/apache-maven-3.5.0
export M2=$M2_HOME/bin
export MAVEN_OPTS="-Xms128m -Xmx1024m"
export PATH=$M2:$PATH
```

<!--more-->

* 配置maven
编辑_/usr/local/apache-maven-3.5.0/conf/settings.xml_，修改内容如下：
```
<?xml version="1.0" encoding="UTF-8"?>

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <pluginGroups>
  </pluginGroups>
  <proxies>
  </proxies>
  <servers>
  </servers>
  <mirrors>
      <mirror>
          <id>alimaven</id>
          <mirrorOf>central</mirrorOf>
          <name>aliyun maven</name>
          <url>http://maven.aliyun.com/nexus/content/repositories/central/</url>
      </mirror>
  </mirrors>
  <profiles>
      <profile>
          <id>jdk-1.8</id>
          <activation> 
              <activeByDefault>true</activeByDefault>
              <jdk>1.8</jdk> 
          </activation>
          <properties> 
              <maven.compiler.source>1.8</maven.compiler.source> 
              <maven.compiler.target>1.8</maven.compiler.target> 
              <maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion> 
          </properties> 
      </profile>
  </profiles>
</settings>
```
* 验证maven配置
输入
```
$ mvn -v
```
出现如下结果即为配置成功
```
Apache Maven 3.5.0 ...
Maven home: ...
Java version: 1.8.0 ...
Java home: ...
Default locale: ...
OS name: ...
```
---
## Jenkins插件源码编辑
>本教程使用[Dingding Notification Plugin](https://github.com/jenkinsci/dingding-notifications-plugin)为例；
>更多hpi教程请移步Jenkins官网。
* 拉取github代码：
```
$ git clone https://github.com/jenkinsci/dingding-notifications-plugin.git
```
* 编辑代码
_略_
---
## 编译与使用hpi(Hudson Plugin)
* 编译hpi
由于“钉钉通知器”插件的代码库中有pom.xml，因此直接运行：
```
mvn package
```
即可生成hpi与jar包。
* 使用hpi
在Jenkins的插件管理中，选择本地上传hpi；
上传后重启Jenkins即可生效。
---
## 参考资料
[1]. 十佳菜鸟,《[deepin安装maven](http://blog.csdn.net/h13140995776/article/details/76652651)》,CSDN,08,2017
[2]. supereagle,《[Jenkins的plugin开发](http://blog.csdn.net/jmyue/article/details/9855449)》,CNBLOG,08,2013
