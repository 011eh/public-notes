

# Docke架构

![Docker Architecture Diagram](D:\011eH\Documents\笔记\图片\Docker-1)

# 开始

## 卸载Docker

```bash
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

## 安装yum 工具

```bash
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

## 设置阿里云镜像仓库

```bash
sudo yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

## 更新yum软件包索引

```bash
yum makecache
```

## 安装Docker

```bash
sudo yum install docker-ce docker-ce-cli containerd.io
```

## 查看版本

```bash
docker version
```



```bash
Client: Docker Engine - Community
 Version:           20.10.11
 API version:       1.41
 Go version:        go1.16.9
 Git commit:        dea9396
 Built:             Thu Nov 18 00:36:58 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.11
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.16.9
  Git commit:       847da18
  Built:            Thu Nov 18 00:35:20 2021
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.12
  GitCommit:        7b11cfaabd73bb80907dd23182b9347b4245eb5d
 runc:
  Version:          1.0.2
  GitCommit:        v1.0.2-0-g52b36a2
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

## Hello World

```bash
docker run hello-world
```

### 第一次运行

```bash
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
2db29710123e: Already exists 
Digest: sha256:cc15c5b292d8525effc0f89cb299f1804f3a725c8d05e158653a563f15e4f685
Status: Downloaded newer image for hello-world:latest
```

> 本地找不到该镜像，从docker镜像库中拉取

### 运行结果

```bash
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

## 卸载

### 卸载Docker引擎

```bash
sudo yum remove docker-ce docker-ce-cli containerd.io
```

### 删除镜像、容器、卷

```bash
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```





# 命令

## 镜像命令

### Docker信息

```bash
docker version
docker info

#帮助
docker [OPTIONS] COMMAND
docker [命令] --help
```
### 查看本地镜像

```bash
docker images [OPTIONS] [REPOSITORY[:TAG]]

#选项
-a, --all             Show all images (default hides intermediate images)
-q, --quiet           Only show image IDs
```
### 搜索

```bash
docker search [OPTIONS] TERM

#例子
docker search centos
```
### 拉取镜像

```bash
docker pull [OPTIONS] NAME[:TAG|@DIGEST]

#例子
docker pull centos
```
### 删除镜像

```bash
docker rmi [OPTIONS] IMAGE [IMAGE...]

#选项
-f 强制删除

#例子
docker rmi -f centos
```

### 运行命令

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

#选项
--name [名字]        给容器设置名字
-p, --publish list  Docker端口到主机端口的映射

-i, --interactive	Keep STDIN open even if not attached
-t, --tty           Allocate a pseudo-TTY
-P, --publish-all   Publish all exposed ports to random ports
-d, --detach        Run container in background and print container ID

##数据卷
-v, --volume list                    Bind mount a volume
    --volume-driver string           Optional volume driver for the container
    --volumes-from list              Mount volumes from the specified container(s)

##网络
	——net	指定网络模式

#例子
docker run -it centos /bin/bash	交互模式，控制台使用bash，启动、进入centos

docker run -d centos	后台运行，直接运行，会立刻停止
```





## 容器命令

### 列出容器

```bash
docker ps [OPTIONS]

#参数
-a, --all             Show all containers (default shows just running)	包括历史运行
-f, --filter filter   Filter output based on conditions provided
-q, --quiet           Only display container IDs	只显示ID
```

### 在容器中执行命令

```bash
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]

#选项
-d, --detach               Detached mode: run command in the background
-i, --interactive          Keep STDIN open even if not attached
-t, --tty                  Allocate a pseudo-TTY
```



```bash
docker attach [OPTIONS] CONTAINER

#选项
--detach-keys string   Override the key sequence for detaching a container
--no-stdin             Do not attach STDIN
--sig-proxy            Proxy all received signals to the process (default true)
```

### 退出容器

`ctrl + p + q`

- 不会关闭容器

### 启动、停止

```bash
docker start
docker restart
docker stop
docker kill
```

### 删除容器

```bash
docker rm [OPTIONS] CONTAINER [CONTAINER...]

#选项
-f, --force     Force the removal of a running container (uses SIGKILL)
```





## 其他命令

### 查看日志

```bash
docker logs [OPTIONS] CONTAINER

    --details		显示额外内容
-f, --follow		追加形式，显示日志打印
    --since 时间	   显示从 时间 开始的日志
-n, --tail N		显示 N 条最近日志
-t, --timestamps	显示时间戳
    --until 时间	   显示从 时间 结束的日志
```

