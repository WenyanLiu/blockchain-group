# 区块链学习

## 目录

* [开始使用以太坊](./开始使用以太坊.md)
* 以太坊源码学习
  * 以太坊结构框架
      *[以太坊框架梳理](./以太坊架构梳理.md)
  * 数据结构与存储
      * trie: [Modified Merkle Patricia Tree](./trie.md)
      * [区块与交易数据结构初探](./blockTransactionDataStructure.md)
      * [物理存储](./physicalView.md)
  * （待整理）
      * [fallback函数](./fallbackFunction.md)
      * [Sender函数](./sender.md)
      * [世界状态](./accountBalance.md)
      * [EVM解释器](./evmInterpreter.md)
      * [黄皮书拾遗](./yellowpaperOmissions.md)
      * [《以太坊技术详解与实战》拾遗](./yyanBookOmissions.md)
      * [隐私保护与数据安全](./yyBookPrivacy.md)
      * [区块验证细节](./verifyBlock.md)
      * [《以太坊技术详解与实战》数字资产发行和数据查询分析工具](./以太坊数字资产发行和数据查询分析工具.md)
  * 交易与区块的打包、提交与执行
      * [Transaction与block的打包与提交](./transactionAndBlock.md)
      * [Gas管理 DoS攻击 交易打包与执行 交易堵塞]()
      * [Gas价目与虚拟机外的交易执行](./applyTransaction.md)
  * 以太坊虚拟机
      * [EVM初探](./evm.md) 
      * [虚拟机基本内容](./evm学习.md)
      * [EVM存储设计](./evm存储.md)
      * [Memory 使用与新增Opcode](./evm存储.md#存储管理)
* 以太坊分析工具
  * [plyvel](./plyvel.md)：数据库访问工具
  * [log4go-ethereum](./log4go.md)：针对Go Ethereum的日志记录工具

## 路线

| 日期 | 主题 | 作者 | 存档 |
| :-: | :-: | :-: | :-: |
| 2018/3/15 | [搭建以太坊私有链](./firstTryEthBuildPrivateChain.md) | 王浩 | :checkered_flag: |
| 2018/3/20 | [以太坊架构梳理](./以太坊架构梳理.md) | 袁佳豪 | :checkered_flag:  |
| 2018/3/20 | [区块与交易的数据结构初探](./blockTransactionDataStructure.md) | 刘文炎 | :checkered_flag: |
| 2018/3/20 | [共识机制与Consensus包](./consensus.md) | 王浩 |  |
| 2018/3/24 | [Transaction与block的打包与提交](./transactionAndBlock.md) | 袁佳豪 | :checkered_flag: |
| 2018/3/24 | [以太坊物理存储分析](./physicalView.md)<br>[数据库访问工具](./plyvel.md) | 刘文炎 | :checkered_flag: |
| 2018/3/24 | [Miner包中的挖矿流程](./minerPackage.md) | 王浩 |:checkered_flag:  |
| 2018/3/27 | Gas管理 DoS攻击 交易打包与执行 交易堵塞 | 袁佳豪 |  |
| 2018/3/27 | [从世界状态获取账户余额](./accountBalance.md) | 刘文炎 | :checkered_flag: |
| 2018/3/27 | worker中新区块组装与交易提交| 王浩 ||
| 2018/3/30 | [Gas价目与虚拟机外的交易执行](./applyTransaction.md) | 袁佳豪 | :checkered_flag: |
| 2018/3/30 | [改进的默克尔·帕特里夏树](./trie.md) | 刘文炎 | :checkered_flag: |
| 2018/3/30 | [新区块的插入](./newBlockInsert.md)<br>[fetcher与downloader](./fetcherAndDownLoader.md) | 王浩 |  |
| 2018/4/4 | [EVM初探](./evm.md) | 袁佳豪 | :checkered_flag: |
| 2018/4/4 | [日志记录工具](./log4go.md)<br>[Sender函数](./sender.md) | 刘文炎 | :checkered_flag: |
| 2018/4/12 | [虚拟机基本内容](./evm学习.md) | 袁佳豪 | :checkered_flag: |
| 2018/4/12 | [以太坊虚拟机解释器](./evmInterpreter.md) | 刘文炎 | :checkered_flag: |
| 2018/4/12 | [新区块写入WriteBlockWithState方法](./insertChainAndWriteBlockWithState.md) | 王浩 |  :checkered_flag: |
| 2018/4/17 | [EVM存储设计](./evm存储.md)| 袁佳豪 |  |
| 2018/4/17 | [以太坊虚拟机释疑](./evmInterpreter.md#%E9%87%8A%E7%96%91%EF%B8%8F) | 刘文炎 | :checkered_flag: |
| 2018/4/17 | [以太坊三棵树](./threeTrees.md) | 王浩 |  |
| 2018/4/23 | [Memory 使用与新增Opcode](./evm存储.md#存储管理) | 袁佳豪 | :checkered_flag: |
| 2018/4/23 | [以太坊虚拟机释疑2](./evmInterpreter.md#%E9%87%8A%E7%96%91%EF%B8%8F)<br>[智能合约的调用方式](./evmInterpreter.md#%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6%E8%B0%83%E7%94%A8%E7%9A%84%E6%96%B9%E5%BC%8F)<br>[fallback函数](./fallbackFunction.md) | 刘文炎 | :checkered_flag: |
| 2018/4/28 | [区块验证细节](./verifyBlock.md) | 袁佳豪 |  |
| 2018/4/28 | [黄皮书拾遗](./yellowpaperOmissions.md) | 刘文炎 | :checkered_flag: |
| 2018/5/4 | [《以太坊技术详解与实战》数字资产发行和数据查询分析工具](./以太坊数字资产发行和数据查询分析工具.md) | 袁佳豪 |  |
| 2018/5/4 | [《以太坊技术详解与实战》拾遗](./yyanBookOmissions.md)<br>[《以太坊技术详解与实战》隐私保护与数据安全](./yyBookPrivacy.md) | 刘文炎 | :checkered_flag: |
| 2018/5/4 | [《以太坊技术详解与实战》框架、共识、雷电网络、分片](./ethArchitecture.md) | 王浩 |  |

## 参考文献
1. Wood, G., 2014. [Ethereum: A secure decentralised generalised transaction ledger](https://ethereum.github.io/yellowpaper/paper.pdf), Ethereum Project Yellow Paper.

