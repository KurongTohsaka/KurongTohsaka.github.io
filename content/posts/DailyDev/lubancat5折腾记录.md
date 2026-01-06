---
title: "lubancat5折腾记录"
date: 2026-01-06
aliases: ["/Daily Dev"]
tags: ["Practise"]
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

## 系统烧录

原则上的指导文档：
https://doc.embedfire.com/linux/rk356x/quick_start/zh/latest/quick_start/flash_img/flash_img.html
LubanCat5的讲解：
https://doc.embedfire.com/linux/rk3588/quick_start/zh/latest/quick_start/lubancat/lubancat5.html

各种工具、镜像的下载都可以在上述链接中找到。

按照步骤进行操作，然后就会碰到两种不同的烧录方式：

1. Recovery/MASKROM
   - 注意这两种模式在LubanCat5下有实体按键的，具体见讲解链接。
   - 烧录需要用到USB OTG，注意不要把线插错位置。
   - 之后操作可以继续按照指导文档进行。
2. SD卡
   - 这种方式就是将系统镜像烧录到SD卡中，私认为这种方式是最方便、最快的。
   - 然后插到TF卡槽中即可。

不管哪种，启动后可能会发现之前的MIPI屏无法使用了，这是因为设备树没有启动：https://doc.embedfire.com/linux/rk356x/quick_start/zh/latest/quick_start/screen/screen.html#id11。可以暂时用HDMI显示器代替。

此外所有有关屏幕的设置也都可以在上面的链接找到，在此不再赘述。



## 远程连接

详情见链接https://doc.embedfire.com/linux/rk356x/quick_start/zh/latest/quick_start/board_startup/board_startup.html

建议选择SSH连接，镜像默认开启了这种方式。一种可能的连接方式：

- 电脑开启网络共享，然后通过网线连接板子；
- 之后调整双方的IP到同一子网下，然后就可以尝试SSH连接。



## 环境安装与部署

这部分可以参考HMI的用户手册，里面的很详细。

部署方式建议选择docker容器部署，部署前确认后要安装的docker版本。

可能的步骤：

1. 安装docker。这里以Ubuntu为例：

   ```shell
   sudo apt-get update
   sudo apt-get install -y \
       ca-certificates \
       curl \
       gnupg \
       lsb-release
    
   # Add Docker's official GPG key
   sudo install -m 0755 -d /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   sudo chmod a+r /etc/apt/keyrings/docker.gpg
   
   # Update the index again
   sudo apt-get update
   
   # Define the target version
   VERSION_STR="5:20.10.15~3-0~ubuntu-focal"
   sudo apt-get install -y \
       docker-ce=$VERSION_STR \
       docker-ce-cli=$VERSION_STR \
       containerd.io \
       docker-compose-plugin \
       docker-buildx-plugin
   
   # Prevent Docker from being upgraded automatically
   sudo apt-mark hold docker-ce docker-ce-cli
   
   # Start Docker service
   sudo systemctl enable --now docker
   ```

2. 按照手册进行操作，完成镜像的加载、启动，并访问`localhost:8081/admin/index`验证。



## 定时清理/root/hmi_data/db/file文件夹的脚本

假设脚本放在`/home/cat/Downloads/manage_file.sh`：