### 查看容器内部进程信息

```bash
docker top CONTAINER [ps OPTIONS]
```

### 查看容器元数据

```bash
docker inspect [OPTIONS] NAME|ID [NAME|ID...]

#选项
-f, --format string   Format the output using the given Go template
-s, --size            Display total file sizes if the type is container
	--type string     Return JSON for specified type
```

### 从容器中复制文件

```bash
#容器到主机
docker cp [OPTIONS] 容器:源路径 目的路径

#主机到容器
docker cp [OPTIONS] 源路径 容器:目的路径

#选项
-a, --archive       Archive mode (copy all uid/gid information)
-L, --follow-link   Always follow symbol link in SRC_PATH
```

### 提交镜像

```bash
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]

#选项
-a, --author string    Author (e.g., "John Hannibal Smith <hannibal@a-team.com>")
-c, --change list      Apply Dockerfile instruction to the created image
-m, --message string   Commit message
-p, --pause            Pause container during commit (default true)
```





# 数据卷

1. 数据可以使容器的数据持久化，容器被删除，数据也不会丢失
2. 实现容器间的数据共享
3. 方便在主机对容器相关数据进行管理
4. 数据卷绕过Docker镜像的层叠机制，对镜像的修改变成了直接存储

## 数据卷容器

1. 是一个容器，主要目的是提供数据卷为其他容器挂载
2. 使用数据卷容器，方便实现多个容器共享一些持续更新的数据



```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

#数据卷选项
-v, --volume list                    Bind mount a volume
    --volume-driver string           Optional volume driver for the container
    --volumes-from list              Mount volumes from the specified container(s)
#例子
docker run 镜像 -v 主机路径:容器内路径

docker run 镜像 -v 名字:容器内路径:[ro,rw]

#数据容器例子
docker run -it --name centos01 mycentos
docker run -it --volumes-from centos01 --name centos02 mycentos
docker run -it --volumes-from centos02 --name centos03 mycentos
```





# DockerFile

```dockerfile
from		#指定基础镜像
maintainer	#镜像维护者信息，姓名+邮箱
run			#指定构建时，需要运行的命令
add			#添加镜像
workdir		#镜像工作目录
volume		#设置挂载目录
expose		#声明暴露的端口
cmd			#容器运行时运行的命令，会覆盖前面的cmd
entrypoint	#容器运行时运行的命令，能追加命令，不会覆盖
onbuild		#构建一个被继承的Dockerfile时，就会执行onbuild的指令
copy		#将文件拷贝到镜像
env			#构建时，设置环境变量
```

## 构建镜像

```bash
docker build -f docker文件 -t  名字:标签 .
```

- `命令最后有点符号：“.”`

## 例子

``` dockerfile
FROM centos
  
VOLUME ["volume01","volume02"]

CMD echo "-----build finish-----"
CMD /bin/bash
```



```dockerfile
From centos
MAINTAINER 011eh<011eh@011.com>
ENV MYPATH /usr/local
WORKDIR $MYPATH

RUN yum -y install vim 
RUN yum -y install net-tools

EXPOSE 80
CMD echo $MYPATH
CMD echo "-----finish-----"
CMD /bin/bash
```



```dockerfile
From centos
MAINTAINER 011eh<011eh@011.com>

COPY readme.txt /usr/local/readme.txt
ADD jdk8_312.tar.gz /usr/local/
ADD apache-tomcat-9.0.55.tar.gz /usr/local/

RUN yum -y install vim

ENV MYPATH /usr/local
WORKDIR $MYPATH

ENV JAVA_HOME /usr/local/jdk8u312-b07
ENV CLASSPATH $JAVA_HOME/lib/dt.jar;$JAVA_HOME/lib/tools.jar

ENV CATALINA_HOME /usr/local/apache-tomcat-9.0.55
ENV CATALINA_BASH /usr/local/apache-tomcat-9.0.55

ENV PATH $PATH:$JAVA_HOME/bin;$CATALINA_HOME/lib;$CATAINA_HOME/bin

EXPOSE 8080

CMD /usr/local/apache-tomcat-9.0.55/bin/startup.sh && tail -F /usr/local/apache-tomcat-9.0.55/logs/catalina.out
```

