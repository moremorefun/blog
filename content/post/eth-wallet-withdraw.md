---
title: 以太坊(eth)交易所钱包开发 - 4 - 提币
date: 2020-04-20 10:11:25
tags:
    - eth
    - wallet
    - address
    - 以太坊
    - 收提币
    - 钱包
---

我们需要有一个数据表来保存用户的提币请求和处理状态，基本数据内容如下:

| id  | 提币地址 | 提币金额 | 提币序列号                           | 提币状态 |
| --- | -------- | -------- | ------------------------------------ | -------- |
| 1   | 0xa      | 0.11     | c39b6ff7-b787-41a5-a3e5-d086c80acd3c | 0        |
| 2   | 0xb      | 0.12     | c39b6ff7-b787-41a5-a3e5-d086c80acd3d | 0        |
| 3   | 0xa      | 0.08     | c39b6ff7-b787-41a5-a3e5-d086c80acd3e | 0        |
| 4   | 0xc      | 0.17     | c39b6ff7-b787-41a5-a3e5-d086c80acd3f | 0        |

数据表结构具体如下:
```sql
CREATE TABLE `t_withdraw` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `to_address` varchar(128) NOT NULL DEFAULT '' COMMENT '提币地址',
  `balance_real` varchar(128) NOT NULL DEFAULT '' COMMENT '提币金额',
  `out_serial` varchar(64) NOT NULL DEFAULT '' COMMENT '提币唯一标示',
  `tx_hash` varchar(128) NOT NULL DEFAULT '' COMMENT '提币tx hash',
  `create_time` bigint(20) unsigned NOT NULL COMMENT '创建时间',
  `handle_status` int(11) NOT NULL COMMENT '处理状态',
  `handle_msg` varchar(128) NOT NULL COMMENT '处理消息',
  `handle_time` bigint(20) unsigned NOT NULL COMMENT '处理时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `out_serial` (`out_serial`),
  KEY `t_withdraw_tx_hash_idx` (`tx_hash`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

之后我们需要一个热钱包，这里面储存着部分资产用于转账给用户。

<!-- more -->

# 生成提币交易

这部分功能的大概流程是:
![](/images/eth-wallet-withdraw/eth-withdraw-0.png)

1. 获取热钱包地址和热钱包私钥
2. 获取热钱包余额
3. 获取gasPrice
4. 获取待处理提币条目
   1. 判断热钱包余额是否足够提币
   2. 获取热钱包nonce
   3. 创建提币交易
   4. 将创建的交易插入待发送队列
   5. 更改提币条目状态为已生成提币交易
   
这部分的功能代码基本上在上一篇 [以太坊(eth)交易所钱包开发 - 3 - 零钱整理](/2020/04/19/eth-wallet-org/) 中都有体现。

# 发送提币交易

这一部分流程和零钱整理中的相似，只不过在处理完成后需要找到对应的提币条目，更改其状态为`已发送`

# 检测提币交易是否已打包

这一部分流程和零钱整理中的相似，只不过在处理完成后需要找到对应的提币条目，更改其状态为`已到账`

具体实现的代码已经传到了github [https://github.com/moremorefun/go-dc-wallet](https://github.com/moremorefun/go-dc-wallet)

功能入口在 
- `cmd/eth_withdraw/main.go` 
- `cmd/eth_raw_tx_send/main.go` 
- `cmd/eth_raw_tx_confirm/main.go` 

***

![](/images/mp-qr-search-v2.png)