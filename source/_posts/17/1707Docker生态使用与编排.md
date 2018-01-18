---
title: Docker生态使用与编排
date: 2017-07-25 23:18:47
tags: [Docker, Docker Swarm]
---

# Docker生态使用与编排

## Docker 介绍

Docker 是一种容器技术，强依赖linux内核

### 容器（Container）
有时候也被称为操作系统级虚拟化，以区别传统的Hypervisor虚拟技术。它不对硬件进行模拟，只是作为普通进程运行于宿主机的内核之上。

容器运行的一般都是一个简单的Linux系统，有root用户权限，进程id，及网络等属性。

<!-- more -->

## Container vs. VM

### Virtual Machine

```             
             +----------+
             |  VM      |
             |          |
+----------+ +----------+ +----------+
|  App A   | |  App B   | |  App C   |
+----------+ +----------+ +----------+
+----------+ +----------+ +----------+
| Bins/Libs| | Bins/Libs| | Bins/Libs|
+----------+ +----------+ +----------+
+----------+ +----------+ +----------+
| Guest OS | | Guest OS | | Guest OS |
+----------+ +----------+ +----------+
+------------------------------------+
|                                    |
|          Hypervisor                |
|                                    |
+------------------------------------+
+------------------------------------+
|                                    |
|          Infrastructure            |
|                                    |
+------------------------------------+
```

### Docker

```             
             +----------+
             | Container|
             |          |
+----------+ +----------+ +----------+
|  App A   | |  App B   | |  App C   |
+----------+ +----------+ +----------+
+----------+ +----------+ +----------+
| Bins/Libs| | Bins/Libs| | Bins/Libs|
+----------+ +----------+ +----------+
+------------------------------------+
|             Docker                 |
+------------------------------------+
+------------------------------------+
|                                    |
|             Host OS                |
|                                    |
+------------------------------------+
+------------------------------------+
|                                    |
|          Infrastructure            |
|                                    |
+------------------------------------+
```

容器是共享单个内核，容器镜像中包含唯一的信息就是可执行文件以及包的依赖项，而这些依赖永远不需要安装在宿主上。这些进程跟本机进程一样运行，并且可以单独的管理他们。这就决定了Docker一个特性`runs anywhere`，因为没有了奇怪复杂的配置项目（依赖已经被容器携带了），一个容器化的程序可以很方便的运行在任何地方。

## 基本概念

- 镜像（Image）
- 容器（Container）
- 仓库（Repository）

### Docker Image

Docker 镜像是一个特殊的文件系统，除了包含容器运行时所需的库，资源，配置等文件外，还包含了一些为运时准备的配置参数，镜像不包含任何动态数据，其内容在构建后也不会改变。

