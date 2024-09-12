---
title: Jenkins部署hexo-blog项目
date: 2024-09-12 15:52:06
tags: 基础建设
---
# 使用Docker安装Jenkins
&nbsp;&nbsp;&nbsp;&nbsp;本文在腾讯云轻量级服务器上，使用Docker搭建了Jenkins，并将使用了hexo的框架的博客部署在服务器上，通过CI/CD流程实现自动化部署项目。

## 配置需求
&nbsp;&nbsp;&nbsp;&nbsp;参考官网的最小资源配置：
> Minimum hardware requirements:
> * 256 MB of RAM
> * 1 GB of drive space (although 10 GB is a recommended minimum if running Jenkins as a Docker container)

## 部署方案

1. 进入服务端控制台
2. 创建一个桥接网络jenkins：`docker network create jenkins`​
3. 为了在Jenkins节点内执行Docker命令，请使用以下docker run命令下载并运行docker:dind Docker映像：
   ```bash
    docker run \
      --name jenkins-docker \
      --rm \
      --detach \
      --privileged \
      --network jenkins \
      --network-alias docker \
      --env DOCKER_TLS_CERTDIR=/certs \
      --volume jenkins-docker-certs:/certs/client \
      --volume jenkins-data:/var/jenkins_home \
      --publish 2376:2376 \
      docker:dind \
      --storage-driver overlay2
    ```

4. 在指定的目录位置创建一个Dockerfile文件，写入以下内容，如果遇到下载网络卡顿无法下载，多执行几次。

    ```bash
    FROM jenkins/jenkins:2.462.2-jdk17
    USER root
    RUN apt-get update && apt-get install -y lsb-release
    RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
      https://download.docker.com/linux/debian/gpg
    RUN echo "deb [arch=$(dpkg --print-architecture) \
      signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
      https://download.docker.com/linux/debian \
      $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
    RUN apt-get update && apt-get install -y docker-ce-cli
    USER jenkins
    RUN jenkins-plugin-cli --plugins "blueocean docker-workflow"
    ```

5. 启动docker container，注意参数配置！此处和ngnix配置不一致会导致404或者302跳转过多等问题。

    ```bash
    docker run \
      --name jenkins \
      --restart=on-failure \
      --detach \
      --network jenkins \
      --env DOCKER_HOST=tcp://docker:2376 \
      --env DOCKER_CERT_PATH=/certs/client \
      --env DOCKER_TLS_VERIFY=1 \
      --env JENKINS_OPTS="--prefix=/jenkins" \
      --env JENKINS_ARGS="--prefix=/jenkins" \
      --publish 8082:8080 \
      --publish 50000:50000 \
      --user root \
      --env TZ="Asia/Shanghai" \
      --volume {{local_pos}}:/var/jenkins_home \
      --volume jenkins-docker-certs:/certs/client:ro \
      --volume {{local_position}}:/www/hexo \
      myjenkins-blueocean:2.462.2-1 
    ```

6. 修改openresty中的nginx配置，增加以下内容，注意和docker配置的前缀保持一致。

    ```bash
    	location /jenkins/ {
    		proxy_pass http://127.0.0.1:8082/jenkins/;
    		proxy_set_header Host $host;
    		proxy_set_header X-Real-IP $remote_addr;
    		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    ```

7. 重启openresty
8. 第一次启动需要初始密码，初始密码的位置在：`/var/jenkins_home/secrets/initialAdminPassword`​，使用命令`docker exec -it {{container_id}} /bin/bash`​进入docker内部，访问指定路径上的文件获取密码即可。

# Jenkins部署hexo-blog项目
&nbsp;&nbsp;&nbsp;&nbsp;由于项目发布的目录在挂载卷`/www/hexo`​ ，结合openresty的nginx配置情况，采用使用内置nodejs + pipeline的部署方式。

## 部署流程

