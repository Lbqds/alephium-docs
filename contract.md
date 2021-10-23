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
  'http://127.0.0.1:12973/contracts/compile-contract' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "code": "TxContract MyToken(owner: Address, mut remain: U256) {\npub payable fn buy(from: Address, alfAmount: U256) -> () {\nlet tokenAmount = alfAmount * 1000\nassert!(remain >= tokenAmount)\nlet tokenId = selfTokenId!()\ntransferAlf!(from, owner, alfAmount)\ntransferTokenFromSelf!(from, tokenId, tokenAmount)\nremain = remain - tokenAmount\n}\n}"
}'
```

执行上述request后，会返回contract binary code:

```json
{
  "code": "0201402c01010204001616011343e82c1702a0011602344db117031600a0001601a7160016031602aba00116022ba101"
}
```

下面开始创建unsigned transaction:

```shell
curl -X 'POST' \
  'http://127.0.0.1:12973/contracts/build-contract' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "fromPublicKey": "022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f402",
  "code": "0201402c01010204001616011343e82c1702a0011602344db117031600a0001601a7160016031602aba00116022ba101",
  "gas": 100000,
  "gasPrice": "100000000000",
  "state": "[@15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46,10000000000000000000000000000u]",
  "issueTokenAmount": "10000000000000000000000000000"
}'
```

这里简单介绍一下上面的几个参数:

* fromPublicKey: 创建合约的用户public key
* code: contract source code
* state: 合约需要的两个初始状态，对应合约代码的两个参数
* issueTokenAmount: 合约发行token的总数

执行上述request后，会返回类似下面的response:

```json
{
  "unsignedTx": "010101010100000007150040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95313c1e8d4a51000a21440300201402c01010204001616011343e82c1702a0011602344db117031600a0001601a7160016031602aba00116022ba10114403102040040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95302c8204fce5e3e2502611000000013c8204fce5e3e25026110000000ae800186a0c1174876e80002c168f9330e156dcad570f73323a970080432ac0000a0c39abac91586c1f90dc3a6d373f500022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f402c168f9338ca37e7cd81c8e3cbd61376586a740efc3751e37333d50654e1df8791a9f066f00022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f40200",
  "hash": "189ea4dd5978ac97ec83d0632750bb811dbcb540808908258c28fd155662e00b",
  "contractId": "c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80",
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
  "data": "189ea4dd5978ac97ec83d0632750bb811dbcb540808908258c28fd155662e00b"
}'
```

上面的request会请求对`txHash`签名，得到的结果为:

```json
{
  "signature": "067a8a69c40e850c9895e296cf15a6094d41574c7932bc63a3c7726405f397922910af6f6c478b529d5cc0e7559f133417f8b5ba55e79344e745a6e51d6054d4"
}
```

现在我们所有创建合约所需的数据都已经准备好了，下面执行submit tx:

```shell
$ curl -X 'POST' \
  'http://127.0.0.1:12973/transactions/submit' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "unsignedTx": "010101010100000007150040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95313c1e8d4a51000a21440300201402c01010204001616011343e82c1702a0011602344db117031600a0001601a7160016031602aba00116022ba10114403102040040e767b37fb1db1af9c5a56e322c9d92e0f718a6ece581214319db0af62ee95302c8204fce5e3e2502611000000013c8204fce5e3e25026110000000ae800186a0c1174876e80002c168f9330e156dcad570f73323a970080432ac0000a0c39abac91586c1f90dc3a6d373f500022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f402c168f9338ca37e7cd81c8e3cbd61376586a740efc3751e37333d50654e1df8791a9f066f00022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f40200",
  "signature": "067a8a69c40e850c9895e296cf15a6094d41574c7932bc63a3c7726405f397922910af6f6c478b529d5cc0e7559f133417f8b5ba55e79344e745a6e51d6054d4"
}' 
```

执行上述request后，如果合约是有效的，会返回类似如下结果:

```json
{
  "txId": "189ea4dd5978ac97ec83d0632750bb811dbcb540808908258c28fd155662e00b",
  "fromGroup": 3,
  "toGroup": 3
}
```

这时我们就可以等待tx直到confirmed，我们可以直接在testnet explorer上查看[这笔交易](https://testnet.alephium.org/#/transactions/189ea4dd5978ac97ec83d0632750bb811dbcb540808908258c28fd155662e00b)。

为了更清楚地了解创建合约的过程，下面我们来看看上述tx的具体内容(目前暂时还没有查看tx详细信息的接口，所以我们这里直接使用获取block的接口，后面我们会提供针对tx的接口):

```shell
$ curl -X 'GET' \
  'http://127.0.0.1:12973/blockflow/blocks/000000065796124c9950f913e98fcb1a9f15ae937b92419e4f10e05ac6f3474f' \
  -H 'accept: application/json'
