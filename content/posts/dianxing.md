---
title: 获取天翼网关超级管理员密码
subtitle:
date: 2023-04-22T15:07:26+08:00
draft: true
author:
  name: yxx
  link:
  email:
  avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  - 网关
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
获取天翼网关超级密码
<!--more-->
***

## 前言
因为自己在家用nas搭建了个影音系统。但是由于运营商不给ipv4 只能退而求其次使用ipv6用来外网访问。但是在老家中的电信宽带没有开启ipv6 无法观看。

## 获取超级管理员密码
天翼网关上贴的密码，默认是管理员，只有更改密码名称等低级权限。如果要把网关改成桥接模式，只能用超级管理员账号。但是问了宽带小哥，也不清楚。只能自己动手破机了。

神秘代码
```
http://192.168.1.1:80/telandftpcfg.cmd?action=add&telusername=光猫用户名&telpwd=密码&telport=23&telenable=1&ftpusername=光猫用户名&ftppwd=密码&ftpport=21&ftpenable=1&sessionKey=[你的当次浏览器sessionKey]
```
注：这里的光猫用户名 是天翼网关后面贴的默认用户名，密码也是。
sessionKey获取：用天翼网关默认的账号登录后，鼠标右键选择查看源码。在源码中搜索sessionKey，后面会有对应的随机数。每次登录这个sessionKey对应的随机数都不一样。先不要关闭当前登录的网关系统窗口。另起一个窗口。然后把神秘代码复制进去。回车访问。
然后会得到telent的账号密码以及端口

***
用telent软件或者黑窗口访问
```
telnet 192.168.1.1 端口
```
然后会让你输入用户名和密码。正常是网关后面贴的默认账号密码。然后通过下面命令获取超级管理员密码

```
telecomadmin get
```
密码就会出现。