一个重要的概念就是<b>分层存储</b>，Docker 充分利用 [Union FS](https://en.wikipedia.org/wiki/Union_mount)技术，将其设计成分层存储的架构。

镜像构建时，会一层层的构建，上一层是下一层的基础，没一层构建完就不会改变。这个特性就使得，镜像的定制与复用变得非常容易。

通常 Union FS 有两个用途, 一方面可以实现不借助 LVM、RAID 将多个 disk 挂到同一个目录下,另一个更常用的就是将一个只读的分支和一个可写的分支联合在一起，Live CD 正是基于此方法可以允许在镜像不变的基础上允许用户在其上进行一些写操作。 Docker 在 AUFS 上构建的容器也是利用了类似的原理。(注：CentOS/RHEL 用户并没有UnionFS使用)

### Docker Container

可以这么理解，一个`Container`就是运行中的`Image`实例。容器可以被创建，启动，停止，删除，暂停等。

容器运行时，是以镜像为基础，它会在创建一个当前容器的存储层，当容器消失，容器的存储也会消失（例如你把container删除了）。所以，任何在保存在容器里的内容都会由于容器的删除而丢失。

因此，容器不应该写入需要保留的数据，所有文件的写入操作应该使用`Volume`，或者绑定宿主目录（但是，万事不是绝对的，京东就把MySQL数据库Docker化了 [京东MySQL数据库Docker化最佳实践](http://www.dockerinfo.net/4391.html)）

### Docker Repository

顾名思义，就是个放镜像的地方。Docker仓库负责存储，分发镜像服务。

## 为什么要用Docker

- 更高效利用系统资源
- 更快启动时间（对比与虚拟化技术）
- 一致的运行环境（这个就相当爽了，环境问题都不是问题）
- 持续交付与部署
- 更轻松的迁移（废话）
- 更轻松的维护和扩展


## 从一个 Dockerfile 入手

```
# 基础镜像
FROM java:8

# 环境变量 Rocketmq version
ENV ROCKETMQ_VERSION 4.0.0-incubating

# Rocketmq home
ENV ROCKETMQ_HOME  /opt/rocketmq-${ROCKETMQ_VERSION}

# working dir 指定工作目录，如果不存在，则创建该目录
WORKDIR  ${ROCKETMQ_HOME}

# Run 执行命令
RUN mkdir -p \
		/opt/logs \
	    /opt/store \
	/opt/conf

RUN curl https://dist.apache.org/repos/dist/release/incubator/rocketmq/${ROCKETMQ_VERSION}/rocketmq-all-${ROCKETMQ_VERSION}-bin-release.zip -o rocketmq.zip \
          && unzip rocketmq.zip \
          && mv apache-rocketmq-all/* . \
          && rmdir apache-rocketmq-all  \
          && rm rocketmq.zip

COPY runbroker.sh bin/runbroker.sh

COPY broker* bin/

RUN chmod +x bin/mqbroker

CMD cd ${ROCKETMQ_HOME}/bin && export JAVA_OPT=" -Duser.home=/opt"


VOLUME /opt/logs \
		/opt/store \
			/opt/conf

ENTRYPOINT ["/opt/rocketmq-4.0.0-incubating/bin/mqbroker"]
```


## Docker 编排与调度

### 为什么需要编排与调度？

如果你应用也就那么一个节点，并不复杂。当然也就不需要编排与调度了。当应用被扩展到多主机，多节点，如果还是人工管理这些节点，这些集群，则会变得很无力与很蠢。

### Docker swarm

Docker的Swarm是一个2014年12月发布的调度器。它旨在提供一个健壮的调度器，采用Docker原生句法使得可以在宿主机上启动容器和进行供应。

作为容器集群管理器，Swarm 最大的优势之一就是 100% 支持标准的 Docker API。各种基于标准 API 的工具比如 Compose 各种管理软件，甚至 Docker 本身等都可以很容易的与 Swarm 进行集成。

#### swarm mode

在 docker 1.12 以上，Docker Engine 已经集成了 swarm mode，这个特性可以让 Docker Engine 更容易管理多host与多container（不需要额外的 etcd 或者 zk 或者 consul），当然传统模式还是用上较好，则只是一个可选的特性。

官方也更推荐用分布式的kv做为发现服务
- Consul 0.5.1+
- Etcd 2.0+
- ZooKeeper 3.4.5+

这个特性安装与加入十分简单，他有一个swarm init命令

```sh
docker swarm init --advertise-addr 192.168.77.131
```

他默认是监听`0.0.0.0:2377`作为通讯端口
可以看一下

```sh
root@ubuntu:/home/hp# lsof -i TCP:2377
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
dockerd 1476 root   16u  IPv6  20684      0t0  TCP *:2377 (LISTEN)
dockerd 1476 root   22u  IPv4  20703      0t0  TCP bogon:42632->bogon:2377 (ESTABLISHED)
dockerd 1476 root   23u  IPv6  20704      0t0  TCP bogon:2377->bogon:42632 (ESTABLISHED)
dockerd 1476 root   24u  IPv6  20706      0t0  TCP bogon:2377->bogon:48716 (ESTABLISHED)
dockerd 1476 root   27u  IPv6  22112      0t0  TCP bogon:2377->bogon:34842 (ESTABLISHED)
```

这样就创建了manager节点了。manager节点可以集群化的，后面再说

node 节点的加入就更简单了

```sh
docker swarm join --token SWMTKN 192.168.77.131:2377
```

这里的token是你的 worker token

我们看一下命令

```sh
root@ubuntu:/home/hp# docker swarm join-token -q
"docker swarm join-token" requires exactly 1 argument(s).
See 'docker swarm join-token --help'.

Usage:  docker swarm join-token [OPTIONS] (worker|manager)

Manage join tokens
```

根据列出来的token类型，你加入集群的 Docker 节点的角色就决定了，非常简单。

把节点都加上去了，这样一个简单的swarm集群就完成了

我们看一下节点信息

```sh
root@ubuntu:/home/hp# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
5z6isoxa3albvz3bs8oxrgavq     ubuntu              Ready               Active              
guhnaiojhv8ipwqgglp62266h *   ubuntu              Ready               Active              Leader
rgi3io84yji23r6oc7r130a4d     ubuntu              Ready               Active              

```

#### swarm web-ui （portainer）

```sh
docker service create \
--name portainer \
--publish 9000:9000 \
--constraint 'node.role == manager' \
--mount type=bind,src=//var/run/docker.sock,dst=/var/run/docker.sock \
portainer/portainer \
-H unix:///var/run/docker.sock
```

找了一个google排第一的 [portainer](https://portainer.io/) ，页面是这样的

![portainer-img](/upload/2017/swarm_portainer.jpg)

则就很方便了可以在web上管理我们的集群了。基本的信息都有了。

#### swarm 负载均衡

swarm manager 使用 ingress load balancing 来暴露你想从外部访问的服务，外部组件可以访问集群中任意节点的端口来访问服务，无论改节点是否正在运行此服务，例如上面的 portainer ，你可以访问集群中任意ip:9000，就可以访问 portainer 服务了。为什么呢，因为他在启动的时候用了一个参数 publish，这就把服务发布在PublishedPort上。
Swarm mode 有一个内部的 DNS 组件，为集群的每一个服务分配 DNS 条目，swarm manager使用内部的负载均衡，根据服务的DNS名称在集群内部服务分配请求。

![ingress-routing-mesh](/upload/2017/ingress-routing-mesh.png)

#### swarm 网络

swarm 中有三种重要的网络概念

- Overlay networks 管理swarm集群里面各 Docker daemon 之间的通信. 你可以创建 overlay 网络, 方法跟在独立容器中创建用户自定网络相同, 你也可以将服务添加到一个或者多个现有的覆盖玩过, 以实现服务间通信. Overlay networks 就是使用 overlay 网络驱动的 Docker networks.

- ingress network 是一个特殊的 Overlay network, 用于服务节点间的负载均衡. 当集群节点在已经发布的端口上接收到请求时, 他负责把请求以前给被调用模块的`IPVS`. `IPVS`跟踪参与该服务的所有ip地址, 并选择其中一个, 通过`ingress`路由请求.

- docker_gwbridge 是一个桥接网络用来连接 overlay networks 到单独的 Docker daemon 的物理网络中.

>IPVS
>IPVS（IP虚拟服务器）在 Linux 内核中执行传输层的负载均衡，因此被叫做 “Layer-4转换”。它是一个被整合进 Linux 内核的负载均衡模块，基于 Netfllter（网络过滤器）。它支持 TCP、SCTP 和 UDP、v4 和 v7。IPVS 在真正的服务器集群之前先在主机上工作，作为负载均衡器。它能把对服务（基于 TCP/UDP）的需求引导到真正的服务器上去，让真正服务器的服务以虚拟服务的方式呈现在单个 IP 地址上。

#### 利用 swarm 进行编排

我们利用一个获取hostname的小程序来测试一下swarm的负载均衡

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
)

func getHost(w http.ResponseWriter, r *http.Request) {
	host, err := os.Hostname()
	if err != nil {
		fmt.Printf("%s\n", err)
	} else {
		fmt.Printf("%s\n", host)
	}
	w.Write([]byte(host))
	return
}

func main() {
	http.HandleFunc("/", getHost)
	err := http.ListenAndServe("0.0.0.0:8888", nil)
	if err != nil {
		log.Fatal("ListenAndServe: ", err)
	}
}

```

这次省事, 直接用二进制文件了
Dockerfile
```
FROM scratch
ADD hostname /
CMD ["/hostname"]
```

此处省略私有库搭建与上传和使用

创建服务

`docker service create --replicas 3 --name hostname --publish 8888:8888 '192.168.52.128:5000/hostname'`

```
root@ubuntu1:/home/hp# docker service ps hostname
ID                  NAME                IMAGE                                 NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
uizw9zjf4qnw        hostname.1          192.168.52.128:5000/hostname:latest   ubuntu3             Running             Running 46 seconds ago                       
q59cl3gl1x29        hostname.2          192.168.52.128:5000/hostname:latest   ubuntu2             Running             Running 47 seconds ago                       
y615gsl4tz77        hostname.3          192.168.52.128:5000/hostname:latest   ubuntu1             Running             Running 47 seconds ago                       
```

可以看到, hostname 落在了三个节点上.

我们访问8888端口
```
root@ubuntu1:/home/hp# curl 192.168.52.128:8888
191cdd3fc152
root@ubuntu1:/home/hp# curl 192.168.52.128:8888
e1584585aa8c
root@ubuntu1:/home/hp# curl 192.168.52.128:8888
b37513c9b9b0
root@ubuntu1:/home/hp# curl 192.168.52.128:8888
191cdd3fc152
root@ubuntu1:/home/hp# curl 192.168.52.128:8888
e1584585aa8c
root@ubuntu1:/home/hp# curl 192.168.52.128:8888
b37513c9b9b0
root@ubuntu1:/home/hp# curl 192.168.52.128:8888
191cdd3fc152
root@ubuntu1:/home/hp# curl 192.168.52.128:8888
e1584585aa8c
root@ubuntu1:/home/hp# curl 192.168.52.128:8888
b37513c9b9b0
```

可以看到, 请求是分散在每个节点上的, 因为服务内分发请求到容器的默认方法是`顺序轮询`

如果此时, 有一台物理机故障了怎么办.我们简单粗暴的直接把Ubuntu3关机.

```
root@ubuntu1:/home/hp# docker service ps hostname
ID                  NAME                IMAGE                                 NODE                DESIRED STATE       CURRENT STATE                    ERROR               PORTS
ovbuip43u760        hostname.1          192.168.52.128:5000/hostname:latest   ubuntu2             Running             Running less than a second ago                       
uizw9zjf4qnw         \_ hostname.1      192.168.52.128:5000/hostname:latest   ubuntu3             Shutdown            Running 20 seconds ago                               
q59cl3gl1x29        hostname.2          192.168.52.128:5000/hostname:latest   ubuntu2             Running             Running 5 minutes ago                                
y615gsl4tz77        hostname.3          192.168.52.128:5000/hostname:latest   ubuntu1             Running             Running 5 minutes ago                           
```

可以看到. swarm 帮我们把服务挂到了节点Ubuntu2上.这个过程都是全自动的.

swarm 还支持 [滚动更新](https://docs.docker.com/engine/swarm/swarm-tutorial/rolling-update/) . 这个特性就不在这里演示了. 滚动更新主要的流程就是:
- 停止第一个任务
- 对停止的任务进行更新
- 启动 container (已经更新的任务)
- 如果任务返回状态是`RUNNING`,则等待指定的时间(可以配置), 然后进行下一个更新任务
- 如果在更新期间的任意时刻, 任务返回`FAILED`, 则停止更新.

### 实战 (actuator+jolokia+telegraf+influxdb+grafana)

[简介](/2017/07/25/mbean信息采集系统/)

#### 1. 创建一个自定义 overlay 网络

```sh
docker network create -d overlay influx
```

创建 overlay 网络的一个目的就是, 这个采集系统不需要与其他 Container "交流", 采集系统是可以完全独立的一个存在, 所以对他进行网络隔离会好一点.


#### 2. 启动 influxdb 服务

```sh
docker service create --replicas 1 --name influxdb --network influx --publish 8086:8086 '192.168.52.128:5000/influxdb'
```

#### 3. 启动 telegraf 采集

创建telegraf的配置文件, 因为 swarm 是集群模式, 如果用docker本身的挂载主机目录的方法, 则它就无法部署在其他节点上, 所以, 我们可以通过挂载配置文件的方式进行这类应用的部署.

```sh
docker config create telegraf telegraf.cnf
```

启动 telegraf 服务

```sh
docker service create --replicas 1 --name telegraf --network influx --config src=telegraf,target=/etc/telegraf/telegraf.conf 192.168.52.128:5000/ydhtelegraf
```

#### 4. 启动 grafana 服务

```sh
docker service create --replicas 1 --name grafana --network influx --publish 3000:3000 '192.168.52.128:5000/grafana'
```

做一些基础的设置

![grafana_datasource](/upload/2017/grafana_datasource.png)

  这里有要讲解的地方
  - 在 docker 里, 因为是在同一个network下了, host可以通过 name 来直接访问, 因此, influxdb 的地址就是 http://influxdb:8086 , docker 会自动帮你处理映射问题. (在以前的版本有个 --link 的参数, 这是一个非常坑的东西, 后面被废弃了)
  - 可以看到, 我特定把配置 access 解释放出来, 就是为了说明这件事.
  - 当然, 在 telegraf 的采集配置文件下也要进行对应的 url 地址修改

简单的配置一下展示页面,效果如下

![grafana](/upload/2017/grafana.png)

#### 我们最后来看一下总体的服务运行状况

``` sh
root@ubuntu1:~# docker service ps $(docker service ls -q)
ID                  NAME                IMAGE                                    NODE                DESIRED STATE       CURRENT STATE               ERROR                              PORTS
9925er3baoao        telegraf.1          192.168.52.128:5000/ydhtelegraf:latest   ubuntu3             Running             Running 22 minutes ago                                         
zr9bkz5crzl0        influxdb.1          192.168.52.128:5000/influxdb:latest      ubuntu2             Running             Running about an hour ago                                      
dlxl3zlgpx8j        grafana.1           192.168.52.128:5000/grafana:latest       ubuntu3             Running             Running 2 hours ago                                            
ovbuip43u760        hostname.1          192.168.52.128:5000/hostname:latest      ubuntu2             Running             Running 6 hours ago                                            
h4oqfvhp37u4        registry.1          registry:2                               ubuntu2             Running             Running 6 hours ago                                            
kttlrq3spsxw        portainer.1         portainer/portainer:latest               ubuntu1             Running             Running 7 hours ago                                            
wlav3nc6ftop        hostname.2          192.168.52.128:5000/hostname:latest      ubuntu3             Running             Running 6 hours ago                                            
nv11s85ozpda        hostname.3          192.168.52.128:5000/hostname:latest      ubuntu1             Running             Running 6 hours ago                                            

```
