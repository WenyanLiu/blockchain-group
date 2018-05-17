# Gas 机制
在以太坊的设计中，为了避免对于区块链网络资源的滥用，每一笔交易的执行都需要支付一定的费用，用于补偿矿工执行交易所需要的计算
开销，促使Dapp开发者合理使用区块链网络资源。

每一条交易的Gas费用由两部分组成：固定费用和动态费用。
- 固定费用：就是执行转账、合约创建等基本功能需要扣除的基本费用，即使是转账金额为0的交易，这部分费用都是必不可少的。
- 动态费用：根据交易的复杂程度计算的Gas费用。对于以太坊中的智能合约，每一个字节码的运行都是明码标价，有着明确的Gas值
具体的内容参见以太坊黃皮书內容[]()。

## 交易Gas
以下是Go语言中的代码片段，对于每一条Transaction，都有着明确的GasPrice和GasLimit：
```
type txdata struct {
	AccountNonce uint64          `json:"nonce"    gencodec:"required"`
	Price        *big.Int        `json:"gasPrice" gencodec:"required"`
	GasLimit     uint64          `json:"gas"      gencodec:"required"`
	Recipient    *common.Address `json:"to"       rlp:"nil"` // nil means contract creation
	Amount       *big.Int        `json:"value"    gencodec:"required"`
	Payload      []byte          `json:"input"    gencodec:"required"`

	// Signature values
	V *big.Int `json:"v" gencodec:"required"`
	R *big.Int `json:"r" gencodec:"required"`
	S *big.Int `json:"s" gencodec:"required"`

	// This is only used when marshaling to JSON.
	Hash *common.Hash `json:"hash" rlp:"-"`
}
```
在交易的执行设计中，矿工会根据交易的支付费用有选择的执行，对于一条Transaction而言：
- 最大愿意支付的费用 = Price * GasLimit
- 实际支付的费用 = Price * GasUsed

从代码中，我们可以知道，Price和GasLimit是用户可以自行设置的内容，而GasUsed会根据实际的交易和合約內容來決定，Price將會決
定矿工的打包的优先级。在实际的交易执行中，会首先冻结最大的愿意支付的费用，最后会根据实际支付的费用进行回退。因此，为了确
保交易的执行，往往会适当更多的设置交易的Gaslimit，因为如果由于GasLimit设置不足而导致的执行失败，扣除的固定费用将不会返还。

## 区块中的Gas
如下的代码片段所示，在以太坊中的区块头中，主要有两个和Gas相关的变量：
```
type Header struct {
	ParentHash  common.Hash    `json:"parentHash"       gencodec:"required"`
	UncleHash   common.Hash    `json:"sha3Uncles"       gencodec:"required"`
	Coinbase    common.Address `json:"miner"            gencodec:"required"`
	Root        common.Hash    `json:"stateRoot"        gencodec:"required"`
	TxHash      common.Hash    `json:"transactionsRoot" gencodec:"required"`
	ReceiptHash common.Hash    `json:"receiptsRoot"     gencodec:"required"`
	Bloom       Bloom          `json:"logsBloom"        gencodec:"required"`
	Difficulty  *big.Int       `json:"difficulty"       gencodec:"required"`
	Number      *big.Int       `json:"number"           gencodec:"required"`
	GasLimit    uint64         `json:"gasLimit"         gencodec:"required"`
	GasUsed     uint64         `json:"gasUsed"          gencodec:"required"`
	Time        *big.Int       `json:"timestamp"        gencodec:"required"`
	Extra       []byte         `json:"extraData"        gencodec:"required"`
	MixDigest   common.Hash    `json:"mixHash"          gencodec:"required"`
	Nonce       BlockNonce     `json:"nonce"            gencodec:"required"`
}
```
- GasLimit: 用于规定这个区块中所允许包含的最大Gas总量。
- GasUsed：该区块的实际Gas需求量
GasUsed是当前区块中，所有交易所需要的实际Gas值，而Gaslimit相对而言较为复杂，特殊的情况下矿工可以通过投票的方式进行更改。
由于在当前以太坊的运行机制中，GasLimit的设计在实际上决定了区块的容量大小，随后的内容我们将重点介绍区块中的GasLimit

## 区块中的GasLimit
在以太坊的设计中，通过创世区块将GasLimit默认设计为4712388,并且会有一个自适应的机制，使得当前区块的GasLimit只能基于上一个
区块的GasLimit 上下波动1/1024

在矿工的执行函数commitNewWork中，我们可以发现当前区块的GasLimit由CalcGasLimit来进行计算，它的输入参数是父区块，我们看看代码中的具体逻辑：

```
// CalcGasLimit computes the gas limit of the next block after parent.
// This is miner strategy, not consensus protocol.
func CalcGasLimit(parent *types.Block) uint64 {
	// contrib = (parentGasUsed * 3 / 2) / 1024
	contrib := (parent.GasUsed() + parent.GasUsed()/2) / params.GasLimitBoundDivisor

	// decay = parentGasLimit / 1024 -1 
	decay := parent.GasLimit()/params.GasLimitBoundDivisor - 1

	/*
		strategy: gasLimit of block-to-mine is set based on parent's
		gasUsed value.  if parentGasUsed > parentGasLimit * (2/3) then we
		increase it, otherwise lower it (or leave it unchanged if it's right
		at that usage) the amount increased/decreased depends on how far away
		from parentGasLimit * (2/3) parentGasUsed is.
	*/
	limit := parent.GasLimit() - decay + contrib
	if limit < params.MinGasLimit {
		limit = params.MinGasLimit
	}
	// however, if we're now below the target (TargetGasLimit) we increase the
	// limit as much as we can (parentGasLimit / 1024 -1)
	if limit < params.TargetGasLimit {
		limit = parent.GasLimit() + decay
		if limit > params.TargetGasLimit {
			limit = params.TargetGasLimit
		}
	}
	return limit
}
```
总结而言，GasLimit的计算逻辑为：GasLimit = parent.GasLimit - (parent.GasLimit - 1.5*parent.GasUsed)/1024，因此：
1. 当父区块的GasUsed 大于 GasLimit的三分之二时，会适当增加GasLimit
2. 当父区块的GasUsed 小于 GasLimit的三分之二时，会适当减少GasLimit
3. 增加和减少的幅度，取决于大于或者小于GasLimit三分之二的数量，但是最多不超过1/1024（GasUsed为0时）

同时，还有一个限制是TargetGasLimit，创世区块中将这个值设置为默认值，而这个值可以由矿工通过投票来进行修改。当前区块的GasLimit
小于这个值时，会将其设置为父区块增加1/1024

但是，这个和实际的情况有一些出入：
1. 从去年以来，由于ICO的火热进行，看绝大多数的区块GasUsed都接近GasLimit，我们看到其并不是一个平滑的上升趋势，而是始终
存在一个突变后稳定的维持在一定范围内。因此，可以猜测 在达到targetGasLimit后，无法再随意上升

2. 网上的资料中，比较频繁的谈到矿工具有“投票”改变GasLimit的权力，这一点尚未在代码中得到证实。



