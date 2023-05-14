---
title: gunicorn 文档学习笔记与配置实践
date: 2021-08-26 23:55:35 +0800
categories: [Webservice, 阅读笔记]
tags: [deploy, 阅读笔记] 
toc: true
comments: true
math: true
mermaid: true

---

### 1.安装

worker【处理请求的app实例在此处称为worker】还可以使用 eventlet 或 gevent

安装方法
```
$ pip install greenlet            # Required for both
$ pip install eventlet            # For eventlet workers
$ pip install gunicorn[eventlet]  # Or, using extra
$ pip install gevent              # For gevent workers
$ pip install gunicorn[gevent]    # Or, using extra
```

greenlet 都需要，如果安装失败，需要装 Python headers，apt-get install python-dev（ubuntu）
gevent 还需要 libevent（注意版本）

如果有多个gunicorn 实例，想给每个process起个名字，还要安装 setproctitle，示例 ```gunicorn[setproctitle]```


### 2.启动
```
$ gunicorn [OPTIONS] $(MODULE_NAME):$(VARIABLE_NAME)
```

命令行可以传参数，但建议使用 环境变量 详见设置


#### 2.1 Commonly Used Arguments

```bash
-c CONFIG, --config=CONFIG - Specify a config file in the form $(PATH), file:$(PATH), or python:$(MODULE_NAME).

-b BIND, --bind=BIND - Specify a server socket to bind. Server sockets can be any of $(HOST), $(HOST):$(PORT), fd://$(FD), or unix:$(PATH). An IP is a valid $(HOST).

-w WORKERS, --workers=WORKERS - The number of worker processes. This number should generally be between 2-4 workers per core in the server. Check the FAQ for ideas on tuning this parameter.

-k WORKERCLASS, --worker-class=WORKERCLASS - The type of worker process to run. You’ll definitely want to read the production page for the implications of this parameter. You can set this to $(NAME) where $(NAME) is one of sync, eventlet, gevent, tornado, gthread. sync is the default. See the worker_class documentation for more information.

-n APP_NAME, --name=APP_NAME - If setproctitle is installed you can adjust the name of Gunicorn process as they appear in the process system table (which affects tools like ps and top).
```


### 3.配置


#### 3.1配置优先级 

Framework Settings < Configuration File < Command Line

#### 3.2 检测配置
```
$ gunicorn --check-config APP_MODULE
```
#### 3.3 命令行

只要Command Line能写的（有的不能写），会覆盖一切
```
$ gunicorn -h
```
查看

#### 3.4 配置文件

不必在模块路径内 module path (sys.path, PYTHONPATH)
一条条列出来即可
```
import multiprocessing

bind = "127.0.0.1:8000"
workers = multiprocessing.cpu_count() * 2 + 1
```

#### 3.5 framework 配置

目前只支持 Paster applications 



### 4 设置

有的设置只能写到配置文件里

所有命令行配置 都可以写到 环境变量 GUNICORN_CMD_ARGS 里，如
```
$ GUNICORN_CMD_ARGS="--bind=127.0.0.1 --workers=3" gunicorn app:app
```

#### 4.1 常用配置项


* 配置文件

```
-c CONFIG, --config CONFIG
None (默认值)
        A string of the form PATH, file:PATH, or python:MODULE_NAME.

```

**【Debugging】**

* reload

```
  --reload False
```

改代码时自动重启workers

* reload_engine
```
        * --reload-engine STRING
        * auto

        Valid engines are:

            ‘auto’
            ‘poll’
            ‘inotify’ (requires inotify)
```


* reload_extra_files

* spew

* check_config


**【logging】**


* accesslog
```
        * --access-logfile FILE
        * None
        '-' means log to stdout.
```


* disable_redirect_access_to_syslog

* access_log_format

* errorlog

* loglevel

* capture_output

* logger_class

* logconfig

* logconfig_dict




**【Process Naming】**


* proc_name
```
        * -n STRING, --name STRING
        * None
```

* default_proc_name
        * gunicorn


**【Security】**


* limit_request_line 抗DDOS

* limit_request_fields

* limit_request_field_size 抗DDOS




**【Server Hooks】**


**【Server Mechanics】**


* daemon

        * -D, --daemon
        * False



* raw_env

        * -e ENV, --env ENV
        * []



* pidfile


        * -p FILE, --pid FILE
        * None


* worker_tmp_dir





**【Server Socket】**


* bind

        * -b ADDRESS, --bind ADDRESS
        * ['127.0.0.1:8000']



* backlog


        * --backlog INT
        * 2048




**【Worker Processes】**

* workers

```
The number of worker processes for handling requests.
* -w INT, --workers INT
* 1
```

*By default, the value of the WEB_CONCURRENCY environment variable. If it is not defined, the default is 1.*


* worker_class

* threads

会强制切换 worker_class 到 gthread 模式
详见设计

* worker_connections

* max_requests
```
        * --max-requests INT
        * 0
```


*The maximum number of requests a worker will process before restarting.*

*Any value greater than zero will limit the number of requests a worker will process before automatically restarting. This is a simple method to help limit the damage of memory leaks.
If this is set to zero (the default) then the automatic worker restarts are disabled.*

* max_requests_jitter
```
* --max-requests-jitter INT
* 0
```
The maximum jitter to add to the max_requests setting.

The jitter causes the restart per worker to be randomized by randint(0, max_requests_jitter). This is intended to stagger worker restarts to avoid all workers restarting at the same time.


* timeout
```
        * -t INT, --timeout INT
        * 30
```
Workers silent for more than this many seconds are killed and restarted.

Generally set to thirty seconds. Only set this noticeably higher if you’re sure of the repercussions for sync workers. For the non sync workers it just means that the worker process is still communicating and is not tied to the length of time required to handle a single request.


* graceful_timeout
        * --graceful-timeout INT
        * 30
Timeout for graceful workers restart.

After receiving a restart signal, workers have this much time to finish serving requests. Workers still alive after the timeout (starting from the receipt of the restart signal) are force killed.

* keepalive





### 5.仪表盘

     现已接入 statusD ，可通过UDP协议，监控各进程，并且不会阻塞 gunicorn 服务！！！



### 6.部署


*     推荐使用 nginx
*     把gunicorn服务放到监控设施里启动以达到监控作用
*     日志轮转



### 7.信号处理

     不同信号以使gunicorn 改变状态


### 8.自定义应用

### 9.设计








