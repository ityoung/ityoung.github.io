---
title:        Docker - 相似的命令
date:         2018-05-30
tags:
    - Docker
categories:     Docker
---

## kill vs stop

两个命令都是停止docker，不同之处在于：

- `docker stop`: 先发`SIGTERM`信号给docker，允许其在一定时间（默认10s）内进行一些操作（例如资源回收），若这段时间内容器未停止，则发送`SIGKILL`信号强行杀死容器；
- `docker kill`: 直接发送`SIGKILL`信号杀死容器。

`SIGTERM`与`SIGKILL`的区别在于，前者是告知你的租期到了，请你赶紧收拾行李离开；后者是你的租期到了，直接将你扫地出门。

简而言之，相比kill，stop给了容器自行处理结束的时间，**更为优雅**。

## run vs start

- `docker run`: 从镜像启动一个容器;
- `docker start`: 运行已停止的容器。

<!--more-->

例如你需要从`docker images`列表中启动一个镜像，你需要执行

```
docker run <IMAGE ID>
```

你在该镜像中做了某些修改，然后结束（kill/stop）该容器，通过`docker ps -a`可以看到一个状态为*EXITED*的容器，此时你可以通过执行

```
docker start <CONTAINER ID>
```

再次启动该容器。通过start启动的容器会保留上一次结束前做的变动，不会像run一样执行的是全新的容器。

## export vs save

- `docker export`: 持久化**容器**，类似给一个虚拟机打快照，会保留容器启动后的一系列修改；
- `docker save`: 持久化一个**镜像**，只是将镜像导出，如果通过镜像启动容器，那么容器中的修改并不会写入镜像中，也因此不会被save。

要注意的是，export导出后的镜像会丢失历史，等于没法回滚到以前的某个状态，而save则保留全部的历史；也因此，export导出后的镜像更小。

另外，export一次只能导出一个容器，而save可以一次性导出多个镜像。

实际使用中，仅仅导出镜像使用save，而如果对容器做了修改后需要保存，可以export，也可以先commit再save。

## import vs load

- `docker import`: 从一个tar包创建一个镜像，可以指定新的NAME:TAG；
- `docker load`: 可以同时将多个tar包导入到仓库中，不可指定新的NAME:TAG。

二者几乎与export/save相呼应，若将save保存的tar包镜像用import导入回去，那么新生成的镜像会丢失原来镜像的历史。

## rm vs rmi

- `docker rm <CONTAINER ID>`: 删除容器
- `docker rmi <IMAGE ID>`: 删除镜像

## image vs container

如果你是一个刚接触Docker的新手，那么你可能会对这两者有疑惑。

其实很容易区分二者，`image`即镜像，类似操作系统的iso文件，而`container`则是用这iso文件安装好的一个操作系统。

## 参考

[1] [Use the Docker command line](https://docs.docker.com/engine/reference/commandline/cli/), Docker Documentation
[2] [what's the difference between docker stop and docker kill?](https://superuser.com/questions/756999/whats-the-difference-between-docker-stop-and-docker-kill), superuse, Steffen Opel
[3] [What is the difference between save and export in Docker?](https://stackoverflow.com/questions/22655867/what-is-the-difference-between-save-and-export-in-docker), Stack Overflow, mbarthelemy
[4] [Difference between save and export](https://tuhrig.de/flatten-a-docker-container-or-image), Thomas
[5] [What is the difference between import and load in Docker?](https://stackoverflow.com/questions/36925261/what-is-the-difference-between-import-and-load-in-docker), Stack Overflow, VonC
