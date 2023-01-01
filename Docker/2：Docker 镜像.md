<h1>Docker 镜像</h1>

> 《掘金小册-开发者必备的 Docker 实践指南》学习笔记

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
