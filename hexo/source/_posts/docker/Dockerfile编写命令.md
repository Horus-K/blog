---
title: Dockerfile编写命令
date: 2020-03-09 13:34:15
tags:
- docker
categories: docker
---

```shell
FROM: 指定基础镜像
RUN： 构建镜像过程中需要执行的命令。可以有多条。
CMD：添加启动容器时需要执行的命令。多条只有最后一条生效。可以在启动容器时被覆盖和修改。
ENTRYPOINT：同CMD，但这个一定会被执行，不会被覆盖修改。
LABEL ：为镜像添加对应的数据。
MLABELAINTAINER：表明镜像的作者。将被遗弃，被LABEL代替。
EXPOSE：设置对外暴露的端口。
ENV：设置执行命令时的环境变量，并且在构建完成后，仍然生效
ARG：设置只在构建过程中使用的环境变量，构建完成后，将消失
ADD：将本地文件或目录拷贝到镜像的文件系统中。能解压特定格式文件，能将URL作为要拷贝的文件
COPY：将本地文件或目录拷贝到镜像的文件系统中。
VOLUME：添加数据卷
USER：指定以哪个用户的名义执行RUN, CMD 和ENTRYPOINT等命令
WORKDIR：设置工作目录
ONBUILD：如果制作的镜像被另一个Dockerfile使用，将在那里被执行Docekrfile命令
STOPSIGNAL：设置容器退出时发出的关闭信号。
HEALTHCHECK：设置容器状态检查。
SHELL：更改执行shell命令的程序。Linux的默认shell是[“/bin/sh”, “-c”]，Windows的是[“cmd”, “/S”, “/C”]
```

