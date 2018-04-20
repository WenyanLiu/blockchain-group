# 区块链学习

## 目录

* [开始使用以太坊](./开始使用以太坊.md)
* 以太坊源码学习
  * 数据结构
      * trie: [Modified Merkle Patricia Tree](./trie.md)
  * （待整理）
      * [Sender函数](./Sender.md)
      * [EVM解释器](./EVMInterpreter.md)
* 以太坊分析工具
  * [plyvel](./plyvel.md)：数据库访问工具
  * [log4go-ethereum](./log4go.md)：针对Go Ethereum的日志记录工具

## 路线

| 日期 | 主题 | 作者 | 存档 |
| :-: | :-: | :-: | :-: |
| 2018/3/15 | [搭建以太坊私有链](./firstTryEthBuildPrivateChain.md) | 王浩 | :checkered_flag: |
| 2018/3/20 | [以太坊架构梳理](./以太坊架构梳理.md) | 袁佳豪 | :checkered_flag:  |
| 2018/3/20 | 区块与交易的数据结构初探 | 刘文炎 |  |
| 2018/3/20 | Miner包 Consensus包 | 王浩 |  |
| 2018/3/24 | Transaction与block的打包与提交 | 袁佳豪 |  |
| 2018/3/24 | 以太坊物理存储分析 | 刘文炎 |  |
| 2018/3/24 | [Miner包中的挖矿流程](./minerPackage.md) | 王浩 |:checkered_flag:  |
| 2018/3/24 | 以太坊物理存储分析<br>[数据库访问工具](./plyvel.md) | 刘文炎 |  |
| 2018/3/27 | Gas管理 DoS攻击 交易打包与执行 交易堵塞 | 袁佳豪 |  |
| 2018/3/27 | 从世界状态获取账户余额 | 刘文炎 |  |
| 2018/3/27 | worker中新区块组装与交易提交| 王浩 ||
| 2018/3/30 | [Gas价目与虚拟机外的交易执行](./ApplyTransaction.md) | 袁佳豪 | :checkered_flag: |
| 2018/3/30 | [改进的默克尔·帕特里夏树](./trie.md) | 刘文炎 | :checkered_flag: |
| 2018/3/30 | [新区块的插入](./newBlockInsert.md) | 王浩 | :checkered_flag:  |
| 2018/3/30 | [fetcher与downloader](./fetcherAndDownLoader.md) | 王浩 |  |
| 2018/4/4 | [EVM初探](./EVM.md) | 袁佳豪 |  |
| 2018/4/4 | [日志记录工具](./log4go.md)<br>[Sender函数](./Sender.md) | 刘文炎 | :checkered_flag: |
| 2018/4/12 | ??? | 袁佳豪 |  |
| 2018/4/12 | [以太坊虚拟机解释器](./EVMInterpreter.md) | 刘文炎 |  |
| 2018/4/12 | [新区块写入WriteBlockWithState方法](./InsertChainAndWriteBlockWithState.md) | 王浩 |  :checkered_flag: |
| 2018/4/17 | ??? | 袁佳豪 |  |
| 2018/4/17 | [以太坊虚拟机释疑](./EVMInterpreter.md#%E9%87%8A%E7%96%91%EF%B8%8F) | 刘文炎 |  |
| 2018/4/17 | [以太坊三棵树](./threeTrees.md) | 王浩 |  |
| 2018/4/24 | ??? | 袁佳豪 |  |
| 2018/4/24 | [以太坊虚拟机释疑2](./EVMInterpreter.md#%E9%87%8A%E7%96%91%EF%B8%8F)<br>[智能合约的调用方式](./EVMInterpreter.md#) | 刘文炎 |  |
| 2018/4/24 | ??? | 王浩 |  |


  

## 参考文献
1. Wood, G., 2014. [Ethereum: A secure decentralised generalised transaction ledger](https://ethereum.github.io/yellowpaper/paper.pdf), Ethereum Project Yellow Paper.

