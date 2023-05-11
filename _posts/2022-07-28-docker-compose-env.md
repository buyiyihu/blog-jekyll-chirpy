---
title: Docker compose环境变量的优先级顺序
date: 2022-07-28 21:14:34 +0800
categories: [Docker]
tags: [docker]
comments: true
---

最近写docker-compose file时需要处理众多的环境变量，这些变量特性不一处理方式各异，特地去[官方文档](https://docs.docker.com/compose/environment-variables/#set-environment-variables-with-docker-compose-run)研究了一下docker-compose对环境变量的处理。



官方介绍的优先级顺序：

1. Compose file
2. Shell environment variables
3. Environment file
4. Dockerfile
5. Variable is not defined


根据实际测试得到：

1. compose文件中的environment项

2. compose文件中的env_file项

3. 用于compose的.env文件


有趣而特别需要注意的地方在于：

用于compose的.env文件 > compose文件中的默认值  > compose文件中的env_file项

