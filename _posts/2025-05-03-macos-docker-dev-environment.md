---
layout: mypost
title: MacOS使用Docker配置开发环境
categories: [环境配置]
---

### 下载与安装

> 目前配置了 mysql, redis, rabbitmq, nginx. 均使用 arm64v8 版本. 所有挂载均使用数据卷形式, 根据情况选择是否创建

拉取镜像:

```bash
docker pull arm64v8/mysql:8.4.2
docker pull arm64v8/redis:6.2.18
docker pull arm64v8/rabbitmq:4.0.9
docker pull arm64v8/nginx:1.27.5
```

创建数据卷:

```bash
docker volume create mysql-data
docker volume create mysql-config
docker volume create mysql-log

docker volume create redis-data
docker volume create redis-conf
docker volume create redis-log

docker volume create rabbitmq-data

docker volume create nginx-data
docker volume create nginx-conf
docker volume create nginx-logs
```

### 部署

> 只在本地使用, 所以仅有最基本的配置.

##### Mysql

```bash
docker run --name mysql \
    -e MYSQL_ROOT_PASSWORD=root \
    -v mysql-data:/var/lib/mysql \
    -v mysql-log:/var/log/mysql \
    -v mysql-config:/etc/mysql \
    -p 3306:3306 \
    -d arm64v8/mysql:8.4.2
```

##### Redis

首先在 redis-conf 中创建 `redis.conf` 文件, 内容如下:

```conf
port 6379
bind 0.0.0.0
protected-mode yes
requirepass 123456

logfile "/var/log/redis/redis.log"
loglevel notice

save ""

dir /data
dbfilename dump.rdb

appendonly no

daemonize no

timeout 0
```

启动:

```bash
docker run -d \
  --name redis \
  -p 6379:6379 \
  -v redis-data:/data \
  -v redis-conf:/usr/local/etc/redis \
  -v redis-log:/var/log/redis \
  arm64v8/redis:6.2.18 \
  redis-server /usr/local/etc/redis/redis.conf
```

##### Rabbitmq

```bash
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=ahci \
  -e RABBITMQ_DEFAULT_PASS=ahci \
  -v rabbitmq-data:/var/lib/rabbitmq \
  arm64v8/rabbitmq:4.0.9
```

启动成功后执行 `docker ps` 命令查看 rabbitmq 对应的容器 ID, 然后进入容器内部开启管理页面.

```bash
docker ps

# 将 xxx 替换为容器ID
docker exec -it xxx /bin/bash
```

成功进入容器内部后, 终端开头应该变成 `root@xxx:/#` 或类似, 然后执行下面的命令开启管理页面

```bash
rabbitmq-plugins enable rabbitmq_management
```

最后重启容器

```bash
# 将 xxx 替换为容器ID
docker restart xxx
```

访问 `http://localhost:15672/#/` 出现 RabbitMQ 管理页面即可.

##### Nginx

```bash
docker run -d \
  --name nginx \
  -p 80:80 \
  -v nginx-data:/usr/share/nginx/html \
  -v nginx-conf:/etc/nginx \
  -v nginx-logs:/var/log/nginx \
  arm64v8/nginx:1.27.5
```

打开 `http://localhost/` 出现 Welcome to nginx 即可.
