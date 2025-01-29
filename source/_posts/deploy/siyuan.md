---
title: 本地化思源服务器的搭建与开发
date: 2024-09-23 14:25:03
banner_img: /img/bg/pngtree-technology.jpg
tags: 运维
excerpt: 本文介绍一种采用docker方式进行部署思源笔记，最好用的开源知识库。
categories: [技术文摘, 腾讯云服务器应用]
---


# 个人知识管理系统思源的服务器搭建

# 前置准备
1. gitlab服务端（可选）
2. docker服务端（必须）
3. 创建docker映射目录 /siyuan/workspace 并设置权限
```
    chown -R 1000:1000 /siyuan/workspace
```

# docker部署

```bash
    docker pull b3log/siyuan

	docker run -d \
	--name siyuan \
	-v /siyuan/workspace:/siyuan/workspace \
	-p {{your port}}:{{your port}} \
	-u 1000:1000 \
	--restart always \
	b3log/siyuan \
	--workspace=/siyuan/workspace  --lang=zh_CN --accessAuthCode={{your password}}
```
&nbsp;&nbsp;&nbsp;&nbsp;参考文档： [https://hub.docker.com/r/b3log/siyuan](https://hub.docker.com/r/b3log/siyuan)

# 版本升级
&nbsp;&nbsp;&nbsp;&nbsp;升级思源系统前请注意备份。备份的2种方式：系统快照备份和gitlab同步备份，注意备份/data文件夹下的所有内容。

&nbsp;&nbsp;&nbsp;&nbsp;操作步骤如下：

```bash
	docker ps   # 查找对应的容器id
	docker stop {{containerId}}  
	docker container prune  # 清理关联的container
	docker images  # 查找老版本镜像
	docker image rm xxxxx  # 删除旧镜像
	docker pull b3log/siyuan
	docker run -d \
	--name siyuan \
	-v /siyuan/workspace:/siyuan/workspace \
	-p {{port}}:{{port}} \
	-u 1000:1000 \
	--restart always \
	b3log/siyuan \
	--workspace=/siyuan/workspace  --lang=zh_CN --accessAuthCode={{your password}}
```

&nbsp;&nbsp;&nbsp;&nbsp;升级完成后，查看对应版本已是否更新到。

&nbsp;&nbsp;&nbsp;&nbsp;坑点注意：如果在升级完毕后启动docker container,  访问页面发生container状态变为exit，查看是否在读写上有权限问题，示例如下：

```bash
	docker logs  # 查看container日志
	# F 2024/02/27 11:35:39 filelock.go:158: write file [/siyuan/workspace/data/storage/petal/siyuan-plugin-background-cover/bg-cover-setting.json] failed: open /siyuan/workspace/data/storage/petal/siyuan-plugin-background-cover/bg-cover-setting.json20sw65i.tmp: permission denied
```
&nbsp;&nbsp;&nbsp;&nbsp;如果遇到版本更新导致无法正常使用，需要进行回滚操作，准备多个版本的images，然后启动container。

## 配置Websockt反向代理

1. 进入openresty的usr目录： `/usr/local/openresty`
2. 在目录下touch一个conf文件，vim进行编辑，参考配置如下。

&nbsp;&nbsp;&nbsp;&nbsp;注意：需要启用ws反向代理，proxy_set_header设置，prox_pass指向本地docker对外端口，将443端口内容转到指向你的本地端口。

```nginx
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
            listen       443 ssl;
            server_name  {{your server host}};
       
            ssl_certificate /usr/local/openresty/nginx/conf/cert/server.crt; 
            ssl_certificate_key /usr/local/openresty/nginx/conf/cert/server.key; 
            ssl_session_timeout 5m;
            ssl_protocols TLSv1.2 TLSv1.3; 
            ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
            ssl_prefer_server_ciphers on;

            access_log              /var/log/nginx/siyuan.log measured;
            error_log               /var/log/nginx/siyuan.err.log notice;
            default_type            application/json;
            keepalive_timeout       75s;
            charset_types           text/plain application/json;
            charset                 utf-8;
            location / {
                    proxy_pass      http://localhost:{{your port}};
                    proxy_set_header Upgrade $http_upgrade;
                    proxy_set_header Connection $connection_upgrade;
                    proxy_set_header Host $host;
                    proxy_set_header X-Real-IP $remote_addr;
            }

    }
```
‍
‍## TODO
	
&nbsp;&nbsp;&nbsp;&nbsp;增加一些高级玩法功能： 

[X] 使用域名访问思源笔记

[ ] 编写bash脚本同步到自建的gitlab或者云仓库，同步

[ ] 自动升级脚本

  ‍

‍

‍
