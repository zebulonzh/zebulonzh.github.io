---
layout: post
title: '基于linode搭建vpn'
date: 2015-01-07 19:44
categories: NetWork
excerpt:
---

* content
{:toc}

##0.写在前面

1. 为什么要选择Linode

简单看了一下国外的vps的主机，这个牌子比较老，以前的价格也相当高，不过最近几年加量不加价，反而很有竞争力，$10的最低配足够满足个人日常需求了。

2. [怎么购买Linode主机](http://my.oschina.net/denglz/blog/313858)

需要注意的的是
* 只支持信用卡付款；
* 选站点的时候，建议别选日本（虽然速度真的超快），GFW对于日本的站点封的挺死的。

##1.搭建环境

1. 我Rebuilt的是Ubuntu系统，所以根据Linode的说明手册，我们得更新一下apt-get

``` shell
apt-get update
```
