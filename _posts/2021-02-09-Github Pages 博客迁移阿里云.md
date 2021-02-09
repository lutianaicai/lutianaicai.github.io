---
title: Github Pages 博客迁移阿里云
date: 2021-02-09 10:15:00
categories: 技术杂谈
tags: [nginx, Travis, 博客迁移, 服务器]
---

虽然没什么人看，但是还是想把博客搭建在自己服务器上，也算是锻炼锻炼动手能力（纯折腾），也给想要迁移的同学做一个简单的分享。

### 前期准备

#### 博客打包文件夹

我的博客使用 [jekyll](https://jekyllrb.com/) 搭建

编译

```
jekyll build
```

项目中的 _site 目录就是编译后的博客

#### 服务器购买

我选择了一台阿里云 CentOS 7.3 轻量应用服务器

#### 域名绑定

购买一个喜欢的域名阿里云也有相应[支持](https://wanwang.aliyun.com/domain/), 然后进行服务器绑定。

### 服务器配置

#### 博客项目部署

把博客项目放入指定文件夹

```
scp -r _site文件夹本地目录 root@服务器公网ip:目标目录
```

#### nginx 配置

使用 nginx 管理 Web 服务

安装

```
yum install nginx
```

打开配置文件

```
vi /etc/nginx/nginx.conf
```

修改相应配置

* blog location 替换成刚才 scp 命令指定的目录也就是博客 _site 文件夹地址

```
server {
    listen       80 default_server;
    listen       [::]:80 default_server;
    server_name  _; 
    root         blog location;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    location / {
    }

    error_page 404 /404.html;
    location = /404.html {
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
    }
}
```

按 Esc 键、输入 :wq 并按 Enter 键，保存修改后的配置文件并退出编辑模式。

启动 nginx

```
systemctl start nginx
```

Tips:
我在这里有报错

```
>>> service nginx restart
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xe" for details.
```

原因是阿里云服务器已经把 80 端口占用了

解除 80 端口占用

```
fuser -k 80/tcp
```

再次启动 nginx

```
systemctl start nginx
```

这个时候浏览器输入绑定的域名就可以访问博客了，不过这个时候迁移工作只能算是进行了一半。

#### https

域名旁边的 Not Secure 好刺眼，不能忍。

阿里云可以 [申请免费DV证书](https://help.aliyun.com/document_detail/156645.html)。
根据流程提示下载证书文件

我的下好了是 xxx.key 和 xxx.pem 两个文件。如果是其他格式请[参考](https://help.aliyun.com/document_detail/42214.html)


再次打开 nginx 配置文件

```
vi /etc/nginx/nginx.conf
```

* yourdomain.com：替换成证书绑定的域名。
* cert-file-name.pem：替换成证书文件的名称。
* cert-file-name.key：替换成证书密钥文件的名称。

```
server {
    listen       443 ssl http2 default_server;
    listen       [::]:443 ssl http2 default_server;
    server_name  yourdomain.com;
    root         blog location;

    ssl_certificate "cert-file-name.pem";
    ssl_certificate_key "cert-file-name.key";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    location / {
    }

    error_page 404 /404.html;
    location = /404.html {
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
    }
}
```

设置 http 自动跳转 https

nginx 配置

```
server {
    listen 80;
    server_name yourdomain.com; #需要将yourdomain.com替换成证书绑定的域名。
    rewrite ^(.*)$ https://$host$1; #将所有HTTP请求通过rewrite指令重定向到HTTPS。
    location / {
        index index.html index.htm;
    }
}
```

更多细节请参考[在Nginx（或Tengine）服务器上安装证书](https://help.aliyun.com/document_detail/98728.html)

nginx 重新加载配置文件

```
nginx -s reload
```

这下看着顺眼多了。

### 自动部署

先给服务器开通下免密登陆，防止密码暴力破解，也方便后续自动部署的设置。已经免密的跳过此步骤。

#### 免密登陆

##### 生成 SSH 公钥

检查本机器有无生成过密钥

```
cd ~/.ssh
ls
id_rsa      id_rsa.pub  known_hosts
```

如果有 id_rsa 和 id_rsa.pub 两个文件则生成过，`.pub` 扩展名的文件就是你的公钥。如果没有则需要生成。

生成密钥

```
ssh-keygen
```

一路回车就可以

##### 本地公钥上传服务器

```
ssh-copy-id -i ~/.ssh/id_rsa.pub root@公网ip
```

然后就可以免密登陆了

```
ssh root@公网ip
```

#### Travis CI 配置

自动部署我选择[Travis CI](https://travis-ci.com/)

##### 注册

选择 github 账号注册，添加自己的博客项目仓库，开启 Travis 监听

##### 配置 .travis.yml

项目中添加一个 .travis.yml 文件，功能是告诉 Travis 需要做什么。

```
os: linux
dist: bionic

language: ruby
rvm: 2.7.0

cache: bundler

addons:
  // 服务器信任
  ssh_known_hosts: $server_ip
    
script:
  // 博客编译打包
  - JEKYLL_ENV=production bundle exec jekyll b

before_install:
  // 密钥加密储存自动生成
  - openssl aes-256-cbc -K $encrypted_2f88525daf9f_key -iv $encrypted_2f88525daf9f_iv
    -in id_rsa.enc -out ~/.ssh/id_rsa -d
  // 修改权限以防 warning
  - chmod 600 ~/.ssh/id_rsa
  
after_success:
  // 部署打包后文件夹 _site 到服务器
  - scp -o StrictHostKeyChecking=no -r "$SOURCE_PATH" "$server_user"@"$server_ip":"$BLOG_PATH"
```

服务器密钥加密这里说一下

先安装 travis

```
gem install travis
```

然后登录 travis

```
travis login

We need your GitHub login to identify you.
This information will not be sent to Travis CI, only to api.github.com.
The password will not be displayed.

Try running with --github-token or --auto if you don't want to enter your password anyway.
```

这里还是推荐 [github token](https://github.com/settings/tokens) 方式登录

然后切换到博客项目目录下

```
travis encrypt-file ~/.ssh/id_rsa --add
```

这一步就是把之前免密登陆的私钥加密保存到了 travis 
.travis.yml 文件会自动生成 

```
before_install:
  // 密钥加密储存自动生成
  - openssl aes-256-cbc -K $encrypted_2f88525daf9f_key -iv $encrypted_2f88525daf9f_iv
    -in id_rsa.enc -out ~\/.ssh/id_rsa -d
```

需要你提交到 github，提交前注意删除 `~\/.ssh/id_rsa -d` 的反斜杠

travis 项目 Settings 里也会多两个变量

![](/assets/images/2021/travis-1.png)

你也可以添加其他变量，在 .travis.yml 中直接 $变量名 进行引用。

最后，在 after_success: hook 里面执行 scp 命令就可以了

参数 `StrictHostKeyChecking=no` 要注意添加，否则报错。

更多细节[Travis官方文档](https://docs.travis-ci.com/user/tutorial/)

### 结语

到这里，每次发文章前，本地检查无误之后，只需要提交项目，几分钟后服务器就会自动更新了。

干到这才算稍微有点模样。

这一套流程下来我也踩了不少坑，分享出来如果有错误还请大家不吝赐教。

一起加油～




