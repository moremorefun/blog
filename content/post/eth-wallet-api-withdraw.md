---
title: 以太坊(eth)交易所钱包开发 - 6 - 申请提币
date: 2020-04-21 23:11:25
tags:
    - eth
    - wallet
    - address
    - 以太坊
    - 收提币
    - 钱包
---
提币的总体时序图大概是这样的:
![](/images/eth-wallet-api-withdraw/s-withdraw.jpg)

[<以太坊(eth)交易所钱包开发 - 5 - 获取冲币地址>](/2020/04/21/eth-wallet-api-address/)中已经介绍了API请求数据的验证方式，这部分验证都是一样的，用gin的自定义的中间件完成就可以了。

提币请求的参数为:
```golang
var req struct {
    Symbol    string `json:"symbol" binding:"required" validate:"oneof=eth"`
    OutSerial string `json:"out_serial" binding:"required" validate:"max=40"`
    Address   string `json:"address" binding:"required"`
    Balance   string `json:"balance" binding:"required"`
}
```
我们需要注意`OutSerial`字段，这个字段是为了避免同一提币请求被多次处理添加的，数据库会给这个字段做一个唯一索引，相同的提币便无法被重复插入了。
<!-- more -->
之后我们只需要验证一下参数是否合法，然后插入数据库就可以了。

1. 把地址小写，验证地址是否合法。
    
    ```golang
    // 验证地址
    req.Address = strings.ToLower(req.Address)
    re := regexp.MustCompile("^0x[0-9a-fA-F]{40}$")
    if !re.MatchString(req.Address) {
        c.JSON(http.StatusOK, gin.H{
            "error":   hcommon.ErrorAddressWrong,
            "err_msg": hcommon.ErrorAddressWrongMsg,
        })
        return
    }
    ```

2. 验证提币金额。eth支持的小数位数最大为18位。
    
    ```golang
    // 验证金额
	balanceObj, err := decimal.NewFromString(req.Balance)
	if err != nil {
		hcommon.Log.Errorf("err: [%T] %s", err, err.Error())
		c.JSON(
            http.StatusOK, 
            gin.H{
                "error":   hcommon.ErrorBalanceFormat,
                "err_msg": hcommon.ErrorBalanceFormatMsg,
            },
        )
		return
	}
	if balanceObj.LessThanOrEqual(decimal.NewFromInt(0)) {
		c.JSON(
            http.StatusOK, 
            gin.H{
                "error":   hcommon.ErrorBalanceFormat,
                "err_msg": hcommon.ErrorBalanceFormatMsg,
            },
        )
		return
	}
	if balanceObj.Exponent() < -18 {
		c.JSON(
            http.StatusOK, 
            gin.H{
                "error":   hcommon.ErrorBalanceFormat,
                "err_msg": hcommon.ErrorBalanceFormatMsg,
            },
        )
		return
	}
    ```

插入成功后[以太坊(eth)交易所钱包开发 - 4 - 提币](/2020/04/20/eth-wallet-withdraw/)就可以去处理了

具体实现的代码已经传到了github [https://github.com/moremorefun/go-dc-wallet](https://github.com/moremorefun/go-dc-wallet)

功能入口在 `cmd/api/main.go` 

***

![](/images/mp-qr-search-v2.png)