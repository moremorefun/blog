---
title: 以太坊(eth)交易所钱包开发 - 9 - erc20代币零钱整理
date: 2020-04-24 23:12:25
tags:
    - eth
    - wallet
    - erc20
    - 以太坊
    - 收提币
    - 钱包
---
# 认识erc20转账的交易信息

从etherscan中我们可以看到erc20交易的信息如下:
![](/images/eth-wallet-erc20-org/erc20-tx.jpg)

可以看到，我们在进行erc20交易的时候需要支付两笔费用:
- eth手续费
- erc20代币金额
<!-- more -->

# 分析erc20零钱整理需求
![](/images/eth-wallet-erc20-org/erc-org.jpg)

用户的冲币地址是我们生成的，新生成的地址(例如0xA)是没有eth金额的。当用户向这个地址(0xA)充入某个erc20代币后，这个地址(0xA)便之后erc20代币却没有eth余额。这时候我们如果想把冲进来的erc20整理到冷钱包，则需要先给0xA冲入一定数量的eth作为0xA转移erc20代币时需要的eth手续费。

# 详细处理流程
![](/images/eth-wallet-erc20-org/erc20-server.jpg)

生成erc20交易的关键代码:
```golang
rpcTx, err := ethclient.RpcGenTokenTransfer(
    context.Background(),
    tokenRow.TokenAddress,
    &bind.TransactOpts{
        Nonce:    big.NewInt(nonce),
        GasPrice: big.NewInt(gasPriceRow.V),
        GasLimit: uint64(erc20GasRow.V),
    },
    tokenRow.ColdAddress,
    orgInfo.TokenBalance,
)
if err != nil {
    hcommon.Log.Warnf("err: [%T] %s", err, err.Error())
    continue
}
signedTx, err := types.SignTx(rpcTx, types.NewEIP155Signer(big.NewInt(chainID)), privateKey)
if err != nil {
    hcommon.Log.Warnf("RpcNetworkID err: [%T] %s", err, err.Error())
    continue
}
ts := types.Transactions{signedTx}
rawTxBytes := ts.GetRlp(0)
rawTxHex := hex.EncodeToString(rawTxBytes)
txHash := strings.ToLower(signedTx.Hash().Hex())
```

# 整合代码

具体实现的代码已经传到了github [https://github.com/moremorefun/go-dc-wallet](https://github.com/moremorefun/go-dc-wallet)

功能入口在`cmd/erc20_tx_org/main.go` 

***

![](/images/mp-qr-search-v2.png)