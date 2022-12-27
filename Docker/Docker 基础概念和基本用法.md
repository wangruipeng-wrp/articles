---
title: Docker 基础概念和基本用法
abbrlink: 24100
date: 2022-09-13 16:15:21
categories:
 - 分布式
---

本文主要记录一些 Docker 中的基本概念，以及一些常用命令，主要是为了理清楚 `镜像`、`容器`、`网络`、`数据卷` 这几个概念、作用、操作。当然能够帮到大家就更好了。

# Docker 介绍

**Docker 的四大组成对象：**

- Docker 镜像：相当于软件的安装包
- Docker 容器：相当于安装完正在运行的软件
- Docker 网络：负责软件之间的相互通信
- Docker 数据卷：将软件内外的数据相互映射，提供容器内外数据交互功能

Docker 是以 服务端/客户端 的形式对外提供服务的，服务端是 docker daemon，客户端是 docker CLI。所有的镜像模块、容器模块、网络模块、数据卷模块都是实现在 docker daemon 之中的。

docker daemon 暴露了一套 RESTFul API 接口以对外提供服务，我们可以在控制台通过编写 http 请求来控制 docker daemon，但是这样太繁琐了。所以，Docker 提供了 docker CLI 可以在控制台更方便的控制 docker daemon。

# 搭建 Docker 运行环境

Docker 主要是运行在 Linux 系统中的，也有提供运行在 Windows、Mac 系统上的桌面软件，运行在 Windows、Mac 系统上的 Docker 也是搭了一层 Linux 的隔离。Windows 和 Mac 操作系统的 Docker 环境下载连接如下：

