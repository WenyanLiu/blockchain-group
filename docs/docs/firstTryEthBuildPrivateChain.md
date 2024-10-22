# 以太坊使用，私有链搭建

### 1.以太坊客户端安装
我们使用了LINUX，MAC OSX，WINDOWS三种平台，安装运行了go-ethereum。


##### 1.1 go语言安装


**LINUX 安装go语言**  
1). 下载源码  

下载go语言最新的linux版本，go1.10.1.linux-amd64.tar.gz，解压到/usr/local/go

下载地址，https://golang.org/dl/


2). 配置环境变量 
```
命令行输入sudo gedit ~/.bashrc  
在打开的文件最后加上两行代码：

export GOPATH=/usr/local/go
export PATH=$GOPATH/bin:$PATH

命令行输入 source ~/.bashr  使配置生效
命令行输入 go version  验证配置是否成功
```


**MAC OSX 安装go语言**  
http://blog.csdn.net/soindy/article/details/70239442  


**WINDOWS 安装go语言**  
https://studygolang.com/articles/7585

### 2. go-ethereum安装


**LINUX**

    git clone https://github.com/ethereum/go-ethereum
    sudo apt-get install -y build-essential golang
    cd go-ethereum
    make geth

**MAC OSX**

    首先确保已安装 homebrew，没有安装过的可以在命令行下执
    /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)" 进行安装
    brew tap ethereum/ethereum
    brew install ethereum

**WINDOWS**

    访问 https://geth.ethereum.org/downloads/
    下载并安装 Geth for Windows

下面以linux为例：
下面以linux为例：
下面以linux为例：

在环境变量中添加

export PATH=$PATH:/usr/local/ethernum/go-ethereum/build/bin:$PATH


在命令行下输入 geth -h , 有如下显示表示成功

![eth安装成功.png](https://upload-images.jianshu.io/upload_images/1819486-45666a75fdce172f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




### 3. 创建节点和创世区块

以linux为例

##### 3.1  编辑一个区块链文件 genesis.json
```
{
  "config": {
        "chainId": 109,
        "homesteadBlock": 0,
        "eip155Block": 0,
        "eip158Block": 0
    },
  "alloc"      : {},
  "coinbase"   : "0x0000000000000000000000000000000000000000",
  "difficulty" : "0x10000",
  "extraData"  : "",
  "gasLimit"   : "0xffffff",
  "nonce"      : "0x0000000000000077",
  "mixhash"    : "0x0000000000000000000000000000000000000000000000000000000000000000",
  "parentHash" : "0x0000000000000000000000000000000000000000000000000000000000000000",
  "timestamp"  : "0x00"
}
```
##### 3.2 创建节点

任意创建一个文件夹ethereumNode，在其中创建文件data1和data2

在ethereumNode文件下打开两个终端A,B


**终端A中**

对创世区块进行初始化，输入：
```
geth -datadir data1 init genesis.json
```
启动节点  端口号自行设定

参数说明  
networkid     网络标识符  
datadir     	设置当前区块链网络数据存放的位置  
console     	启动命令行模式，可以在Geth中执行命令  
ipcdisable                禁用IPC-RPC服务器
```
geth --networkid 123 --datadir data1 --ipcdisable --port 61910 --rpcport 8200 console
```
**终端B中**  
对创世区块进行初始化，输入：
```
geth -datadir data1 init genesis.json
```
启动节点  修改文件夹data2   端口号修改

```
geth --networkid 123 --datadir data2 --ipcdisable --port 61911 --rpcport 8201 console
```
进入>命令行即为启动成功


### 4. 连接节点

**终端A的命令行中查看节点enode**  
```
>admin.nodeInfo.enode
"enode://2700fee9b8575b3a6df5146b192c74cc0a4eb832a8a3b95a80cc2a9aa73c7abb1a4cf2e734bb0228789611f86c95bcfe2654a187f3fa5ea58b49d245cf014e35@[::]:61910"
```
**终端B的命令行中添加邻居**
```
>admin.addPeer("enode://2700fee9b8575b3a6df5146b192c74cc0a4eb832a8a3b95a80cc2a9aa73c7abb1a4cf2e734bb0228789611f86c95bcfe2654a187f3fa5ea58b49d245cf014e35@[::]:61910")
```
在这里如果是多台机器之间的链接添加，需要将[::]改为对应机器的ip地址  

**终端A的命令行中查看邻居**
```
admin.peers
[{
    caps: ["eth/63"],
    id: "2700fee9b8575b3a6df5146b192c74cc0a4eb832a8a3b95a80cc2a9aa73c7abb1a4cf2e734bb0228789611f86c95bcfe2654a187f3fa5ea58b49d245cf014e35",
    name: "Geth/v1.8.3-unstable-6a2d2869/linux-amd64/go1.9.2",
    network: {
      inbound: true,
      localAddress: "[::1]:61910",
      remoteAddress: "[::1]:50340",
      static: false,
      trusted: false
    },
    protocols: {
      eth: {
        difficulty: 7955328,
        head: "0x78c2aec0fa9d1a5c5477b32117f2b51cf7de622cc14c835f8e17ee81e6435509",
        version: 63
      }
    }
}]

```
可以看到这里的id与终端B节点的的ip是相同的  
同样，也可以在终端B中查看邻居，会得到节点B的信息  
至此，两个节点就在一个区块链上连接成功

## 5. 挖矿与交易

挖矿前要创建账户
```
创建新账号
personal.newAccount()
或者 personal.newAccount("123456")
```
```
挖矿
开始挖矿 miner.start(1)
停止挖矿 miner.stop()
```
挖矿一些时间我们可以查看产生了多少区块等信息
```
eth.blockNumber  查看区块数量
eth.getBlock(1) 通过区块号查看区块
eth.getTransaction("0x0c59f431068937cbe9e230483bc79f59bd7146edc8ff5ec37fea6710adcab825")
```
挖矿成功也会得到token
```
查看账户余额
eth.getBalance(eth.accounts[0])
或者 web3.fromWei(eth.getBalance(eth.accounts[0]), "ether")
```


发起交易前要解锁账户
```
解锁账号
personal.unlockAccount(eth.accounts[0])
使用账户资金前都需要先解锁账号
```
查看终端B的账户信息
```
eth.accounts[0]
"0xe4be4471c30c2552e14d1e26e8384db67b6d7e62"
```

在终端A中查看余额并转账
```
查看余额 单位是wei
> eth.getBalance(eth.accounts[0])
1000000000000000000

转账  0.5个ether
eth.sendTransaction({from:eth.accounts[0],to:"0xe4be4471c30c2552e14d1e26e8384db67b6d7e62",value:web3.toWei(0.5,"ether")})
使用 txpool.status 可以看到交易状态

挖矿确认交易
miner.start()
当有新的区块产生时停止挖矿
miner.stop()
```

查看节点B的余额，转账成功
```
eth.getBalance(eth.accounts[0])
500000000000000000
```

查看节点A的余额，数量反而变多，因为使用节点A进行了挖矿
```
eth.getBalance(eth.accounts[0])
25500000000000000000
```


###   


