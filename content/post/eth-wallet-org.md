---
title: 以太坊(eth)交易所钱包开发 - 3 - 零钱整理
date: 2020-04-19 19:00:25
tags:
    - eth
    - wallet
    - address
    - 以太坊
    - 收提币
    - 钱包
---
当用户充入eth后，我们需要定时把这部分资产转移到冷钱包。一方面是为了安全，一方面是为了更方便的管理资产。

# 生成交易数据

<!-- more -->

基本的流程是下面这个样子:
![](/images/eth-wallet-org/eth-org-0.png)

用户充入的eth我们都会在`t_tx`表中记录，记录的大概内容是:

| id  | 交易id | 冲币地址 | 冲币金额 | 零钱整理状态 |
| --- | ------ | -------- | -------- | ------------ |
| 1   | 0x1    | 0xa      | 0.11     | 0            |
| 2   | 0x2    | 0xb      | 0.12     | 0            |
| 3   | 0x3    | 0xa      | 0.08     | 0            |
| 4   | 0x4    | 0xc      | 0.17     | 0            |

这里我们可以看到有这样一种情况，`id` 是1和3的交易记录都是同一个地址的交易记录，为了节省手续费，我们会把这两个记录一起整理。

生成交易后，我们需要把生成的数据存入数据库，数据格式如下
```sql
CREATE TABLE `t_send` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `related_type` tinyint(4) NOT NULL COMMENT '关联类型 1 零钱整理 2 提币',
  `related_id` int(11) unsigned NOT NULL COMMENT '关联id',
  `tx_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'tx hash',
  `from_address` varchar(128) NOT NULL DEFAULT '' COMMENT '打币地址',
  `to_address` varchar(128) NOT NULL COMMENT '收币地址',
  `balance` bigint(20) NOT NULL COMMENT '打币金额 Wei',
  `balance_real` varchar(128) NOT NULL COMMENT '打币金额 Ether',
  `gas` bigint(20) NOT NULL COMMENT 'gas消耗',
  `gas_price` bigint(20) NOT NULL COMMENT 'gasPrice',
  `nonce` int(11) NOT NULL COMMENT 'nonce',
  `hex` varchar(2048) NOT NULL COMMENT 'tx raw hex',
  `create_time` bigint(20) NOT NULL COMMENT '创建时间',
  `handle_status` tinyint(4) NOT NULL COMMENT '处理状态',
  `handle_msg` varchar(1024) NOT NULL DEFAULT '' COMMENT '处理消息',
  `handle_time` bigint(20) NOT NULL COMMENT '处理时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `related_id` (`related_id`,`related_type`) USING BTREE,
  KEY `tx_id` (`tx_id`) USING BTREE,
  KEY `t_send_from_address_idx` (`from_address`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

1. 首先我们会从数据库中检索所有还没有整理的冲币记录，也就是`零钱整理状态`为0的记录。
2. 然后合并相同`冲币地址`的记录，记录为`map[冲币地址]SUM(冲币金额)`。
3. 合并完成后我们遍历这个map，每次处理map中的一个元素。
    1. 从获取`冲币地址`的私钥
    2. 因为转账都是需要eth`手续费`的，所以我们首先要判断该地址的冲币金额总和>转账所需手续费地址。
    3. 获取nonce值
       1. 从数据库中获取该这个地址记录的最大nonce，如果没有设置为-1，然后+1返回
       2. 从RPC获取该地址的nonce
       3. 返回两者的最大值
    4. 生成交易并用之前的私钥签名。
       关键代码为:
       ```golang
        // 生成tx
        var data []byte
        tx := types.NewTransaction(
            uint64(nonce),
            coldAddress,
            big.NewInt(sendBalance),
            uint64(gasLimit),
            big.NewInt(gasPrice),
            data,
        )
        chainID, err := ethclient.RpcNetworkID(context.Background())
        if err != nil {
            hcommon.Log.Warnf("RpcNetworkID err: [%T] %s", err, err.Error())
            return
        }
        signedTx, err := types.SignTx(tx, types.NewEIP155Signer(big.NewInt(chainID)), privateKey)
        if err != nil {
            hcommon.Log.Warnf("RpcNetworkID err: [%T] %s", err, err.Error())
            return
        }
        ts := types.Transactions{signedTx}
        rawTxBytes := ts.GetRlp(0)
        rawTxHex := hex.EncodeToString(rawTxBytes)
        txHash := strings.ToLower(signedTx.Hash().Hex())
       ```
    5. 遍历`冲币地址`对应的记录id
       1. 如果索引值是0，则插入待发送数据到待发送表
       2. 如果索引值非0，则插入hash id，不插入具体内容
       3. 这样做的原因是，避免被重复整理
4. 更新待处理记录状态为已生成交易。

# 发送交易

处理方式与之前类似。

1. 遍历待发送数据
   1. 根据数据库中存储的数据序列化成交易数据
        ```golang
        rawTxBytes, err := hex.DecodeString(sendRow.Hex)
		if err != nil {
			hcommon.Log.Errorf("err: [%T] %s", err, err.Error())
			return
		}
		tx := new(types.Transaction)
		err = rlp.DecodeBytes(rawTxBytes, &tx)
		if err != nil {
			hcommon.Log.Errorf("err: [%T] %s", err, err.Error())
			return
		}
        ```
   2. 调用RPC接口发送数据
       ```golang
        txHash := common.HexToHash(txHashStr)
        tx, isPending, err := client.TransactionByHash(ctx, txHash)
        if err != nil {
            return nil, err
        }
        if isPending {
            return nil, nil
        }
       ```
2. 更新待发送数据的状态为已发送

# 检测交易到账

1. 遍历发送数据中状态为已发送的数据
   1. 调用RPC检测交易处理状态，如果已经打包完成，则记录到状态更改id数组中
    ```golang
    txHash := common.HexToHash(txHashStr)
	tx, isPending, err := client.TransactionByHash(ctx, txHash)
	if err != nil {
		return nil, err
	}
	if isPending {
		return nil, nil
	}
	return tx, nil
    ```
2. 将状态更改数据中的数据更改状态为已发送成功。

以上功能便完成了零钱整理功能。


具体实现的代码已经传到了github [https://github.com/moremorefun/go-dc-wallet](https://github.com/moremorefun/go-dc-wallet)

功能入口在 
- `cmd/eth_address_org/main.go` 
- `cmd/eth_raw_tx_send/main.go` 
- `cmd/eth_raw_tx_confirm/main.go` 

***

![](/images/mp-qr-search-v2.png)