1. Jenkins 从 GitLab 拉取 Hexo 项目代码。
2. Jenkins 构建 Hexo 项目，生成静态文件。
3. 将静态文件拷贝到`/www/hexo`​ 文件夹中。

## Node插件安装
&nbsp;&nbsp;&nbsp;&nbsp;这个插件自带Nodejs运行环境，可选择不用的安装版本，配合在pipeline脚本中使用。配置操作步骤如下：

1.  系统管理 -\> 全局工具配置 -\> NodeJS 安装 -> 点击新增

	配置完成后参考：​![Nodejs配置安装](/img/posts/deploy-hexo/image-20240912144803-7g8yzww.png)​

    注意Nodejs官方源下载受网络影响，最好实用镜像代替，如使用阿里镜像源：`https://mirrors.aliyun.com/nodejs-release/`​。有其他需要可以安装cnpm or yarn等其他包管理工具。

    这里起一个别名，脚本会使用到。

2. 安装完成后，进入系统设置，选择`Managed files`​
    
    如图：![Managed files](/img/posts/deploy-hexo/image-20240912145711-5rl92ey.png)​

    选择新建一个配置`Add a new Config`​ -> 然后选择 `Npm config file`​，然后点击 `Next`​。
    
    如图: ![Add a new Config](/img/posts/deploy-hexo/image-20240912145804-0qovwl6.png)​

    参考如下图配置：![npm-mirror](/img/posts/deploy-hexo/image-20240912145930-5zt4lqp.png)​

&nbsp;&nbsp;&nbsp;&nbsp;保存后node插件就安装完成了。

## 配置 Jenkins Pipeline
&nbsp;&nbsp;&nbsp;&nbsp;接下来，配置 Jenkins 来拉取GitHub上的Hexo项目代码、构建并发布到服务器的 `/www/hexo`​ 目录中。

1. 新建一个Pipeline（流水线）项目：在Jenkins UI 中，点击 "新建任务" 选择 "流水线" 类型，然后给你的项目命名。

    ​![](/img/posts/deploy-hexo/image-20240912150518-zr39obs.png)​

2. 添加配置Github基本项配置：

    ​![](/img/posts/deploy-hexo/image-20240912150748-kdxpwsw.png)​

3. 配置构建触发器，这里可以使用webhook来做，由于这块配置出现问题卡主了暂时列为TODO，后续有研究成果后再添加。

    ​![](/img/posts/deploy-hexo/image-20240912150903-catwwfp.png)​

4. 添加流水线脚本

    ​![](/img/posts/deploy-hexo/image-20240912151131-06r5l79.png)​

    ```bash
    pipeline {
        agent any
      
        tools {
            nodejs "node-20-17-0"  // 使用你在 Jenkins 中配置的 Node.js 版本名称
        }

        environment {
            GITHUB_REPO = 'git@github.com:xxxxx/hexo-blog.git'  // GitHub 仓库地址
            DEPLOY_DIR = '/www/hexo'  // 发布目录
        }

        stages {
            stage('Clone Repository') {
                steps {
                    // 克隆 GitHub 项目
                    git branch: 'main', url: "${GITHUB_REPO}", credentialsId: '{{credentialsIds}}'
                }
            }
          
            stage('Configure npm Registry') {
                steps {
                    // 切换 npm 的 registry 为阿里云镜像
                    sh 'npm config set registry https://registry.npmmirror.com'
                    // 验证 npm registry 设置
                    sh 'npm config get registry'
                }
            }

            stage('Install Dependencies') {
                steps {
                    script {
                        // 安装 Node.js 项目的依赖
                        sh 'npm install'
                        sh 'npm install hexo-cli -g'
                    }
                }
            }

            stage('Build Hexo Site') {
                steps {
                    script {
                        // 生成 Hexo 静态文件
                        sh 'hexo generate'
                    }
                }
            }
          
            stage('Deploy to Local Directory') {
                steps {
                    script {
                        // 将生成的静态文件复制到 /www/hexo 目录
                        sh "cp -r public/* ${DEPLOY_DIR}"
                    }
                }
            }
        }
    }

    ```

