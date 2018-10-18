# 以太坊源码学习

[Go Ethereum](https://github.com/ethereum/go-ethereum)的架构、数据存储、虚拟机和共识协议等关键技术的学习文档

## 目录

* [开始使用以太坊](./docs/firstTryEthBuildPrivateChain.md)
* 以太坊源码学习
  * [以太坊框架梳理](./docs/以太坊架构梳理.md)
  * 数据结构与存储
      * [区块与交易数据结构初探](./docs/blockTransactionDataStructure.md)
      * [数据存储与组织](./docs/dataStructureStorage.md)
          * [从世界状态获取账户余额](./docs/accountBalance.md)
  * 邻居区块插入
      - [新区块插入](./docs/newBlockInsert.md)
      - [fetcher与downloader](./docs/fetcherAndDownLoader.md)
      - [新区块写入区块链](./docs/insertChainAndWriteBlockWithState.md)
      - [以太坊中三棵树的存储](./docs/threeTrees.md)
  * 区块挖掘
      * [Miner包中的挖矿流程](./docs/minerPackage.md)
      * [本地区块的组装](./docs/commitNewWork.md)
      * [共识机制与Consensus](./docs/consensus.md)
  * 交易的执行
      * [Transaction与block的打包与提交](./docs/transactionAndBlock.md)
      * [Gas管理](./docs/GasAndGasLimit.md)
      * [Gas价目与虚拟机外的交易执行](./docs/applyTransaction.md)
      * [EVM解释器](./docs/evmInterpreter.md)
  * 共识算法
      * [Ethash](./docs/Ethash.md)
      * [难度计算](./docs/totalDifficult.md)
      * [区块验证细节](./docs/verifyBlock.md)
          * [Sender函数](./docs/sender.md)
          * [双花问题](./docs/doublePay.md)
  * 以太坊虚拟机
      * [EVM初探](./docs/evm.md) 
      * [EVM基本内容](./docs/evm学习.md)
      * [EVM解释器](./docs/evmInterpreter.md)
          * [fallback函数](./docs/fallbackFunction.md)
      * [EVM存储设计](./docs/evm存储.md)
      * [Memory使用与新增Opcode](./docs/evm存储.md#存储管理)
* 以太坊分析工具
  * [plyvel](./docs/plyvel.md)：数据库访问工具
  * [log4go-ethereum](./docs/log4go.md)：针对Go Ethereum的日志记录工具
* 《以太坊技术详解与实战》
  * [数字资产发行和数据查询分析工具](./docs/以太坊数字资产发行和数据查询分析工具.md)
  * [框架、共识、雷电网络、分片](./docs/ethArchitecture.md)
  * [隐私保护与数据安全](./docs/yyBookPrivacy.md)
  * [《以太坊技术详解与实战》拾遗](./docs/yyanBookOmissions.md)
* 以太坊黄皮书
   * [以太坊黄皮书拾遗](./docs/yellowpaperOmissions.md)  

<details>
    <summary>路线</summary>

| 日期 | 主题 | 作者 | 存档 |
| :-: | :-: | :-: | :-: |
| 2018/3/15 | [搭建以太坊私有链](./docs/firstTryEthBuildPrivateChain.md) | 王浩 | :checkered_flag: |
| 2018/3/20 | [以太坊架构梳理](./docs/以太坊架构梳理.md) | 袁佳豪 | :checkered_flag:  |
| 2018/3/20 | [区块与交易的数据结构初探](./docs/blockTransactionDataStructure.md) | 刘文炎 | :checkered_flag: |
| 2018/3/20 | [共识机制与Consensus包](./docs/consensus.md) | 王浩 |  |
| 2018/3/24 | [Transaction与block的打包与提交](./docs/transactionAndBlock.md) | 袁佳豪 | :checkered_flag: |
| 2018/3/24 | [以太坊物理存储分析](./docs/dataStructureStorage.md#数据的物理存储)<br>[数据库访问工具](./docs/plyvel.md) | 刘文炎 | :checkered_flag: |
| 2018/3/24 | [Miner包中的挖矿流程](./docs/minerPackage.md) | 王浩 |:checkered_flag:  |
| 2018/3/27 | Gas管理 DoS攻击 交易打包与执行 交易堵塞 | 袁佳豪 |  |
| 2018/3/27 | [从世界状态获取账户余额](./docs/accountBalance.md) | 刘文炎 | :checkered_flag: |
| 2018/3/27 | worker中新区块组装与交易提交| 王浩 ||
| 2018/3/30 | [Gas价目与虚拟机外的交易执行](./docs/applyTransaction.md) | 袁佳豪 | :checkered_flag: |
| 2018/3/30 | [改进的默克尔·帕特里夏树](./docs/dataStructureStorage.md#数据的组织形式) | 刘文炎 | :checkered_flag: |
| 2018/3/30 | [新区块的插入](./docs/newBlockInsert.md)<br>[fetcher与downloader](./docs/fetcherAndDownLoader.md) | 王浩 |  |
| 2018/4/4 | [EVM初探](./docs/evm.md) | 袁佳豪 | :checkered_flag: |
| 2018/4/4 | [日志记录工具](./docs/log4go.md)<br>[Sender函数](./docs/sender.md) | 刘文炎 | :checkered_flag: |
| 2018/4/12 | [虚拟机基本内容](./docs/evm学习.md) | 袁佳豪 | :checkered_flag: |
| 2018/4/12 | [以太坊虚拟机解释器](./docs/evmInterpreter.md) | 刘文炎 | :checkered_flag: |
| 2018/4/12 | [新区块写入WriteBlockWithState方法](./docs/insertChainAndWriteBlockWithState.md) | 王浩 |  :checkered_flag: |
| 2018/4/17 | [EVM存储设计](./docs/evm存储.md)| 袁佳豪 |  |
| 2018/4/17 | [以太坊虚拟机释疑](./docs/evmInterpreter.md#%E9%87%8A%E7%96%91%EF%B8%8F) | 刘文炎 | :checkered_flag: |
| 2018/4/17 | [以太坊三棵树](./docs/threeTrees.md) | 王浩 |  |
| 2018/4/23 | [Memory 使用与新增Opcode](./docs/evm存储.md#存储管理) | 袁佳豪 | :checkered_flag: |
| 2018/4/23 | [以太坊虚拟机释疑2](./docs/evmInterpreter.md#%E9%87%8A%E7%96%91%EF%B8%8F)<br>[智能合约的调用方式](./docs/evmInterpreter.md#%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6%E8%B0%83%E7%94%A8%E7%9A%84%E6%96%B9%E5%BC%8F)<br>[fallback函数](./docs/fallbackFunction.md) | 刘文炎 | :checkered_flag: |
| 2018/4/28 | [区块验证细节](./docs/verifyBlock.md) | 袁佳豪 |  |
| 2018/4/28 | [黄皮书拾遗](./docs/yellowpaperOmissions.md) | 刘文炎 | :checkered_flag: |
| 2018/5/4 | [《以太坊技术详解与实战》数字资产发行和数据查询分析工具](./docs/以太坊数字资产发行和数据查询分析工具.md) | 袁佳豪 |  |
| 2018/5/4 | [《以太坊技术详解与实战》拾遗](./docs/yyanBookOmissions.md)<br>[《以太坊技术详解与实战》隐私保护与数据安全](./docs/yyBookPrivacy.md) | 刘文炎 | :checkered_flag: |
| 2018/5/4 | [《以太坊技术详解与实战》框架、共识、雷电网络、分片](./docs/ethArchitecture.md) | 王浩 |  |
| 2018/5/6 | [总结汇报](./docs/总结汇报.md) | 袁佳豪<br>王浩<br>刘文炎 | :checkered_flag: |
| 2018/5/18 | [Gas机制总结](./docs/GasAndGasLimit.md) | 袁佳豪 |  |
| 2018/5/18 | [查询任务实战](./docs/dataStructureStorage.md#查询任务实战) | 刘文炎 | :checkered_flag: |

</details>

## 参考文献
1. Wood, G., 2014. [Ethereum: A secure decentralised generalised transaction ledger](https://ethereum.github.io/yellowpaper/paper.pdf), Ethereum Project Yellow Paper.


