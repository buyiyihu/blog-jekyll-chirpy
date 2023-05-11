---
title: 遇到的docker相关小问题点列表
date: 2021-10-13 22:15:44 +0800
categories: [Docker, curated-list]
tags: [curated-list, docker] 
comments: true

---


## 1.默认shell不一致引发环境变量差异导致的docker容器测试和运行不一致

开发调试同事交付的某个docker容器时遇到一个问题：

使用以下命令
```docker
docker run --rm -it <container_name> bash
```
调试时，一切正常。但真正跑起来时却报错：找不到某命令，或缺包

找了半天发现，运行时报错是因为缺少一个环境变量，而负责打镜像同事把这个环境变量写在了`~/.bashrc`里，而很多Alpine的docker image里使用的shell是ash，所以当默认使用bash进入容易调试，就正常，而实际运行时就会出现问题。


看来把环境变量写到镜像中的`~/.bashrc`是一个十分不合适的开发习惯。

