# 以太坊重要结构体



Ethereum结构体实现了一个全节点，它的不同属性负责实现节点的不同功能

```
// Ethereum implements the Ethereum full node service.
// 实现了以太坊全节点服务
type Ethereum struct {
	// 配置
	config      *Config
	chainConfig *params.ChainConfig
	
	// Channel for shutting down the service
	// 停止通道
	shutdownChan  chan bool    // Channel for shutting down the ethereum
	stopDbUpgrade func() error // stop chain db sequential key upgrade
	
	// Handlers
	txPool          *core.TxPool
	blockchain      *core.BlockChain
	protocolManager *ProtocolManager
	lesServer       LesServer
	
	// DB interfaces
	// 数据库接口
	chainDb ethdb.Database // Block chain database
	
	eventMux       *event.TypeMux
	engine         consensus.Engine
	accountManager *accounts.Manager
	
	bloomRequests chan chan *bloombits.Retrieval // Channel receiving bloom data retrieval requests
	bloomIndexer  *core.ChainIndexer             // Bloom indexer operating during block imports
	
	ApiBackend *EthApiBackend
	
	// 挖矿对象
	miner     *miner.Miner
	gasPrice  *big.Int
	etherbase common.Address
	
	// 网络id
	networkId     uint64
	netRPCService *ethapi.PublicNetAPI
	
	lock sync.RWMutex // Protects the variadic fields (e.g. gas price and etherbase)
}
```


blockchain代表了一个节点的基于某个创世区块的主链，进行区块加入、回退，重新组织链的管理。

```
type  BlockChain struct {
   chainConfig *params.ChainConfig // Chain & network configuration
   cacheConfig *CacheConfig        // Cache configuration for pruning

   // 底层的持久化数据库，存储最终结果
   db     ethdb.Database // Low level persistent database to store final content in
   triegc *prque.Prque   // Priority queue mapping block numbers to tries to gc
   // 累计主链的执行
   gcproc time.Duration  // Accumulates canonical block processing for trie dumping

   hc            *HeaderChain
   rmLogsFeed    event.Feed
   chainFeed     event.Feed
   chainSideFeed event.Feed
   chainHeadFeed event.Feed
   logsFeed      event.Feed
   scope         event.SubscriptionScope
   genesisBlock  *types.Block

   mu      sync.RWMutex // global mutex for locking chain operations
   chainmu sync.RWMutex // blockchain insertion lock
   procmu  sync.RWMutex // block processor lock

   checkpoint       int          // checkpoint counts towards the new checkpoint
   currentBlock     atomic.Value // Current head of the block chain
   currentFastBlock atomic.Value // Current head of the fast-sync chain (may be above the block chain!)

   stateCache   state.Database // State database to reuse between imports (contains state cache)
   bodyCache    *lru.Cache     // Cache for the most recent block bodies
   bodyRLPCache *lru.Cache     // Cache for the most recent block bodies in RLP encoded format
   blockCache   *lru.Cache     // Cache for the most recent entire blocks
   futureBlocks *lru.Cache     // future blocks are blocks added for later processing

   quit    chan struct{} // blockchain quit channel
   running int32         // running must be called atomically
   // procInterrupt must be atomically called
   procInterrupt int32          // interrupt signaler for block processing
   wg            sync.WaitGroup // chain processing wait group for shutting down

   engine    consensus.Engine
   // 区块执行接口
   processor Processor // block processor interface
   validator Validator // block and state validator interface
   vmConfig  vm.Config

   badBlocks *lru.Cache // Bad block cache
}
```

Block结构体代表了一个block的内容，主要包括header指针，叔链头部指针数组，交易数组
```
// Block represents an entire block in the Ethereum blockchain.
// block结构
type Block struct {
   // header  叔链 交易
   header       *Header
   uncles       []*Header
   transactions Transactions

   // caches
   // 缓存
   hash atomic.Value
   size atomic.Value

   // Td is used by package core to store the total difficulty
   // of the chain up to and including the block.
   // 包括本block的总难度
   td *big.Int

   // These fields are used by package eth to track
   // inter-peer block relay.
   // 接收到block的时间和邻居来源
   ReceivedAt   time.Time
   ReceivedFrom interface{}
}
```
block header 中包含一个block的部分摘要信息，拥有状态树、交易树、收据树根的hash值，以及父块和叔块的hash值。

```
// Header represents a block header in the Ethereum blockchain.
type Header struct {
   // 父块hash 叔块hash
   ParentHash  common.Hash    `json:"parentHash"       gencodec:"required"`
   UncleHash   common.Hash    `json:"sha3Uncles"       gencodec:"required"`
   // 挖出区块的节点的地址
   Coinbase    common.Address `json:"miner"            gencodec:"required"`
   // 状态的root
   Root        common.Hash    `json:"stateRoot"        gencodec:"required"`
   // 交易树的root
   TxHash      common.Hash    `json:"transactionsRoot" gencodec:"required"`
   // 收据树的root
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


StateDB
```
// 用来存储任何在默克尔树中的。关心caching和存储的嵌套状态，
type StateDB struct {
   db   Database // 就是bc.stateCache
   trie Trie

   // live 的对象，执行一个状态交易时会改变
   // This map holds 'live' objects, which will get modified while processing a state transition.
   stateObjects      map[common.Address]*stateObject
   stateObjectsDirty map[common.Address]struct{}

   // DB error.
   // State objects are used by the consensus core and VM which are
   // unable to deal with database-level errors. Any error that occurs
   // during a database read is memoized here and will eventually be returned
   // by StateDB.Commit.
   dbErr error

   // The refund counter, also used by state transitioning.
   refund uint64

   thash, bhash common.Hash
   txIndex      int
   logs         map[common.Hash][]*types.Log
   logSize      uint

   preimages map[common.Hash][]byte

   // Journal of state modifications. This is the backbone of
   // Snapshot and RevertToSnapshot.
   journal        journal
   validRevisions []revision
   nextRevisionId int

   lock sync.Mutex
}

```

stateObject一个正在改变的账户对象

```
type stateObject struct {
   address  common.Address
   addrHash common.Hash // hash of ethereum address of the account
   // 账户data
   data     Account
   db       *StateDB

   // DB error.
   // State objects are used by the consensus core and VM which are
   // unable to deal with database-level errors. Any error that occurs
   // during a database read is memoized here and will eventually be returned
   // by StateDB.Commit.
   dbErr error

   // Write caches.
   // 写缓存
   trie Trie // storage trie, which becomes non-nil on first access
   code Code // contract bytecode, which gets set when code is loaded

   cachedStorage Storage // Storage entry cache to avoid duplicate reads
   dirtyStorage  Storage // Storage entries that need to be flushed to disk

   // Cache flags.
   // When an object is marked suicided it will be delete from the trie
   // during the "update" phase of the state transition.
   dirtyCode bool // true if the code was updated
   suicided  bool
   touched   bool
   deleted   bool
   onDirty   func(addr common.Address) // Callback method to mark a state object newly dirty
}
```