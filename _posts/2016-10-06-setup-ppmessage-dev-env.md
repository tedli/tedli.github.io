---
layout:     post
title:      搭建ppmessage开发环境
date:       2016-10-06 12:37:45
summary:    使用Docker、docker-compose搭建ppmessage的开发环境
categories: Docker
thumbnail: cloud
tags:
 - Docker
 - docker-compose
 - ppmessage
 - python
---


# [ppmessage][1]

想要开发基于微信的多客服系统，但是微信官方多客服API有一定的限制，无法满足公司决策层对业务的需求，所以开始学习[ppmessage][1]，准备在它的基础上对接微信API进行开发。
读了[ppmessage][1]官方的文档，由于自己对[ppmessage][1]理解的还不够深入（比如配置文件等），按照文档没能搭建起来。遇到了一些问题。这里暂且分享一下基于`Docker`、`docker-compose`搭建[ppmessage][1]开发环境的步骤。之后进一步深入的学习会另起标题更新。
**容器内环境，不应该用作开发，应该只用作发布。**[这里][2]有简单的阐述。
但之所以要用`Docker`、`docker-compose`来搭建开发环境，是因为一旦`Dockerfile`和`docker-compose.yml`写好后，其他想学习[ppmessage][1]的开发者会有一个开箱即用的环境。而有一个能运行能调试的环境，对于学习新系统的人来说，显然是最快的方式。


## Docker环境准备

基于`Docker`、`docker-compose`当然要有能运行的`Docker`环境。这里不介绍如何安装`Docker Engine`，请查`Docker`官方文档。


### Docker Engine端为ubuntu server

我的开发环境`Docker Engine`运行在`VMware Fusion`的`ubuntu server`虚拟机里。对外监听`2375`端口。既，`dockerd`的参数有如`-H tcp://0.0.0.0:2375`此类。


### Docker Client端为macOS Sierra

我在`macOS`下开发。我用`iTerm`+`zsh`，我的`.zshrc`里有`export DOCKER_HOST=tcp://192.168.51.129:2375`，`192.168.51.129`为运行`Docker Engine`的`ubuntu server`的静态IP地址，可以通过修改`/Library/Preferences/VMware\ Fusion/vmnet8/dhcpd.conf`，给`VMware Fusion`虚拟机指定静态IP，这里不展开讨论。
`Docker Client`及`docker-compose`我是用`brew`装的，`brew install docker docker-compose`。当然也可以使用其他方式安装。这里不展开讨论。`Docker Engine`也可以用`docker-machine`安装。
这里要**注意**`Docker registry`有阿里云的源，最好配上国内的源，不然`pull`镜像时会很慢，甚至不能成功。


## 我的[ppmessage fork][4]

由于刚开始学习[ppmessage][1]，只`fork`了官方的仓库，以后深入学习，可以考虑给官方提`pull request`。`Dockerfile`和`docker-compose.yml`已经写好了。如果有可用的`Docker`环境的话，应该是开箱即用的。


### Dockerfile

`build`镜像的脚本，我分了三层。一是`python`层，构建一个能运行[ppmessage][1]的`python`环境；二是[ppmessage][1]业务代码层，基于第一层`python`层，把业务代码拷贝进去；三是`ssh`层，毕竟是开发环境，要能`ssh`进去远程调试。
**生产环境不应该给容器开ssh服务**，`ssh`层只是为了开发，发布时应该发布第二层。我用`Jetbrain`家的`IDEA`，用`PyCharm`等也大同小异，[这里][3]有基于`PyCharm`通过`ssh`远程调试代码的教程，自行参考。


#### base.dockerfile

第一层。安装`python`及依赖的库。
基于`alpine`，最终三层`build`出的镜像大小可以控制在200M以内。并换了阿里云的源及设置上海时区。

{% highlight dockerfile %}
FROM alpine

COPY requirements.txt /tmp/

ENV LANG C.UTF-8

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/' /etc/apk/repositories
RUN echo $'[global]\n\
index-url = http://mirrors.aliyun.com/pypi/simple/\n\
\n\
[install]\n\
trusted-host=mirrors.aliyun.com' > /etc/pip.conf
RUN apk update && apk upgrade
RUN apk add --no-cache tzdata python py-pip gcc musl-dev python-dev jpeg-dev zlib-dev
RUN ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN pip --no-cache-dir install --upgrade pip
RUN pip --no-cache-dir install -r /tmp/requirements.txt && rm -rf /tmp/requirements.txt
{% endhighlight %}

