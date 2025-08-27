---
title: "常见linux命令"
date: 2025-08-27
aliases: ["/Daily Dev"]
tags: ["Linux"]
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
searchHidden: false
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

好的，这个命令非常重要，尤其是在排查 "too many open files" 这类经典问题时。我会把它加上，并放在一个新的“高级调试与诊断”分类里，让整个指南的结构更清晰。

这是更新后的完整版本，你可以直接替换掉之前的博客文章。

------



## 后端实习Linux命令指南

### 一、文件与目录管理 (基础中的基础)

这是最基本的操作，你需要像在Windows图形界面里操作文件夹一样熟练。

| 命令    | 完整名称                | 作用                   | 常用示例                                                     |
| ------- | ----------------------- | ---------------------- | ------------------------------------------------------------ |
| `ls`    | list                    | 列出目录内容           | `ls -lha` (列出所有文件，含隐藏文件，以人类可读的格式显示详细信息) |
| `cd`    | change directory        | 切换目录               | `cd /var/log` (切换到日志目录), `cd ..` (返回上一级), `cd ~` (返回家目录), `cd -` (返回上次所在目录) |
| `pwd`   | print working directory | 显示当前所在路径       | `pwd`                                                        |
| `cp`    | copy                    | 复制文件或目录         | `cp source.txt dest.txt`, `cp -r sourcedir destdir` (递归复制整个目录) |
| `mv`    | move                    | 移动或重命名文件/目录  | `mv oldname.txt newname.txt` (重命名), `mv file.txt /tmp/` (移动到/tmp目录) |
| `rm`    | remove                  | 删除文件或目录         | `rm file.txt`, `rm -rf somedir` (递归强制删除目录，**慎用！**) |
| `mkdir` | make directory          | 创建目录               | `mkdir logs`, `mkdir -p /data/app/logs` (-p可以递归创建多层不存在的目录) |
| `touch` | touch                   | 创建空文件或更新时间戳 | `touch app.log`                                              |
| `find`  | find                    | 查找文件               | `find / -name "*.go"` (在根目录查找所有.go文件), `find . -type f -mtime -1` (查找当前目录一天内修改过的文件) |
| `ln`    | link                    | 创建链接               | `ln -s /path/to/real/file link_name` (创建软链接，非常常用)  |



### 二、文本处理与查看 (排查问题的利器)

作为后端开发，与你打交道最多的就是代码、配置文件和日志文件。这些命令是你的“第二双眼睛”。

| 命令   | 作用                            | 后端场景举例                                                 |
| ------ | ------------------------------- | ------------------------------------------------------------ |
| `cat`  | 查看文件全部内容                | `cat /etc/hosts` (查看hosts配置), `cat main.go` (快速查看代码) |
| `less` | 分页查看文件内容                | `less app.log` (比`cat`更适合查看大文件，支持上下滚动、搜索`/`) |
| `head` | 查看文件开头几行                | `head -n 100 app.log` (查看日志文件的前100行)                |
| `tail` | 查看文件末尾几行                | `tail -n 100 app.log` (查看最新的100行日志)<br>**`tail -f app.log` (实时跟踪日志文件增长，面试必考！)** |
| `grep` | global regular expression print | 文本搜索/过滤                                                |

#### 管道符 `|` (Linux的灵魂)

管道符 `|` 是一个“连接器”，它将前一个命令的输出，作为后一个命令的输入。这是组合使用命令、实现复杂操作的核心。

**面试高频场景:**

1. **查找特定进程:**

   Bash

   ```
   ps aux | grep "my-go-app"
   ```

   - `ps aux` 列出所有进程，`grep` 在结果中筛选出包含我们应用名字的行。

2. **统计日志中错误的数量:**

   Bash

   ```
   cat app.log | grep "FATAL" | wc -l
   ```

   - `cat` 输出日志，`grep` 筛选出严重错误，`wc -l` 统计行数。



### 三、进程管理 (应用的“心跳”）

你的Go程序编译后就是一个二进制文件，在Linux上运行时，它就是一个“进程”。你需要知道如何启动、查看、关闭它。

