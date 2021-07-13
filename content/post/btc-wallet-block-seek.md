---
title: 比特币(btc)交易所钱包开发 - 3 - 检测交易
date: 2020-05-08 14:25:50+00:00
tags:
    - btc
    - wallet
    - 比特币
    - 收提币
    - 钱包
---
# 钱包节点

检测交易就需要调用比特币钱包的RPC接口了，所以我们首先要部署一个比特币钱包节点。官方的节点在[https://bitcoin.org/en/bitcoin-core/](https://bitcoin.org/en/bitcoin-core/)。但是为了之后同时支持USDT之类的比特币代币，我们这里使用[Omni CORE](https://www.omnilayer.org/download.html)。`Omni CORE`和`bitcoin-core`是兼容了，并且`bitcoin-core`是`Omni CORE`的子集。

<!-- more -->

# 钱包节点RPC接口

RPC接口的说明文件可以从这里找到[https://bitcoin.org/en/developer-reference#bitcoin-core-apis](https://bitcoin.org/en/developer-reference#bitcoin-core-apis)

例如获取最新的区块数

GetBlockCount

The getblockcount RPC returns the number of blocks in the longest blockchain.

Parameters: none

Result


| Name | Type | Presence | Description |
| ---- | ---- | ---- | ---- |
| result | number (int) | Required(exactly 1) | The current block count |


Example

bitcoin-cli getblockcount

那么用Postman请求实际上是这样的请求
![](/images/btc-wallet-block-seek/rpc-postman.jpg)

# 使用Golang调用钱包RPC接口

```golang
var client *gorequest.SuperAgent
var rpcURI string

// StRpcRespError rpc返回错误信息
type StRpcRespError struct {
	Code    int64  `json:"code"`
	Message string `json:"message"`
}

func (e *StRpcRespError) Error() string {
	return fmt.Sprintf("%d %s", e.Code, e.Message)
}

// StRpcReq rpc请求信息
type StRpcReq struct {
	Jsonrpc string        `json:"jsonrpc"`
	ID      string        `json:"id"`
	Method  string        `json:"method"`
	Params  []interface{} `json:"params"`
}

// StRpcResp rpc通用返回字段
type StRpcResp struct {
	ID    string          `json:"id"`
	Error *StRpcRespError `json:"error"`
}

// InitClient 初始化客户端
func InitClient(omniRPCHost, omniRPCUser, omniRPCPwd string) {
	rpcURI = omniRPCHost
	client = gorequest.New().SetBasicAuth(omniRPCUser, omniRPCPwd).Timeout(time.Minute * 5)
}

// doReq 调用http请求
func doReq(method string, arqs []interface{}, resp interface{}) error {
	_, body, errs := client.Post(rpcURI).Send(StRpcReq{
		Jsonrpc: "1.0",
		ID:      hcommon.GetUUIDStr(),
		Method:  method,
		Params:  arqs,
	}).EndBytes()
	if errs != nil {
		return errs[0]
	}
	err := json.Unmarshal(body, resp)
	if err != nil {
		return err
	}
	return nil
}

// RpcGetBlockCount 获取block number
func RpcGetBlockCount() (int64, error) {
	resp := struct {
		StRpcResp
		Result int64 `json:"result"`
	}{}
	err := doReq(
		"getblockcount",
		nil,
		&resp,
	)
	if err != nil {
		return 0, err
	}
	if resp.Error != nil {
		return 0, resp.Error
	}
	return resp.Result, nil
}
```

# 检测交易流程

用时序图展示这个流程就是

![](/images/btc-wallet-block-seek/btc-seek-seq.jpg)

比较麻烦的是，vin中没有包含地址信息，所以我们还需要根据vin的txid再查询一遍rawtransaction。为了减少请求，我们把这个数据存入map，如果有则直接读取，没有便请求rpc。

```golang
vinTxMap := make(map[string]*omniclient.StTxResult)
rpcVinTx, ok := vinTxMap[vin.Txid]
if !ok {
    rpcVinTx, err = omniclient.RpcGetRawTransactionVerbose(vin.Txid)
    if err != nil {
        hcommon.Log.Errorf("err: [%T] %s", err, err.Error())
        return
    }
    vinTxMap[vin.Txid] = rpcVinTx
}
```

# 整合代码

具体实现的代码已经传到了github [https://github.com/moremorefun/go-dc-wallet](https://github.com/moremorefun/go-dc-wallet)

功能入口在`cmd/btc_block_seek/main.go` 

***

![](/images/mp-qr-search-v2.png)