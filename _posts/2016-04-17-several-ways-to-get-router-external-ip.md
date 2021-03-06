---
layout: post
title: 内网机器获取路由器外网IP的方法  
date: 2016-04-17
categories:
- tool
- funny
tags: [funny,toss]
status: publish
type: post
published: true
author:
  login: slayer
  email: dongyado@gmail.com
  display_name: slayer
---

最近一直在折腾，利用arduino做了一个远程控制开门的东西，流程如下：
手机->访问路由器->路由器转发到内网的一台电脑（web服务器）->调用开门api->api发送开门信号->arduino->开门

我是直接通过 http://路由器ip/arduino/?action=open 来调用的，虽然电信给了我一个外网ip，但是经常变，所以在ip变的时候得到一个通知。

由于内网机器没法直接拿到路由器的外网IP，所以只能通过特殊的方法去拿。也可以采用花生壳，但是速度巨慢，所以就找了其他方法，自己也分析想出了几个方法。

### 内网机器访问外网的一个服务，服务拿到访问IP并返回
网上几乎全部都是这种解决方法。
比如ip.cn，这些网站提供了获取来源地址并返回的功能，内网机器要做的就是定期的去访问一下外网的服务器，拿到响应数据中的来源IP，IP改变了则发个邮件，或者通过其他方式通知就行了。
但是访问其他服务有可能会被封禁，所以如果自己有服务器或者虚拟主机，完全可以在自己的服务中加一个API供调用就行。


### 本机轮询
运营商分配给路由器的IP一般会固定在一个IP段内，比如219.133.xxx.1-219.133.xxx.255中间，这就为轮询提供了条件。
首先在内网的一台机器（加入192.168.1.102）提供一个特殊HTTP api 供调用，我们在路由器中设置一个端口转发，比如把路由器88端口（80端口电信不能用）转发到192.168.1.102的80端口，这样我们就可以通过路由器ip:88直接访问内网192.168.1.102的web服务了。

现在在192.168.1.102上跑一个定时任务，每次对219.133.xxx.1-219.133.xxx.255的每个ip,调用一下那个特殊的api, 比如 http:// 219.133.xxx.2:88/path/to/api。
如果可以调用，返回的数据也是对的，说明肯定就是自己的路由器IP，但是要注意超时问题，其实这种调用1s之内就应该返回数据，如果无法返回，说明IP不对。

ip完全可以通过外网IP推断出来，也可以通过traceroute拿到，一般第2个就是运营商的机器地址，traceroute只会显示219.133.xxx.1。

    slayer@dongyado:~$ traceroute dongyado.com
    traceroute to dongyado.com (192.30.252.153), 30 hops max, 60 byte packets
    1  192.168.1.1 (192.168.1.1)  2.082 ms  5.627 ms  5.553 ms
    2  219.133.xxx.1 (219.133.xxx.1)  134.392 ms  134.398 ms  134.383 ms
    3  183.56.68.45 (183.56.68.45)  31.544 ms  31.551 ms  35.154 ms


### 最折腾的方法，脚本模拟登录路由器面板，从面板页面拿到IP
路由器的管理面板肯定是有外网IP的，只要找到模拟登录的方法，定期去拿，肯定是能拿到的，为啥会有这个方法，因为喜欢折腾，所以最后我采用的是这个方法。

自己用的tplink的路由器，tplink做了一下加密，断点几个小时才真正摸清的它的会话机制。

然后重写了一些加密函数，直接用curl访问就行了。

我用的路由器的型号是TL-WR842N，软件版本是2.3.8 Build 150425 Rel.60768n，需要源码的可以联系我dongyado#gmail.com。

最简单的方法就是第一种了，不喜欢依赖其他服务的（有强迫症的），可以用第二种或者第三种，第三种的风险大，说不定下次就换了其他品牌的路由器，会话机制以及加密方法全变了，得重写一个。

但是如果喜欢折腾，毫无疑问，第三种最适合你。
