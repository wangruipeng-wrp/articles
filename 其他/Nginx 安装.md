---
title: Nginx 安装和配置
abbrlink: 62048
date: 2020-07-06 21:28:40
hide: true
categories:
 - 代码段记录
---

记录一些 Nginx 的安装步骤、配置项信息等，主要为了方便自己后续查看，如果能帮到其他人就更好了。

# 安装

## 二进制安装

### 依赖安装

```
yum -y install gcc-c++ pcre pcre-devel slib zlib-devel openssl openssl-devel
```

### 下载解压

从官网[下载](http://nginx.org/en/download.html) Nginx，各个版本区别：

- Mainline version：开发版本
- Stable version：最新稳定版本
- Legacy versions：历史版本

推荐大家下载最新稳定版本：

```
wget http://nginx.org/download/nginx-1.20.1.tar.gz
```

下载完成后解压即可：

```
tar -zxvf nginx-1.20.1.tar.gz
```

### 编译安装

首先需要配置安装参数，在安装目录下通过 `configure` 文件进行配置，如果全部使用默认的配置的话可以直接运行 `./configure`。

**参数说明**

| 参数 | 说明 |
| -- | -- |
| - - prefix | 指定 nginx 安装目录 |
| - - pid-path | 指定 nginx 的pid |
| - - lock-path | 锁定安装文件，防止而已篡改或者误操作 |
| - - error-log | 错误日志 |
| - - http-log-path | http 日志 |
| - - with-http_gzip_static_module | 启用 gzip 模块，在线实时压缩输出数据流 |
| - - http-client-body-temp-path | 设定客户端请求的临时目录 |
| - - http-proxy-temp-path | 设定 http 代理临时目录 |
| - - http-fastcgi-temp-path | 设定 fastcgi 临时目录 |
| - - http-uwsgi-temp-path | 设定 uwsgi 临时目录 |
| - - http-scgi-temp-path | 设定 scgil 临时目录 |

```
./configure \
--pid-path=/var/run/nginx.pid \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log
```

如果配置 https 访问的话，需要在编译时添加 `http_v2_module` 和 `http_ssl_module` 这两个模块

```
./configure \
--pid-path=/var/run/nginx.pid \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--with-http_v2_module \
--with-http_ssl_module
```

**编译安装**

```
make && make install 
```

### 启动测试

在安装目录的 `sbin` 目录下执行 `./ngnix` 就可以将 Nginx 的服务成功启动起来。

**踩坑提醒：**

1. 如果在配置时是直接运行的 `./configure` 可能会报 `error.log` 和 `access.log` 这两个日志文件找不到，直接按照提示创建这两个文件即可启动 nginx。

2. 如果启动403错误的话，可以查看 nginx 的启动线程是哪个用户然后在配置文件中修改 user 的值为改成哪个用户。

### 环境变量

编辑 `/etc/profile` 文件，在最后添加以下代码即可

```
export PATH=$PATH:"nginx 安装目录"/sbin
```

保存退出后再执行一下 `source /etc/profile` 使配置生效。

### 常用命令

| 说明 | 命令 |
| -- | -- |
启动 | `nginx`
暴力停止 | `nginx -s stop`
平滑停止 | `nginx -s quit`
重新加载配置文件 | `nginx -s reload`
验证配置文件 | `nginx -t`

## yum 源安装

### 添加 nginx 到 yum 源

```
rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
```

### 使用 yum 安装 nginx

```
yum install -y nginx
```

### 常用命令

| 说明 | 命令 |
| -- | -- |
| 启动 | `systemctl start nginx`|
| 停止 | `systemctl stop nginx`|
| 重启 | `systemctl restart nginx`|
| 开机启动 | `systemctl enable nginx`|
| 查看 nginx 状态 | `systemctl status nginx` |
| 指定配置文件启动 | `nginx -c nginx.conf` |

### 配置文件

- 网站文件存放默认目录：`/usr/share/nginx/html`
- 网站默认站点配置：`/etc/nginx/conf.d/default.conf`
- 自定义 nginx 站点配置文件存放目录：`/etc/nginx/conf.d/`
- nginx 全局配置：`/etc/nginx/nginx.conf`

# 配置

> 小操作：  
> 为了方便管理，配置可以不直接写入 `nginx.conf` 文件。在外面写 `.conf` 配置文件，之后在 `nginx.conf` 文件中引入 `include myconf/*.conf;`

## 运行配置

```txt
user root;                # 指定运行 worker 进程的用户
worker_processes  3;      # worker 进程的数量

# 日志文件输出配置 可根据日志级别输出到不同的日志文件中
# error_log  logs/error.log;
# error_log  logs/error.log  notice;
# error_log  logs/error.log  info;

pid  logs/nginx.pid;  # 指定 pid

# 工作模式
events {
    use epoll;                 # 默认使用 epoll，采用异步非阻塞的处理方式
    worker_connections  1024;  # 每个 worker 进程的客户端最大连接数
}
```

## 传输配置

```txt
http {
    # 导入请求类型
    include       mime.types;
    default_type  application/octet-stream;

    # 日志格式，main 为格式的名称，可以通过 main 来指定输出此日志格式。
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    # 定义日志输出格式
    access_log  logs/access.log  main;

    # sendfile 高效传输文件，开启可提升传输性能，
    # 开启后才可以使用 tcp_nopush，当数据累积到一定大小后才发送，提高了效率
    sendfile      on;
    tcp_nopush    on;

    # 设置客户端与服务端请求的超时时间，保证客户端多次请求的时候不会重复建立新的连接，节约资源资源损耗
    keepalive_timeout  65;

    gzip  on; # 启用 gzip 压缩，静态资源文件压缩后传输会更快一些
}
```

**日志格式：**

名称 | 解释
-- | --
$remote_addr | 客户端IP 
$remote_user | 远程客户端用户名，一般为：'-'
$time_local | 时间和时区
$request | 请求的 url 和 method
$status | 响应状态码
$body_bytes_send | 响应客户端内容字节数
$http_referer | 记录用户从哪个链接跳转过来的
$http_user_agent | 用户所使用的代理，一般情况下为浏览器
$http_x_forwarded_for | 通过代理服务器来记录客户端的IP

## 虚拟主机配置

```txt
# 虚拟主机配置块，位于 http 块中
server {
    listen      80;         # 监听端口
    server_name localhost;  # 请求域名

    # 设置请求头，一般用于解决跨域问题
    add_header 'Access-Control_Allow_Origin' *;             # 允许跨域请求的域
    add_header 'Access-Control_Allow_Credentials' 'true';   # 允许带上 cookie 请求
    add_header 'Access-Control_Allow_Methods' *;            # 允许请求的方法（GET/POST/PUT/DELETE）
    add_header 'Access-Control_Allow_Headers' *;            # 允许请求的 header

    # 对源站点进行验证（防盗链）
    valid_referers *.wrp.cool;
    if ($valid_referers) {
        return 403;
    }

    # 请求路由映射，匹配拦截
    location / {
        root        html;           # 网站根路径（配置负载均衡之后不需要配置根路径）
        index       index.html;     # 默认首页
        expires     10s;            # 设置缓存
        # alias 可以为请求路径配置一个别名
    }

    # ssl 证书配置
    ssl on                              # 开启ssl
    ssl_certificate "ssl证书路径";       # 配置 ssl 证书
    ssl_certificate_key "ssl证书路径";   # 配置 ssl 证书密钥
    ssl_session_cache shared:SSL:10m;   # ssl 会话 cache
    ssl_session_timeout 10m;            # ssl 会话超时时间
    
    # 配置加密套件
    ssl_protocols TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
}
```

**location 的匹配规则：**
- `空格`：普通匹配
- `=`：精确匹配
- `~*`：匹配正则表达式，不区分大小写
- `~`：匹配正则表达式，区分大小写
- `^~`：以某个字符路径开头

**expires 指令**
- `expires 10s`：10s之后过期
- `expires @22h30m`：22h30m这个时间点之后过期
- `expires -1h`：一个小时之前就过期了
- `expires epoch`：关闭缓存
- `expires off`：使用浏览器默认缓存

## 负载均衡配置

```txt
http {

    # 负载均衡配置快
    upstream load_balance {
        
        # 使用 ip hash 作为负载均衡算法，使用 ip hash 算法之后不能把服务器直接移除，只能标记为 down
        ip_hash;

        # 使用 url hash 作为负载均衡算法
        hash $request_uri;

        # 使用 least_conn 作为负载均衡算法，哪台服务器连接数少就去请求哪台服务器
        least_conn;

        # 配置上游服务器
        server localhost:81;
        server localhost:82;
        server localhost:83;

        # 常链接数量，用于提高网络吞吐量，相当于连接池
        keepalive   32;
    }

    # 将一些负载均衡节点中的静态资源文件缓存在nginx服务器当中
    proxy_cache_path /usr/local/cache # 指令设置缓存保存的目录
    keys_zone=mycache:5m              # 指令设置共享缓存空间的名称和索引信息大小
    max_size=1g                       # 指令指定缓存大小
    inactive=1h                       # 指令指定超过这个时间则清理此缓存
    use_temp_path=off;                # 指令指定临时目录，使用后会影响 nginx 性能

    # 拦截请求使用负载均衡
    server {
        listen      80;         # 监听端口
        server_name localhost;  # 请求域名

        proxy_cache mycache;           # 开启并且使用缓存
        proxy_cache_valid 200 304 8h;  # 指定命中缓存返回码

        # 请求路由映射，匹配拦截
        location / {
            proxy_pass http://load_balance;
        }
    }
}
```

**upstream 模块参数：**

- `weight`：配置每台服务器的权重
- `max_conns`：配置服务器最大连接数，多个 `worker` 线程可能会超过指定值
- `slow_start`：设置服务器权重从零开始至设置值的时间（付费）
- `down`：表示该服务器已经宕机
- `backup`：表示该服务器为备用机，所有服务器都无法访问才会访问备用机
- `max_fails`：最大失败次数，超过这个次数时，默认该服务器已经宕机。
- `fail_timeout`：当超过最大失败次数时，经过多少时间去重新访问这台服务器，默认值为10s。

**以上参数使用示例：**

```txt
upstream load_balance {
    server localhost:81 weight=1 max_conns=200 slow_start=60s;
    server localhost:82 weight=2 max_fails=50 file_timeout=10s;
    server localhost:83 down;
    server localhost:84 backup;
}
```