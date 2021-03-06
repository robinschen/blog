---
title: Docker 问题记录
date: 2018-12-13 14:49:18
description: 这里记录下日常 docker 使用遇到的问题,现在整理如下
categories: [Docker篇]
tags: [Docker]
---
<!-- more -->
### 时区问题

#### 构建镜像
时区的配置在 `/etc/localtime`

localtime 文件会指向 `/usr/share/zoneinfo/Asia/` 目录下的某个文件

我们只需要将其指向 ShangHai 即可

Dockerfile 可以这样配置

``` bash
    RUN rm -rf /etc/localtime && \
        ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

先删除,在创建一个软连接即可



如果是在容器内直接操作的话:

1. apt-get install tzdata
2. 然后依次选择 6 , 70 即可
3. 使用 dpkg-reconfigure tzdata 来重写选择


#### 已构建好的镜像
启动一个容器，加上如下参数，即可使用宿主机时间

```bash
    -v /etc/localtime:/etc/localtime:ro
```

#### 已经在运行的容器

```bash
    docker cp /etc/localtime [container]:/etc/localtime
```

在检查下是否修改成功:

```bash
    docker exec [container] date
```

#### springboot 应用的时区

如果构建的 springboot 项目的镜像,基于 jib 插件构建的话,并且基础镜像选择的是 openjdk:8  
那么这时 `jvmFlags` 参数加上一个 `-Duser.timezone=GMT+08` boot 服务启动时会使用东八区的时间  
而且这不会改变容器的时区,如果你进入容器,执行 date 打印的话,还是会发现时间少 8 个小时  
但是对于应用来说已经没有问题了

#### mariadb 容器的时区
默认官方的 mariadb 的镜像时区是 0 时区的,想要改变的话,添加执行参数 `--default-time-zone='+8:00'`  
完整 docker-compose.yml

```yaml
      db:
        image: mariadb
        container_name: mariadb
        restart: always
        networks:
          - job-net
        ports:
          - 3306:3306
        volumes:
          - /home/liufa/joy-job/db-data:/var/lib/mysql
        environment:
          - MYSQL_ROOT_PASSWORD=123456
          - MYSQL_ROOT_HOST=%
        command: --default-time-zone='+8:00'
```

### pm2-web 命令错误问题

通常我们都是将 node_modules 文件夹直接复制到镜像中

有时候会出现问题,就比如 pm2-web ,构建成镜像后,命令无法使用

原因在于开发的机器的操作系统和镜像的操作系统不一致,会导致一些包出问题

解决的方式就是重新 `nmp install`

Dockerfile 如下:

``` bash
    RUN rm -rf ./node_modules && \
        rm -rf ./package-lock.json && \
        npm install
```


### 更改 docker数据目录
默认安装的 docker 数据目录在 `/var/lib/docker` 下
想要更改的话  
在 docker 的 service 文件添加参数 `--graph=/home/lifa`  
之后重启 docker 服务

### 查看 docker 使用的空间
有时我们下载镜像和运行的容器多了,会占用很多磁盘,该怎么查看呢?
`du sh * `  
或者  
`du -m --max-depth=1`

### 基于已有 docker 镜像制作自己的镜像

#### 编写 Dockerfile 文件
这里就不细说了,了解 Dockerfile 文件里的命令即可

#### 使用 docker commit提交容器
比如说 openjdk:8 ,我想让他支持 node , php , python 等该怎么办?  
有个简单方法  
先 pull 下来
再  
`docker run -it openjdk:8 /bin/bash`
这样就进入到该镜像了,我们可以安装自己需要的东西了
我自己遇到了棘手的问题
就是 openjdk:8 是 debian 系统的,使用的是 apt-get 包管理器
要安装其他东西就要先更新源
可以使用 `apt update` 更新
但是我遇到的问题是更新了也无法安装....
于是我就想着切换到阿里或者科大的源
但是发现这个简单的镜像连 vi 和 gedit 等编辑器都没有,又没法安装他们
无法编辑 `/etc/apt/source.list` 源文件
后来我是 cp 先备份,在 `echo "xxxx" >> /ect/apt/source.list` 更新的  
之后再 update 就可以安装了
安装好之后退出容器,之后
`docker commit containID xxx/xxxx:latest`
提交容器的改变,之后就看到一个新的镜像了

这里有个问题,就是我想定义自己的启动脚本,貌似无法做到,后来看了官方文档
docker commit 有个 -c 参数,解释是这样的:

> commit Apply Dockerfile instruction to the created image

以为使用 Dockerfile 的语法来创建镜像, 还有 3 个参数

-a, --author string    制定个作者
-m, --message string   本次提交信息
-p, --pause            提交镜像时暂停容器,默认是 true

假如我有个 blog 的容器, 示例:

> docker commit -c 'CMD ["sh", "/my-blog/bash/init.sh"]' -c "EXPOSE 80" -c "EXPOSE 8080" -a "JoyLau" -m "JoyLau's Blog Docker Image"  blog nas.joylau.cn:5007/joy/blog.joylau.cn:2.1


### docker 安装在内网服务器, 如何 pull 镜像
在命令行使用 export HTTP_PROXY=xxxx:xx , 命令行里绝大部分命令都可以使用此代理联网,但是安装的 docker 不行,无法 pull 下来镜像文件,想要 pull 使用代理的话,需要添加代理的变量
vim /usr/lib/systemd/system/docker.service
添加
Environment=HTTP_PROXY=http://xxxx:xxx
Environment=HTTPS_PROXY=http://xxxx:xxx
保存
systemctl deamon-reload
systemctl restart docker