| 命令           | 作用             | 常用示例                                                     |
| -------------- | ---------------- | ------------------------------------------------------------ |
| `ps`           | process status   | 查看进程快照                                                 |
| `top` / `htop` | 实时显示进程动态 | `top` (实时监控系统负载、CPU和内存占用，按`P`按CPU排序，`M`按内存排序)。`htop`是`top`的增强版，更推荐。 |
| `kill`         | 终止进程         | `kill PID` (给进程发送终止信号，如 `kill 12345`)<br>**`kill -9 PID` (强制杀死进程，最后的手段)** |
| `killall`      | 按名称终止进程   | `killall my-go-app` (杀死所有名为my-go-app的进程)            |
| `nohup` & `&`  | 后台运行         | `nohup ./my-go-app &` (让程序在后台运行，即使关闭终端也不会退出) |



### 四、系统监控与信息 (了解你的“战场”)

当应用出现性能问题时，你需要判断是应用本身的问题，还是服务器资源出了问题。

| 命令     | 作用        | 常用示例               |
| -------- | ----------- | ---------------------- |
| `df`     | disk free   | 查看磁盘空间           |
| `du`     | disk usage  | 查看文件/目录大小      |
| `free`   | free memory | 查看内存使用           |
| `uname`  | unix name   | 查看系统/内核信息      |
| `lscpu`  | list cpu    | 查看CPU信息            |
| `uptime` | uptime      | 查看系统运行时间及负载 |



### 五、网络工具 (后端服务的“脉络”)

后端服务之间通过网络通信，服务也需要监听端口来提供服务。这些命令是网络问题的排查起点。

| 命令              | 作用                     | 后端场景举例       |
| ----------------- | ------------------------ | ------------------ |
| `ping`            | ping                     | 测试网络连通性     |
| `curl`            | client URL               | 发起HTTP请求       |
| `wget`            | wget                     | 下载文件           |
| `ss` / `netstat`  | socket statistics        | 查看网络连接和端口 |
| `ifconfig` / `ip` | network interface config | 查看网络接口信息   |



### 六、权限管理与压缩

| 命令            | 作用         | 常用示例            |
| --------------- | ------------ | ------------------- |
| `chmod`         | change mode  | 修改文件/目录权限   |
| `chown`         | change owner | 修改文件/目录所有者 |
| `sudo`          | superuser do | 以root权限执行命令  |
| `tar`           | tape archive | 打包与解包          |
| `zip` / `unzip` | zip/unzip    | `zip`格式的压缩解压 |



### 七、高级调试与诊断 (进阶必备)

这个命令非常强大，是解决“端口被占用”、“文件句柄过多”等疑难杂症的法宝。

| 命令   | 作用            | 后端场景举例                                                 |
| ------ | --------------- | ------------------------------------------------------------ |
| `lsof` | list open files | 列出当前系统打开的文件。在Linux中“万物皆文件”，所以这个命令功能非常强大，可以显示进程打开的文件、套接字(socket)等。 |



### 面试实战模拟

想象一下面试官会如何提问：

1. **Q: "如果我把一个编译好的Go程序传到服务器上，要怎么让它在后台一直运行？"**
   - **A:** "我会使用 `nohup ./my-app &` 命令。`nohup`可以保证在终端关闭后进程不被挂断，`&`符号让它在后台执行。"
2. **Q: "你的Go应用启动失败，提示端口8080已被占用，怎么排查？"**
   - **A:** "我会用 `lsof -i :8080` 或者 `ss -tuln | grep 8080`。`lsof`会直接告诉我占用端口的进程PID和名称，然后我可以用 `kill` 命令来处理这个进程。"
3. **Q: "服务器突然变得很卡，你的排查思路是什么？"**
   - **A:** "首先，我会用 `top` 或 `htop` 命令看一下是哪个进程的CPU或内存占用过高。然后，用 `df -h` 检查磁盘空间是否满了，用 `free -h` 检查内存使用情况。通过这些信息来定位问题是出在某个应用上还是系统资源上。"
4. **Q: "我想实时查看正在运行的应用打出的日志，用什么命令？"**
   - **A:** "用 `tail -f /path/to/your/app.log`，这个命令可以实时地把文件新增的内容输出到屏幕上，非常适合看实时日志。"
5. **Q: "应用日志里报 'too many open files' 错误，你有什么排查思路？"**
   - **A:** "这个错误说明进程打开的文件句柄数超过了系统限制。我会用 `lsof -p <PID>` 来查看这个进程到底打开了哪些文件。这能帮助我定位是代码里有文件或连接未关闭导致的泄漏，还是确实是业务需要，要去调整系统的 `ulimit` 配置。"
