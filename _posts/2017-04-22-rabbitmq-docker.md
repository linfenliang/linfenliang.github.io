---
layout: post
title:  "在Docker中安装并使用RabbitMQ"
subtitle: ""
date:   2017-04-22
header-img: "img/post-bg-tech.jpg"
tags:
    - 工作
    - mq
categories: 工作 消息队列
---

#Docker下安装并运行rabbitmq


`
sudo docker pull rabbitmq
`

拉取到rabbitmq后启动rabbitmq：

`docker run -d --hostname my-rabbit --name some-rabbit -p 15672:15672 rabbitmq:management`

该命令定义了rabbitmq的主机名称（该名称将在集群的名称中使用，另外在docker下hostname是容器启动的时候自动生成的，不指定的话每次可能都不一样，会导致数据丢失情况）：my_rabbit
以及定义了rabbitmq的名称：some-rabbit
同时对docker容器中的端口做了映射到本机15672（rabbitmq的web端口）
最后指定了安装rabbitmq的Web管理插件版本：rabbitmq:management

打开浏览器输入：[http://localhost:15672](http://localhost:15672)
可以看到rabbitmq已经安装并启动起来了
##注意
注意如果运行时参数出错或想重新运行，需要删除该container，

先找到该container：

`docker ps -a`

删除：

`docker rm containerId`

如果发现需要删除image，重新拉取，则：

`docker rmi imageId`



#rabbitmq基本原理


#在Java中调用rabbitmq

#参考

	https://hub.docker.com/_/rabbitmq/
	http://blog.csdn.net/evankaka/article/details/50495437
	