### 创建镜像

```
docker build -t imagefirst:1.0 .
```

### 运行容器

```bash
docker run -it -p 8888:8080 --name first -v /home/011eh/docker/mount/first/tomcat/webapps:/usr/local/apache-tomcat-9.0.55/webapps -v /home/011eh/docker/mount/first/tomcat/logs:/usr/local/apache-tomcat-9.0.55/logs imagefirst:1.0
```

# Docker网络

1. 启动2个容器
2. 分别执行`ip addr`命令

## centos01打印信息

```bash
124: eth0@if125: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    
    #地址为172.17.0.2/16
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

## centos02打印信息

```bash
126: eth0@if127: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    
    #地址为172.17.0.3/16
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

## 主机打印信息

```bash
#docker0
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:39:0c:eb:bd brd ff:ff:ff:ff:ff:ff
    
    #地址为172.17.0.1/16
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:39ff:fe0c:ebbd/64 scope link 
       valid_lft forever preferred_lft forever

#vech-pair技术，对应centos01
125: veth2e4dc22@if124: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether fe:42:45:d2:fd:62 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::fc42:45ff:fed2:fd62/64 scope link 
       valid_lft forever preferred_lft forever

#vech-pair技术，对应centos02
127: vethb659505@if126: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether 8e:e7:6f:ac:58:51 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::8ce7:6fff:feac:5851/64 scope link 
       valid_lft forever preferred_lft forever
```

## docker的link选项

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

--link list	Add link to another container	链接容器，实质时hosts文件的映射
```

## 

### 自定义网络

```bash
docker network create -d bridge --subnet 192.168.0.0/24 --gateway 192.168.0.1 mynet
docker run -it -P --name centos02 --net mynet centos
```

#### 查看主机Ip信息

```bash
130: br-c6cd484e2c02: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:cc:6a:62:25 brd ff:ff:ff:ff:ff:ff
    
    #自定义的网络
    inet 192.168.0.1/24 brd 192.168.0.255 scope global br-c6cd484e2c02
       valid_lft forever preferred_lft forever
    inet6 fe80::42:ccff:fe6a:6225/64 scope link 
       valid_lft forever preferred_lft forever
```

### 将主机连接到另一个网络

```bash
docker network connect mynet centos03
```



## 相关命令

```bash
docker network COMMAND

#命令
connect     Connect a container to a network
create      Create a network
disconnect  Disconnect a container from a network
inspect     Display detailed information on one or more networks
ls          List networks
prune       Remove all unused networks
rm          Remove one or more networks
```

### create

```bash
docker network create [OPTIONS] NETWORK

#选项
      --attachable           Enable manual container attachment
      --aux-address map      Auxiliary IPv4 or IPv6 addresses used by Network driver (default map[])
      --config-from string   The network from which to copy the configuration
      --config-only          Create a configuration only network
  -d, --driver string        Driver to manage the Network (default "bridge")
      --gateway strings      IPv4 or IPv6 Gateway for the master subnet
      --ingress              Create swarm routing-mesh network
      --internal             Restrict external access to the network
      --ip-range strings     Allocate container ip from a sub-range
      --ipam-driver string   IP Address Management Driver (default "default")
      --ipam-opt map         Set IPAM driver specific options (default map[])
      --ipv6                 Enable IPv6 networking
      --label list           Set metadata on a network
  -o, --opt map              Set driver specific options (default map[])
      --scope string         Control the network's scope
      --subnet strings       Subnet in CIDR format that represents a network segment
```

### connect

```bash
docker network connect [OPTIONS] NETWORK CONTAINER

#选项
--alias strings           Add network-scoped alias for the container
--driver-opt strings      driver options for the network
--ip string               IPv4 address (e.g., 172.30.100.104)
--ip6 string              IPv6 address (e.g., 2001:db8::33)
--link list               Add link to another container
--link-local-ip strings   Add a link-local address for the container
```

# redis集群

## 准备工作

### 创建网络

```bash
docker network create redis --subnet 172.38.0.0/16
```

#### 创建节点

```sh
for port in $(seq 1 6);
do
mkdir -p /home/011eh/redis/cluster/node-${port}/conf
touch /home/011eh/redis/cluster/node-${port}/conf/redis.conf
cat  <<EOF> /home/011eh/redis/cluster/node-${port}/conf/redis.conf
port 6379
bind 0.0.0.0
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-announce-ip 172.38.0.1${port}
cluster-announce-port 6379
cluster-announce-bus-port 16379
appendonly yes
EOF
done
```

## 运行容器

```sh
for port in $(seq 1 6);
do
docker run -p 637${port}:6379 -p 1637${port}:16379 --name redis-${port} \
    -v /home/011eh/redis/cluster/node-${port}/data:/data \
    -v /home/011eh/redis/cluster/node-${port}/conf/redis.conf:/etc/redis/redis.conf \
    -d --net redis --ip 172.38.0.1${port} redis redis-server /etc/redis/redis.conf
