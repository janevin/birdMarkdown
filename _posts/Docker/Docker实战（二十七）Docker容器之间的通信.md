---
title: "Docker实战（二十七）Docker容器之间的通信"
date: 2017-05-02 14:00:38
tags: [Docker命令, Dockerfile]
categories: [Docker]
---

最近在修改我以前写的Docker镜像，才发现我一直都没有把Docker用好，连Docker的容器之前如何通信都不知道。之前的做法是把不同的环境安装在一个Docker容器中，就不存在容器间通信的问题。但是Docker推荐的用法是一个Docker容器只运行一个进程，所以我将以前写的Docker镜像进行了重构。下面来总结下Docker容器之间的通信。

### Docker的网络模式

docker目前支持以下5种网络模式：

docker run 创建 Docker 容器时，可以用 --net 选项指定容器的网络模式。

- host模式 : 使用 --net=host 指定。与宿主机共享网络，此时容器没有使用网络的namespace，宿主机的所有设备，如Dbus会暴露到容器中，因此存在安全隐患。
- container模式 : 使用 --net=container:NAME_or_ID 指定。指定与某个容器实例共享网络。
- none模式 : 使用 --net=none 指定。不设置网络，相当于容器内没有配置网卡，用户可以手动配置。
- bridge模式 : 使用 --net=bridge 指定，默认设置。此时docker引擎会创建一个veth对，一端连接到容器实例并命名为eth0，另一端连接到指定的网桥中（比如docker0），因此同在一个主机的容器实例由于连接在同一个网桥中，它们能够互相通信。容器创建时还会自动创建一条SNAT规则，用于容器与外部通信时。如果用户使用了-p或者-Pe端口端口，还会创建对应的端口映射规则。
- 自定义模式 : 使用自定义网络，可以使用docker network create创建，并且默认支持多种网络驱动，用户可以自由创建桥接网络或者overlay网络。

默认是桥接模式，网络地址为172.17.0.0/16，同一主机的容器实例能够通信，但不能跨主机通信。

#### host模式

如果启动容器的时候使用 host 模式，那么这个容器将不会获得一个独立的 Network Namespace，而是和宿主机共用一个 Network Namespace。容器将不会虚拟出自己的网卡，配置自己的 IP 等，而是使用宿主机的 IP 和端口。

#### container模式

这个模式指定新创建的容器和已经存在的一个容器共享一个 Network Namespace，而不是和宿主机共享。新创建的容器不会创建自己的网卡，配置自己的 IP，而是和一个指定的容器共享 IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。两个容器的进程可以通过 lo 网卡设备通信。

#### none模式

这个模式和前两个不同。在这种模式下，Docker 容器拥有自己的 Network Namespace，但是，并不为 Docker容器进行任何网络配置。也就是说，这个 Docker 容器没有网卡、IP、路由等信息。需要我们自己为 Docker 容器添加网卡、配置 IP 等。

#### bridge模式

bridge 模式是 Docker 默认的网络设置，此模式会为每一个容器分配 Network Namespace、设置 IP 等，并将一个主机上的 Docker 容器连接到一个虚拟网桥上。

当 Docker server 启动时，会在主机上创建一个名为 docker0 的虚拟网桥，此主机上启动的 Docker 容器会连接到这个虚拟网桥上。虚拟网桥的工作方式和物理交换机类似，这样主机上的所有容器就通过交换机连在了一个二层网络中。

接下来就要为容器分配 IP 了，Docker 会从 RFC1918 所定义的私有 IP 网段中，选择一个和宿主机不同的IP地址和子网分配给 docker0，连接到 docker0 的容器就从这个子网中选择一个未占用的 IP 使用。如一般 Docker 会使用 172.17.0.0/16 这个网段，并将 172.17.42.1/16 分配给 docker0 网桥（在主机上使用 ifconfig 命令是可以看到 docker0 的，可以认为它是网桥的管理接口，在宿主机上作为一块虚拟网卡使用）

当创建一个 Docker 容器的时候，同时会创建了一对 veth pair 接口（当数据包发送到一个接口时，另外一个接口也可以收到相同的数据包）。这对接口一端在容器内，即 eth0；另一端在本地并被挂载到 docker0 网桥，名称以 veth 开头（例如 vethAQI2QT）。通过这种方式，主机可以跟容器通信，容器之间也可以相互通信。Docker 就创建了在主机和所有容器之间一个虚拟共享网络。

