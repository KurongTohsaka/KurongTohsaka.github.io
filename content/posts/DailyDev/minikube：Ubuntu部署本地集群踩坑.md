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



## 仪表盘

使用 `minikube dashboard` 启动仪表盘。如果在服务器上启动、本地访问，就运行类似于的下面命令：

```bash
kubectl proxy --address='0.0.0.0' --disable-filter=true
```

然后把 http://127.0.0.1:46749/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ 这样的 URL 改为 http://ServerIP:8001/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ ，就可以外网直接访问了（注意服务器开端口）。



## Helm 安装

直接使用 snap 安装，方便快捷：

```bash
sudo snap install helm --classic
```



## kubectl 安装

snap 上大分：

```bash
snap install kubectl --classic
```



## 安装可能遇到的问题的解决方案

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



## 启动仪表盘可能遇到的问题的解决方案

### 卡在 “正在验证 proxy 运行状况 ...”

先用 `minikube dashboard --alsologtostderr -v=1` 查看是否存在 `503` 报错，如果有继续执行下面命令：

```bash
kubectl get pods --all-namespaces
```

会看到所有命名空间下的 Pod 的状态，着重看 `dashboard-metrics-scraper` 和 `kubernetes-dashboard` 开头的两个 Pod，分别运行 `kubectl logs --namespace=kubernetes-dashboard [PodName]` 查看 Pod 的日志，并通过 `kubectl describe pod [PodName] -n [NameSpace]` ，查看事件详情。

如果遇到的是 `ImagePullBackOff` 镜像拉取错误，步骤如下：

- 先记录下要拉取的镜像名，类似于 `docker.io/kubernetesui/metrics-scraper:v1.0.8` 和 `docker.io/kubernetesui/dashboard:v2.7`  。
- 然后 `minikube ssh` 进入容器，分别拉取 `docker.io/kubernetesui/metrics-scraper:v1.0.8` 和 `kubernetesui/dashboard:v2.7.0` 镜像。
- 如果拉取后的镜像 tag 与查看事件详情中所描述要拉取的镜像名不同，则通过类似 `docker tag kubernetesui/dashboard:v2.7.0 docker.io/kubernetesui/dashboard:v2.7` 来修改 tag 。
- 最后重新启动集群 `minikube start` ，并运行 `minikube dashboard` 。

如果遇到的是 `CrashLoopBackOff` 路由错误，步骤如下：

- 刷新 iptabels：

  ```bash
  systemctl stop kubelet
  systemctl stop docker
  iptables --flush
  iptables -tnat --flush
  systemctl start kubelet
  systemctl start docker
  ```

- 重新运行 `minikube dashboard`

