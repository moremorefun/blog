---
title: 以太坊(eth)简介
date: 2020-04-17 10:43:25
tags:
    - eth
    - 以太坊
---

理解以太坊(eth)需要理解其中比较重要的几个概念
- 地址和私钥
- 区块(block)和交易(transaction)
<!-- more -->
# 地址和私钥

地址和私钥相当于我们日常生活中的账户和密码，不过也不完全相同。

- 地址和私钥是一对一关系，私钥决定地址，不可修改
- 地址可公开用于身份标示，私钥不可公开用于签名交易
- 私钥可以导出地址，地址无法反推私钥

![](/images/eth-address-0.png)

我们可以用下面的代码生成一个地址

```golang
package main

import (
	"crypto/ecdsa"
	"fmt"
	"log"

	"github.com/ethereum/go-ethereum/common/hexutil"
	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	// 生成私钥
	privateKey, err := crypto.GenerateKey()
	if err != nil {
		log.Fatal(err)
	}
	privateKeyBytes := crypto.FromECDSA(privateKey)
	fmt.Println("私钥为: " + hexutil.Encode(privateKeyBytes))
	// 私钥导出地址
	publicKey := privateKey.Public()
	publicKeyECDSA, ok := publicKey.(*ecdsa.PublicKey)
	if !ok {
		log.Fatal("cannot assert type: publicKey is not of type *ecdsa.PublicKey")
	}
	address := crypto.PubkeyToAddress(*publicKeyECDSA).Hex()
	fmt.Println("地址为: " + address)
}
```

运行将看到类似的输出
```
私钥为: 0x7e311abd6aa962e388ffba96cec31053ee617c4f238ef47ad8213785db01ebba
地址为: 0xab8599e59B6B6e0BB574403a9daB6864dd94c3EE
```

# 区块(block)和交易(transaction)

以太坊(eth)本质是一个账本。账本由顺序排列的区块(block)组成，而区块(block)由若干个交易(transaction)组成。

![](/images/eth-block-1.png)

深入理解区块和交易我们可以借助eth区块链浏览器，其中比较著名的eth区块链浏览器为[etherscan](https://etherscan.io/)

我们重点关注一下交易(transaction)里的数据内容。

![](/images/eth-tx-2.jpg)

这里我们重点介绍一下 **gas和gasprice** 以及 **nonce**

## Gas和Gas Price以及手续费

Gas直接翻译是汽油的意思，在这里我们可以理解为买个交易(transaction)消耗的资源。

- Gas Limit: 在创建这个交易时，允许交易消耗的最大的gas数值，如果交易在实际执行中，所消耗的gas大于了这个值，则交易直接失败，不生效，但是手续费还是需要扣除。**单纯的eth转账交易，gas消耗固定为21000**。非固定消耗来源与合约交易，我们暂不考虑。
- Gas Used by Transaction: 交易中真实使用的gas值。在合约中，成功的交易大部分这个数值会小于Gas Limit。

Gas Price是Gas的单价。交易需要矿工打包，打包的优先级和Gas Price成正比，**注意，不是和手续费总额成正比。**也就是说，Gas Price越高，将被越早打包。这个值的单位显示一般为xxGwei。这里介绍一下eth的金额单位。
- 1 Ether = 1,000,000,000,000,000,000 wei （10的18次方）
- 1 Ether = 1,000,000,000,000,000 Kwei （10的15次方）
- 1 Ether = 1,000,000,000,000 Mwei （10的12次方）
- 1 Ether = 1,000,000,000 Gwei （10的9次方）
- 1 Ether = 1,000,000 Szabo （10的6次方）
- 1 Ether = 1,000 Finney （10的3次方）
- 1 Ether = 0.001 Kether
- 1 Ether = 0.000001 Mether
- 1 Ether = 0.000000001 Gether
- 1 Ether = 0.000000000001 Tether

手续费是本次交易实际上付出的eth金额

以截图中的交易为例，Gas为21000，Gas Price为2.6Gwei

手续费则为 
> 21000 * 2.6Gwei = 54600Gwei = 54600 * 10^9 Wei = 54600000000000 Wei = (54600000000000 / 10^18)Ether = 0.0000546Ether

真正交易的时候，我们想尽快打包的情况下如何更节省手续费呢？这是我们可以参考一个叫[ethgasstation](https://ethgasstation.info/)的网站，里面为我们提供了可以参考的Gas Price的数值，并同时提供了API可以实时查询。

![](/images/eth-gas-3.jpg)

## nonce

nonce都是相对与From而言的，也就是打款地址。nonce的出现是为了避免重放攻击。例如0xA地址广播了一笔转账给0xB的交易，因为交易数据都是公开的，谁都能拿到这个交易的数据，如果0xB再广播一次这个交易，是不是0xB会再收到一笔eth呢？

nonce就避免了这种情况的出现。

假设0xA->0xB的第一笔交易nonce为0并已经打包，0xB再广播一遍的时候，大家发现，0xA的nonce的0已经用过了，于是就忽略了这个广播，就避免了二次打款的情况。

但是有一个问题需要注意，**nonce相对与一个地址是从0开始的，并且是顺序的。在同一个地址nonce为5的交易没有被打包之前该地址nonce>5的交易永远不会被打包**。

这里还涉及到一个交易替换的问题，**比如0xA有一个nonce为1的交易是给0xB打款，`在还没有打包前`，0xA又广播了一个nonce为1但是Gas Price比之前那个交易高的给0xC打款的交易，则给0xC打款的交易会顶替给0xB打款的交易。也就是给0xB打款的交易会失效**。

以上就是关于以太坊(eth)比较简略的一个说明。

***

![](/images/mp-qr-search-v2.png)