![Docker bridge模式](http://img.blog.csdn.net/20170502213752341?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### 同主机不同容器之间通信

这里同主机不同容器之间通信主要使用Docker桥接（Bridge）模式。该bridge接口在本地一个单独的Docker宿主机上运行，并且它是我们后面提到的所有三种连接方式的背后机制。

```
$ ifconfig docker0
docker0   Link encap:Ethernet  HWaddr 56:84:7a:fe:97:99  
          inet addr:172.17.42.1  Bcast:0.0.0.0  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

#### 连接方式

- 方式一：可以通过使用容器的IP地址来通信。这种方式会导致IP地址的硬编码，不方便迁移，并且容器重启后IP地址可能会改变，除非使用固定的IP地址。
- 方式二：可以通过宿主机的IP加上容器暴露出的端口号来通信。这种方式比较单一，只能依靠监听在暴露出的端口的进程来进行有限的通信。
- 方式三：可以使用容器名，通过docker的link机制通信。这种方式通过docker的link机制可以通过一个name来和另一个容器通信，link机制方便了容器去发现其它的容器并且可以安全的传递一些连接信息给其它的容器。使用name给容器起一个别名，方便记忆和使用。即使容器重启了，地址发生了变化，不会影响两个容器之间的连接。

```
# 查看容器的内部IP
$ docker inspect --format='{{.NetworkSettings.IPAddress}}' $CONTAINER_ID
```

```
# Elasticsearch容器
$ docker inspect --format='{{.NetworkSettings.IPAddress}}' 4d5e7a1058de
172.17.0.2

# Kibana容器
$ docker inspect --format='{{.NetworkSettings.IPAddress}}' 4f26e64bfe82
172.17.0.4
```

方式一：使用容器的IP地址来通信

```
# 进入Kibana容器
$ docker exec -it 4f26e64bfe82 /bin/bash

# 在Kibana容器使用ES容器的IP地址来访问ES服务
$ curl -XGET 'http://172.17.0.2:9200/_cat/health?pretty'
1493707223 06:40:23 ben-es yellow 1 1 11 11 0 0 11 0 - 50.0%
```

方式二：使用宿主机的IP加上容器暴露出的端口号来通信

```
# 进入Kibana容器
$ docker exec -it 4f26e64bfe82 /bin/bash

# 在Kibana容器使用宿主机的IP地址来访问ES服务（我这里本机的IP地址是10.10.1.129）
$ curl -XGET 'http://10.10.1.129:9200/_cat/health?pretty'
1493707223 06:40:23 ben-es yellow 1 1 11 11 0 0 11 0 - 50.0%
```

方式三：使用docker的link机制通信

```
# 先启动ES容器，并且使用--name指定容器名称为：elasticsearch_2.x_yunyu
$ docker run -itd -p 9200:9200 -p 9300:9300 --name elasticsearch_2.x_yunyu birdben/elasticsearch_2.x:v2

# 启动Kibana容器，并且使用--link指定关联的容器名称为ES的容器名称：elasticsearch_2.x_yunyu
$ docker run -itd -p 5601:5601 --link elasticsearch_2.x_yunyu --name kibana_4.x_yunyu birdben/kibana_4.x:v2

# 查看运行的容器
$ docker ps
CONTAINER ID        IMAGE                               COMMAND                  CREATED             STATUS              PORTS
4f26e64bfe82        birdben/kibana_4.x:v2               "docker-entrypoint..."   25 hours ago        Up 15 minutes       0.0.0.0:5601->5601/tcp                                                       kibana_4.x_yunyu
4d5e7a1058de        birdben/elasticsearch_2.x:v2        "docker-entrypoint..."   26 hours ago        Up 19 hours         0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp                               elasticsearch_2.x_yunyu

# 在Kibana容器使用--link的容器名称来访问ES服务
$ curl -XGET 'http://elasticsearch_2.x_yunyu:9200/_cat/health?pretty'
1493707223 06:40:23 ben-es yellow 1 1 11 11 0 0 11 0 - 50.0%
```

实际上--link机制就是在Docker容器中的/etc/hosts文件中添加了一个ES容器的名称解析。有了这个名称解析后就可以不使用IP来和目标容器通信了，除此之外当目标容器重启，Docker会负责更新/etc/hosts文件，因此可以不用担心容器重启后IP地址发生了改变，解析无法生效的问题。

Kibana容器的/etc/hosts文件

```
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.2      4d5e7a1058de
```

ES容器的/etc/hosts文件

```
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.2      elasticsearch_2.x_yunyu 4d5e7a1058de
172.17.0.4      4f26e64bfe82
```

当docker引入网络新特性后，link机制变的有些多余，但是为了兼容早期版本，–link机制在默认网络上的功能依旧没有发生变化，docker引入网络新特性后，内置了一个DNS Server，但是只有用户创建了自定义网络后，这个DNS Server才会起作用。

### 跨主机不同容器之间通信

（待续）

### 使用DockerCompose

（待续）

参考文章：

- https://jiajially.gitbooks.io/dockerguide/content/chapter_network_pro/index.html
- https://opskumu.gitbooks.io/docker/content/chapter6.html
- http://tonybai.com/2016/01/15/understanding-container-networking-on-single-host/
- https://yq.aliyun.com/articles/55912
- https://yq.aliyun.com/articles/30345?spm=5176.100239.blogcont40494.28.FnfzAV
- http://int32bit.me/2016/05/10/Docker%E5%AE%9E%E7%8E%B0%E8%B7%A8%E4%B8%BB%E6%9C%BA%E9%80%9A%E4%BF%A1/