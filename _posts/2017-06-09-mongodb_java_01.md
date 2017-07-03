---
layout: post
title:  "spring-data整合MongoDB代码测试"
subtitle: ""
date:   2017-06-09
header-img: "img/post-bg-tech.jpg"
tags:
    - 技术学习 mongodb
categories: 学习
---

# 场景

目前业务场景中有用MySQL存储车辆轨迹数据以及温度计温湿度数据，数据增长非常快,每个月有大概200万到300万的增长速度，并且随着下一步接入企业的进一步增加，温度计数据与车辆轨迹数据还会持续增加（业务需求，每辆车都必须要安装GPS），之前做过数据分库分表，数据库MySQL分库分表导致运维查询会很麻烦，并且不利于数据的统计分析。
目前的业务需求是希望数据稳定的持续的存储以及能够实现实时查询功能，并能够为将来数据的统计分析做好准备，同时数据包含GIS信息，后期可能会需要按照GIS查询。

# 实现思路

# 准备工作

# 代码实战




# MongoDB客户端管理工具

[RoboMongo](https://robomongo.org/)

![MongoDB客户端管理工具RoboMongo使用](/img/post-images/2017-06-16/201706161814.png)

# 参考

# 结束语
