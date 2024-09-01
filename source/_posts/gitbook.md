---
title: 自搭建Gitbook手册
date: 2024-08-29 16:31:35
tags: 基础建设
---

# 使用对象

	<span data-type="text" style="font-size: 19px;">对技术感兴趣，喜欢自己搭建的道友，自使用需求，用于记录用户日常的操作行为</span>

# 提前准备

* <span data-type="text" style="font-size: 19px;">一台ECS机器，可根据实际情况选择配置，推荐使用阿里云或者腾讯云进行搭建</span>
* <span data-type="text" style="font-size: 19px;">Nodejs ( version &lt;= v12.16.3 )</span>
* <span data-type="text" style="font-size: 19px;">nvm ( node包管理工具)</span>
* <span data-type="text" style="font-size: 19px;">openresty ( 集成化的ngnix, 十分好用！)</span>

【**注意**<span data-type="text" style="font-size: 19px;">】不同版本的Nodejs会影响gitserver的安装，亲测试v12.16.3有效</span>

# 搭建

	好，let's do it !

## nvm

```bash
	# 安装nvm
	$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
	$ source $NVM_DIR/nvm.sh 或者 . $NVM_DIR/nvm.sh

	# 使用
	$ nvm install v12.16.3
	$ nvm alias default v12.16.3
	$ npm config set registry https://registry.npm.taobao.org/
```

## gitbook server

```bash
$ npm i -g gitbook-cli

# 如果发现版本不一致，根据提示修复即可，如下命令可参考
$ npm i -g npm@8.0.0
$ npm audit fix
$ npm i --package-lock-only

# 查看是否安装成功
$ gitbook -V

# 初始化gitbook
$ gitbook init

# 启动server
$ gitbook server
```

    Great, 现在可以启动服务了，默认的端口是4000，这个可以在config中进行配置。

【**注意**】如果想更进一步开放给外网使用，在ecs的安全组设置中，放开4000端口的流入和流出。

### openresty

1. 编写安装脚本

```
    #! /bin/bash
  
    OPS_DIR=
    PREFIX=/usr/local/openresty
  
    #
    # The official built openresty package is recommended, see http://openresty.org/cn/linux-packages.html
    #
  
    wget -qO - https://openresty.org/package/pubkey.gpg | apt-key add -
    apt-get -y install software-properties-common
    add-apt-repository -y "deb http://openresty.org/package/ubuntu $(lsb_release -sc) main"
    apt-get update
    apt-get -y install openresty
  
    mkdir -p $PREFIX/nginx/conf/sites-enabled
    mkdir -p $PREFIX/nginx/conf/conf.d
    mkdir -p /var/log/nginx
    ln -sf $OPS_DIR/openresty/nginx.conf $PREFIX/nginx/conf/nginx.conf
  
    cat > $PREFIX/nginx/conf/proxy_params << EOF
    proxy_set_header Host \$http_host;
    proxy_set_header X-Real-IP \$remote_addr;
    proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto \$scheme;
    EOF
  
    ln -sf $OPS_DIR/openresty/nginx_logrotate /etc/logrotate.d/nginx
    # 在自己指定的文件夹路径放置脚本，例如OPS_DIR=/opt/{{自定义路径}}
```

```bash
# 创建脚本路径
$ mkdir /opt/{{自定义路径}} && cd /opt/{{自定义路径}}
$ mkdir openresty && cd openresty
$ touch install.sh
$ vim install.sh
$ 复制脚本文件保存
```

2. 编写logrotate配置

```
    /var/log/nginx/*.log {
        daily
        dateext
        dateformat -%Y%m%d
        missingok
        rotate 52
        compress
        delaycompress
        notifempty
        create 0640 www-data adm
        sharedscripts
        prerotate
                if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
                        run-parts /etc/logrotate.d/httpd-prerotate; \
                fi \
        endscript
        postrotate
                [ -s /usr/local/openresty/nginx/logs/nginx.pid ] && kill -USR1 `cat /usr/local/openresty/nginx/logs/nginx.pid`
        endscript
    }
```

```bash
$ touch nginx_logrotate
$ vim nginx_logrotate
$ 复制脚本并保存
```

3. 编写nginx配置

```
    #
    # replace the default nginx.conf in /usr/local/openresty/nginx/
    #
  
    user www-data;
    worker_processes auto;
    # pid logs/nginx.pid;
  
    events {
        worker_connections 65535;
    }
  
    http {
  
        ##
        # Basic Settings
        ##
  
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 3s 3s;
        types_hash_max_size 2048;
        server_tokens off;
  
        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;
  
        include mime.types;
        default_type application/octet-stream;
  
        ##
        # Logging Settings
        ##
  
        log_format measured '$remote_addr - $remote_user [$time_local] '
                            '"$request" $status $body_bytes_sent '
                            '"$http_referer" "$http_user_agent" '
                            '$request_time $upstream_response_time';
  
        log_format mixed    '$remote_addr - $remote_user [$time_local] '
                            '"$request" $status $body_bytes_sent '
                            '"$http_referer" "$http_user_agent" '
                            '$host $request_time';
  
        access_log /var/log/nginx/access.log mixed;
        error_log /var/log/nginx/error.log;
  
        ##
        # Gzip Settings
        ##
  
        gzip on;
        gzip_disable "msie6";
  
        gzip_vary on;
        gzip_proxied any;
        gzip_min_length 1k;
        gzip_comp_level 6;
        gzip_buffers 16 8k;
        gzip_http_version 1.1;
        gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
  
        ##
        # Cache for file/directory info
        ##
  
        open_file_cache          max=4096 inactive=5s;
        open_file_cache_valid    10s;
        open_file_cache_min_uses 1;
        open_file_cache_errors   on;
  
        ##
        # Temp pathes
        ##
  
        # client_body_temp_path /tmp/nginx_client_body_temp;
        # fastcgi_temp_path     /tmp/nginx_fastcgi_temp;
        # proxy_temp_path       /tmp/nginx_proxy_temp;
        # scgi_temp_path        /tmp/nginx_scgi_temp;
        # uwsgi_temp_path       /tmp/nginx_uwsgi_temp;
  
        ##
        # Virtual Host Configs
        ##
  
        # Deny access via unknown server_name
        # server {
        #    listen       8080;
        #    listen       80  default_server;
        #    server_name  _;
        #    access_log   /var/log/nginx/unknown.log mixed;
        #    return       444;
        #}
  
        include conf.d/*.conf;
        include sites-enabled/*;
    }
  
    include conf.d/*.mainconf;
```

```bash
$ touch nginx.conf
$ vim nginx.conf
$ 复制脚本并保存
```

4. 执行bash install.sh脚本进行安装
5. 使用openresty配置gitbook server

```
    server {
           listen       4000;
           server_name  localhost;
  
           location /wiki {
               alias /opt/yessirwiki/_book;
               index  index.html index.htm;
               autoindex on;
           }
    }
```

```bash
	# 检测openresty是否完成安装, active状态是active (running)
	$ systemctl status openresty
	$ cd /usr/local/openresty/nginx/conf/sites-enabled

	# 新建一个自己nginx配置
	$ touch yessirwiki.conf

	# 复制如如上配置, alias根据实际路径自己配置 
	# build gitbook server
	$ cd {gitbook workpath} && gitbook build

	# 重启openresty
	$ systemctl restart openresty
```

	<span data-type="text" style="font-size: 19px;">Done! 然后我们看看服务重启后，是否可以正常访问， 至此我们就搭建完成啦！</span>

# gitbook更新计划

* [X] gitbook侧边栏收起
* [ ] gitbook主题式样优化变更