---
title: "minikube：Ubuntu部署本地集群踩坑"
date: 2025-01-25
aliases: ["/Daily Dev"]
tags: ["minikube", "Kubernetes"]
categories: ["Daily Dev"]
author: "Kurong"
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
description: ""
disableHLJS: false # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: false
searchHidden: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

# minikube：Ubuntu 部署本地集群踩坑

## 官方安装教程

[minikube start | minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download)

再次感叹 homebrew 的伟大，一行命令就安装、后续也没有问题。



## 可能遇到的问题的解决方案

### The "docker" driver should not be used with root privileges

使用命令 `minikube start` 启动，参数 `driver` 默认为 `docker` 。如果以 root 用户运行该命令会出现如标题所示的错误，下面是解决方法：

[minikube - Why The "docker" driver should not be used with root privileges - Stack Overflow](https://stackoverflow.com/questions/68984450/why-the-docker-driver-should-not-be-used-with-root-privileges)

然而直接在命令最后加上 `–-force` 也不是不行。

### Unable to find image ……. locally

minikube考虑到了墙的问题，并提供了解决方案。那就是启动的时候添加两个限定国内镜像仓库的参数，这样可以避免后面minikube拉不到镜像的问题.

```bash
 minikube start --image-mirror-country='cn'  --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'
```

如果还是解决不了，就指定镜像，先用 docker 拉取镜像：

```bash
docker pull docker-0.unsee.tech/kicbase/stable:v0.0.46
```

然后再指定镜像：

```bash
minikube start --driver=docker --base-image="docker-0.unsee.tech/kicbase/stable:v0.0.46" --force
```

