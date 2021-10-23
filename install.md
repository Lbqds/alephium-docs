# 安装alephium full node

## Ubuntu 20.04

### private network

首先介绍如何在本地搭建private network.

第一步先下载java(如果本地已经有java运行环境的可以直接跳过这一部分)，这里使用[sdkman](https://sdkman.io/)下载java，sdkman可以非常方便的管理不同版本的java，依次执行下列命令:

```sh
$ curl -s "https://get.sdkman.io" | bash
$ source "$HOME/.sdkman/bin/sdkman-init.sh"
$ sdk install java 11.0.11.hs-adpt
```

接下来我们从[alephium release](https://github.com/alephium/alephium/releases)页面上下载最新版本(当前最新版本为0.11.3)，下载完成功后开始创建配置文件:

```sh
$ cd; mkdir .alephium; cd .alephium
$ echo "alephium.network.network-id = 10\nalephium.discovery.bootstrap = []\n" > user.conf
```

上面我们在`user.conf`中设置了两个参数:

* network-id: alephium通过`network-id`来区分不同的网络环境，默认情况下0为`mainnet`，1为`testnet`，由于这里我们是搭建私有网络，可以随意指定`network-id`
* bootstrap: bootstrap node用于在启动时连接网络中的其他节点，由于我们这里是搭建私有网络，所以不需要bootstrap node

下面我们就可以直接运行alephium full node了:

```sh
$ 切换到之前下载的`alephium-0.11.3.jar`目录下
$ java -jar -Xmx500m alephium-0.11.3.jar
```

当看到类似下面的信息时表明节点已经运行起来了:

```
2021-09-12 13:27:31,828 [main]          INFO  o.a.f.s.Configs$                       - Using user configuration file at /home/lbqds/.alephium/user.conf 

2021-09-12 13:27:31,846 [main]          INFO  o.a.f.s.Configs$                       - Using system configuration file at /home/lbqds/.alephium/system.conf 

2021-09-12 13:27:31,870 [main]          INFO  o.a.f.s.Configs$                       - Using network configuration file at /home/lbqds/.alephium/network.conf 

2021-09-12 13:27:41,242 [-dispatcher-4] INFO  a.e.s.Slf4jLogger                      - Slf4jLogger started
2021-09-12 13:27:41,481 [main]          INFO  o.a.f.c.BlockFlow$                     - Initialize storage for BlockFlow
2021-09-12 13:27:41,604 [dispatcher-12] INFO  o.a.f.n.CliqueManager                  - Intra clique manager is ready
2021-09-12 13:27:41,608 [dispatcher-11] INFO  o.a.f.n.TcpController                  - Node bound to /0:0:0:0:0:0:0:0:9973
2021-09-12 13:27:41,614 [dispatcher-11] INFO  o.a.f.n.DiscoveryServer                - UDP server bound to /0.0.0.0:9973
2021-09-12 13:27:41,637 [main]          INFO  o.a.app.BootUp                         - Build info: name: alephium-app, scalaVersion: 2.13.5, sbtVersion: 1.3.10, commitId: 5bb6610cd0118f6407dab5364baebea2282275a0, branch: 5bb6610cd0118f6407dab5364baebea2282275a0, releaseVersion: 0.11.3
2021-09-12 13:27:41,641 [main]          INFO  o.a.app.BootUp                         - Genesis digests: 028ca2f0-2ece6961-fbef2de2-0c4e2f73-1f8342a4-304417d5-c43d6e06-be5e5d67-cc0289d8-61cd2bc9-99bbaa6a-a12f0ceb-6a1fec4c-a26f079d-f8531fbe-06dbf8af
2021-09-12 13:27:42,613 [ext-global-59] INFO  o.a.a.WebSocketServer                  - Listening ws request on 11973
2021-09-12 13:27:42,613 [ext-global-55] INFO  o.a.app.RestServer                     - Listening http request on 12973
2021-09-12 13:27:42,616 [-dispatcher-9] INFO  o.a.f.m.MinerApiController             - Miner API server bound to /127.0.0.1:10973
```

因为整个网络中只有我们一个节点，所以下面需要开启挖矿，首先我们需要创建钱包:

```sh
$ curl -X 'POST' \
  'http://127.0.0.1:12973/wallets' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "password": "000000",
  "walletName": "alph-wallet",
  "isMiner": true
}'
```

返回的结果为:

```json
{
  "walletName": "alph-wallet",
  "mnemonic": "buzz coconut bubble bargain green bean type parent marine sunny current door glimpse dirt assume pizza tag insect valid opinion girl giraffe street trouble"
}
```

执行上述命令会创建一个名称为`alph-wallet`的钱包，并返回助记词(在mainnet下需要妥善保管)，下面我们需要生成用于挖矿的地址:

```sh
$ curl -X 'POST' \
  'http://127.0.0.1:12973/wallets/alph-wallet/derive-next-miner-addresses' \
  -H 'accept: application/json' \
  -d ''
```

返回结果为:

```json
[
  {
    "address": "1G7Azxs6y7PYGr65CnbGHMXY1F4CjgLn7nXb4m7eX9sNL",
    "group": 0
  },
  {
    "address": "14zzLoo8thi1mqU7N82oStk81aoiDt7gEky2HXa4kAcr",
    "group": 1
  },
  {
    "address": "1EqDgin6jcmH6LG7zQbG3xiQDLZUSWnHqjF7P8jbMhCuD",
    "group": 2
  },
  {
    "address": "15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46",
    "group": 3
  }
]
```

alephium将所有用户划分为多个group(即分片)，默认会分为4个group，所以这里返回了四个地址，分别对应4个group，有了这些地址后我们可以开始挖矿。

我们可以通过两种方法来更新mining addresses:

第一种是直接修改`user.conf`，好处是下次重启时开启挖矿会继续使用这些地址进行挖矿:

```sh
$ cd ~/.alephium/
$ echo 'alephium.mining.miner-addresses = ["1G7Azxs6y7PYGr65CnbGHMXY1F4CjgLn7nXb4m7eX9sNL", "14zzLoo8thi1mqU7N82oStk81aoiDt7gEky2HXa4kAcr", "1EqDgin6jcmH6LG7zQbG3xiQDLZUSWnHqjF7P8jbMhCuD", "15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46"]' >> user.conf
```

**NOTE**: 需要将上述的地址修改为你自己生成的地址，并且要按照先后顺序

下面我们停掉刚刚启动的节点并清理数据库:

```sh
$ cd ~/.alephium/
$ rm -rf db*
```

第二种方法是直接通过REST API来更新挖矿地址，好处是不需要重新启动full-node:

```sh
$ curl -X 'PUT' \
  'http://127.0.0.1:12973/miners/addresses' \
  -H 'accept: */*' \
  -H 'Content-Type: application/json' \
  -d '{
  "addresses": ["1G7Azxs6y7PYGr65CnbGHMXY1F4CjgLn7nXb4m7eX9sNL", "14zzLoo8thi1mqU7N82oStk81aoiDt7gEky2HXa4kAcr", "1EqDgin6jcmH6LG7zQbG3xiQDLZUSWnHqjF7P8jbMhCuD", "15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46"]
}'
```

上面的两种方法可以任意选取一种， 然后通过下面的命令开启挖矿:

```sh
$ curl -X 'POST' \
  'http://127.0.0.1:12973/miners?action=start-mining' \
  -H 'accept: application/json' \
  -d ''
```

如果看到类似下面的输出就表明已经开始挖矿:

```
2021-09-12 14:06:09,843 [dispatcher-10] INFO  o.a.f.m.CpuMiner                       - Start mining
2021-09-12 14:06:09,893 [dispatcher-11] INFO  o.a.f.m.CpuMiner                       - Subscribed for mining tasks
2021-09-12 14:06:26,611 [dispatcher-10] INFO  o.a.f.m.CpuMiner                       - A new block 0000310343a5302efaba55f124407b2c1eb1544e1e8ee3d4357100d3f095b2ab got mined for ChainIndex(2, 3), tx: 1, miningCount: 2164406, target: Target(1e3fffff), miner: 15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46
2021-09-12 14:06:26,718 [dispatcher-10] INFO  o.a.f.h.BlockChainHandler              - f095b2ab is validated
2021-09-12 14:06:26,743 [dispatcher-10] INFO  o.a.f.h.BlockChainHandler              - hash: 0000310343a5302efaba55f124407b2c1eb1544e1e8ee3d4357100d3f095b2ab; ChainIndex(2, 3); height: 1/1; total: 17; targetRatio: 1.0, blockTime: 400420281109ms #tx: 1
2021-09-12 14:06:27,507 [dispatcher-11] INFO  o.a.f.m.CpuMiner                       - A new block 000037106ceaf302230634eb99c110b0c7d99d3d61462611bc1cc06219fda984 got mined for ChainIndex(1, 0), tx: 1, miningCount: 2283734, target: Target(1e3fffff), miner: 1G7Azxs6y7PYGr65CnbGHMXY1F4CjgLn7nXb4m7eX9sNL
2021-09-12 14:06:27,512 [dispatcher-10] INFO  o.a.f.h.BlockChainHandler              - 19fda984 is validated
2021-09-12 14:06:27,516 [dispatcher-10] INFO  o.a.f.h.BlockChainHandler              - hash: 000037106ceaf302230634eb99c110b0c7d99d3d61462611bc1cc06219fda984; ChainIndex(1, 0); height: 1/1; total: 18; targetRatio: 1.0, blockTime: 400420281790ms #tx: 1
2021-09-12 14:06:31,405 [dispatcher-10] INFO  o.a.f.m.CpuMiner                       - A new block 00001b5c11a311b38bf7d3e228906182d50e34f27598468ece0a4563d6093ffe got mined for ChainIndex(3, 2), tx: 1, miningCount: 2785228, target: Target(1e3fffff), miner: 1EqDgin6jcmH6LG7zQbG3xiQDLZUSWnHqjF7P8jbMhCuD
2021-09-12 14:06:31,409 [dispatcher-20] INFO  o.a.f.h.BlockChainHandler              - d6093ffe is validated
2021-09-12 14:06:31,411 [dispatcher-20] INFO  o.a.f.h.BlockChainHandler              - hash: 00001b5c11a311b38bf7d3e228906182d50e34f27598468ece0a4563d6093ffe; ChainIndex(3, 2); height: 1/1; total: 19; targetRatio: 1.0, blockTime: 400420284559ms #tx: 1
2021-09-12 14:06:32,732 [dispatcher-10] INFO  o.a.f.m.CpuMiner                       - A new block 00003be557e82e2124953231d0b4a316561bc469090e6d58917958fdd2ae0e91 got mined for ChainIndex(0, 1), tx: 1, miningCount: 2964847, target: Target(1e3fffff), miner: 14zzLoo8thi1mqU7N82oStk81aoiDt7gEky2HXa4kAcr
2021-09-12 14:06:32,737 [dispatcher-11] INFO  o.a.f.h.BlockChainHandler              - d2ae0e91 is validated
```

**NOTE**: 如果挖矿比较慢的话，可以尝试追加下面两个参数到`user.conf`中(需要清空数据库重新启动):

* alephium.consensus.block-target-time = 30 seconds  设置出块时间
* alephium.consensus.num-zeros-at-least-in-hash = 10 降低挖矿难度

接下来我们可以通过下面的命令查看我们的挖矿奖励:

```sh
$ curl -X 'POST' \
  'http://127.0.0.1:12973/wallets/alph-wallet/unlock' \
  -H 'accept: */*' \
  -H 'Content-Type: application/json' \
  -d '{
  "password": "000000"
}'

$ curl -X 'GET' \
  'http://127.0.0.1:12973/wallets/alph-wallet/balances' \
  -H 'accept: application/json' 
```

返回结果为:

``` json
{
  "totalBalance": "1875000000000000000",
  "balances": [
    {
      "address": "18zY2hb3PqVFj7UiWcBp4A5GGYBbCmjmmz7WFYPALZGYk",
      "balance": "0"
    },
    {
      "address": "19pxkRbqShQLW5EiLEeAmhCDATTvEz7HPbT8TDgkXuLDR",
      "balance": "0"
    },
    {
      "address": "17rnsock4fRrU6wYxCtoogit96nKjWzqjKSMA2se5J2QK",
      "balance": "0"
    },
    {
      "address": "1FcFfMU5NmQY4Tj71ApytZpsgAEYLhkX3s4ZWy3PVqVeR",
      "balance": "0"
    },
    {
      "address": "1G7Azxs6y7PYGr65CnbGHMXY1F4CjgLn7nXb4m7eX9sNL",
      "balance": "0"
    },
    {
      "address": "14zzLoo8thi1mqU7N82oStk81aoiDt7gEky2HXa4kAcr",
      "balance": "0"
    },
    {
      "address": "1EqDgin6jcmH6LG7zQbG3xiQDLZUSWnHqjF7P8jbMhCuD",
      "balance": "0"
    },
    {
      "address": "15NMkQuPg7GE4eiXAed3rUJV8NswipscoBUY22mqZSX46",
      "balance": "1875000000000000000"
    }
  ]
}
```

根据返回结果可以看到我们的地址已经收到了挖矿奖励，表明我们已经成功地搭建了一个alephium私有网络，如果你感兴趣可以继续下面的操作:

* 在局域网内启动一个新节点，将刚刚的节点作为新节点的bootstrap-node，看看这两个节点是否能够正常同步
* 参照`127.0.0.1:12973/docs`下的文档，给其他的账户发送ALPH，看看交易是否能够成功

**NOTE**: 上述所有通过`curl`发起的请求都可以在`127.0.0.1:12973/docs`下执行，而且更加方便。

### 加入testnet

为了能够加入testnet，在上面private network的基础上，我们只需要修改`user.conf`中的两个参数(如果修改了`consensus`的参数也要删除掉):

* alephium.network.network-id = 1
* alephium.discovery.bootstrap = ["18.185.54.38:9973", "3.141.11.239:9973", "54.79.138.71:9973"]

由于private network跟testnet的数据不兼容，所以还需要删除数据库，然后重新运行full node即可加入testnet，节点启动后会自动从网络同步，当同步完成后如果想要在testnet下挖矿，可以通过上述private network同样的命令开启挖矿。

### 通过Docker运行full node

在Ubuntu 20.04上可以通过[snap](https://snapcraft.io/about)安装docker(通过snap安装docker也包括了下面要用到的`docker-compose`)，执行:

```sh
$ sudo snap install docker
```

安装完成后将`alephum`代码clone到本地:

```sh
$ git clone git@github.com:alephium/alephium.git
```

然后切换到`docker-compose.yml`所在的目录:

```sh
$ cd alephium/docker
```

通过`docker-compose`运行full node:

```sh
$ sudo docker-compose up
```

**NOTE**: 如果看到`exec user process caused: operation not permitted`错误提示，将`docker-compose.yml`中的`no-new-privileges:true`修改为`no-new-privileges:false`后重新运行上述命令

当看到下面的输出时就表示运行成功了:

```
alephium_1    | 2021-09-12 04:38:03,528 [dispatcher-14] INFO  o.a.f.n.i.OutboundBrokerHandler        - Connected to /91.158.35.136:9973
alephium_1    | 2021-09-12 04:38:03,535 [dispatcher-20] INFO  o.a.f.n.i.OutboundBrokerHandler        - Connected to /65.21.246.250:9973
alephium_1    | 2021-09-12 04:38:03,568 [dispatcher-18] INFO  o.a.f.n.i.OutboundBrokerHandler        - Connected to /3.66.121.189:9973
alephium_1    | 2021-09-12 04:38:03,568 [dispatcher-18] INFO  o.a.f.n.i.OutboundBrokerHandler        - Connected to /34.125.183.112:9973
alephium_1    | 2021-09-12 04:38:03,568 [dispatcher-18] INFO  o.a.f.n.i.OutboundBrokerHandler        - Connected to /3.15.45.47:9973
alephium_1    | 2021-09-12 04:38:03,572 [dispatcher-13] INFO  o.a.f.n.i.OutboundBrokerHandler        - Connected to /3.122.234.1:9973
alephium_1    | 2021-09-12 04:38:03,579 [dispatcher-12] INFO  o.a.f.n.i.OutboundBrokerHandler        - Connected to /3.126.139.161:9973
alephium_1    | 2021-09-12 04:38:03,659 [dispatcher-12] INFO  o.a.f.n.i.OutboundBrokerHandler        - Connected to /13.211.88.121:9973
alephium_1    | 2021-09-12 04:38:03,737 [-dispatcher-7] INFO  o.a.f.n.i.OutboundBrokerHandler        - Connected to /13.239.2.91:9973
alephium_1    | 2021-09-12 04:38:03,751 [ext-global-58] INFO  o.a.a.WebSocketServer                  - Listening ws request on 11973
alephium_1    | 2021-09-12 04:38:03,751 [ext-global-64] INFO  o.a.app.RestServer                     - Listening http request on 12973
alephium_1    | 2021-09-12 04:38:03,754 [dispatcher-12] INFO  o.a.f.m.MinerApiController             - Miner API server bound to /127.0.0.1:10973
grafana_1     | t=2021-09-12T04:38:14+0000 lvl=info msg="Request Completed" logger=context userId=0 orgId=0 uname= method=GET path=/ status=302 remote_addr=172.19.0.1 time_ms=0 size=29 referer=
alephium_1    | 2021-09-12 04:38:14,860 [dispatcher-12] INFO  o.a.f.h.BlockChainHandler              - 07c48721 is validated
alephium_1    | 2021-09-12 04:38:14,876 [dispatcher-12] INFO  o.a.f.h.BlockChainHandler              - hash: 000024cb15990177f0704b3cba973c8742debcf6684dd93f255c08c007c48721; ChainIndex(0, 1); height: 132/132; total: 2017; targetRatio: 0.99999976, blockTime: 45508ms #tx: 1
alephium_1    | 2021-09-12 04:38:14,878 [dispatcher-12] INFO  o.a.f.h.BlockChainHandler              - 35d19832 is validated
alephium_1    | 2021-09-12 04:38:14,881 [dispatcher-12] INFO  o.a.f.h.BlockChainHandler              - b38ba781 is validated
alephium_1    | 2021-09-12 04:38:14,881 [dispatcher-12] INFO  o.a.f.h.BlockChainHandler              - hash: 00002d464dfd0c951ba2be84919b8785f368539912de99a70c8e6d3235d19832; ChainIndex(0, 2); height: 121/121; total: 2018; targetRatio: 0.9999473, blockTime: 77551ms #tx: 1
alephium_1    | 2021-09-12 04:38:14,882 [dispatcher-12] INFO  o.a.f.h.BlockChainHandler              - hash: 000000dbe2aa453b7350a6b2b2fa293be5fa8c2b4fd52cc8541a150fb38ba781; ChainIndex(0, 1); height: 133/133; total: 2019; targetRatio: 0.99999976, blockTime: 73157ms #tx: 1
alephium_1    | 2021-09-12 04:38:14,885 [dispatcher-12] INFO  o.a.f.h.BlockChainHandler              - 19f77701 is validated
alephium_1    | 2021-09-12 04:38:14,886 [dispatcher-12] INFO  o.a.f.h.BlockChainHandler              - 525fe6d0 is validated
alephium_1    | 2021-09-12 04:38:14,887 [dispatcher-12] INFO  o.a.f.h.BlockChainHandler              - hash: 00001aa595cc09ca40cde0401b0022737b2c476952fd49775440fd0b19f77701; ChainIndex(0, 1); height: 134/134; total: 2020; targetRatio: 0.99963665, blockTime: 98559ms #tx: 1
alephium_1    | 2021-09-12 04:38:14,888 [dispatcher-12] INFO  o.a.f.h.BlockChainHandler              - 5be98f01 is validated
alephium_1    | 2021-09-12 04:38:14,890 [dispatcher-12] INFO  o.a.f.h.BlockChainHandler              - hash: 00000a857fc65b9a4f1d5f9b3a422b726df1aa1b646606bd6eaf3d585be98f01; ChainIndex(0, 1); height: 135/135; total: 2021; targetRatio: 0.999614, blockTime: 4505ms #tx: 1
```

**NOTE**: 也可以通过`sudo docker-compose up -d`命令在后台运行

通过上面的`docker-compose`我们一共运行了三个container，分别是: `alephium-full-node`, `prometheus`和`grafana`，后面两个用于监控alephium full node的运行状态，我们可以在浏览器中输入`127.0.0.1:3000`进入监控页面。同样地也可以通过`127.0.0.1:12973/docs`查看链上的数据。

TODO: 通过mining-companion挖矿

### Windows 10

Windows 10需要设置环境变量`HOME`为`C:\Users\your-user-name`，然后在`HOME`目录下创建`.alephium`目录，其余步骤和上述private network步骤类似.

在Windows环境下可能还会遇到其他奇怪的问题，如果有问题欢迎提issue。

