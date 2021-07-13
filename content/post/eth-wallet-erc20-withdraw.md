---
title: 以太坊(eth)交易所钱包开发 - 10 - erc20代币提币
date: 2020-04-26 9:39:25
tags:
    - eth
    - wallet
    - erc20
    - 以太坊
    - 收提币
    - 钱包
---
# 数据结构的一些说明

erc20的提币和eth的提币处理方式基本相同，只是生成交易数据的逻辑有些区别。但是我们要注意一种情况，就是多个erc20的代币热钱包可能是相同的，这样我们在查询代币热钱包余额的时候需要查询不同代币的余额。

比如代币表数据为

| 代币id | 代币合约地址 | 代币单位 | 热钱包地址 |
| ------ | ------------ | -------- | ---------- |
| 1      | 0xA          | usdt     | 0x1        |
| 2      | 0xB          | tcp      | 0x1        |
| 3      | 0xC          | pc       | 0x1        |

提币数据为

| 提币id | 代币单位 | 提币地址 | 提币金额 |
| ------ | -------- | -------- | -------- |
| 1      | usdt     | 0x2      | 0.11     |
| 2      | tcp      | 0x3      | 1.23     |
| 3      | pc       | 0x4      | 9.8      |

所以我们获取热钱包0x1的代币余额的时候需要生成类似这样一个map, `map[热钱包地址-代币id] = 热钱包余额`的结构

<!-- more -->

# 处理流程

![](/images/eth-wallet-erc20-withdraw/erc20-withdraw.jpg)

# 关键代码

## 获取token余额

```golang
// RpcTokenBalance 获取token余额
func RpcTokenBalance(ctx context.Context, tokenAddress string, address string) (int64, error) {
	tokenAddressHash := common.HexToAddress(tokenAddress)
	instance, err := NewEth(tokenAddressHash, client)
	if err != nil {
		return 0, err
	}
	balance, err := instance.BalanceOf(&bind.CallOpts{}, common.HexToAddress(address))
	if err != nil {
		return 0, err
	}
	return balance.Int64(), nil
}
```

## 生成token交易

```golang
// RpcGenTokenTransfer 生成token转账交易
func RpcGenTokenTransfer(ctx context.Context, tokenAddress string, opts *bind.TransactOpts, to string, balance int64) (*types.Transaction, error) {
	address := common.HexToAddress(tokenAddress)
	instance, err := NewEth(address, client)
	if err != nil {
		return nil, err
	}
	tx, err := instance.Transfer(opts, common.HexToAddress(to), big.NewInt(balance))
	if err != nil {
		return nil, err
	}
	return tx, nil
}

// 获取nonce值
nonce, err := GetNonce(dbTx, hotAddress)
if err != nil {
    return err
}
rpcTx, err := ethclient.RpcGenTokenTransfer(
    context.Background(),
    tokenRow.TokenAddress,
    &bind.TransactOpts{
        Nonce:    big.NewInt(nonce),
        GasPrice: big.NewInt(gasPrice),
        GasLimit: uint64(gasLimit),
    },
    withdrawRow.ToAddress,
    tokenBalance,
)
if err != nil {
    return nil
}
signedTx, err := types.SignTx(rpcTx, types.NewEIP155Signer(big.NewInt(chainID)), key)
if err != nil {
    return err
}
ts := types.Transactions{signedTx}
rawTxBytes := ts.GetRlp(0)
rawTxHex := hex.EncodeToString(rawTxBytes)
txHash := strings.ToLower(signedTx.Hash().Hex())
```

# 整合代码

具体实现的代码已经传到了github [https://github.com/moremorefun/go-dc-wallet](https://github.com/moremorefun/go-dc-wallet)

功能入口在`cmd/erc20_withdraw/main.go` 

***

![](/images/mp-qr-search-v2.png)