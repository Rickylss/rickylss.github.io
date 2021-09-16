---
layout: post
title:  "科学上网"
subtitle: ""
date:   2019-2-20 16:56:09 +0800
tags:
  - shadowsock
categories: [others]
---

 本教程适合有相应知识、经验的同学参考。后续或补充细节，若有疑问可以给我发邮件，欢迎打扰。

 <!-- more -->

## 挑选境外服务器

境外服务器提供商有很多家，但是挑选时一定请**注意安全**，由于我们要做的事情比较特殊，因此*不建议挑选任何国内提供商在境外的机房*。我曾帮助过一个老哥在“四十大盗”的香港机房搭了这个服务，结果用了两天就受到短信警告，不过，速度是非常爽的。

以本人的经验来说，vultr 和 digital ocean 都是不错的选择，决定速度的关键之一在于机房的位置，下面我会教大家如何挑选。

大家在注册账号的时候可以通过下面链接注册，类似一个推广活动，这样我们都能够获得奖励：

vultr：[https://www.vultr.com/?ref=7791816-4F](https://www.vultr.com/?ref=7791816-4F)

Digital Ocean：[https://m.do.co/c/d1698500bee0](https://m.do.co/c/d1698500bee0)

### 机房位置

以 vultr 为例，机房位置以美国居多，日本和新加坡虽然看起来物理距离相对较近，但是实际效果却未必有那么好。

<img class="col-lg-12 col-md-12 mx-auto" src="\pictures\vultr_serverlocation.png"/>

要判断你所居住的地方购买那个机房的服务比较好，可以访问官方的[测速网站](https://www.vultrvps.com/test-server)，但是这个网站测出来的速度到底几分真假，我就不清楚了，**自己动手 ping 出来的才是最真实的**。

下面给大家提供一个好用的工具——17monipdb。

> 链接: [https://pan.baidu.com/s/1ViYNKE4ugLSQZguPtiD18w](https://pan.baidu.com/s/1ViYNKE4ugLSQZguPtiD18w ) 
>
> 提取码: 1s36 

使用这个工具就可以跟踪你到固定 ip 地址的路由信息，并且查看延迟（直接 ping 也行）。

<img class="col-lg-12 col-md-12 mx-auto" src="\pictures\17inodb_trac.png"/>

### 服务器价格

挑最便宜的！！！！这个不用考虑，相信我，你不天天下动作片，这个带宽你是用不完的，但是注意最好挑选带 IPv4 的服务器，IPv6 暂时不清楚如何开 vpn。

以 vultr 为例：

<img class="col-lg-12 col-md-12 mx-auto" src="\pictures\vultr_create.png"/>

最便宜的每个月$5，但是带宽有 1T/mo，这个够你用了。注意：*vultr 曾推出过$2.5 每月的机器，但是只有 IPv6*。

### 添加 SSH Keys

挑选好了地址和配置，最好能够添加一个 ssh key 这样下次使用 ssh 连接服务器的时候就不用去敲那段又臭又长的密码了。

<img class="col-lg-12 col-md-12 mx-auto" src="\pictures\vultr_sshkey.png"/>

## 创建并连接远程服务器

使用 MobaXterm 连接远程服务器

<img class="col-lg-12 col-md-12 mx-auto" src="\pictures\mobaXterm_ssh.png"/>

如图所示创建一个 ssh 连接，设置 ip 和 private key（1.3 里的 ssh private key），可指定用户名，也可连接后再指定。点击确定连接远程服务器。若连接不上，先 ping 一下远程服务器，确保网络通畅。

*注意：登陆远程服务器后，还要做一个确认操作，在远程服务器上 ping 一下你本地公网 IP 或者国内网站，以确定该 IP 没有被反向墙。*

> 我曾经就被反向墙过，能够连接境外服务器，但是服务器没法把信息传回来，就是因为该 IP 被墙盯上了，只能换一台服务器，换一个 IP。

## 配置服务器

安装包，开启对应端口的防火墙，在这里是 9010~9015 一共五个端口。

``` shell
yum update
yum install python-setuptools libevent python-devel openssl-devel swig
easy_install pip 
pip install gevent shadowsocks

array_port=("9010" 
            "9011"
            "9012"
            "9013"
            "9014"
            "9015")

for i in ${array_port[@]}
do
    firewall-cmd --zone=public --add-port=$i/tcp --permanent
    firewall-cmd --zone=public --add-port=$i/udp --permanent
done
firewall-cmd --reload
```

shadowsocks 配置文件`/etc/shadowsocks.json`

```json
{
    "server":"xxx.xxx.xxx.xxx",（如果是遇到阿里云这样的公网IP 无法ifcfg的填0.0.0.0）
    "local_address":"127.0.0.1",
    "local_port":1080,
    "port_password":{
         "9010":"IamThanos",
         "9011":"IamScarletWitch",
         "9012":"IamThor",
         "9013":"IamIronMan",
         "9014":"IamSpiderMan",
         "9015":"IamCaptainAmerica"
    },
    "timeout":300,
    "method":"rc4-md5",
    "fast_open": false
}
```

server 填服务器 IP 或者直接 0.0.0.0，port_password 可理解为账号，每个不同端口使用不同的密码，这样就可以控制多个用户了。

开启服务，设置开机启动服务。

```shell
$ ssserver -c /etc/shadowsocks.json -d start
$ echo 'ssserver -c /etc/shadowsocks.json -d start' >> /etc/rc.local
```

## 下载客户端

shadowsocks 的客户端可到 github 下载，其他下载渠道大多已被屏蔽。

直接搜索 shadowsocks，可找到安卓、windows、linux、mac、ios 等基本上所有系统的安装包，进入项目选择 relleases 下载最新的版本。

[https://github.com/search?utf8=%E2%9C%93&q=shadowsocks&type=](https://github.com/search?utf8=%E2%9C%93&q=shadowsocks&type=)