```

得到的结果如下:

```json
{
  "hash": "000000065796124c9950f913e98fcb1a9f15ae937b92419e4f10e05ac6f3474f",
  "timestamp": 1634954617055,
  "chainFrom": 3,
  "chainTo": 3,
  "height": 20871,
  "deps": [
    "00000005608265594fb2e1c1edf7e3c913701ed88e87fefd68c1fb61e152ce90",
    "00000003177290f7d4afbe01fb64feaf09f42cbd31db6c37a5838aef3982b335",
    "000000090b07066aeb8c6fa8513101d84cfa87779ff0d75fef5b2bc5cc904b0a",
    "0000000330143898b3db70c47ac7b4326f148cdf73801e05ec6b60b25ae2a0ac",
    "000000019310fe3c960922bb23cb873e0fc723dd1fab42d7f9f64765e545732d",
    "000000071d82202649a568a56ab618c25cefea6f07d6aa11d5e6610b7270b39e",
    "0000000773419f867842b8b51ecfa76cfa09dca7196415fb13590d1ed3ac6dff"
  ],
  "transactions": [
    {
      "id": "189ea4dd5978ac97ec83d0632750bb811dbcb540808908258c28fd155662e00b",
      "inputs": [
        {
          "type": "asset",
          "outputRef": {
            "hint": -1050085069,
            "key": "0e156dcad570f73323a970080432ac0000a0c39abac91586c1f90dc3a6d373f5"
          },
          "unlockScript": "00022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f402"
        },
        {
          "type": "asset",
          "outputRef": {
            "hint": -1050085069,
            "key": "8ca37e7cd81c8e3cbd61376586a740efc3751e37333d50654e1df8791a9f066f"
          },
          "unlockScript": "00022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f402"
        }
      ],
      "outputs": [
        {
          "type": "contract",
          "amount": "1000000000000",
          "address": "28GkvM9xuRB5Ku33ZstXxr81ydWQ8WQoWxNVBsL8StbmR",
          "tokens": [
            {
              "id": "c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80",
              "amount": "10000000000000000000000000000"
            }
          ]
        },
        {
          "type": "asset",
          "amount": "3989999000000000000",
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
      "id": "2c4c024f9e992b28cdf57b2be924d3690550e6147c9a64159a7777547f372f2f",
      "inputs": [],
      "outputs": [
        {
          "type": "asset",
          "amount": "2311101497821509838",
          "address": "18skQAqJSUbsDgik2pS6HW5tuMGNuCyPYGuHfn3C4S9Q3",
          "tokens": [],
          "lockTime": 1634955217055,
          "additionalData": "03030000017caae3a0df"
        }
      ],
      "gasAmount": 20000,
      "gasPrice": "1000000000"
    }
  ]
}
```

这里我们只关注txId为`189ea4dd5978ac97ec83d0632750bb811dbcb540808908258c28fd155662e00b`的交易:

```json
{
  "id": "189ea4dd5978ac97ec83d0632750bb811dbcb540808908258c28fd155662e00b",
  "inputs": [
    {
      "type": "asset",
      "outputRef": {
        "hint": -1050085069,
        "key": "0e156dcad570f73323a970080432ac0000a0c39abac91586c1f90dc3a6d373f5"
      },
      "unlockScript": "00022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f402"
    },
    {
      "type": "asset",
      "outputRef": {
        "hint": -1050085069,
        "key": "8ca37e7cd81c8e3cbd61376586a740efc3751e37333d50654e1df8791a9f066f"
      },
      "unlockScript": "00022ef57a83ccaa8b72f1c7c68b0d784ccf734e4383f9dbd6982c1c271eedd9f402"
    }
  ],
  "outputs": [
    {
      "type": "contract",
      "amount": "1000000000000",
      "address": "28GkvM9xuRB5Ku33ZstXxr81ydWQ8WQoWxNVBsL8StbmR",
      "tokens": [
        {
          "id": "c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80",
          "amount": "10000000000000000000000000000"
        }
      ]
    },
    {
      "type": "asset",
      "amount": "3989999000000000000",
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

我们可以看到这笔tx有一个类型为`contract`的output，下面来解释contract output的每个字段:

* type: tx output的类型，`contract`或者`asset`
* amount: **ALPH**的数量，这里`1000000000000`为创建合约时默认从用户地址转给合约的数量(可以为任意值)
* address: contract address的base58编码
* tokens: 其中包括了我们刚刚创建的token amount和token id

合约创建成功后，我们可以查看contract state:

```shell
$ curl -X 'GET' \
  'http://127.0.0.1:12973/contracts/28GkvM9xuRB5Ku33ZstXxr81ydWQ8WQoWxNVBsL8StbmR/state?group=3' \
  -H 'accept: application/json'
```

得到的contract state为:

```json
{
  "fields": [
    {
      "type": "address",
      "value": "15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46"
    },
    {
      "type": "u256",
      "value": "10000000000000000000000000000"
    }
  ]
}
```

我们可以看到contract state即为创建合约时初始化的state

合约创建成功后，我们接下来开始调用合约，即通过`MyToken.buy`向合约支付**ALPH**来换取token

# Call contract

首先我们需要写script来调用上面创建的contract:

```
TxScript Main {
  pub payable fn main() -> () {
    let address = @1FcFfMU5NmQY4Tj71ApytZpsgAEYLhkX3s4ZWy3PVqVeR 
    let amount = 1000000000000000000
    let contractId = #c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80
    approveAlf!(address, amount)
    let contract = MyToken(contractId)
    contract.buy(address, amount)
  }
}
```

简单解释一下这一段代码:

* 通过`approveAlf!`授权contract代为支付**ALPH**
* `MyToken(contractId)` 会从world state中加载contract
* `contract.buy(address, amount)` 会调用`MyToken.buy`

同样，我们需要先编译为二进制代码(需要注意的是这里也要带上被调用的合约代码):

```shell
$ curl -X 'POST' \
  'http://127.0.0.1:12973/contracts/compile-script' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "code": "TxScript Main {\n pub payable fn main() -> () {\n let address = @1FcFfMU5NmQY4Tj71ApytZpsgAEYLhkX3s4ZWy3PVqVeR \n let amount = 1000000000000000000\n let contractId = #c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80\n approveAlf!(address, amount)\n let contract = MyToken(contractId)\n contract.buy(address, amount)\n }\n }\n TxContract MyToken(owner: Address, mut remain: U256) {\npub payable fn buy(from: Address, alfAmount: U256) -> () {\nlet tokenAmount = alfAmount * 1000\nassert!(remain >= tokenAmount)\nlet tokenId = selfTokenId!()\ntransferAlf!(from, owner, alfAmount)\ntransferTokenFromSelf!(from, tokenId, tokenAmount)\nremain = remain - tokenAmount\n}\n}"
}'
```

得到script binary code:

```json
{
  "code": "0101010004000f1500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea170013c40de0b6b3a76400001701144020c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80170216001601a2160217031600160116030100"
}
```

然后我们根据binary code来创建unsigned tx(这里我们使用新的地址):

```shell
$ curl -X 'POST' \
  'http://127.0.0.1:12973/contracts/build-script' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "fromPublicKey": "02b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b",
  "code": "0101010004000f1500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea170013c40de0b6b3a76400001701144020c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80170216001601a2160217031600160116030100",
  "gas": 100000,
  "gasPrice": "100000000000"
}'
```

返回结果为:

```json
{
  "unsignedTx": "01010101010004000f1500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea170013c40de0b6b3a76400001701144020c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80170216001601a2160217031600160116030100800186a0c1174876e800017aa0f6ff37c5bc885bd198d0a6cac7c87b6967b3f23636ff01cdc966b949ea785373c8ed0002b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b00",
  "hash": "df4c3bbad175c38409aefdb9c27046fb04578dfd915625334a95153e11040020",
  "fromGroup": 3,
  "toGroup": 3
}
```

然后对txHash签名，得到的签名为:

```json
{
  "signature": "73291f3bcc5083874305c945dd848abdeffc160e5af767d3f875aea1d19e3a7c696150b3073e240d01555957f0a7c9bd5bba5f32e948adb3cad9aa372d0ee1e7"
}
```

得到签名后，提交tx:

```shell
$ curl -X 'POST' \
  'http://127.0.0.1:12973/transactions/submit' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "unsignedTx": "01010101010004000f1500d90b85d7ab9aec01def906cb80331a19a7fea1a0fc6fd536b556362a4e405aea170013c40de0b6b3a76400001701144020c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80170216001601a2160217031600160116030100800186a0c1174876e800017aa0f6ff37c5bc885bd198d0a6cac7c87b6967b3f23636ff01cdc966b949ea785373c8ed0002b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b00",
  "signature": "73291f3bcc5083874305c945dd848abdeffc160e5af767d3f875aea1d19e3a7c696150b3073e240d01555957f0a7c9bd5bba5f32e948adb3cad9aa372d0ee1e7"
}' 
```

同样，可以在testnet上找到[这笔交易](https://testnet.alephium.org/#/transactions/df4c3bbad175c38409aefdb9c27046fb04578dfd915625334a95153e11040020)。

下面我们来看看这笔tx的具体内容:

```json
{
  "id": "df4c3bbad175c38409aefdb9c27046fb04578dfd915625334a95153e11040020",
  "inputs": [
    {
      "type": "asset",
      "outputRef": {
        "hint": 2057369343,
        "key": "37c5bc885bd198d0a6cac7c87b6967b3f23636ff01cdc966b949ea785373c8ed"
      },
      "unlockScript": "0002b79c9fc0e442d9c196ef7b09cd1410665396dd30ffc5d05f5b24b816f67d480b"
    },
    {
      "type": "contract",
      "outputRef": {
        "hint": -425827896,
        "key": "c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80"
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
      "amount": "990000000000000000",
      "address": "1FcFfMU5NmQY4Tj71ApytZpsgAEYLhkX3s4ZWy3PVqVeR",
      "tokens": [
        {
          "id": "c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80",
          "amount": "1000000000000000000000"
        }
      ],
      "lockTime": 0,
      "additionalData": ""
    },
    {
      "type": "contract",
      "amount": "1000000000000",
      "address": "28GkvM9xuRB5Ku33ZstXxr81ydWQ8WQoWxNVBsL8StbmR",
      "tokens": [
        {
          "id": "c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80",
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
    "hint": -425827896,
    "key": "c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80"
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
  "amount": "990000000000000000",
  "address": "1FcFfMU5NmQY4Tj71ApytZpsgAEYLhkX3s4ZWy3PVqVeR",
  "tokens": [
    {
      "id": "c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80",
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
  "address": "28GkvM9xuRB5Ku33ZstXxr81ydWQ8WQoWxNVBsL8StbmR",
  "tokens": [
    {
      "id": "c9d105c99b1fb26f213fb80797e76959c137659fb13bd77e935d05d557c59e80",
      "amount": "9999999000000000000000000000"
    }
  ]
}
```

这里我们可以看到，contract output的token数量从原来的`10000000000000000000000000000`变为现在的`9999999000000000000000000000`，少的数量刚好是我们兑换的token数量。

同样，我们可以再次查看contract state，得到的结果为:

```json
{
  "fields": [
    {
      "type": "address",
      "value": "15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46"
    },
    {
      "type": "u256",
      "value": "9999999000000000000000000000"
    }
  ]
}
```

我们可以看到token数量发生了变化

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

对比上述合约调用前后的contract state可以发现调用`MyToken.buy`之后，`remain`及`contractOutputRef`都发生了变化。

另外简单说下在创建和调用合约时可能遇到的错误及解决办法:

* NotEnoughBalance: 这个只能通过mining获取奖励或者别人转账来解决
* OutOfGas: 默认的gas比较小，创建和调用合约时通常不够用，所以在创建unsigned tx时一般要手动指定消耗的gas
* InvalidOutputStats: 为了避免遭受到攻击，系统会拒绝amount过于小的output，如果想要了解更多可以参考[这里](https://github.com/alephium/alephium/wiki/On-dust-outputs-and-state-explosion)

感兴趣的同学可以尝试在testnet上创建合约，将ETH应用迁移到alephium上来

