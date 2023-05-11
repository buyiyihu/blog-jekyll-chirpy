---
title: Docker-out-of-docker的用户组问题 
date: 2022-07-21 21:14:34 +0800
categories: [Docker]
tags: [hack,docker] 
comments: true
pin: true

---



工作中遇到一个需求，通过一个docker里的服务触发一个机器学习训练任务，训练相关的程序需要在另一个docker容器里进行，这些容器事先都已经准备好。此处采用Docker-out-of-Docker的方式，即通过挂载docker API的socket文件，达到从一个容器内部触发主机启动另一个容器的目的。

正常情况下，讲容器内的用户加入主机上的docker组，就可以调用了。

```docker
RUN useradd -u $UID -ms /bin/bash $USER && \
  usermod -aG docker $USER
```

但是实际开发调试中发现，容器内的docker组和host的docker组的group id不一致，总是报权限问题。不知道哪里出了错，看了很多资料也没找到解决方法，时间紧迫，于是想出了一个曲线救国的方式：dockerfile里，在容器里新建一个组，叫dockerr（这是为了容易联想特地取了个类似typo的名字），然后group id和host的group id一致。试了下，完全没问题。

dockerfile的新代码
```docker
RUN useradd -u $UID -ms /bin/bash $USER && \
  usermod -aG docker $USER && \
  groupadd -g $DGID dockerr && \
  usermod -aG dockerr $USER
```

实话说，这个方法让自己感到既尴尬，又有成就感，十分复杂的感受。

后来在stackoverflow上看到了同样问题的提问，题主说自己直接加root运行了解决了。我见没有更好的回答，就把自己的这个方法po上去了。说个插曲，本来以为既然没有更好的方法，我这个问题应该会被采纳吧，结果我回答了后12个小时不到，题主就直接采纳了自己的答案，让我顿感搞笑。

<br>

---

[原回答](https://stackoverflow.com/a/73030787/11135916)


> I met the same problem several days ago and fixed it by adding an another group as below:
>
```docker
RUN useradd -u $UID -ms /bin/bash $USER && \
  usermod -aG docker $USER && \
  groupadd -g $DGID dockerr && \
  usermod -aG dockerr $USER
```
>
> The $DGID here is the gid of group docker at the host.
>
> After installing docker client inside the docker image, a docker group is created, but the gid of this docker group could be different from that at host, while the gid, not the group name is the key.
>
> So I created a new group with the gid of docker group at host and put my new user into this group. With this gid, this user then can access the /var/run/docker.sock.
>
> PS.
>
> I tried to make the docker group inside docker share the same gid with the host, but just couldn't make it.
>
> This did fix my problem, however I'm wondering whether there could be a better solution.

