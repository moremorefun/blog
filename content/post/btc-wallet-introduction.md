---
title: 比特币(btc)交易所钱包开发 - 1 - 认识数据结构
date: 2020-04-28 14:47:50+00:00
tags:
    - btc
    - wallet
    - 比特币
    - 收提币
    - 钱包
---
# 比特币记账方式

比特币和以太坊一样，也是一个账本。不过比特币和以太坊的记账方式差别还是很大的。

以太坊的记账方式是`余额`，余额以地址一一对应。

比特币的记账方式为`UTXO(Unspent Transaction Output)`，翻译出来大概是`未消费交易输出`。而地址的余额就是该地址所剩余的所有的`UTXO`金额的总和。

<!-- more -->

比特币的交易信息我们可以从区块链浏览器[blockchair](https://blockchair.com/bitcoin)上查看。

例如，我们查看地址`1ANjYHibCQ6FzagLfeXubC8SQYDfUS5wAJ`的余额为`23,969.00001200 BTC`。

![](/images/btc-wallet-introduction/address-balance.jpg)

然后我们查看这个地址的`UXTO`有4个

![](/images/btc-wallet-introduction/address-uxto.jpg)

我们把这4个`UXTO`的金额`0.00000600+0.00000600+23,968.00000000+1.00000000`加起来便是`23,969.00001200 BTC`。

**所以比特币没有余额只有`UXTO`**

# 比特币转账方式

我们再看一眼比特币的交易信息

![](/images/btc-wallet-introduction/btc-tx.jpg)

我们看到，交易分为输入和输出。输入可以有多个，输出也可以有多个。

输入的金额总和-输出的金额总和=手续费。

所以我们需要注意一种情况，比如我们有一个地址1A的UXTO金额为0.5BTC，我们想给地址1B打0.1个BTC。那么在不考虑手续费的情况下，我们则需要构造这样一个交易，**注意，程序是不会自动帮我们创建找零输出的**

![](/images/btc-wallet-introduction/make-tx.png)

# 转账详细数据

```json
{
    "txid": "766ea01f70f4259597fb6d07af42b9408dfe8147e1fecd6dfd16663054c4b1d6",
    "hash": "766ea01f70f4259597fb6d07af42b9408dfe8147e1fecd6dfd16663054c4b1d6",
    "version": 1,
    "size": 225,
    "vsize": 225,
    "weight": 900,
    "locktime": 0,
    "vin": [
        {
            "txid": "5c2e50b7cb317a36a8fa52d32846727340925cf40868ad47f66ecf9cb2350509",
            "vout": 1,
            "scriptSig": {
                "asm": "30440220344879b01bc208a5e3203505ce3be289fa3ec34e313d35219fee7bb0f4e41f9202204901ce1cf45706695f049df0da70652e98c21814f1f229bbecf6c00b32f4fccf[ALL] 02d62848a4d57a7fb4ff324202084d6ce1b0507d01a30646a96db1d9bdb8b5bb62",
                "hex": "4730440220344879b01bc208a5e3203505ce3be289fa3ec34e313d35219fee7bb0f4e41f9202204901ce1cf45706695f049df0da70652e98c21814f1f229bbecf6c00b32f4fccf012102d62848a4d57a7fb4ff324202084d6ce1b0507d01a30646a96db1d9bdb8b5bb62"
            },
            "sequence": 4294967295
        }
    ],
    "vout": [
        {
            "value": 23968,
            "n": 0,
            "scriptPubKey": {
                "asm": "OP_DUP OP_HASH160 66d5609d270a82f82fa15e74fce4f27fd178eb1d OP_EQUALVERIFY OP_CHECKSIG",
                "hex": "76a91466d5609d270a82f82fa15e74fce4f27fd178eb1d88ac",
                "reqSigs": 1,
                "type": "pubkeyhash",
                "addresses": [
                    "1ANjYHibCQ6FzagLfeXubC8SQYDfUS5wAJ"
                ]
            }
        },
        {
            "value": 61977.12686354,
            "n": 1,
            "scriptPubKey": {
                "asm": "OP_DUP OP_HASH160 4d4bbd82d5a2350a4d48922483068f56d43f00dc OP_EQUALVERIFY OP_CHECKSIG",
                "hex": "76a9144d4bbd82d5a2350a4d48922483068f56d43f00dc88ac",
                "reqSigs": 1,
                "type": "pubkeyhash",
                "addresses": [
                    "183hmJGRuTEi2YDCWy5iozY8rZtFwVgahM"
                ]
            }
        }
    ]
}
```

`vin[].txid` 为输入的`UXTO`的交易id，`vin[].vout`为交易的输出索引。这两个数据便标示了唯一一个输出。所以这里需要注意，创建交易的输入数据是以之前的交易数据为基础的，而不是以地址为基础的。

`vout`里主要包含地址和金额两个数据。需要注意的是，这里的地址是一个数组格式，因为这里涉及到多重签名。

这样我们就了解了比特币和以太坊交易的数据结构的区别，而在链的概念上他们基本是相同的。

***

![](/images/mp-qr-search-v2.png)