## 配置 GitHub SSH 凭证

1. 获取SSH秘钥
    1. 如果已存在，从 `.ssh/id_rsa`​中获取私钥
    2. 如果没有现成的 SSH 密钥

        ```bash
        ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
        ```

2. 然后将公钥添加到 GitHub 仓库的 `Settings -> Deploy keys`​。
3. **在 Jenkins 中配置 SSH 凭证**：

    打开Jenkins的"凭据管理"，添加SSH私钥凭证，用于从GitHub拉取代码。

    ​![image-20240912152105-773sk76](/img/posts/deploy-hexo/image-20240912152105-773sk76.png)​

## 测试 Jenkins Pipeline

1. 手动触发 Jenkins Pipeline 构建，确认项目能正确拉取、构建，并将静态文件发布到 `/www/hexo`​ 目录。
2. 确保 `/www/hexo`​ 目录有正确的静态文件，并配置 Nginx 或 Apache 指向这个目录。

## 关于部署踩坑问题
&nbsp;&nbsp;&nbsp;&nbsp;Q: Pipeline过程中：  npm: not found

&nbsp;&nbsp;&nbsp;&nbsp;A:  脚本中没有指定：`credentialsId: '{{credentials_id}}'`​

‍&nbsp;&nbsp;&nbsp;&nbsp;Q: 发生错误`stderr: fatal: unable to access 'https://github.com/': GnuTLS recv error (-110): The TLS connection was non-properly terminated.  `​

&nbsp;&nbsp;&nbsp;&nbsp;A: 这个错误表明 Jenkins 在拉取 GitHub 仓库时遇到了 TLS 连接的问题。通常这是由于 Jenkins 容器或环境中缺少某些网络或 SSL/TLS 库引起的。解决方案是使用ssh方式拉取代码。

‍&nbsp;&nbsp;&nbsp;&nbsp;Q：`/www/hexo`​ 没有权限，导致cp -R 操作失败，无法复制。

&nbsp;&nbsp;&nbsp;&nbsp;A:  权限提权问题，尝试了多种方案后，只能在docker run启动时，增加 --user root 参数配置，让jenkins使用root权限进行操作。唯一的缺点是安全方面有隐患，如果个人使用其实还好。

## 参考文章

1. [Jenkins部署官方文档](https://www.jenkins.io/doc/book/installing/docker/)
2. [linux环境下docker中搭建 jenkins 及自定义访问路径，利用nginx反向代理](https://blog.csdn.net/qq_44850489/article/details/129199842)
3. [docker 启动 jenkins 挂载目录权限问题 Permission denied](https://blog.51cto.com/u_1472521/6037094)
4. [Docker+Jenkins 入门教程 (包含自己遇到的一些坑跟解决方法)](https://testerhome.com/topics/11935)
5. [CI/CD 工具Jenkins的基本使用（Hexo为例）](https://marlous.github.io/2019/03/17/CICD%20%E5%B7%A5%E5%85%B7%20Jenkins%20%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%EF%BC%88Hexo%20%E4%B8%BA%E4%BE%8B%EF%BC%89)
6. [Docker+Jekins+GitHub 持续集成配置（详细操作过程）](https://blog.csdn.net/weixin_52021698/article/details/130110694)
7. [Gitlab+Jenkins+Docker+Harbor+K8s集群搭建CICD平台(持续集成部署Hexo博客Demo)](https://www.cnblogs.com/misakivv/p/18075229)
8. [Jenkins自动部署前端项目](https://bald3r.wang/2024/04/07/Jenkins%E8%87%AA%E5%8A%A8%E9%83%A8%E7%BD%B2%E5%89%8D%E7%AB%AF%E9%A1%B9%E7%9B%AE/#%E5%AE%89%E8%A3%85%E6%8F%92%E4%BB%B6-2)

‍
