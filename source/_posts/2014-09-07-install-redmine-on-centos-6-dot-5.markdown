---
layout: post
title: "在CentOS 6.5环境下安装Redmine"
date: 2014-09-07 16:33:08 +0800
comments: true
categories: CentOS, GitLab, Aliyun, VMWare
---
GitLab的Issue管理比较弱，如果开发测试为同一人的话，用用也还好。对于一个团队来讲，还是搭建个Redmine吧。

<!--more-->

## 准备
在[bitnami](http://baike.baidu.com/view/6313045.htm)上下载[Redmine](http://baike.baidu.com/view/2228665.htm)的[Linux版](https://bitnami.com/redirect/to/40137/bitnami-redmine-2.5.2-2-linux-x64-installer.run)。

## 安装
```sh
chmod +x bitnami-redmine-2.5.2-2-linux-x64-installer.run
sudo ./bitnami-redmine-2.5.2-2-linux-x64-installer.run 
```

根据相应的提示，选择语言、所需的组件等，非常简单。端口么，本人设的8081，如果启用了防火墙，或许你还需要做如下配置：
```sh
sudo vi /etc/sysconfig/iptables
```

添加-A INPUT -m state --state NEW -m tcp -p tcp --dport 8081 -j ACCEPT这条记录即可。打开浏览器访问http://<ip_address>:8081/redmine，应该就OKay了。

## 集成
要用Redmine替换GitLab内嵌的Issue跟踪功能，只需做如下配置：
```sh
sudo vi /opt/gitlab/embedded/service/gitlab-rails/config/gitlab.yml
```

然后在External issue trackers部分，配置一下Redmine地址即可：
```sh
## External issues trackers
issues_tracker:
    redmine:
      title: "Redmine"
      project_url: "http://<ip_address>:8081/redmine/projects/:issues_tracker_id"
      issues_url: "http://<ip_address>:8081/redmine/issues/:id"
      new_issue_url: "http://<ip_address>:8081/redmine/projects/:issues_tracker_id/issues/new"
```

在浏览器中访问GitLab具体某个项目，此时项目的设置页面里已经可以选择Issue跟踪系统了。我们选择Redmine，同时将Redmine里对应的项目名称填在Project name or id in issues tracker区域。

OK，Redmine的安装、与GitLab的集成，均已大功告成，在GitLab里点击Issue标签会自动跳转到Redmine。

## 后记
至于如何将每次commit消息里的Fixes #issue_id和Redmine里的issue关联，暂时还没有研究。

另外需要注意的是，直接在gitlab.yml中做的修改会随sudo gitlab-ctl reconfigure的执行而丢失。按照gitlab.yml文件顶部注释的说法，应在gitlab.rb文件中配置，但具体怎么在这个文件里配Redmine，暂时也没有研究。

"暂时没有研究"的事实是，研究了一小会儿后无功而返，也不想再花更多时间在上面，毕竟不影响我正常使用GitLab及Redmine。
