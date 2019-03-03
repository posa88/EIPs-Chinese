---
eip: 55
title: 混合大小写校验地址编码
author: Vitalik Buterin
type: Standards Track
category: ERC
status: Final
created: 2016-01-14
---

# 规范

代码:

``` python
from ethereum import utils

def checksum_encode(addr): # Takes a 20-byte binary address as input
    o = ''
    v = utils.big_endian_to_int(utils.sha3(addr.hex()))
    for i, c in enumerate(addr.hex()):
        if c in '0123456789':
            o += c
        else:
            o += c.upper() if (v & (2**(255 - 4*i))) else c.lower()
    return '0x'+o

def test(addrstr):
    assert(addrstr == checksum_encode(bytes.fromhex(addrstr[2:])))

test('0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed')
test('0xfB6916095ca1df60bB79Ce92cE3Ea74c37c5d359')
test('0xdbF03B407c01E7cD3CBea99509d93f8DDDC8C6FB')
test('0xD1220A0cf47c7B9Be7A2E6BA89F429762e7b9aDb')

```

在英文里，将地址转换为十六进制，但如果第`i`位是一个字母(即是`abcdef`中的一个)，那么如果小写的十六进制地址的哈希值的第`4*i`位是1则用大写打印，否则用小写。

# 基本原理

好处:
- 向后兼容许多接受混合大小写的十六进制解析器，允许随着时间的推移很容易地引入它。
- 保持长度在40字符。
- 平均每个地址有15个检查位，如果输入错误，随机生成的地址意外通过检查的概率是0.0247%。这是对ICAP的一个~50倍的改进，但不如一个4字节的校验码好。

# 实现

使用javascript:

```js
const createKeccakHash = require('keccak')

function toChecksumAddress (address) {
  address = address.toLowerCase().replace('0x', '')
  var hash = createKeccakHash('keccak256').update(address).digest('hex')
  var ret = '0x'

  for (var i = 0; i < address.length; i++) {
    if (parseInt(hash[i], 16) >= 8) {
      ret += address[i].toUpperCase()
    } else {
      ret += address[i]
    }
  }

  return ret
}
```

```
> toChecksumAddress('0xfb6916095ca1df60bb79ce92ce3ea74c37c5d359')
'0xfB6916095ca1df60bB79Ce92cE3Ea74c37c5d359'
```

注意Keccak256哈希函数的输入是小写十六进制字符串(即用ASCII编码的十六进制地址):

```
    var hash = createKeccakHash('keccak256').update(Buffer.from(address.toLowerCase(), 'ascii')).digest()
```

# 测试用例

```
# 全大写
0x52908400098527886E0F7030069857D2E4169EE7
0x8617E340B3D01FA5F11F306F4090FD50E238070D
# 全小写
0xde709f2102306220921060314715629080e2fb77
0x27b1fdb04752bbc536007a920d24acb045561c26
# 正常
0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed
0xfB6916095ca1df60bB79Ce92cE3Ea74c37c5d359
0xdbF03B407c01E7cD3CBea99509d93f8DDDC8C6FB
0xD1220A0cf47c7B9Be7A2E6BA89F429762e7b9aDb
```

# 采用

| 钱包                     | 展示校验的地址                   | 拒绝无效的混合大小写编码地址   | 拒绝过短地址        | 拒绝过长地址       |
|--------------------------|--------------------------------|----------------------------|-------------------|------------------|
| Etherwall 2.0.1          | Yes                            | Yes                        | Yes               | Yes              |
| Jaxx 1.2.17              | No                             | Yes                        | Yes               | Yes              |
| MetaMask 3.7.8           | Yes                            | Yes                        | Yes               | Yes              |
| Mist 0.8.10              | Yes                            | Yes                        | Yes               | Yes              |
| MyEtherWallet v3.9.4     | Yes                            | Yes                        | Yes               | Yes              |
| Parity 1.6.6-beta (UI)   | Yes                            | Yes                        | Yes               | Yes              |
| Jaxx Liberty 2.0.0       | Yes                            | Yes                        | Yes               | Yes              |
| Coinomi 1.10             | Yes                            | Yes                        | Yes               | Yes              |

### 交易所对混合大小写校验地址的支持，更新到2017-05-27:

| 交易所        | 展示校验的充值地址                       | 拒绝无效的混合大小写编码地址   | 拒绝过短地址        | 拒绝过长地址       |
|--------------|----------------------------------------|----------------------------|-------------------|------------------|
| Bitfinex     | No                                     | Yes                        | Yes               | Yes              |
| Coinbase     | Yes                                    | No                         | Yes               | Yes              |
| GDAX         | Yes                                    | Yes                        | Yes               | Yes              |
| Kraken       | No                                     | No                         | Yes               | Yes              |
| Poloniex     | No                                     | No                         | Yes               | Yes              |
| Shapeshift   | No                                     | No                         | Yes               | Yes              |

# 引用

1. EIP 55 问题和讨论 https://github.com/ethereum/eips/issues/55
2. @Recmo 报告的Python例子  https://github.com/ethereum/eips/issues/55#issuecomment-261521584
3. Python 实现，在[`ethereum-utils`](https://github.com/pipermerriam/ethereum-utils#to_checksum_addressvalue---text)
4. Ethereumjs-util 实现 https://github.com/ethereumjs/ethereumjs-util/blob/75f529458bc7dc84f85fd0446d0fac92d991c262/index.js#L452-L466
5. Swift实现，在[`EthereumKit`](https://github.com/yuzushioh/EthereumKit/blob/master/EthereumKit/Helper/EIP55.swift)
