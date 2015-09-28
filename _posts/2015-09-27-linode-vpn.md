---
layout: post
title: '基于linode搭建vpn'
date: 2015-09-27 17:51
categories: NetWork
excerpt:
---

* content
{:toc}

## 0.写在前面
1. 为什么要选择Linode
简单看了一下国外的vps的主机，这个牌子比较老，以前的价格也相当高，不过最近几年加量不加价，反而很有竞争力，$10的最低配足够满足个人日常需求了。
2. [怎么购买Linode主机](http://my.oschina.net/denglz/blog/313858)
需要注意的的是

* 只支持信用卡付款；
* 选站点的时候，建议别选日本（虽然速度真的超快），GFW对于日本的站点封的挺死的。

## 1.搭建环境
1. 我Rebuilt的是Ubuntu系统，所以根据Linode的说明手册，我们得更新一下apt-get
```
apt-get update
```
2. 安装pptpd

```
apt-get install pptpd
```

## 2.配置pptp

	1.基本配置(/etc/pptpd.conf)
		/etc/pptpd.conf文件是pptp服务器的配置文件，翻滚到文件最下面有localIp和remoteIp两项，remoteip指的是将来分配给VPN Client的IP，localip则是将来VPN Client看到的远端地址。去掉一组注释，设置好remoteip的区间：

		```
		localip 192.168.0.1
		remoteip 192.168.0.234-238,192.168.0.245
		```

		我们的系统是Ubuntu，所以需要在这个文件中指定log文件的位置，否则就需要在`/etc/pptpd.conf`中注释掉`logwtmp`	以彻底关闭log。

		```
		logfile /var/log/pptpd.log
		```
	2. 账号配置(/etc/ppp/chap-secrets)

	每个用户对应一行数据，用户名，服务器，密码和分配ip。用户名和密码都是明文的，服务器默认是pptpd，ip可以指定一个（需要在romoteip范围内），也可以用*代表随机ip。
	3. Dns配置(/etc/ppp/options)

		把添加可以使用的Dns，比如说：
		```
		ms-dns 8.8.8.8
		ms-dns 8.8.4.4
		```
	4. 开启Ipv4转发(/etc/sysctl.conf)

		编辑/etc/sysctl.conf文件，去掉`net.ipv4.ip_forward=1`前的注释,并执行`sysctl -p`使协议生效
	5. 重启pptp服务

		/etc/init.d/pptpd restart

	6. 安装并开启itables转发

		192.168.0.0/24 对应remoteip的网段

		```
		apt-get install iptables
		iptables -A FORWARD -s 192.168.0.0/24 -j ACCEPT
		iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o eth0 -j MASQUERADE
		```
	7. 保存转发规则到`/etc/iptables-rules`

		```
		iptables-save > /etc/iptables-rules
		```

		创建新文件`/etc/network/if-up.d/iptables`,并填充一下内容,这样每次网卡启动时都会重新从iptables-rules读取iptables的转发规则
		
		```
		#!/bin/sh
		iptables-restore < /etc/iptables-rules
		chmod +x /etc/network/if-up.d/iptables
		```