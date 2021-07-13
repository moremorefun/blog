---
title: 以太坊(eth)交易所钱包开发 - 1 - 创建地址
date: 2020-04-17 14:43:25
tags:
    - eth
    - wallet
    - address
    - 以太坊
    - 收提币
    - 钱包
---

交易所收币的过程大概可以用如下的时序图说明:
![](/images/eth-wallet-address-create/sq-check.jpg)

<!-- more -->

**收提币服务**部分是我们需要自己开发的部分。

其中收币部分我们需要开发这几个功能:
- 为用户生成独立的收币地址
- 检测收币地址的到账情况
- 将收币地址到账整理到冷钱包

下面我们首先完成**为用户生成独立的收币地址**功能。

首先设计数据库结构如下
```sql
CREATE TABLE `t_address_key` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `address` varchar(64) NOT NULL COMMENT '地址',
  `pwd` varchar(512) NOT NULL COMMENT '加密私钥',
  `use_tag` int(11) NOT NULL DEFAULT '0' COMMENT '占用标志 -1 作为热钱包占用\n0 未占用\n>0 作为用户冲币地址占用',
  PRIMARY KEY (`id`),
  UNIQUE KEY `id` (`id`),
  UNIQUE KEY `t_address_key_address_idx` (`address`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

这里我们建议事先创建一定数量的地址存入数据库，当用户获取地址时直接从数据库中获取，所以在数据库中添加了`use_tag`字段
```
占用标志
-1 作为热钱包占用
0 未占用
>0 作为用户冲币地址占用
```

`pwd`建议存储对称加密过的私钥，这样减少dba获得私钥的可能性。

地址的创建流程大概时如下的样子
![](/images/eth-wallet-address-create/fl-create-address.png)

当用户请求冲币地址的时候，直接在数据库中获取一个`use_tag`为0的地址，然后充值`use_tag`为1，并将`address`返回给用户就可以了。

至于如何通过代码创建地址，参考我们之前的文章就可以[以太坊(eth)简介](/2020/04/17/eth-introduction/)

实现完整功能的代码可以从这里找到
[https://github.com/moremorefun/go-dc-wallet](https://github.com/moremorefun/go-dc-wallet)

***

![](/images/mp-qr-search-v2.png)