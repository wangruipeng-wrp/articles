<h1>Docker 网络</h1>

> 《掘金小册-开发者必备的 Docker 实践指南》学习笔记

![docker容器网络模型](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/docker容器网络模型.png)

在 Docker 网络中，有三个比较核心的概念，也就是：`沙盒（Sandbox）`、`网络（Network）`、`端点（Endpoint）`。这三者形成了 Docker 网络的核心模型，也就是容器网络模型 ( Container Network Model )。

- **沙盒：** 每个容器完全独立的网络环境，隔离了容器网络与宿主机网络。
- **网络：** Docker 内部的虚拟子网，网络内的参与者相互可见并能够进行通讯。
- **端点：** 可控封闭网络环境的出入口，当端口相互配对后，就能进行数据传输了。

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
