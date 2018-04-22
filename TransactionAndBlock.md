# Transaction
- 在提交交易后（submitTransaction），最终会调用tx_pool.go中的add函数，将交易加入到交易池中
- tx_pool中包含所有已知的未处理交易

流程如下：
![submitTransaction](img/20180422/submitTransaction.jpg)

1. 正常提交不会有问题,否则直接报错
2. 验证交易是否合法：
   - 大小不超过（32KB）
   - 金额不能小于0
   - gas量不能超过交易池限制
   - 验证签名
   - 验证非本地交易gas费用
   - 验证交易Nonce
   - 验证账户余额
   - 验证交易金额是不是小于当前账户的金额
3. 验证交易池是否超量
   - 总数量不超过4096+1024
   - 若超过，将会比较Gas大小，保留交易费用较大的一个交易
4. 检查交易Nonce值
   - 相同Nonceti替换掉pending队列中的交易
   - 否则，加入到queue队列等待执行
   - 仍然以gas费用决定是否抛弃交易

# Block
Block的打包在work.go中完成，由commitNewWork()完成:
- 创建Header对象，确定父区块哈希，区块号，时间，准备GasLimit等
- 根据header，创建新的work对象，更新work.current成员变量
- 打包新区块中的交易列表，来源是txpool中的pending中所有tx
- 打包叔区块信息
- 验证上一个区块是否已经进入主链
- 将work对象发送给agent（负责调用共识算法的流程）
以太坊对每条交易的大小、每个区块的交易条数作出了限制
第二个限制在gas的设置上，Txpool与block的打包都有相同的最大的gas设置
Gas limit的管理是一个复杂的问题（由矿工决定Limit, 市场供求决定最终的消耗）

