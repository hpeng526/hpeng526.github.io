---
title: Docker 体验
date: 2016-05-29 20:18:38
tags: [Docker, Flask, uwsgi, Python, Redis, nsenter]
---

周末了，很早就想体验一下 Docker ，所以，就把很久以前用 Python + Flask 写的小项目用 Docker 部署一下。项目还用到了 Redis 所以，这次就简单的把所有东西揉在一个 Docker image 里好了。

### Docker-flask

在 docker Hub 上搜了一下，发现有简单的 docker-flask 集成了，所以，基础的配置也就参考了 [p0bailey/docker-flask](https://github.com/p0bailey/docker-flask) 

<!-- more -->

### Dockerfile

因为项目用到 Redis ，同时，redis-server 也是部署在同一个 image 里，// 最好分开单独的 redis-server，便于扩展与维护，所以在原有的 Dockerfile 里面要增加 redis-server 的安装，跟 python redis 的依赖

``` Dockerfile
FROM ubuntu:14.04

MAINTAINER Hp <hpeng526@gmail.com>

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update
RUN apt-get -y install nginx  sed python-pip python-dev uwsgi-plugin-python supervisor redis-server

RUN mkdir -p /var/log/nginx/app
RUN mkdir -p /var/log/uwsgi/app/


RUN rm /etc/nginx/sites-enabled/default
COPY flask.conf /etc/nginx/sites-available/
RUN ln -s /etc/nginx/sites-available/flask.conf /etc/nginx/sites-enabled/flask.conf
COPY uwsgi.ini /var/www/app/
RUN echo "daemon off;" >> /etc/nginx/nginx.conf


RUN mkdir -p /var/log/supervisor
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

copy app /var/www/app
RUN pip install -r /var/www/app/requirements.txt

#EXPOSE 80
CMD ["/usr/bin/supervisord"]
```

python 的依赖在 requirements.txt ，增加 redis 就可以了

``` requirements
uwsgi
flask
redis
```

看 Dockerfile 可以知道，Container 启动的时候执行的命令是 supervisord，配置文件为目录下的文件，至于 supervisor 是什么，就去 [官网](http://supervisord.org/) 看下吧。被supervisor 监控的后台进程有 nginx，uwsgi，再把我们需要的redis-server 加进去，配置我采用最简单的默认配置了。这样，我们就完成了 redis-server 的配置了。

``` supervisord.conf
[supervisord]
nodaemon=true

[program:nginx]
command=/usr/sbin/nginx

[program:uwsgi]
command =/usr/local/bin/uwsgi --catch-exceptions --ini  /var/www/app/uwsgi.ini

[program:redis]
command =/usr/bin/redis-server
```

uwsgi倒不用修改多少东西，把 module 改成自己的文件就行了

``` uwsgi.ini
[uwsgi]
#application's base folder
base = /var/www/app
#python module to import
module = overtime
#the variable that holds a flask application inside the module imported at line #6
callable = app
#socket file's location
socket = /var/www/app/uwsgi.sock
#permissions for the socket file
chmod-socket    = 666
#Log directory
logto = /var/log/uwsgi/app/app.log

chdir = /var/www/app

post-buffering = 8192

post-buffering-bufsize = 65536
```

### build 跟 run

在有 Dockerfile 目录下 build 

``` sh
docker build -t hpeng526/flaskredis .

```

build 完成之后，只要暴露 80 端口出来就行了

``` sh
docker run -d -p 80:80 hpeng526/flaskredis
```


### 谈谈踩到的坑

以前写flask项目都没试过自己部署，这次是第一次用 uwsgi 部署项目，遇到了几个坑。

* 明明开 debug = True 但是就是不出 debug ，抛的错误都被吃了，日志也没有，本地调试是正常的
* docker build 总是慢

#### 解决

* 特喵的，uwsgi 不是调用 app.run()的，是调用 app() 所以把run处理的事，扔到外面就好了。[flask-debug-true-does-not-work-when-going-through-uwsgi](http://stackoverflow.com/questions/10364854/flask-debug-true-does-not-work-when-going-through-uwsgi) 看 stackoverflow 最后一个答案，但是年少无知的我心中千万只草泥马在奔腾，程序改改就跑通了。
* 一开始我就接入了 daocloud 的加速服务，但是 docker build 是不走那边的=.=，觉得 docker 有些会用cache里面的，但是到了 pip install 的时候就不走 cache 了，然后由于各种原因，网络不同的时候就各种。。。

### 特殊技巧

在调试的时候，没有把log分离出来，没有挂在vol，这是个极大的失误(小程序就不折腾了)，所以，问题就来了，怎么才能愉快地查看 container 里面的各种东西呢？

* nsenter，进入没有开启sshd服务的容器里，首先要拿到pid

``` sh
docker ps -a # 查看 docker container
docker inspect --format "{{.State.Pid}}" CONTAINER ID # 获取pid
nsenter --target $PID --mount --uts --ipc --net --pid
```

这样就ok了

源码 [docker-flaskredis](https://github.com/hpeng526/docker-flaskredis)