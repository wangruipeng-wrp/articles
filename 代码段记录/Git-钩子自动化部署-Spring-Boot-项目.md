---
title: Git 钩子自动化部署 Spring Boot 项目
abbrlink: 19410
date: 2021-08-17 17:20:20
hide: true
categories:
 - 代码段记录
---

开发过程中，接口开发完毕给测试人员做测试的时候会很经常的需要将一些测试提交的修改更新到测试服务器上去，但是如果每次提交都手动的去测试服务器打包代码，重启服务的话，那太麻烦了。

之前在 Linux 服务器上部署这个博客的时候有了解到 git 里面有一个钩子的东西可以做这种自动化的部署，于是最近研究了一下针对上面的问题可以使用 git 的钩子来做一个项目的自动部署，这样就不用每次都去手动的部署了，要更新的时候只需要把代码提交到 git 上就可以完成自动化部署。

接下来详细说一下部署的过程，也算是做一个记录，方便以后查看。

---

<font size=4>**在测试环境上搭建git仓库**</font>

1. **安装git**
```shell
yum -y install git
```

2. **创建一个 git 用户并且设置密码**
```shell
useradd git
passwd git
```

3. **选定一个目录作为git仓库，并初始化这个git仓库**
```shell
# 创建一个 project.git 的目录，并初始化为 git 仓库
git init --bare projuce.git
```

4. **将本地的 ssh 公钥部署到服务器上**
```shell
# 1) 创建 ssh 公钥
ssh-keygen -t rsa -C "你的邮箱"

# 2) 创建 authorized_keys 文件
touch /home/git/.ssh/authorized_keys

# 3) 将本地创建的公钥复制到 authorized_keys 中，一行一个
```

5. **本地项目添加测试服务器的 git 远程仓库地址**
```shell
git remote add origin git@'测试服务器IP':'仓库路径'
```
接下来就是在本地正常的提交代码到测试服务器的 git 仓库了，就像是平时开发一样提交即可。

---

<font size=4>**将代码部署到测试服务器**</font>

1. **检出代码**
```shell
git --work-tree='要发布的目录' --git-dir='远程仓库地址' checkout -f
```

2. **打包**
```shell
mvn package
```

3. **启动**
```shell
java -jar xxx.jar
```

<font size=4>**自动化部署到测试服务器**</font>

自动化部署的流程跟以上是一样的，只是利用了 git 的钩子来自动的执行一个部署的脚本，以此免去了人工手动部署的工作。

1. **在测试服务器上的 git 远程仓库中有一个 `hooks` 文件夹，在这个文件夹中创建 `post-receive` 文件**
> 这个文件就是钩子，当我们的代码提交到这个远程仓库中就会触发这个文件的执行，于是我们就可以把代码部署的脚本写在这个文件中，利用这个来实现自动部署。

2. **赋予 `post-receive` 可执行权限**
```shell
chmod +x post-receive
```

3. **自动部署代码脚本**
```shell
#!/bin/sh

echo "删除项目目录"
rm -rf "项目目录"

echo "创建项目目录"
mkdir "项目目录"

echo "拉取代码"
git --work-tree='项目根目录' --git-dir='远程仓库地址' checkout -f

echo "进入项目根目录"
cd ~/foodie-prod/foodie/

echo "maven 打包"
/usr/local/apache-maven-3.8.1/bin/mvn package # 此处 maven 打包需要使用全路径

echo "停止正在运行的 Spring Boot"
appid=`ps -ef |grep java|grep foodie|awk '{print $2}'`
kill $appid

echo "进入 jar 包所在路径"
cd "jar 包所在路径"

echo "后台启动 Spring Boot"
nohup java -jar -Dspring.profiles.active="配置文件" foodie-api-0.0.1-SNAPSHOT.jar > ~/temp.log 2>&1 &

echo "休眠 10s 等待 Spring Boot 启动"
sleep 10

echo "自动化发布脚本执行结束！"
```
> 注意：
> 以上脚本在编写时应该对每一步都进行校验以保证最终成功运行。
> 钩子的运行日志可以提交代码时查看。