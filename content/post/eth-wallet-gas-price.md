---
title: 以太坊(eth)交易所钱包开发 - 11 - 动态获取gasprice
date: 2020-04-27 09:48:25
tags:
    - eth
    - wallet
    - 以太坊
    - 收提币
    - 钱包
---
以太坊运行过程中`gasprice`是随着打包需求量不停变化的。所以为了打包速度，我们也需要不停变换我们的gasprice以满足我们打包速度的需求。

那该变化为多少呢？有一个网站为我们提供了参考数值。这个网站就是[ethgasstation](https://ethgasstation.info/)
<!-- more -->
![](/images/eth-wallet-gas-price/ethgasstation-0.jpg)

这里我们可以看到，`ethgasstation`提供了三种gasprice，分别对应三种速度。为了兼顾成本和速度，我们给用户提币的时候选用fast的等级，零钱整理的时候选用standard就可以了。

这个网站也为我们提供了API接口，也就不需要我们去解析网页了。

接口地址为: `https://ethgasstation.info/api/ethgasAPI.json`

用Get方式请求会获得一下数据
```json
{
  "fast": 77,
  "fastest": 199,
  "safeLow": 33,
  "average": 50,
  "block_time": 11.926829268292684,
  "blockNum": 9951653,
  "speed": 0.6112403032824382,
  "safeLowWait": 16.5,
  "avgWait": 1,
  "fastWait": 0.4,
  "fastestWait": 0.4
}
```
所以我们只需要用golang访问这个接口，然后把解析到的数据存入数据库就可以了。

这里我们可以使用一个小工具，这个工具可以把json数据转化为可以序列化这个json的struct结构定义: [json2go](https://www.jidangeng.com/tool-json-to-go/)

![](/images/eth-wallet-gas-price/json2go.jpg)

关键代码为:

```golang
type StRespGasPrice struct {
    Fast        int64   `json:"fast"`
    Fastest     int64   `json:"fastest"`
    SafeLow     int64   `json:"safeLow"`
    Average     int64   `json:"average"`
    BlockTime   float64 `json:"block_time"`
    BlockNum    int64   `json:"blockNum"`
    Speed       float64 `json:"speed"`
    SafeLowWait float64 `json:"safeLowWait"`
    AvgWait     float64 `json:"avgWait"`
    FastWait    float64 `json:"fastWait"`
    FastestWait float64 `json:"fastestWait"`
}
gresp, body, errs := gorequest.New().
    Get("https://ethgasstation.info/api/ethgasAPI.json").
    Timeout(time.Second * 120).
    End()
if errs != nil {
    hcommon.Log.Errorf("err: [%T] %s", errs[0], errs[0].Error())
    return
}
if gresp.StatusCode != http.StatusOK {
    // 状态错误
    hcommon.Log.Errorf("req status error: %d", gresp.StatusCode)
    return
}
var resp StRespGasPrice
err := json.Unmarshal([]byte(body), &resp)
if err != nil {
    hcommon.Log.Errorf("err: [%T] %s", err, err.Error())
    return
}
toUserGasPrice := resp.Fast * int64(math.Pow10(8))
toColdGasPrice := resp.Average * int64(math.Pow10(8))
_, err = app.SQLUpdateTAppStatusIntByK(
    context.Background(),
    app.DbCon,
    &model.DBTAppStatusInt{
        K: "to_user_gas_price",
        V: toUserGasPrice,
    },
)
if err != nil {
    hcommon.Log.Errorf("err: [%T] %s", err, err.Error())
    return
}
_, err = app.SQLUpdateTAppStatusIntByK(
    context.Background(),
    app.DbCon,
    &model.DBTAppStatusInt{
        K: "to_cold_gas_price",
        V: toColdGasPrice,
    },
)
if err != nil {
    hcommon.Log.Errorf("err: [%T] %s", err, err.Error())
    return
}
```

具体实现的代码已经传到了github [https://github.com/moremorefun/go-dc-wallet](https://github.com/moremorefun/go-dc-wallet)

功能入口在`cmd/eth_gas_price/main.go` 

***

![](/images/mp-qr-search-v2.png)