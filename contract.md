这部分内容主要介绍如何在alephium testnet上创建和调用合约，然后根据产生的output来了解合约的状态存储及更新。

# Create Contract

首先我们先创建一个token contract，代码如下:

```
TxContract MyToken(owner: Address, mut remain: U256) {
  pub payable fn buy(from: Address, alfAmount: U256) -> () {
    let tokenAmount = alfAmount * 1000
    assert!(remain >= tokenAmount)
    let tokenId = selfTokenId!()
    transferAlf!(from, owner, alfAmount)
    transferTokenFromSelf!(from, tokenId, tokenAmount)
    remain = remain - tokenAmount
  }
}
```

这是一个非常简单的token contract，用户可以以`1:1000`的比例通过向`owner`支付**ALPH**来换取token，其中以 **\!** 结尾的都是buildin function，这里用到的有:

* assert!(pred): 用于条件判断，pred为false会导致合约执行失败
* selfTokenId!(): 当前的token id，也是当前的contract id
* transferAlf!(from, to, alfAmount): 从`from`向`to`支付数量为`alfAmount`的**ALPH**
* transferTokenFromSelf!(to, tokenId, tokenAmount): contract向`to`支付数量为`tokenAmount`的token

**NOTE**: 这里的`remain`不是必须的，不过`remain`可以帮助我们了解合约的可变状态，后面会简单介绍合约状态是如何存储的

下面我们开始在testnet上创建合约，首先我们需要将上述的合约代码编译为二进制代码:

```shell
$ curl -X 'POST' \
  'http://127.0.0.1:12973/contracts/compile' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "address": "15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46",
  "type": "contract",
  "code": "TxContract MyToken(owner: Address, mut remain: U256) {\npub payable fn buy(from: Address, alfAmount: U256) -> () {\nlet tokenAmount = alfAmount * 1000\nassert!(remain >= tokenAmount)\nlet tokenId = selfTokenId!()\ntransferAlf!(from, owner, alfAmount)\ntransferTokenFromSelf!(from, tokenId, tokenAmount)\nremain = remain - tokenAmount\n}\n}",
  "state": "[@15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46,10000000000000000000000000000u]",
  "issueTokenAmount": "10000000000000000000000000000"
}'
```

这里简单介绍一下上面的几个参数:

* address: 在创建合约时会从这个address向合约transfer一笔数量很小(`dustUtxoAmount`)的**ALPH**，关于`dustUtxoAmount`后面会介绍
* type: 类型为`contract`或者`script`，`script`可以用于创建和调用合约，这里我们调用的compile接口会为我们生成`script`
* code: contract source code
* state: 合约需要的两个初始状态，对应合约代码的两个参数
* issueTokenAmount: 合约发行token的总数

