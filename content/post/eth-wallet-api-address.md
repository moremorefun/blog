---
title: 以太坊(eth)交易所钱包开发 - 5 - 获取冲币地址
date: 2020-04-21 09:17:25
tags:
    - eth
    - wallet
    - address
    - 以太坊
    - 收提币
    - 钱包
---
# 验证请求是否合法

基础功能完成以后我们需要为外部提供服务了。这里我们对外提供RestFul API，使用`gin`来做http server。因为涉及到资产，所以我们需要对请求数据做一些验证，主要字段如需下:
```golang
var req struct {
    AppName string `json:"app_name" binding:"required"`
    Nonce   string `json:"nonce" binding:"required" validate:"max=40"`
    Sign    string `json:"sign" binding:"required"`
}
```
在数据库中，我们同样需要添加一个产品表，用来对应请求中的`AppName`字段。
```sql
CREATE TABLE `t_product` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `app_name` varchar(128) NOT NULL DEFAULT '' COMMENT '应用名',
  `app_sk` varchar(64) NOT NULL DEFAULT '' COMMENT '应用私钥',
  `cb_url` varchar(512) NOT NULL COMMENT '回调地址',
  `whitelist_ip` varchar(1024) NOT NULL DEFAULT '' COMMENT 'ip白名单',
  PRIMARY KEY (`id`),
  UNIQUE KEY `app_name` (`app_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
<!-- more -->
验证请求是否合法的流程如下:
![](/images/eth-wallet-api-address/api-check.png)

主要介绍一下签名生成算法，这个算法实际上和微信支付的签名算法一样。

## 签名生成的通用步骤如下：

1. 设所有发送或者接收到的数据为集合M，将集合M内非空参数值的参数按照参数名ASCII码从小到大排序（字典序），使用URL键值对的格式（即key1=value1&key2=value2…）拼接成字符串stringA。
特别注意以下重要规则：
   * 参数名ASCII码从小到大排序（字典序）；
   * 如果参数的值为空不参与签名；
   * 参数名区分大小写；
   * 验证调用返回或主动通知签名时，传送的sign参数不参与签名，将生成的签名与该sign值作校验。
   * 接口可能增加字段，验证签名时必须支持增加的扩展字段

2. 在stringA最后拼接上key得到stringSignTemp字符串，并对stringSignTemp进行MD5运算，再将得到的字符串所有字符转换为大写，得到sign值signValue。

## 举例说明：

假设传送的参数如下：
```
app_name： wxd930ea5d5a258f4f
nonce： ibuaiVcKdpRxkhJA
```
1. 对参数按照key=value的格式，并按照参数名ASCII字典序排序如下：
stringA="app_name=wxd930ea5d5a258f4f&nonce=ibuaiVcKdpRxkhJA";

2. 拼接API密钥：
stringSignTemp=stringA+"&key=192006250b4c09247ec02edce69f6a2d" //注：key为平台设置的密钥key
sign=MD5(stringSignTemp).toUpperCase()="30A40459EB96131C493486D8013C5D96" //注：MD5签名方式

> API接口协议中包含字段nonce，主要保证签名不可预测。

## 接口签名校验工具
[https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=20_1](https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=20_1)

# 实现接口
其中对外提供的接口有两个
1. 获取冲币地址
2. 提交提币数据

这里首先介绍`获取冲币地址接口`，前面我们生成的待使用地址的数据结构大概如下

| id  | 地址 | 使用标志 |
| --- | ---- | -------- |
| 1   | 0x1  | -1       |
| 2   | 0x2  | 0        |
| 3   | 0x3  | 0        |
| 4   | 0x4  | 0        |

在事物中，从数据库中获取一个`使用标示`为0的地址，然后把这个地址的使用标志设置为`product`的id，提交事物后将这个地址返回给接口调用者。

具体实现的代码已经传到了github [https://github.com/moremorefun/go-dc-wallet](https://github.com/moremorefun/go-dc-wallet)

功能入口在 `cmd/api/main.go` 

***

![](/images/mp-qr-search-v2.png)