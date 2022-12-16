---
title: Keepalived 实现 Nginx 的主备和互备
abbrlink: 42807
date: 2021-08-22 15:05:46
hide: true
categories:
 - 代码段记录
---

> 本文侧重点是讲解 Keepalived 的原理以及使用方式。

**简单讲解一下 Keepalived 的工作原理：**
1. 通过 VRRP 协议将虚拟IP绑定至本机的一张网卡上
2. 将 Nginx 服务器的IP隐藏起来不对用户暴露，用户直接访问虚拟IP
3. 通过虚拟IP对同一个集群内的不同节点网卡的绑定来实现控制用户访问不同的节点

### 实验环境

1. CentOS 7 虚拟机1，IP地址：`192.168.160.136`，为了方便区分取主机名为 `keep_136`
2. CentOS 7 虚拟机2，IP地址：`192.168.160.137`，为了方便区分取主机名为 `keep_137`
3. 分别安装：`nginx-1.20.1`、`keepalived-2.2.4`
4. 绑定虚拟IP为 `192.168.160.161`

### 实验准备工作

1. **安装 Nginx**
    - 详细步骤记录于：[Nginx 安装](https://www.wrp.cool/posts/62048/)


2. **安装 Keepalived**
    - **下载：**https://www.keepalived.org/download.html
    - **依赖：**`yum -y install libnl libnl-level`
    - **配置：**`./configure --prefix=/usr/local/keepalived --sysconf=/etc`
    - **安装：**`make && make install`
    - **配置文件：**`/etc/keepalived/keelalived.conf`


3. **注册 Keepalived 为系统服务**
    - 进入 Keepalived 的安装目录下的`/keepalived/etc/`，以本实验为例：
    ```
    cd ~/software/keepalived-2.2.4/keepalived/etc
    ```
    - 复制安装目录下的`init.d`目录下的`keepalived`文件拷贝到`/etc/init.d`目录中
    ```
    cp -a init.d/keepalived /etc/init.d/
    ```
    - 复制`sysconfig`目录中的`keepalive`文件至`/etc/sysconfig`目录中
    ```
    cp -a sysconfig/keepalived /etc/sysconfig/
    ```
    - 重新加载
    ```
    systemctl daemon-reload
    ```
    - 启动、关闭、重启
    ```
    systemctl start keepalived
    systemctl stop keepalived
    systemctl restart keepalived
    ```


### 主备实验步骤

1. 配置 Keepalived 主机（keep_136）
```
! Configuration File for keepalived

# 全局配置
global_defs {
    router_id keep_136          # 路由ID：当前安装 keepalived 节点主机的标识符，全局唯一
}

# 计算机节点
vrrp_instance VI_1 {
    state MASTER                # 标识当前节点为主节点或者是备节点。MASTER/BACKUP
    interface ens33             # 指定虚拟IP所绑定的本机网卡
    virtual_router_id 51        # 虚拟路由ID，保持主备节点一致即可
    priority 100                # 标识计算节点权重，当主节点宕机后权重高的节点优先成为主节点
    advert_int 1                # 心跳检测间隔时间，单位：秒

    # 认证授权的密码，防止非法节点接入
    authentication {
        auth_type PASS
        auth_pass 1111
    }

    # 虚拟IP
    virtual_ipaddress {
        192.168.160.161
    }
}
```

2. 配置 Keepalived 备用机（keep_137）
```
! Configuration File for keepalived

global_defs {
    router_id keep_137
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.160.161
    }
}
```

### 互备实验步骤

> 做完了主备之后，互备其实很简单，就是在备用机中添加一份主机的配置，在主机中添加一份备用机的配置。

1. 配置 Keepalived 主机（keep_136）
```
# 添加一个备用机节点的配置
vrrp_instance VI_2 {
    state BACKUP
    interface ens33
    virtual_router_id 52
    priority 80
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        192.168.160.162
    }
}
```

2. 配置 Keepalived 备用机（keep_137）
```
# 添加一个主机节点的配置
vrrp_instance VI_2 {
    state MASTER
    interface ens33
    virtual_router_id 52
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        192.168.160.162
    }
}
```

### 验证配置

1. 分别启动两台虚拟机中的 Keepalived 服务
2. 观察此时的虚拟IP绑定在哪台虚拟机中，可使用 `ip addr` 命令查看
3. 打开浏览器访问虚拟IP，此时由主机提供服务
4. 停止提供服务的 Keepalived
5. 浏览器中再次访问该虚拟IP，此时可见由备用机开始提供服务