- [Windows 安装环境下载](https://docs.docker.com/desktop/install/windows-install/)
- [Mac 安装环境下载](https://docs.docker.com/desktop/install/mac-install/)

**CentOS 安装如下：**

```
sudo yum install yum-utils device-mapper-persistent-data lvm2

sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce

sudo systemctl enable docker  # 开机自启动
sudo systemctl start docker   # 启动 Docker 服务
```

安装完成，启动成功之后可以运行两条命令查看 Docker 的一些基本信息：

```
docker version  # Docker 版本信息（包含客户端和服务端的版本信息）
docker info     # Docker 详细信息（包含镜像、容器状态等）
```

**配置国内镜像源：**

修改 `/etc/docker/daemon.json` 文件（如果不存在，直接创建即可）添加以下内容：

```json
{
    "registry-mirrors": [
        "https://registry.docker-cn.com"
    ]
}
```

修改完成，重启 Docker 以生效：`systemctl restart docker`

运行 `docker info` 看看是否已经生效，大概在倒数两三行的位置可以看到 Registry Mirrors 信息。

```
Registry Mirrors:
 https://registry.docker-cn.com/
```

# Docker 镜像

在 [dockerhub](https://hub.docker.com/) 中搜索想要下载的镜像直接下载即可，有标注 `DOCKER OFFICIAL IMAGE` 字样的镜像为官方维护的镜像。也可以在控制台查看 dockerhub 的镜像信息：`docker search image-name`

下载命令：`docker pull image-name`

**镜像命名规则：** `username`/`repository`:`tag`

- **username：** 上传该镜像的用户，由官方维护的镜像无此信息
- **repository：** 镜像名称，通常是软件名
- **tag：** 镜像标签，通常是版本信息，也可以是环境需求、构建方式等信息

当我们在下载镜像时如果没有明确给出 `tag` 时，Docker 会默认使用 `latest` 作为 `tag`，也就是默认下载最新版本的镜像。

**常用命令：**

| 操作 | 命令 |
| :-- | :-- |
| 查看本地镜像信息 | `docker images` |
| 查看本地镜像的详细信息 | `docker inspect image-name` |
| 删除镜像 | `docker rmi image-name` |

**创建镜像：**

提交容器修改保存成镜像：

```
sudo docker commit -m "message" container-name your-image-name
```

创建镜像之后可以使用 `docker images` 命令查看镜像信息。创建时如果忘了指定镜像名称可以通过 `docker tag` 重新命名镜像。

**镜像导入导出：**

```
# 将镜像保存导出
docker save -o save-path/save-name1.tar your-image-name1 your-image-name2
# 将导出的镜像导入
docker load -i image-path/image-name.tar
```

**容器导入导出：**
```
# 导出容器（一次只能导出一个）
docker export -o save-name.tar your-container-name
# 导入容器（导入之后是镜像）
docker import save-name.tar your-image-name
```

# Docker 容器

Docker 镜像运行起来就是 Docker 容器了，也就是一个个运行起来的软件。

**Docker 容器的生命周期：**

![docker容器生命周期](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/docker容器生命周期.png)

Docker 容器的生命周期分为五种状态，分别为：

- **Created：**容器已经被创建，容器所需的相关资源已经准备就绪，但容器中的程序还未处于运行状态。
- **Running：**容器正在运行，也就是容器中的应用正在运行。
- **Paused：**容器已暂停，表示容器中的所有程序都处于暂停 ( 不是停止 ) 状态。
- **Stopped：**容器处于停止状态，占用的资源和沙盒环境都依然存在，只是容器中的应用程序均已停止。
- **Deleted：**容器已删除，相关占用的资源及存储在 Docker 中的管理信息也都已释放和移除。

**常用命令：**

| 操作 | 命令| 常用参数 |
| :-- | :-- | :-- |
| 创建容器 | `docker create image-name` | `--name`（指定容器名称） |
| 启动容器 | `docker start container-name` | 
| 创建并启动容器 | `docker run image-name` | `--name`、`-d`（后台运行）<br> `--rm`（停止后删除）、`-e`（设置环境） |
| 停止容器 | `docker stop container-name` | 
| 删除容器 | `docker rm container-name` | `-v`（删除相关联的数据卷）
| 查看容器状态 | `docker ps` | `-a`（查看全部容器）
| 进入容器 | `docker exec container-name` | `-it`（当前控制台输入输出）<br> `bash`/`sh`（以哪种方式启动）

**写时复制：**

在通过镜像运行容器时，并不是马上就把镜像里的所有内容拷贝到容器所运行的沙盒文件系统中，而是利用 UnionFS 将镜像以只读的方式挂载到沙盒文件系统中。只有在容器中发生对文件的修改时，修改才会体现到沙盒环境上。

容器是基于镜像启动的，在启动容器的时候并不是直接创建一个容器，而是利用镜像直接启动，当发生修改时再利用写时复制机制去修改。这样可以大幅提高容器启动速度，做到秒启动。

# Docker 网络

![docker容器网络模型](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/docker容器网络模型.png)

在 Docker 网络中，有三个比较核心的概念，也就是：`沙盒（Sandbox）`、`网络（Network）`、`端点（Endpoint）`。这三者形成了 Docker 网络的核心模型，也就是容器网络模型 ( Container Network Model )。

- **沙盒：**每个容器完全独立的网络环境，隔离了容器网络与宿主机网络。
- **网络：**Docker 内部的虚拟子网，网络内的参与者相互可见并能够进行通讯。
- **端点：**可控封闭网络环境的出入口，当端口相互配对后，就能进行数据传输了。

目前 Docker 官方为我们提供了五种 Docker 网络驱动，分别是：**Bridge Driver（默认，网桥模式）**、Host Driver、Overlay Driver、MacLan Driver、None Driver。docker daemon 中默认维护了一个 bridge 的网络，所有的容器默认都是加入到此网络中。

## 容器互联

要让一个容器连接到另外一个容器，可以在容器通过 `docker create` 或 `docker run` 创建时通过 `--link` 选项进行配置，同时也可以为连接指定一个别名，这样在代码中可以直接使用别名进行连接。如下就是创建 mysql 容器和 webapp 容器，让 webapp 容器连接至 mysql 容器中，并且在 webapp 容器中为 mysql 容器取别名为 database。

```
docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=root mysql
docker run -d --name webapp --link mysql:database webapp
```

这样在 webapp 中如果需要连接 mysql 可以这么写：

```java
String url = "jdbc:mysql://database:3306/database-name";
```

如果不是连接默认端口，那么需要在启动时通过 `--expose` 参数告诉容器暴露对应端口，使用如下：

```
docker run -d --name mysql --expose=3307 -e MYSQL_ROOT_PASSWORD=root mysql
```

可通过 `docker ps` 命令查看容器对外暴露了哪些端口。

## 创建网络

我们也可以自定义网络，形成虚拟子网的目的，**只有在同一网络之中的容器才可以互联**。

- 创建网络：`docker network create -d bridge network-name`，`-d` 指定网络类型
- 查看网络：`docker network ls`
- 删除网络：`docker network rm network-name`

创建容器时指定网络：

```
docker run -d --name your-container-name --network=network-name container-name
```

## 端口映射

将容器中的端口映射至宿主机的端口。如下：

![docker网络端口映射模型](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/docker网络端口映射模型.png)

要映射端口，我们可以在创建容器时使用 `-p` 或者是 `--publish` 参数。

使用方式：`-p <host-port>:<container-port>`

```
docker run -d --name nginx -p 80:80 nginx
```

可通过 `docker ps` 查看端口映射情况

# Docker 数据卷

数据管理使用挂载的方式，通过数据卷可以方便管理挂载的目录

## 容器数据管理

容器内的文件系统是随着容器的生命周期而创建和移除的，容器内部的数据无法被持久化存储。由于容器隔离，操作容器内部文件也变得很麻烦。Docker 解决这一问题的方式是文件挂载，将宿主操作系统中的文件挂载到容器内部，便可以让容器内外都共享这个文件。通过这种方式可以互通容器内外的文件，那么文件数据持久化以及操作容器内部文件的问题也就得到了解决。

Docker 提供了三种文件挂载的方式：Bind Mount、Volume 和 Tmpfs Mount。

![docker文件挂载](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/docker文件挂载.png)

- **Bind Mount：** 直接将宿主操作系统中的目录和文件挂载到容器内的文件系统中，形成挂载映射关系，在容器内外对文件的读写都相互可见。
    > 使用 `-v` 参数设置，示例：`-v <host-path>:<container-path>:ro`，ro（可选）表示容器内只读。为避免混淆，此处强制使用绝对路径。

- **Volume（数据卷）：** 仅指定容器内的目录，宿主操作系统挂载的目录由 Docker 进行管理，不需要关心具体挂载到了宿主操作系统中的哪里。
    > Bind Mount 模式为指定 `<host-path>` 则为 Volume。此种方式生成的数据在路径上会有 Docker 生成的 ID，所以能够自己命名：`-v name:<container-path>`

- **Tmpfs Mount：** 挂载系统内存中的一部分到容器的文件系统里，不过由于内存和容器的特征，它的存储并不是持久的，其中的内容会随着容器的停止而消失。
    > 使用 `--tmpfs` 参数挂载，`--tmpfs <host-path>`

以上创建出来的数据挂载信息，或者是数据卷信息可以通过 `docker inspect` 命令查看。

## 数据卷管理

上面对数据卷的操作都是基于容器的，多少有点不方便，Docker 提供了独立操作数据卷的方式。

**数据卷使用命令：**

| 操作 | 命令| 
| :-- | :-- |
| 创建数据卷 | `docker volume create volume-name` |
| 查看数据卷 | `docker volume ls` |
| 删除数据卷 | `docker volume rm volume-name` |
| 删除没有被引用且未命名的数据卷 | `docker volume prune -f` |

**数据卷容器：**

创建数据卷容器的方式很简单，由于不需要容器本身运行，因而我们找个简单的系统镜像都可以完成创建。

```
docker create --name appdata -v /webapp/storage ubuntu
```

在使用数据卷容器时，我们不建议再定义数据卷的名称，因为我们可以通过对数据卷容器的引用来完成数据卷的引用。之前我们提到，Docker 的 Network 是容器间的网络桥梁，如果做类比，数据卷容器就可以算是容器间的文件系统桥梁。我们可以像加入网络一样引用数据卷容器，只需要在创建新容器时使用专门的 `--volumes-from` 选项即可。

```
docker run -d --name nginx --volumes-from appdata nginx
```

引用数据卷容器时，不需要再定义数据卷挂载到容器中的位置，Docker 会以数据卷容器中的挂载定义将数据卷挂载到引用的容器中。

使用数据卷容器看起来与使用原始数据卷的方式差不多，但是数据卷容器主要是在迁移数据卷时提供方便。如果迁移数据卷至其他目录，不用修改引用容器中的数据卷，数据卷容器相当于对数据卷的路径做了一层包装或者是取了个别名的意思。

---

> **巨人的肩膀：**
> 
> - 掘金小册-开发者必备的 Docker 实践指南