---
title: MySQL 安装
abbrlink: 7587
date: 2021-08-17 17:19:35
hide: true
categories:
 - 代码段记录
---

记录一下 Linux 中安装 MySQL 的步骤。

<!-- more -->

# 下载
MySQL 下载地址：[https://www.mysql.com/downloads/](https://www.mysql.com/downloads/)

1. **进入下载地址点击这个链接**
![20210818114657](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/20210818114657.png)

2. **选择 MySQL Yum Repository**
![20210818115056](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/20210818115056.png)

3. **进入这个页面之后选择合适的的版本点击 `Download`**
![20210818120024](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/20210818120024.png)

4. **右键这个链接选择“复制链接地址”到 Linux 系统中使用 `wget` 下载即可**
![20210818120520](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/20210818120520.png)

---

# 安装

1. **检查是否安装了 MySQL 或者 Mariadb，如果安装了需要先卸载**
```shell
# 检查是否安装了 MySQL
rpm -qa |grep mysql

# 卸载 MySQL
rpm -e --nodeps "上面查询出来的全部内容"
```

2. **开始安装**
```shell
# 将下载的 MySQL 文件加载进本地 yum 源中
yum -y localinstall mysql80-community-release-el7-3.noarch.rpm

# 通过 yum 源进行安装
yum -y install mysql-community-server
```

3. **启动 MySQL**
```shell
# 启动
service mysqld start

# 查看 MySQL 服务是否启动成功
ps -ef |grep mysql
```

---

# 修改密码

1. **查询 MySQL 临时密码**
```shell
grep 'temporary password' /var/log/mysqld.log
```

2. **登录 MySQL**
```shell
mysql -uroot -p'临时密码'
```

3. **修改 root 用户密码**
```shell
# 调整密码复杂度和长度
set global validate_password.policy=0;
set global validate_password.length=4;

ALTER USER 'root'@'localhost' IDENTIFIED BY "新密码";
```

# SQL的四种语言：DDL、DML、DCL、TCL

- **DDL（Data Definition Language）**
    - 数据定义语言，定义表结构跟约束
    - 关键字：CREATE、ALTER、DROP、TRUNCATE（清空表）

- **DML（Data Manipulation Language）**
    - 数据操纵语言，可操控具体数据
    - 关键字：SELECT、INSERT、UPDATE、DELETE

- **DCL（Data Control Language）**
    - 数据库控制语言，授权，角色控制等
    - 关键字：GRANT、REVOKE

- **TCL（Transaction Control Language）**
    - 事务控制语言
    - 关键字：BEGIN、COMMIT、ROLLBACK