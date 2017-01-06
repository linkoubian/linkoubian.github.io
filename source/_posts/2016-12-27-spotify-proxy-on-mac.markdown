---
layout: post
title: Mac Spotify 代理设置
date: 2016-12-27 09:23:52 +0800
comments: true
categories: Mac
---
Mac 为了无障碍上网，开着 Surge；想用 Spotify 听听歌，却因公司网络而无法登录，怎么办？

<!--more-->

## 应用内代理设置
开启 Surge 会自动配置 HTTP / Socket 代理。代理服务器为本机，即 127.0.0.1，端口分别为 6152 / 6153。

打开 Spotify for Mac 打开高级设置项，选择 Socket 5 代理，填写并保存代理服务器信息。

## Facebook 账号登录
如果依旧无法使用 Facebook 账号登录，请在 /etc/hosts 文件中，将 login.spotilocal.com 映射到 127.0.0.1 即可。也可以在 Spotify Web 设置设备登录密码。

如果还有问题，可以尝试借用手机热点先登录成功。

## 代理规则
有时无法播放，请查看 Spotify 是不是直接连 IP 地址。如果是的话，请在 Surge 规则文件里添加 IP-CIDR 以覆盖对应的 IP 段，终极大法。

Enjoy!