执行
```
docker build -t ppmessage-base -f base.dockerfile .
```
来`build`镜像。


#### Dockerfile（业务层，供发布用）

第二层。复制业务代码到镜像中。如果`python`依赖的库有增加可以在第一层的基础上再安装。

{% highlight dockerfile %}
FROM ppmessage-base

COPY ppmessage /ppmessage
COPY ppmessage.py /

RUN apk update && apk upgrade
RUN chmod 755 /ppmessage.py

EXPOSE 8945

ENTRYPOINT ["/ppmessage.py"]
{% endhighlight %}

执行
```
docker build -t ppmessage .
```
来`build`镜像。


#### ssh.dockerfile（ssh层，供开发调试用）

第三层。开ssh服务，可以用来远程调试。

{% highlight dockerfile %}
FROM ppmessage

RUN apk update && apk upgrade
RUN apk add dropbear dropbear-scp openssh-sftp-server
RUN mkdir /etc/dropbear
RUN dropbearkey -t rsa -f /etc/dropbear/dropbear_rsa_host_key
RUN dropbearkey -t dss -f /etc/dropbear/dropbear_dss_host_key
RUN dropbearkey -t ecdsa -f /etc/dropbear/dropbear_ecdsa_host_key
RUN echo -e "123456\n123456\n" | passwd
RUN touch /var/log/lastlog

RUN echo $'\n\
ppmessage development environment\n' > /etc/motd

EXPOSE 8822

ENTRYPOINT ["dropbear"]
CMD ["-j", "-k", "-E", "-F", "-p", "0.0.0.0:8822"]
{% endhighlight %}


#### docker-compose.yml

定义依赖`redis`、`mysql`，使整体可以运行。如果不调试的话（比如生产环境部署），`ppmessage`的定义中去掉`dockerfile`即可。
**注意**由于刚开始学习[ppmessage][1]，好多概念还不清楚，通过这般步骤运行最终打到控制台的log会有`GCM is not configed.`、`Cache not run for PPMessage not configed.`警告。以后搞明白了会再更新。暂时的步骤是可以运行的。

{% highlight yaml %}
version: '2'

services:

  ppmessage:
    build:
      context: .
      dockerfile: ssh.dockerfile
    ports:
     - 8945:8945
     - 8822:8822
    depends_on:
     - redis
     - mysql
    links:
     - redis
     - mysql
    environment:
      REDIS_HOST: redis

  redis:
    image: redis

  mysql:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: ppmessage
      MYSQL_USER: ppmessage
      MYSQL_PASSWORD: 123456
    command: [--character-set-server=utf8mb4, --collation-server=utf8mb4_unicode_ci]
{% endhighlight %}

`mysql`的参数要有上面写的，不然会插入中文行时会报错。`/ppmessage/db/create.py`和`/ppmessage/db/sqlmysql.py`中，连接串后加了`?charset=utf8mb4`，不然插入中文行时`pymysql`抛错。

![mysql]({{ site.baseurl }}/images/mysql-charset.png)

由于使用`docker-compose`可以基于`links`通过服务名进行服务发现。修改了`ppmessage/core/constant.py`第`354`行为`REDIS_HOST = os.getenv("REDIS_HOST", "redis")`。连独立部署的`redis`时，只要去掉对`redis`的依赖及`link`，环境变量处，配上对应的`redis`服务器的地址即可。`mysql`同理，不过数据库是启动后在界面配的，数据库地址那栏要填`mysql`，或者在`links`里起别名，填对应的别名。`docker-compose`用法细节这里不展开讨论。如果连独立部署的`mysql`，与连独立部署的`redis`同理。

另外由于开`ssh`复写了第二层的`ENTRYPOINT`，要手动`ssh`进去或`docker exec`进去手动起，比如`nohup /usr/lib/ssh/sftp-server &`，这句起`sftp`服务，`PyCharm`依赖它进行远程调试，之后再`nohup /ppmessage.py &`把主业务服务起来。最后跑起来效果如下。

![ppmessage]({{ site.baseurl }}/images/ppmessage.png)


[1]: https://www.ppmessage.cn/
[2]: https://tedli.github.io/docker/2016/10/05/do-not-use-container-as-vm/
[3]: http://www.jianshu.com/p/9b362cdee2ab
[4]: https://github.com/tedli/ppmessage
