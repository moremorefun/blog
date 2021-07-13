---
title: 比特币(btc)交易所钱包开发 - 2 - 创建地址
date: 2020-04-30 09:32:50+00:00
tags:
    - btc
    - wallet
    - 比特币
    - 收提币
    - 钱包
---
# 比特币地址和私钥与以太坊的异同

比特币的地址和私钥的关系和以太坊也是一样的，参见[以太坊(eth)简介](/2020/04/17/eth-introduction/)。

但是在显示上有一些区别

- 比特币的地址用的是Base58的序列化显示方式，这个方式是大小写敏感的，所以要注意比特币的地址是不能转化大小写的。
- 比特币的私钥导出时一般使用WIF格式，为了给私钥加验证，比特币的私钥导出时在首位会加一个netID。这个netID对应不同的网络，例如比特币测试网络，比特币正式网络，莱特币等等；而在末尾会加一个数据校验，用来验证数据是否被损坏。

<!-- more -->

# 比特币及其衍生链的wif和地址填充数据

这里我们列举了几种链的数据的填充字节信息

```golang
var network = map[string]Network{
	"rdd": {name: "reddcoin", symbol: "rdd", xpubkey: 0x3d, xprivatekey: 0xbd},
	"dgb": {name: "digibyte", symbol: "dgb", xpubkey: 0x1e, xprivatekey: 0x80},
	"btc": {name: "bitcoin", symbol: "btc", xpubkey: 0x00, xprivatekey: 0x80},
	"ltc": {name: "litecoin", symbol: "ltc", xpubkey: 0x30, xprivatekey: 0xb0},
}
```

# golang处理比特币数据的开源库

用golang处理比特币数据我们可以利用一个开源的库[https://github.com/btcsuite](https://github.com/btcsuite)

# 检测并生成可用的比特币地址

检测生成比特币地址的逻辑其实和以太坊是一样的，可参阅 [以太坊(eth)交易所钱包开发 - 1 - 创建地址
](/2020/04/17/eth-wallet-address-create/) 。只不过生成私钥和获取地址的函数不一样。

首先我们看一下生成比特币私钥并转换为wif格式导出的代码

```golang
// CreatePrivateKey 生成一个二进制的私钥，并返回一个wif对象
func (network Network) CreatePrivateKey() (*btcutil.WIF, error) {
	secret, err := btcec.NewPrivateKey(btcec.S256())
	if err != nil {
		return nil, err
	}
	// 注意，在返回wif对象的时候，传入了wif填充对象GetNetworkParams
	return btcutil.NewWIF(secret, network.GetNetworkParams(), true)
}

// GetNetworkParams 根据map配置返回链填充数据
func (network Network) GetNetworkParams() *chaincfg.Params {
	networkParams := &chaincfg.MainNetParams
	networkParams.PubKeyHashAddrID = network.xpubkey
	networkParams.PrivateKeyID = network.xprivatekey
	return networkParams
}

// 获取私钥的wif字符串，Base58格式的字符串
// See DecodeWIF for a detailed breakdown of the format and requirements of
// a valid WIF string.
func (w *WIF) String() string {
	// Precalculate size.  Maximum number of bytes before base58 encoding
	// is one byte for the network, 32 bytes of private key, possibly one
	// extra byte if the pubkey is to be compressed, and finally four
	// bytes of checksum.
	encodeLen := 1 + btcec.PrivKeyBytesLen + 4
	if w.CompressPubKey {
		encodeLen++
	}

	a := make([]byte, 0, encodeLen)
	// 首先填充netID, 也就是填充数据
	a = append(a, w.netID)
	// Pad and append bytes manually, instead of using Serialize, to
	// avoid another call to make.
	// 然后填充私钥二进制数据
	a = paddedAppend(btcec.PrivKeyBytesLen, a, w.PrivKey.D.Bytes())
	if w.CompressPubKey {
		a = append(a, compressMagic)
	}
	// 对数据做校验填充
	cksum := chainhash.DoubleHashB(a)[:4]
	a = append(a, cksum...)
	// 返回最终的Base58字符串
	return base58.Encode(a)
}
```

之后我们看一下地址是怎么通过wif导出的

```golang
// NewAddressPubKey 获取一个public key
// address.  The serializedPubKey parameter must be a valid pubkey and can be
// uncompressed, compressed, or hybrid.
func NewAddressPubKey(serializedPubKey []byte, net *chaincfg.Params) (*AddressPubKey, error) {
	pubKey, err := btcec.ParsePubKey(serializedPubKey, btcec.S256())
	if err != nil {
		return nil, err
	}

	// Set the format of the pubkey.  This probably should be returned
	// from btcec, but do it here to avoid API churn.  We already know the
	// pubkey is valid since it parsed above, so it's safe to simply examine
	// the leading byte to get the format.
	pkFormat := PKFUncompressed
	switch serializedPubKey[0] {
	case 0x02, 0x03:
		pkFormat = PKFCompressed
	case 0x06, 0x07:
		pkFormat = PKFHybrid
	}

	return &AddressPubKey{
		pubKeyFormat: pkFormat,
		pubKey:       pubKey,
		// 设置地址填充数据
		pubKeyHashID: net.PubKeyHashAddrID,
	}, nil
}

// EncodeAddress returns the string encoding of the public key as a
// pay-to-pubkey-hash.  Note that the public key format (uncompressed,
// compressed, etc) will change the resulting address.  This is expected since
// pay-to-pubkey-hash is a hash of the serialized public key which obviously
// differs with the format.  At the time of this writing, most Bitcoin addresses
// are pay-to-pubkey-hash constructed from the uncompressed public key.
//
// Part of the Address interface.
func (a *AddressPubKey) EncodeAddress() string {
	return encodeAddress(Hash160(a.serialize()), a.pubKeyHashID)
}

// encodeAddress returns a human-readable payment address given a ripemd160 hash
// and netID which encodes the bitcoin network and address type.  It is used
// in both pay-to-pubkey-hash (P2PKH) and pay-to-script-hash (P2SH) address
// encoding.
func encodeAddress(hash160 []byte, netID byte) string {
	// Format is 1 byte for a network and address class (i.e. P2PKH vs
	// P2SH), 20 bytes for a RIPEMD160 hash, and 4 bytes of checksum.
	return base58.CheckEncode(hash160[:ripemd160.Size], netID)
}
```

# 整合代码

具体实现的代码已经传到了github [https://github.com/moremorefun/go-dc-wallet](https://github.com/moremorefun/go-dc-wallet)

功能入口在`cmd/btc_address/main.go` 

***

![](/images/mp-qr-search-v2.png)