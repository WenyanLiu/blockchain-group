# WriteBlockWithState

此方法用来将block存入本地blockchain中，也就是存入数据库中，新的block会存入blockchain的db，执行交易产生的state会存入statedb。



首先回顾block的数据结构

```
// Block represents an entire block in the Ethereum blockchain.
// 以太坊区块链中一个完整的block
type Block struct {
   header       *Header // 
   uncles       []*Header // 叔链的header
   transactions Transactions // Transaction结构体的数组，存放交易的txdata,hash,size,from

   // caches
   hash atomic.Value
   size atomic.Value

   // Td is used by package core to store the total difficulty
   // of the chain up to and including the block.
   // 当前block的总难度，td大的block将被认可为主链
   td *big.Int

   // These fields are used by package eth to track
   // inter-peer block relay.
   // 接收到block的时间和block来源（peer）
   ReceivedAt   time.Time
   ReceivedFrom interface{}
}
```
header的数据结构
```
// Header represents a block header in the Ethereum blockchain.
type Header struct {
   ParentHash  common.Hash    `json:"parentHash"       gencodec:"required"`
   UncleHash   common.Hash    `json:"sha3Uncles"       gencodec:"required"`
   // 挖出区块的节点的地址
   Coinbase    common.Address `json:"miner"            gencodec:"required"`
   // 状态的root
   Root        common.Hash    `json:"stateRoot"        gencodec:"required"`
   // 交易树的root
   TxHash      common.Hash    `json:"transactionsRoot" gencodec:"required"`
   // 收据树 的root
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