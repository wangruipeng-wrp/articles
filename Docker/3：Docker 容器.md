<h1>Docker 容器</h1>

> 《掘金小册-开发者必备的 Docker 实践指南》学习笔记

Docker 镜像运行起来就是 Docker 容器了，也就是一个个运行起来的软件。

# Docker 容器的生命周期

![docker容器生命周期](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/docker容器生命周期.png)

Docker 容器的生命周期分为五种状态，分别为：

- **Created：** 容器已经被创建，容器所需的相关资源已经准备就绪，但容器中的程序还未处于运行状态。
- **Running：** 容器正在运行，也就是容器中的应用正在运行。
- **Paused：** 容器已暂停，表示容器中的所有程序都处于暂停 ( 不是停止 ) 状态。
- **Stopped：** 容器处于停止状态，占用的资源和沙盒环境都依然存在，只是容器中的应用程序均已停止。
- **Deleted：** 容器已删除，相关占用的资源及存储在 Docker 中的管理信息也都已释放和移除。

# 常用命令

| 操作 | 命令| 常用参数 |
| :-- | :-- | :-- |
| 创建容器 | `docker create image-name` | `--name`（指定容器名称） |
| 启动容器 | `docker start container-name` | 
| 创建并启动容器 | `docker run image-name` | `--name`、`-d`（后台运行）<br> `--rm`（停止后删除）、`-e`（设置环境） |
| 停止容器 | `docker stop container-name` | 
| 删除容器 | `docker rm container-name` | `-v`（删除相关联的数据卷）
| 查看容器状态 | `docker ps` | `-a`（查看全部容器）
| 进入容器 | `docker exec container-name` | `-it`（当前控制台输入输出）<br> `bash`/`sh`（以哪种方式启动）

# 写时复制

在通过镜像运行容器时，并不是马上就把镜像里的所有内容拷贝到容器所运行的沙盒文件系统中，而是利用 UnionFS 将镜像以只读的方式挂载到沙盒文件系统中。只有在容器中发生对文件的修改时，修改才会体现到沙盒环境上。

容器是基于镜像启动的，在启动容器的时候并不是直接创建一个容器，而是利用镜像直接启动，当发生修改时再利用写时复制机制去修改。这样可以大幅提高容器启动速度，做到秒启动。