done
```

## 创建集群

```bash
redis-cli --cluster create 127.38.0.11:6379 127.38.0.12:6379 127.38.0.13:6379 127.38.0.14:6379 127.38.0.15:6379 127.38.0.16:6379 --cluster-replicas 1
```



# docker Compose

## Hello World

### 创建文件夹

```
mkdir composetest
```



### 创建app.py

- Python应用
- 使用了Redis

```python
import time

import redis
from flask import Flask

app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)

@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times.\n'.format(count)
```

### 创建Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.7-alpine
WORKDIR /code
ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0
RUN set -eux && sed -i s/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g /etc/apk/repositories
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
EXPOSE 5000

#复制当前目录到工作目录
COPY .. .
CMD ["flask", "run"]
```

### 创建requirements.txt

```
flask
redis
```

### 创建docker-compose.yml

[Compose yml配置文件](https://docs.docker.com/compose/compose-file/compose-file-v3/)

```yaml
version: "3.9"
services:
  web:
    build: .
    ports:
      - "5000:5000"
  redis:
    image: "redis:alpine"
```

### 使用Copmopse并运行

```bash
docker-compose up
```

### 创建的网络

```bash
docker network ls

#该服务的网络
eb75a39ac6a7   composetest_default   bridge    local
```

### 查看网络

```bash
docker network inspect composetest_default

# 具体容器在该网络中

"Containers": {
            "3b7e7b37a281ba4f296832a7a0ba8d89d6b14e967b305d5356ea75f20f5f1bce": {
                "Name": "composetest_redis_1",
                "EndpointID": "780dffc262fa5686d114dd6d33f74bcb9a75eb44d38051ca1317fd2c0a8d0726",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            },
            "8ffe7c3940dce6298cebfbb49164a722c8a6b30fa09a1116978b45c38473a6ae": {
                "Name": "composetest_web_1",
                "EndpointID": "d20938f938cc15c876588fcbca5ae5cae64d43466a959623a2e653374e4a379f",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            }
},
```



# Docker Swarm

## 节点工作模式

![Swarm mode cluster](D:\011eH\Documents\笔记\图片\Docker-2)

## 初始节点

```bash
docker swarm init --advertise-addr 地址
```

## 加入节点

```bash
#管理加入节点
docker swarm join-token manager
docker swarm join-token worker
```

## 查看节点

```bash
docker node ls

#例子
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
zfr54k4yj10ldl5kzyvsm0g91 *   linux3     Ready     Active         Reachable        20.10.11
osdc1tzptl2dcvt6jsznf7jiu     linux4     Ready     Active         Leader           20.10.11
x9wawnqzsykhbb4a2ylnd8giu     linux4     Down      Active                          20.10.11
s7hjr6p4ef1xsyz148cjitf97     linux5     Ready     Active                          20.10.11
v3zk3nzoqnm0wawxdiranvo34     linux6     Ready     Active         Reachable        20.10.11
```

## 运行服务

- 具有动态扩缩容功能

```bash
docker service create -p 8000:80 --name nginx01 nginx
```

## 查看服务

```bash
docker service ls

#打印信息
ID             NAME      MODE         REPLICAS   IMAGE          PORTS
li7l0igehb4o   nginx01   replicated   1/1        nginx:latest   *:8000->80/tcp

#只有一份副本
#随机分配，当前的nginx服务被安排linux4上
```

## 进行扩容

```bash
docker service update --replicas 3 nginx01
docker service scale nginx01=4

#nginx服务有3个，在任何节点上都能访问
docker service ls

ID             NAME      MODE         REPLICAS   IMAGE          PORTS
li7l0igehb4o   nginx01   replicated   3/3        nginx:latest   *:8000->80/tcp
```



