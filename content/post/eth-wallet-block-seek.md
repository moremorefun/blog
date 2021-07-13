---
title: 以太坊(eth)交易所钱包开发 - 2 - 检测到账
date: 2020-04-17 17:09:25
tags:
    - eth
    - wallet
    - address
    - 以太坊
    - 收提币
    - 钱包
---
之前生成地址的时候不需要和Eth的节点交互，但是从到账检测开始就需要与Eth的RPC接口交互了。

Eth的钱包节点目前比较大众的两个实现分别是
- Golang实现的[Geth](https://github.com/ethereum/go-ethereum)
- Rust实现的[parity](https://www.parity.io)

个人推荐的是`parity`

<!-- more -->

但是为了快捷测试，避免部署节点占用资源，我们可以用一个线上的免费服务[infura](https://infura.io/)。注册完成后我们可以获得`infura`提供的rpc接口地址[https://mainnet.infura.io/v3/0b359d2406a6492fb53883d46921d775](https://mainnet.infura.io/v3/0b359d2406a6492fb53883d46921d775)

Eth的RPC接口实际上是一个Http的调用，接口的参数内容可以在文档中找到[https://github.com/ethereum/wiki/wiki/JSON-RPC](https://github.com/ethereum/wiki/wiki/JSON-RPC)

测试这些RPC接口的参数和返回内容我们可以用[https://postwoman.io/](https://postwoman.io/)来。

例如，我们想查询当前区块链最新的区块数字是多少，我们可以从wiki文档中找到这个接口的说明
![](/images/eth-wallet-block-seek/rpc-block-num.jpg)
在`postwoman`中调用，查看返回结果:
![](/images/eth-wallet-block-seek/rpc-postwoman.jpg)

之后我们要理解一个概念叫`确认数`。

在区块链中存在这样一种情况，两个矿工同时打包提交了一个block，之后的矿工会在这两个不同的block基础上继续打包，系统会判断最长的链有效，并废弃没在最长的链上的区块。那么如果我们检测到了那个被废弃的交易，并为用户做了到账处理，但是被废弃了，交易所就亏了。所以一般在15个确认之后才认为最终到账。

那么这个`确认数`实际上就是当前最新的block number - 区块所在的block number的差值。

在配置表中添加配置字段`block_confirm_num`设置在经过多少次确认之后才认为到账，暂时可以设置为15。

下面我们开始处理到账检测逻辑

我们需要在数据库中储存一个值，这个值为当前处理完成的block number，可以叫做seek_num。这是一个状态字段，我们创建一个表，用于存储程序的状态数据
```sql
CREATE TABLE `t_app_status` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `k` varchar(64) NOT NULL DEFAULT '',
  `v` varchar(128) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  UNIQUE KEY `k` (`k`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

检测收币的处理流程大概如下:
![](/images/eth-wallet-block-seek/f-seek-block.png)

这里着重看一下的`eth_getBlockByNumber`返回
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "difficulty": "0x7cbda41a6a963",
    "extraData": "0x73706964657231310a011999",
    "gasLimit": "0x98112b",
    "gasUsed": "0x9800ec",
    "hash": "0x98dbf36c61fb46a746f0a4c868154335d4d664ea3e8f0251569d54942ffbbcec",
    "logsBloom": "0x408500009092282600c2000a0a12502941e30382050000086009480000a00884882820000004000004014020220e1100b3010402080030414281808e802c81a600000400040900404280082810304e0e020208544040242030e351a28a0020820020820000809010100018204c200000840054621081641904992c90901600080160f9ea9104c400103124a084020d010000200818c09704040820000050042106002040000000c21e0912c81941ac10800500c080415000006042521102010002202c862814b050002610319800000d4a70480210e040042000105a5845000520303510194007cd18010008a0004090080442088200416003101240080021d4",
    "miner": "0x04668ec2f57cc15c381b461b9fedab5d451c8f7f",
    "mixHash": "0x016ef38ff326c871108734969c1b3a2876fb1fbc17b9fa1fd3dd6f8b83e107fc",
    "nonce": "0x382fa580191b9f6a",
    "number": "0x96e50a",
    "parentHash": "0xdd99cc9147b9107580093bddaf230245e43a9ff133c7f3d42ae940db6b04a04f",
    "receiptsRoot": "0x9761f7e4b951bcbb3353c937b635e75c9223cab3dc12eddf701315f90a13d8eb",
    "sha3Uncles": "0x0839e48b84ea34460288867bef862309566145db2c37b6dc349c6c5a4c5a3930",
    "size": "0x6608",
    "stateRoot": "0x3783450469af7d80230231f72afe672e178e54d45a8c6083dddff85456cc04a6",
    "timestamp": "0x5e99731a",
    "totalDifficulty": "0x32e7e3ddd3025aec3e9",
    "transactions": [
      {
        "blockHash": "0x98dbf36c61fb46a746f0a4c868154335d4d664ea3e8f0251569d54942ffbbcec",
        "blockNumber": "0x96e50a",
        "from": "0x805035189339dc2a0b4d19867c4eaa03278e9f97",
        "gas": "0x19a28",
        "gasPrice": "0x826299e00",
        "hash": "0x0c620ff831b4d386777e83dec123ef8f514236c0d575ac7574829e4cd9ec4973",
        "input": "0xa9059cbb0000000000000000000000005041ed759dd4afc3a72b8192c143f72f4724081a0000000000000000000000000000000000000000000000000000000005e69ebf",
        "nonce": "0x0",
        "r": "0xeefa14910ad7db23ed388a8185a218ebbbfe4223932704169e6fc0191b43765e",
        "s": "0xf67d9acf4b748cbba4a40a33e4fbd1e3455876332cfd357cc84e8e6a663c1b9",
        "to": "0xdac17f958d2ee523a2206206994597c13d831ec7",
        "transactionIndex": "0x0",
        "v": "0x1b",
        "value": "0x0"
      },
      {
        "blockHash": "0x98dbf36c61fb46a746f0a4c868154335d4d664ea3e8f0251569d54942ffbbcec",
        "blockNumber": "0x96e50a",
        "from": "0xdc5ab4400d39f80d7c11fe9c3d8e78d469e6bbe1",
        "gas": "0x5208",
        "gasPrice": "0x2540be400",
        "hash": "0x1224e433eac8ca7c33056c23d5c2e8b18df041bc9f4a64e5a781c3df8140a5b4",
        "input": "0x",
        "nonce": "0x4",
        "r": "0x60de3109228d595ecdfdbd29ac5328494642ae45651c1c103b8a47376bbd5104",
        "s": "0x394f775adc20cb047ebffaabd4532501430f52d1a0ebc18a5b56949ee4ca5f63",
        "to": "0x787a5098678cbab92a656be84fc6ffb9345fbeee",
        "transactionIndex": "0x8",
        "v": "0x25",
        "value": "0x8ac7e202f94f2000"
      }
    ],
    "transactionsRoot": "0x2ab6a60513dac10d603db8c4df90f10b1ebe9d66b57fea3832b0eb7f42c3e6c5",
    "uncles": [
      "0xe4e2e48f51f84353244a1eb656897dbd89a506737e3dc76782098f38fbc2e48c"
    ]
  }
}
```

result.transactions[].value如果是0，我们不用关心，一般这是合约交易，不涉及eth转账。

如果`result.transactions[].value`不为0，我们则需取出`result.transactions[].to`的内容与数据库中的冲币地址做比较，如果在地址中，我们则将信息存入待处理数据库。

数据库结构如下:
```sql
CREATE TABLE `t_tx` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `tx_id` varchar(128) NOT NULL DEFAULT '' COMMENT '交易id',
  `from_address` varchar(128) NOT NULL DEFAULT '' COMMENT '来源地址',
  `to_address` varchar(128) NOT NULL DEFAULT '' COMMENT '目标地址',
  `balance` bigint(20) unsigned NOT NULL COMMENT '到账金额Wei',
  `balance_real` varchar(512) NOT NULL COMMENT '到账金额Ether',
  `create_time` bigint(20) unsigned NOT NULL COMMENT '创建时间戳',
  `handle_status` tinyint(4) NOT NULL COMMENT '处理状态',
  `handle_msg` varchar(128) NOT NULL DEFAULT '',
  `handle_time` bigint(20) unsigned NOT NULL COMMENT '处理时间戳',
  PRIMARY KEY (`id`),
  UNIQUE KEY `tx_id` (`tx_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
其中`tx_id`为唯一索引，这样即使我们重复插入一个交易的数据，也不会产生多次到账。

而`handle_status`为处理状态，因为在插入阶段，我们并不会给用户加资产，而是在另外的处理逻辑中，检测`handle_status`为0的数据项，用事物的方式给用户加资产并重置`handle_status`的状态。

在代码中，我们需要用golang调用eth rpc的接口, 官方有一个ethclient库[https://pkg.go.dev/github.com/ethereum/go-ethereum/ethclient?tab=doc](https://pkg.go.dev/github.com/ethereum/go-ethereum/ethclient?tab=doc)。但是这个库没有实现eth_blockNum接口，而且很难扩展，于是我直接把相关的源码下载下来，放到本地直接修改了。添加了eth_blockNum接口。
```golang
func (ec *Client) GetBlockNumber(ctx context.Context) (uint64, error) {
	var hex hexutil.Uint64
	if err := ec.c.CallContext(ctx, &hex, "eth_blockNumber"); err != nil {
		return 0, err
	}
	return uint64(hex), nil
}
```

具体实现的代码已经传到了github [https://github.com/moremorefun/go-dc-wallet](https://github.com/moremorefun/go-dc-wallet)

功能入口在 `cmd/eth_block_seek/main.go` 

里面配置读取，数据库链接，日志已经加入，有兴趣的可以看一下具体代码。

这里主要是搞明白 eth rpc 的本质是什么以及有哪些接口。这就是需要去看一下我们上面提到的接口文档了。

***

![](/images/mp-qr-search-v2.png)