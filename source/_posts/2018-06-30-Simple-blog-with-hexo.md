---
title: Hexo+Github 搭建个人博客
date: 2018-06-30
tags:
    - 随笔
    - 博客
---

## 环境

Ubuntu 18.04
Git

## nodejs安装

官网下载最新的LTS版本的**tar.xz**包，我下载的是*8.11.3*

```
wget https://nodejs.org/dist/v8.11.3/node-v8.11.3-linux-x64.tar.xz
```

解压

```
xz node-v8.11.3-linux-x64.tar.xz    # 得到tar包
tar -xvf node-v8.11.3-linux-x64.tar    # 解压tar包
```

移动到常用应用目录*/usr/local/lib*

```
sudo mv node-v8.11.3-linux-x64 /usr/local/lib/node-v8.11.3
```

将`node`与`npm`可执行文件软链接到*/usr/local/bin*

```
ln -s /usr/local/lib/node-v8.11.3/bin/node /usr/local/bin/node
ln -s /usr/local/lib/node-v8.11.3/bin/npm /usr/local/bin/npm
```

<!--more-->

验证安装完成

```
ityoung@Home:~$ node -v
v8.11.3
ityoung@Home:~$ npm -v
5.6.0
```

## hexo安装

官方中文文档：https://hexo.io/zh-cn/docs/index.html

```
npm install -g hexo-cli
```

（如果你的node用的`apt`安装的可以跳过下面这步）
进入hexo目录，将`hexo`可执行文件软链接到*/usr/local/bin*

```
cd /usr/local/lib/node-v8.11.3/lib/node_modules/hexo-cli
ln -s $(pwd)/bin/hexo /usr/local/bin/hexo
```

## 创建项目

新建一个文件夹用于存放项目相关的文件

```
mkdir first
```

初始化项目

```
hexo init first
```

启动服务就能查看当前的模版了

```
hexo server
```

## 配置主题

选择人气较高的`next`主题

### 下载主题

clone主题到项目的themes文件夹下

```
git clone https://github.com/iissnan/hexo-theme-next themes/next
```

### 启用主题

打开项目根目录的*_config.yml*， 找到 `theme` 字段，并将其值更改为 `next`。

```
theme: next
```

再次启动server验证主题正常启用

## 创建新页面

### 标签页

新建一个页面

```
hexo new page “tags”
```

编辑刚新建的页面，将页面的类型设置为 tags ，主题将自动为这个页面显示标签云。页面内容如下：

```
title: 标签
date: 2014-12-22 12:39:04
type: "tags"
---
```

在菜单中添加链接。编辑主题的*_config.yml* ， 添加 tags 到 menu 中，如下:

```
menu:
  home: /
  archives: /archives
  tags: /tags
```

## 发布到github

参照官方教程： https://hexo.io/zh-cn/docs/deployment.html

## 参考

https://hexo.io/zh-cn/docs/index.html
http://theme-next.iissnan.com/getting-started.html