```shell
#!/bin/bash

# ----------------------------------------------------------------              
# Script Location: /home/cat/Downloads/manage_file.sh
# Description: Monitor DIRECTORY size and clean contents if > 1GB.
# Target: /root/hmi_data/db/file (Directory)
# ----------------------------------------------------------------

# 1. Configuration
# The target is now explicitly treated as a DIRECTORY
readonly TARGET_PARENT="/root/hmi_data/db"
readonly TARGET_DIR="${TARGET_PARENT}/file"
# 1GB in bytes
readonly MAX_SIZE=1073741824
# Safe guard to prevent deleting wrong paths
readonly MIN_PATH_LEN=15

# Commands
readonly CMD_DU="/usr/bin/du"
readonly CMD_RM="/usr/bin/rm"
readonly CMD_DATE="/usr/bin/date"

# 2. Logging
log() {
    echo "[$($CMD_DATE '+%Y-%m-%d %H:%M:%S')] $1"
}

# 3. Path Validation & Initialization
ensure_directory() {
    # Ensure parent exists
    if [ ! -d "$TARGET_PARENT" ]; then
        mkdir -p "$TARGET_PARENT"
    fi

    # Ensure target directory exists
    if [ ! -d "$TARGET_DIR" ]; then
        mkdir -p "$TARGET_DIR"
        log "[INFO] Created directory: $TARGET_DIR"
    fi
}

# 4. Core Logic
monitor_and_clean() {
    # Calculate directory size in bytes (-s: summary, -b: bytes)
    # cut -f1 extracts just the number
    local dir_size
    dir_size=$($CMD_DU -sb "$TARGET_DIR" | cut -f1)

    # Check if calculation succeeded
    if [[ ! "$dir_size" =~ ^[0-9]+$ ]]; then
        log "[ERROR] Failed to calculate size. Output: $dir_size"
        exit 1
    fi

    if [ "$dir_size" -ge "$MAX_SIZE" ]; then
        log "[WARN] Directory size ($dir_size bytes) exceeds limit."
        
        # SAFETY CHECK: Never run rm -rf on short variables or root
        if [ "${#TARGET_DIR}" -lt "$MIN_PATH_LEN" ]; then
             log "[FATAL] Path length sanity check failed. Aborting delete."
             exit 1
        fi

        # Execute Clean: Delete everything INSIDE the directory, but keep the directory itself
        # rm -rf /root/hmi_data/db/file/*
        $CMD_RM -rf "${TARGET_DIR:?}"/*
        
        if [ $? -eq 0 ]; then
            log "[INFO] Directory contents cleaned successfully."
        else
            log "[ERROR] Failed to clean directory."
        fi
    else
        log "[INFO] Directory size: $dir_size bytes. No action taken."
    fi
}

# 5. Execution
main() {
    ensure_directory
    monitor_and_clean
}

main "$@"
```

由于脚本涉及 `/root` 路径的操作，脚本必须以 `root` 权限执行。

1. **设置脚本执行权限**：

   ```shell
   chmod +x /home/cat/Downloads/manage_file.sh
   ```

2. **配置 Crontab**： 执行 `sudo crontab -e` 并添加以下行（设定每 5 分钟检查一次）：

   ```
   */5 * * * * /home/cat/Downloads/manage_file.sh >> /var/log/hmi_db_monitor.log 2>&1
   ```

为了确保它真的能在满 1GB 时自动清空，建议做一个**暴力测试**：

**1. 在该文件夹内创建一个 1.1GB 的模拟文件：**

```
# 在 /root/hmi_data/db/file 目录下创建一个名为 test.img 的 1.1GB 大文件
sudo fallocate -l 1100M /root/hmi_data/db/file/test_data.img
```

**2. 再次运行脚本：**

```
sudo /bin/bash /home/cat/Downloads/manage_file.sh
```

**3. 预期结果：**

- 终端会输出：`[WARN] Directory size ... exceeds limit.`
- 终端会输出：`[INFO] Directory contents cleaned successfully.`
- 再用 `ls` 检查，那个 1.1GB 的文件应该消失了。



## 快捷方式

```shell
# 1. 赋予普通用户对该路径的读取和执行权限 (rx)
sudo chmod 755 /root
sudo chmod 755 /root/hmi_data
sudo chmod 755 /root/hmi_data/db
sudo chmod 777 /root/hmi_data/db/file

# 2. 创建软链接到桌面 (名字叫 HMI_Data)
ln -s /root/hmi_data/db/file /home/cat/Desktop/HMI_Data
```

