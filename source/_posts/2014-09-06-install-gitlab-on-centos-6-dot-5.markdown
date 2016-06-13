---
layout: post
title: "在CentOS 6.5环境下安装GitLab"
date: 2014-09-06 11:32:15 +0800
comments: true
categories: CentOS, GitLab, Aliyun, VMWare
---
公司买了一台阿里云服务器部署项目测试坏境。看着服务器那么高的配置，便打算在上面搭建GitLab，以告别间歇性龟速的Bitbucket服务。当然喽，作为个人私有项目托管，我依然推荐使用BB。

<!--more-->

## 准备
查看服务器系统环境，以下载对应的GitLab包。
```sh
#查看内核版本
uname -r
#查看发行版本
cat /etc/redhat-release
```

根据服务器信息CentOS release 6.5 (Final)，到[GitLab | Package downloads](https://about.gitlab.com/downloads/)下载对应的[RPM](https://downloads-packages.s3.amazonaws.com/centos-6.5/gitlab-7.2.1_omnibus-1.el6.x86_64.rpm)

用curl下载比较慢，所以我改用迅雷下载，然后scp到服务器主目录下。
```sh
#打包上传
tar -zcvf gitlab.tar.gz gitlab-7.2.1_omnibus-1.el6.x86_64.rpm
scp gitlab.tar.gz user_name@ip_address:~/
```

登录服务器，解压rpm文件
```sh
ssh <YOUR_USERNAME>@<YOUR_SERVER_IP>
tar -zxvf gitlab.tar.gz
```

## 安装
```sh
sudo yum install openssh-server
sudo yum install postfix
sudo service postfix start
sudo chkconfig postfix on
sudo rpm -i gitlab-7.2.1_omnibus-1.el6.x86_64.rpm
```

## 配置
```sh
sudo -e /etc/gitlab/gitlab.rb
```

将external_url设成服务器ip地址，然后执行
```sh
sudo gitlab-ctl reconfigure
sudo lokkit -s http -s ssh
```

在浏览器地址栏输入服务器ip，以访问GitLab。管理员用户名为root，初始密码为5iveL!fe，首次登录后会要求改密码。

## 问题
当我在服务器安装之前，先在本地的虚拟机跑了一遍，很正常。但是当安装到真实的服务器时，访问GitLab遇到了502错误。

在命令行执行sudo gitlab-ctl tail可看到错误信息，原来是因为8080端口被项目测试环境占用，unicorn无法启动。

所以，很自然的想到去修改GitLab的配置文件。最终的配置信息如下：
```sh
#Change the external_url to the address your users will type in their browser
external_url 'http://<YOUR_SERVER_IP>:8888'
redis['port'] = 6379
postgresql['port'] = 5432
unicorn['port'] = 9999
nginx['listen_address'] = '<YOUR_SERVER_IP>'

#Limit backup lifetime to 7 days
gitlab_rails['backup_keep_time'] = 604800
```

修改完配置文件，再次执行sudo gitlab-ctl reconfigure，等执行完成后打开浏览器，此时应该就可以访问GitLab了。

若访问被防火墙拦截（比如我在Mac上访问虚拟机里安装的CentOS），则执行下面操作即可：
```sh
sudo vi /etc/sysconfig/iptables
```

然后加入-A INPUT -m state --state NEW -m tcp -p tcp --dport 8888 -j ACCEPT

## 更多
1. 启动GitLab组件: gitlab-ctl start
2. 停止GitLab组件: gitlab-ctl stop
3. 重启GitLab组件: gitlab-ctl restart

## 补记 Sep 19th
今天pull代码时遇到“the requested url returned error 500”这样的错误，到服务端用sudo gitlab-ctl tail查看了下得知是因为“PostgreSQL's request for a shared memory segment exceeded available memory”。

解决的办法是在Gitlab配置文件里加上下面这一行，然后reconfigure即可。
```sh
postgresql['shared_buffers'] = "400MB"
```

