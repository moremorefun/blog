---
title: 以太坊(eth)交易所钱包开发 - 7 - 冲币提币回调
date: 2020-04-21 23:59:25
tags:
    - eth
    - wallet
    - address
    - 以太坊
    - 收提币
    - 钱包
---
回调时序图大概如下
![](/images/eth-wallet-api-cb/cb.jpg)

当用户冲币和提币的时候，我们需要把这些信息通知用户。这个通知我们也是通过http调用实现，只不过我们调用的是`product`中提供的回调地址。

比如用户冲币的时候，我们回调的内容是:
```json
{
    "address":"0xa94ae9cd4d3ad8a29697da180bda45040e633501",
    "app_name":"app_1",
    "balance":"0.0001",
    "notify_type":1,
    "sign":"97E589CEC28A80CD9FBCE0F042C08837",
    "symbol":"eth",
    "tx_hash":"0xd1fe86cc358f525e452cbab251adc403b531da230304e71887fda911a40e8286"
}
```
这里我们用相同的逻辑签名获得`sign`一同返回，因为`tx_hash`每笔交易都不相同，可以作为nonce使用。
<!-- more -->
创建一个表，用来储存待发送给客户端的通知，并保留返回值和处理状态。在发送时，如果遇到返回错误是需要重试的，这样能避免对方服务器出错或者网络出问题而错过消息。
```sql
CREATE TABLE `t_product_notify` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `nonce` varchar(128) NOT NULL DEFAULT '',
  `product_id` int(11) NOT NULL,
  `item_type` int(11) NOT NULL,
  `item_id` int(11) NOT NULL,
  `notify_type` int(11) NOT NULL,
  `url` varchar(512) NOT NULL DEFAULT '',
  `msg` varchar(4089) NOT NULL,
  `handle_status` int(11) NOT NULL,
  `handle_msg` varchar(512) NOT NULL,
  `create_time` bigint(20) unsigned NOT NULL,
  `update_time` bigint(20) unsigned NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `product_id` (`product_id`,`item_type`,`item_id`,`notify_type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

有几个地方处理的时候需要将待发送数据存入待发送表。
1. 检测区块链入账的时候
2. 发送提币交易的时候
3. 提币交易被确认打包的时候

发送通知也很简单，遍历这个表里没有成功的数据，逐个发送就好。
```golang
gresp, body, errs := gorequest.New().Post(initNotifyRow.URL).Timeout(time.Second * 30).Send(initNotifyRow.Msg).End()
if errs != nil {
    hcommon.Log.Errorf("err: [%T] %s", errs[0], errs[0].Error())
    _, err = SQLUpdateTProductNotifyStatusByID(
        context.Background(),
        DbCon,
        &model.DBTProductNotify{
            ID:           initNotifyRow.ID,
            HandleStatus: 1,
            HandleMsg:    errs[0].Error(),
            UpdateTime:   time.Now().Unix(),
        },
    )
    if err != nil {
        hcommon.Log.Errorf("err: [%T] %s", err, err.Error())
    }
    continue
}
if gresp.StatusCode != http.StatusOK {
    // 状态错误
    hcommon.Log.Errorf("req status error: %d", gresp.StatusCode)
    _, err = SQLUpdateTProductNotifyStatusByID(
        context.Background(),
        DbCon,
        &model.DBTProductNotify{
            ID:           initNotifyRow.ID,
            HandleStatus: 1,
            HandleMsg:    fmt.Sprintf("http status: %d", gresp.StatusCode),
            UpdateTime:   time.Now().Unix(),
        },
    )
    if err != nil {
        hcommon.Log.Errorf("err: [%T] %s", err, err.Error())
    }
    continue
}
resp := gin.H{}
err = json.Unmarshal([]byte(body), &resp)
if err != nil {
    hcommon.Log.Errorf("err: [%T] %s", err, err.Error())
    _, err = SQLUpdateTProductNotifyStatusByID(
        context.Background(),
        DbCon,
        &model.DBTProductNotify{
            ID:           initNotifyRow.ID,
            HandleStatus: 1,
            HandleMsg:    body,
            UpdateTime:   time.Now().Unix(),
        },
    )
    if err != nil {
        hcommon.Log.Errorf("err: [%T] %s", err, err.Error())
    }
    continue
}
_, ok := resp["error"]
if ok {
    // 处理成功
    _, err = SQLUpdateTProductNotifyStatusByID(
        context.Background(),
        DbCon,
        &model.DBTProductNotify{
            ID:           initNotifyRow.ID,
            HandleStatus: 2,
            HandleMsg:    body,
            UpdateTime:   time.Now().Unix(),
        },
    )
    if err != nil {
        hcommon.Log.Errorf("err: [%T] %s", err, err.Error())
    }
} else {
    hcommon.Log.Errorf("no error in resp")
    _, err = SQLUpdateTProductNotifyStatusByID(
        context.Background(),
        DbCon,
        &model.DBTProductNotify{
            ID:           initNotifyRow.ID,
            HandleStatus: 1,
            HandleMsg:    body,
            UpdateTime:   time.Now().Unix(),
        },
    )
    if err != nil {
        hcommon.Log.Errorf("err: [%T] %s", err, err.Error())
    }
    continue
}
```

具体实现的代码已经传到了github [https://github.com/moremorefun/go-dc-wallet](https://github.com/moremorefun/go-dc-wallet)

功能入口在 
- `cmd/do_notify/main.go` 
- `cmd/eth_block_seek/main.go` 
- `cmd/eth_raw_tx_send/main.go` 
- `cmd/eth_raw_tx_confirm/main.go` 

***

![](/images/mp-qr-search-v2.png)