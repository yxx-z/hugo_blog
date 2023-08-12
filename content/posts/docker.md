---
title: Docker 网络相关命令
subtitle:
date: 2023-08-12T13:45:25+08:00
draft: true
author:
  name: yxx
  link:
  email: yangxx@88.com
  avatar:
description:
keywords: docker、网络
license:
comment: false
weight: 0
tags:
  - docker
categories:
  - 网络
hiddenFromHomePage: false
hiddenFromSearch: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content/#front-matter
---
docker 网络相关命令
<!--more-->
***

##### 查看所有网络
```
docker network ls
```

##### 删除指定网络
```
docker network rm [网络名称]
```

##### 删除容器的网络
```
docker network disconnect [网络名称] [容器名称]
```

##### 创建容器网络
创建名为xxxx的网络 172.18.0.1为网关地址 可更换，172.18.0.0/16 表示ip范围 可根据自己需求更改，18为ip段 可更换
```
​docker network create --subnet 172.18.0.0/16 --gateway 172.18.0.1 --driver bridge xxxx
```


##### 查看容器网络信息
```
docker inspect [容器名称]
```

##### 将容器加入一个网络中
```
docker network connect [网络名称] [容器名称]
```

##### 将容器加入一个网络中，并指定ip地址
```
docker network connect --ip [ip地址] [网络名称] [容器名称]
```