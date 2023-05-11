---
title: Docker的DooD方式调研
date: 2022-07-21 21:14:34 +0800
categories: [Docker]
tags: [research,docker] 
comments: true
pin: true

---


## 需求

工作中遇到一个需求，通过一个docker里的服务触发一个机器学习训练任务，训练相关的程序需要在另一个docker容器里进行，这些容器事先都已经准备好。鉴于k8s平台尚未完成，于是调研了一下用docker-out-of-docker (DooD)实现的方式。

## DooD and DinD
DooD方式本身原理很简单，将host的docker api的socket挂载到container内，就可以从container内部触发host的docker daemon去执行一个docker命令以启动、控制另一个container。使用命令：

```bash
docker run -it -v /var/run/docker.sock:/var/run/docker.sock DOCKER_IMAGE
```
![img]({{ site.url }}/assets/img/source/dood.png)


相对于DooD，还有另一种方式：DinD[^ref-1] （Docker-in-Docker），就是在一个docker容器内再装一个docker，不过看了下就放弃了，容器里面套容器，属实有点想不开了。对这一方式有一些反对意见，来自《Using Docker-in-Docker for your CI or testing environment? Think twice.》[^ref-2]:


## DooD的优劣

DooD的优劣点，以下内容来自《Secure Docker-in-Docker with System Containers》[^ref-3]:

> This approach has some benefits but also important drawbacks.
>
> One key benefit is that it bypasses the complexities of running the Docker daemon inside a container and does not require an unsecure privileged container.
> 
> It also avoids having multiple Docker image caches in the system (since there is only one Docker daemon on the host), which may be good if your system is constrained on storage space.
> 
> But it has important drawbacks too.
> 
> The main drawback is that it results in poor context isolation because the Docker CLI runs within a different context than the Docker daemon. The former runs within the container’s context; the latter runs within host’s context. This leads to problems such as:
> 
> Container naming collisions: if the container running the Docker CLI creates a container named some_cont, the creation will fail if some_cont already exists on the host. Avoiding such naming collisions may not always trivial depending on the use case.
> 
> Mount paths: if the container running the Docker CLI creates a container with a bind mount, the mount path must be relative to the host (as otherwise the host Docker daemon on the host won’t be able to perform the mount correctly).
> 
> Port mappings: if the container running the Docker CLI creates a container with a port mapping, the port mapping occurs at the host level, potentially colliding with other port mappings.
> 
> This approach is also not a good idea if the containerized Docker is orchestrated by Kubernetes. In this case, any containers created by the containerized Docker CLI will not be encapsulated within the associated Kubernetes pod, and will thus be outside of Kubernetes’ visibility and control.
> 
> Finally, there are security concerns too: the container running the Docker CLI can manipulate any containers running on the host. It can remove containers created by other entities on the host, or even create unsecure privileged containers putting the host at risk.
> 
> Depending on your use case and environment, these drawbacks may void use of this approach.
> 
> For more info on these issues, check these excellent articles from Applatix (now Intuit) on why DooD was not a good fit for them.



---
Refrence


[^ref-1]: [Run docker inside a docker container?](https://stackoverflow.com/questions/26239116/run-docker-inside-a-docker-container)

[^ref-2]: [Using Docker-in-Docker for your CI or testing environment? Think twice.](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/)

[^ref-3]: [Secure Docker-in-Docker with System Container](https://blog.nestybox.com/2019/09/14/dind.html)