**NOTE**: 创建合约时会默认向合约transfer数量为`dustUtxoAmount`的**ALPH**，我们完全可以自己写`TxScript`指定需要向合约transfer多少**ALPH**，具体做法可以参考[这里](https://github.com/alephium/alephium/blob/master/app/src/main/scala/org/alephium/app/ServerUtils.scala#L541-L548)。

执行上述request后，会返回类似下面的response:

```json
{
  "code": "01010100000007150040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95313c1e8d4a51000a21440300201402c01010204001616011343e82c1702a0011602344db117031600a0001601a7160016031602aba00116022ba10114403102040040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95302c8204fce5e3e2502611000000013c8204fce5e3e25026110000000ae"
}
```

这个就是合约`MyToken`的二进制代码了，下面我们开始创建unsigned transaction:

```shell
$ curl -X 'POST' \
  'http://127.0.0.1:12973/contracts/build' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "fromPublicKey": "022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f402",
  "code": "01010100000007150040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95313c1e8d4a51000a21440300201402c01010204001616011343e82c1702a0011602344db117031600a0001601a7160016031602aba00116022ba10114403102040040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95302c8204fce5e3e2502611000000013c8204fce5e3e25026110000000ae",
  "gas": 100000
}'
```

这里参数分别为:

* fromPublicKey: 当前使用的地址对应的Public Key
* code: contract binary code
* gas: 这里我们手动指定了gas，默认的gas对于合约相关的操作可能不够用

执行上述request之后，会返回类似下面的response:

```json
{
  "unsignedTx": "010101010100000007150040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95313c1e8d4a51000a21440300201402c01010204001616011343e82c1702a0011602344db117031600a0001601a7160016031602aba00116022ba10114403102040040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95302c8204fce5e3e2502611000000013c8204fce5e3e25026110000000ae800186a0c1174876e80001c168f933c229678d1fb71105360abbca47cc00e793439c76bb78fea3b4e97cdc963c983300022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f40200",
  "hash": "9999d8638fc458e987bafc1eae311d31e44149d1551398f53af41752868fb0b6",
  "fromGroup": 3,
  "toGroup": 3
}
```

这里我们得到了`unsignedTx`和`txHash`，下面需要对`unsignedTx`签名:

```shell
$ curl -X 'POST' \
  'http://127.0.0.1:12973/wallets/alph-wallet/sign' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "data": "9999d8638fc458e987bafc1eae311d31e44149d1551398f53af41752868fb0b6"
}'
```

上面的request会请求对`txHash`签名，得到的结果为:

```json
{
  "signature": "96b9e91d77cf9976a3e55162f109fcfb99ad2aa1c2550b3fd3dd3957931231f430a92385984478645452d9de0fb0eda017e327ff026f24f75cda8cfd2d334c36"
}
```

现在我们所有创建合约所需的数据都已经准备好了，下面执行submit tx:

```shell
$ curl -X 'POST' \
  'http://127.0.0.1:12973/contracts/submit' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "code": "01010100000007150040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95313c1e8d4a51000a21440300201402c01010204001616011343e82c1702a0011602344db117031600a0001601a7160016031602aba00116022ba10114403102040040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95302c8204fce5e3e2502611000000013c8204fce5e3e25026110000000ae",
  "tx": "010101010100000007150040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95313c1e8d4a51000a21440300201402c01010204001616011343e82c1702a0011602344db117031600a0001601a7160016031602aba00116022ba10114403102040040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95302c8204fce5e3e2502611000000013c8204fce5e3e25026110000000ae800186a0c1174876e80001c168f933c229678d1fb71105360abbca47cc00e793439c76bb78fea3b4e97cdc963c983300022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f40200",
  "signature": "96b9e91d77cf9976a3e55162f109fcfb99ad2aa1c2550b3fd3dd3957931231f430a92385984478645452d9de0fb0eda017e327ff026f24f75cda8cfd2d334c36",
  "fromGroup": 3
}' 
```

执行上述request后，如果合约是有效的，会返回类似如下结果:

```json
{
  "txId": "9999d8638fc458e987bafc1eae311d31e44149d1551398f53af41752868fb0b6",
  "fromGroup": 3,
  "toGroup": 3
}
```

这时我们就可以等待tx直到confirmed，我们可以直接在testnet explorer上查看[这笔交易](https://testnet.alephium.org/#/transactions/9999d8638fc458e987bafc1eae311d31e44149d1551398f53af41752868fb0b6)。

为了更清楚地了解创建合约的过程，下面我们来看看上述tx的具体内容(目前暂时还没有查看tx详细信息的接口，所以我们这里直接使用获取block的接口，后面我们会提供针对tx的接口):

```shell
$ curl -X 'GET' \
  'http://127.0.0.1:12973/blockflow/blocks/00000042cc6dce61cf6e8beb5d90c1a35fcfb28c9deb8f99e2fd37e669315acf' \
  -H 'accept: application/json'
```

得到的结果如下:

```json
{
  "hash": "00000042cc6dce61cf6e8beb5d90c1a35fcfb28c9deb8f99e2fd37e669315acf",
  "timestamp": 1632547455286,
  "chainFrom": 3,
  "chainTo": 3,
  "height": 13264,
  "deps": [
    "0000005551446a346875eb4db15f5aad291833db18408f9aa0f48cd70cdcb6a0",
    "000000ec8abe87275f1735e63b3059b20429d4d7eb4ed305ffdee660df7d04c5",
    "000001275a6c11d3f52148c4eaa24b2f7fc316e5171ade632dae288483b82a3a",
    "000001cb7cc0245d6f1885ad7012081441e316765d50b198293121fae3b250dc",
    "000000ff5011e25ec63a12a62a1659aa5dc048cf13dbd61c21c85140dbbe4c5d",
    "000001ceaf57dc7f8cfce27648729f5c8a14f55a58d122b1a279ce8bf89e1ace",
    "000001806c6d29d968f88db03193a3d5dd60ed002bc7ea82c182b490738e645f"
  ],
  "transactions": [
    {
      "id": "9999d8638fc458e987bafc1eae311d31e44149d1551398f53af41752868fb0b6",
      "inputs": [
        {
          "type": "asset",
          "outputRef": {
            "hint": -1050085069,
            "key": "c229678d1fb71105360abbca47cc00e793439c76bb78fea3b4e97cdc963c9833"
          },
          "unlockScript": "00022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f402"
        }
      ],
      "outputs": [
        {
          "type": "contract",
          "amount": "1000000000000",
          "address": "288xWaVhT4oKRHB8QSsLp4oZg2YvUNHu2rr8Y3mvegeW2",
          "tokens": [
            {
              "id": "c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f",
              "amount": "10000000000000000000000000000"
            }
          ]
        },
        {
          "type": "asset",
          "amount": "26974598000000000000",
          "address": "15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46",
          "tokens": [],
          "lockTime": 0,
          "additionalData": ""
        }
      ],
      "gasAmount": 100000,
      "gasPrice": "100000000000"
    },
    {
      "id": "7abf93fc435eb39c21ee2e7def0d9fdd9b7855d02d2afdddde5e204c0314ade5",
      "inputs": [],
      "outputs": [
        {
          "type": "asset",
          "amount": "1895069812536239624",
          "address": "12oxFH1C8JqKCW5Rv9otymoZwsB6grvWP8jGiLcyaHDZW",
          "tokens": [],
          "lockTime": 1632548055286,
          "additionalData": "03030000017c1b694136"
        }
      ],
      "gasAmount": 20000,
      "gasPrice": "1000000000"
    }
  ]
}
```

这里我们只关注txId为`9999d8638fc458e987bafc1eae311d31e44149d1551398f53af41752868fb0b6`的交易:

```json
{
  "id": "9999d8638fc458e987bafc1eae311d31e44149d1551398f53af41752868fb0b6",
  "inputs": [
    {
      "type": "asset",
      "outputRef": {
        "hint": -1050085069,
        "key": "c229678d1fb71105360abbca47cc00e793439c76bb78fea3b4e97cdc963c9833"
      },
      "unlockScript": "00022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f402"
    }
  ],
  "outputs": [
    {
      "type": "contract",
      "amount": "1000000000000",
      "address": "288xWaVhT4oKRHB8QSsLp4oZg2YvUNHu2rr8Y3mvegeW2",
      "tokens": [
        {
          "id": "c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f",
          "amount": "10000000000000000000000000000"
        }
      ]
    },
    {
      "type": "asset",
      "amount": "26974598000000000000",
      "address": "15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46",
      "tokens": [],
      "lockTime": 0,
      "additionalData": ""
    }
  ],
  "gasAmount": 100000,
  "gasPrice": "100000000000"
}
```

我们可以看到这笔tx有一个input，两个outputs，第一个output类型为`contract`，下面来解释每个字段:

* type: tx output的类型，`contract`或者`asset`
* amount: **ALPH**的数量，我们可以看到这里`1000000000000`为上面说的`dustUtxoAmount`
* address: contract address的base58编码
* tokens: 其中包括了我们刚刚创建的token amount和token id

第二个output为asset output，跟普通的transfer output类似，区别在于这笔tx的两个output都是合约生成的。

这里主要关注创建合约生成的output，后面有时间我会单独介绍tx中每个字段的意义。

合约创建成功后，我们接下来开始调用合约，即通过`MyToken.buy`向合约支付**ALPH**来换取token。

# Call contract

首先我们需要写script来调用上面创建的contract:

```
TxScript Main {
  pub payable fn main() -> () {
    approveAlf!(@1FcFfMU5NmQY4Tj71ApytZpsgAEYLhkX3s4ZWy3PVqVeR, 1000000000000000000)
    let contract = MyToken(#c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f)
    contract.buy(@1FcFfMU5NmQY4Tj71ApytZpsgAEYLhkX3s4ZWy3PVqVeR, 1000000000000000000)
  }
}
```

简单解释一下这一段代码:

* 通过`approveAlf!`授权contract代为支付**ALPH**
* 通过contract id load需要调用的contract，这里的contract id `c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f`是通过上述的contract output address计算得来的(decode base58)，后面我们会提供更便捷的方式
* 调用`MyToken.buy`支付1个**ALPH**来换取token

同样，我们需要先编译为二进制代码(需要注意的是这里也要带上被调用的合约代码):

```shell
$ curl -X 'POST' \
  'http://127.0.0.1:12973/contracts/compile' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "address": "1FcFfMU5NmQY4Tj71ApytZpsgAEYLhkX3s4ZWy3PVqVeR",
  "type": "script",
  "code": "TxScript Main {\n pub payable fn main() -> () {\n approveAlf!(@1FcFfMU5NmQY4Tj71ApytZpsgAEYLhkX3s4ZWy3PVqVeR, 1000000000000000000)\n let contract = MyToken(#c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f)\n contract.buy(@1FcFfMU5NmQY4Tj71ApytZpsgAEYLhkX3s4ZWy3PVqVeR, 1000000000000000000)\n }\n }\n \nTxContract MyToken(owner: Address, mut remain: U256) {\n pub payable fn buy(from: Address, alfAmount: U256) -> () {\n let tokenAmount = alfAmount * 1000\n assert!(remain >= tokenAmount)\n let tokenId = selfTokenId!()\n transferAlf!(from, owner, alfAmount)\n transferTokenFromSelf!(from, tokenId, tokenAmount)\n remain = remain - tokenAmount\n }\n }"
}'
```

这里我们使用另外一个地址来调用contract，执行上述request后，会返回类似下面的response:

```json
response:
{
  "code": "010101000100091500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea13c40de0b6b3a7640000a2144020c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f17001500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea13c40de0b6b3a764000016000100"
}
```

然后我们根据binary code来创建unsigned tx:

```shell
$ curl -X 'POST' \
  'http://127.0.0.1:12973/contracts/build' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "fromPublicKey": "02b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b",
  "code": "010101000100091500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea13c40de0b6b3a7640000a2144020c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f17001500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea13c40de0b6b3a764000016000100",
  "gas": 100000
}'
```

返回结果为:

```json
{
  "unsignedTx": "0101010101000100091500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea13c40de0b6b3a7640000a2144020c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f17001500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea13c40de0b6b3a764000016000100800186a0c1174876e800037aa0f6ff04a938b825c1b7bd6784f0d3ef92693fb1b8e9c2dc6710cf13bf4a8b08795ffb0002b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b7aa0f6ffb32908ca767650d9e81e6c123f36818e7835803fa43647162bdd863bffd04a8a0002b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b7aa0f6ffb849638ad667078d54503b3afaf13251aab2c5d1ad5ef06bd6dd083195091d320002b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b00",
  "hash": "907eac91ff7adb3940eabbe0f850c4978f9e00e35693114d8aff707399aa9086",
  "fromGroup": 3,
  "toGroup": 3
}
```

然后对txHash签名，得到的签名为:

```json
{
  "signature": "25939e984436deeab57a078a692f3d7f01202a1e1b1ab4963ae623027621954b7e575906348c82b5629f16fc3e2363342d19f08fc021354da8a88e3956b98df2"
}
```

得到签名后，提交tx:

```shell
$ curl -X 'POST' \
  'http://127.0.0.1:12973/contracts/submit' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "code": "010101000100091500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea13c40de0b6b3a7640000a2144020c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f17001500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea13c40de0b6b3a764000016000100",
  "tx": "0101010101000100091500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea13c40de0b6b3a7640000a2144020c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f17001500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea13c40de0b6b3a764000016000100800186a0c1174876e800037aa0f6ff04a938b825c1b7bd6784f0d3ef92693fb1b8e9c2dc6710cf13bf4a8b08795ffb0002b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b7aa0f6ffb32908ca767650d9e81e6c123f36818e7835803fa43647162bdd863bffd04a8a0002b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b7aa0f6ffb849638ad667078d54503b3afaf13251aab2c5d1ad5ef06bd6dd083195091d320002b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b00",
  "signature": "25939e984436deeab57a078a692f3d7f01202a1e1b1ab4963ae623027621954b7e575906348c82b5629f16fc3e2363342d19f08fc021354da8a88e3956b98df2",
  "fromGroup": 3
}'  
```

同样，可以在testnet上找到[这笔交易](https://testnet.alephium.org/#/transactions/907eac91ff7adb3940eabbe0f850c4978f9e00e35693114d8aff707399aa9086)。

下面我们来看看这笔tx的具体内容:

```json
{
  "id": "907eac91ff7adb3940eabbe0f850c4978f9e00e35693114d8aff707399aa9086",
  "inputs": [
    {
      "type": "asset",
      "outputRef": {
        "hint": 2057369343,
        "key": "04a938b825c1b7bd6784f0d3ef92693fb1b8e9c2dc6710cf13bf4a8b08795ffb"
      },
      "unlockScript": "0002b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b"
    },
    {
      "type": "asset",
      "outputRef": {
        "hint": 2057369343,
        "key": "b32908ca767650d9e81e6c123f36818e7835803fa43647162bdd863bffd04a8a"
      },
      "unlockScript": "0002b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b"
    },
    {
      "type": "asset",
      "outputRef": {
        "hint": 2057369343,
        "key": "b849638ad667078d54503b3afaf13251aab2c5d1ad5ef06bd6dd083195091d32"
      },
      "unlockScript": "0002b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b"
    },
    {
      "type": "contract",
      "outputRef": {
        "hint": 1829167778,
        "key": "c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f"
      }
    }
  ],
  "outputs": [
    {
      "type": "asset",
      "amount": "1000000000000000000",
      "address": "15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46",
      "tokens": [],
      "lockTime": 0,
      "additionalData": ""
    },
    {
      "type": "asset",
      "amount": "4990000000000000000",
      "address": "1FcFfMU5NmQY4Tj71ApytZpsgAEYLhkX3s4ZWy3PVqVeR",
      "tokens": [
        {
          "id": "c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f",
          "amount": "1000000000000000000000"
        }
      ],
      "lockTime": 0,
      "additionalData": ""
    },
    {
      "type": "contract",
      "amount": "1000000000000",
      "address": "288xWaVhT4oKRHB8QSsLp4oZg2YvUNHu2rr8Y3mvegeW2",
      "tokens": [
        {
          "id": "c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f",
          "amount": "9999999000000000000000000000"
        }
      ]
    }
  ],
  "gasAmount": 100000,
  "gasPrice": "100000000000"
}
```

我们可以看到，这里有一个contract input:

```json
{
  "type": "contract",
  "outputRef": {
    "hint": 1829167778,
    "key": "c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f"
  }
}
```

这里的outputRef指向的就是我们前面创建合约的contract output，其中key就是contract id，然后我们来看看这三个outputs，第一个output:

```json
{
  "type": "asset",
  "amount": "1000000000000000000",
  "address": "15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46",
  "tokens": [],
  "lockTime": 0,
  "additionalData": ""
}
```

我们的script通过支付1个**ALPH**给`15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46`来换取token，这个output就是支付给`15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46`的**ALPH**输出。

第二个output:

```json
{
  "type": "asset",
  "amount": "4990000000000000000",
  "address": "1FcFfMU5NmQY4Tj71ApytZpsgAEYLhkX3s4ZWy3PVqVeR",
  "tokens": [
    {
      "id": "c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f",
      "amount": "1000000000000000000000"
    }
  ],
  "lockTime": 0,
  "additionalData": ""
}
```

除了找零的数量为`amount`的**ALPH**之外，我们还得到了合约发行的token。

第三个output:

```json
{
  "type": "contract",
  "amount": "1000000000000",
  "address": "288xWaVhT4oKRHB8QSsLp4oZg2YvUNHu2rr8Y3mvegeW2",
  "tokens": [
    {
      "id": "c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f",
      "amount": "9999999000000000000000000000"
    }
  ]
}
```

这里我们可以看到，contract output的token数量从原来的`10000000000000000000000000000`变为现在的`9999999000000000000000000000`，少的数量刚好是我们兑换的token数量。

# Contract state

从上面的创建和调用合约的交易数据我们可以看出:

* 创建合约时会生成一个contract output(无论是否发行token都会生成contract output)，如果发行token的话，output的tokens列表中会有初始token的数量
* 调用合约会消耗contract output，同时生成一个新的contract output，在上述例子中我们可以看到，消耗了创建合约时生成的contract output，然后生成了新的contract output
* 调用合约同样可能会修改合约状态，上面的例子中调用`MyToken.buy`后会修改`remain`

下面我们来看看contract state具体包括哪些内容:

```scala
final case class ContractState private (
    code: StatefulContract.HalfDecoded,
    initialStateHash: Hash,
    fields: AVector[Val],
    contractOutputRef: ContractOutputRef
)
```

其中:

* code: `HalfDecoded`主要是考虑到在调用合约时可能只会涉及到部分代码，所以没必要完全decode
* initialStateHash: 合约创建时的state hash
* fields: 以`MyToken`合约为例，这里会包含`owner`和`remain`
* contractOutputRef: 指向contract output

调用合约及合约状态变更的过程大概为:

* 从WorldState中load contract state
* 根据contract state中的`contractOutputRef`去load contract output(在method为payable时执行)
* 合约执行时涉及到修改合约状态时会更新WorldState中的合约状态
* 如果合约生成新的contract output，会更新合约状态中的`contractOutputRef`，同时删除老的contract output

我们以`MyToken`合约为例，来看看合约状态的变更，目前还没有提供直接读取合约状态的接口，所以这里我直接从WorldState中读取`MyToken`的状态数据:

```
创建合约后的状态:
fields: Iterable(Address(P2PKH(Blake2b(hex"40e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee953"))), U256(10000000000000000000000000000))
contractOutputRef: ContractOutputRef(Hint(1829167778),Blake2b(hex"c7d17b2eebf64b13f66c34d3c65df0aa7720b4ddee8efc342a6051e6e293ff4f"))

调用合约后的状态:
fields: Iterable(Address(P2PKH(Blake2b(hex"40e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee953"))), U256(9999999000000000000000000000))
contractOutputRef: ContractOutputRef(Hint(1829167778),Blake2b(hex"5e00b698308b49768e9137e710a0ed57ff96a083a573e53d5aa81478d5c93878"))
```

这里我们可以看到，调用`MyToken.buy`之后，`remain`及`contractOutputRef`都发生了变化。

不同于[eUTXO](https://iohk.io/en/research/library/papers/the-extended-utxo-model/)，在eUTXO中合约的状态也是保存在contract output中，这就导致了cardano上经常提到的[concurrency issue](https://iohk.io/en/blog/posts/2021/09/10/concurrency-and-all-that-cardano-smart-contracts-and-the-eutxo-model/)。alephium的stateful UTXO不仅避免了concurrency issue，而且迁移ETH合约也会变得简单很多。

另外简单说下在创建和调用合约时可能遇到的错误及解决办法:

* NotEnoughBalance: 这个只能通过mining获取奖励或者别人转账来解决
* OutOfGas: 默认的gas比较小，创建和调用合约时通常不够用，所以在创建unsigned tx时一般要手动指定消耗的gas
* AmountIsDustOrZero: 为了避免遭受到攻击，系统会拒绝amount过于小的output，如果想要了解更多可以参考[这里](https://github.com/alephium/alephium/wiki/On-dust-outputs-and-state-explosion)

感兴趣的同学可以尝试在testnet上创建合约，将ETH应用迁移到alephium上来。

