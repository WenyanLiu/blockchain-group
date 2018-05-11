# 数据结构与存储

* [数据的组织形式](#数据的组织形式)
   * [预备知识](#预备知识)
       * [前缀树](#前缀树)
       * [压缩前缀树](#压缩前缀树)
       * [哈希树](#哈希树)
   * [理论基础](#理论基础)
       * [Modified Merkle Patricia tree](#modified-merkle-patricia-tree)
       * [存疑](#存疑)
       * [释疑️](#释疑️)
  * [源码解读](#源码解读)
       * [trie/database.go](#triedatabasego)
       * [trie/encoding.go](#trieencodinggo)
       * [trie/errors.go](#trieerrorsgo)
       * [trie/hasher.go](#triehashergo)
       * [trie/iterator.go](#trieiteratorgo)
       * [trie/node.go](#trienodego)
       * [trie/proof.go](#trieproofgo)
       * [trie/secure_trie.go](#triesecure_triego)
       * [trie/sync.go](#triesyncgo)
       * [trie/trie.go](#trietriego)
* [数据的物理存储](#数据的物理存储)
    * [Block Header的存储](#block-header的存储)
    * [Block Body的存储](#block-body的存储)
    * [其他存储](#其他存储)
    * [附录](#附录)
* [查询任务实战](#查询任务实战)
    * [第一种类型的查询](#第一种类型的查询)
    * [第二种类型的查询](#第二种类型的查询)
    * [第三种类型的查询](#第三种类型的查询)
    * [第四种类型的查询](#第四种类型的查询)
    * [第五种类型的查询](#第五种类型的查询)

## 数据的组织形式

trie包实现了改进的默克尔・帕特里夏树（Modified Merkle Patricia Tree, MPT），提供了任意长度的二进制数据（字节数组）之间的映射的持久化存储。

以太坊的区块头部中的`stateRoot`、`transacionsRoot`、`receiptsRoot`使用MPT存储。

### 预备知识

MPT是一种前缀树的变种，基本思想来自[前缀树](#前缀树)、[压缩前缀树](#压缩前缀树)和[哈希树](#哈希树)。

#### 前缀树

前缀树（trie）是一种用于快速搜索的数据结构。如图所示，该前缀树存储了key为"A"、"to"、"tea"、"ted"、"ten"、"in"和"inn"，每个完整的英语单词都有与其对应的整数值。

![trie](./img/trie_trie.png)

前缀树的性质归纳为：
1. 根结点不包含字符，其他节点包含一个字符；
1. 从根节点到某个节点遍历时经过的字符连接起来，构成该节点的字符串；
1. 每个节点的子节点包含的字符串均不相同。

#### 压缩前缀树

压缩前缀树（Patricia Trie）是一种用于快速搜索，并对空间进行优化的数据结构。如图展示了一棵存储了7个key的压缩前缀树。

由于前缀树为每个节点分配一个字符，在字符串没有公共前缀的情况下，前缀树退化成一条链。与前缀树不同，压缩前缀树将唯一的子节点与父节点合并，允许使用多个元素标记边。

![patriciaTrie](./img/trie_patriciaTrie.png)

#### 哈希树

哈希树（Merkle Tree）是一种用于高效验证数据内容的数据结构。如图所示，哈希树的叶子节点是数据块的哈希值，非叶子节点是其子节点串联字符串的哈希值。

![merkleTree](./img/trie_merkleTree.png)

### 理论基础

详见[以太坊黄皮书](https://ethereum.github.io/yellowpaper/paper.pdf)Appendix D、[以太坊Wiki](https://github.com/ethereum/wiki/wiki/Patricia-Tree)和[StackExchange](https://ethereum.stackexchange.com/questions/268/ethereum-block-architecture)。

#### Modified Merkle Patricia tree

假设有输入值`J`，包含字节数组对的集合：
![187](./img/trie_187.png)

处理该集合时，使用数值下表标识元组的key或者value，有：
![188](./img/trie_188.png)

给定字节存储顺序，任意字节可以被表示为半字节（nibbles）。假设使用高位优先的存储顺序，有：
![189](./img/trie_189.png)
![190](./img/trie_190.png)

定义表示集合的trie的根节点`TRIE`函数：
![191](./img/trie_191.png)

当节点的RLP少于32字节时，直接存储其RLP编码；否则，存储字节数组的Keccak哈希值。定义节点的容量`n`函数：
![192](./img/trie_192.png)

在MPT中有三种类型的节点：

* **叶子节点（Leaf）：**`RLP(HP(尚未串联的key的半字节编码, true), value)`
* **扩展节点（Extension）：**`RLP(HP(至少2个尚未累加的key公共半子节编码, false), 剩余部分)`
* **分支节点（Branch）：**`{0, 1, ..., f, $}`

定义节点的结构组成函数`c`：
![193](./img/trie_193.png)

如图展示了一颗MPT树。假设MPT包含四个key-value对`('a711355', '45.0 ETH')`、`('a77d337', '1.00 WEI')`、`('a7f9365', '1.1 ETH')`和`('a77d397', '0.12 ETH')`。

![MPT](./img/trie_MPT.png)

首次，将key和value转换为`bytes`。*注：为了方便理解，key使用`<>`、value使用`''`标记，在实现中他们都是字节数组。*

然后，构建如下MPT：

```
root:   [<a7>, HASH(A)]
HASH(A):[<>, HASH(B), <>, <>, <>, <>, <>, HASH(C), <>, <>, <>, <>, <>, <>, <>, HAHS(D), <>]
HASH(B):[<1355>, '45.0 ETH']
HAHS(C):[<dc>, HASH(E)]
HASH(D):[<9365>, '1.1 ETH']
HASH(E):[<>, <>, <>, HASH(F), <>, <>, <>, <>, <>, HASH(G), <>, <>, <>, <>, <>, <>, <>]
HASH(F):[<7>, '1.00 WEI']
HASH(G):[<7>, '0.12 ETH']
```

#### 存疑❓

```diff
+ 类似前缀树，从根节点到叶子节点遍历MPT树，构建唯一的key-value对。key通过遍历从每个分支节点获取单个半字节。
+ 与前缀树不同，当多个key共享相同的前缀或者单个key拥有唯一的后缀时，使用优化节点，在遍历时从扩展节点和叶子节点获取多个半字节。
- MPT如何体现哈希树的性质？
```

#### 释疑❗️

> ...; we simply define the identity function mapping the key-value set `J` to a 32-byte hash and assert that only a single such hash exists for any `J`, which though not strictly true is accurate within acceptable precision given the Keccak hash's collision resistance. In reality, a sensible implementation will not fully recompute the trie root hash for each set.

[trie包的hasher go文件的`hashChildren`函数](#triehashergo)

### 源码解读

trie包的内容如下：

```
trie
  ├─database.go
  ├─encoding.go
  ├─errors.go
  ├─hasher.go
  ├─iterator.go
  ├─node.go
  ├─proof.go
  ├─secure_trie.go
  ├─sync.go
  └─trie.go
```  

#### trie/database.go

`Database`是在MPT数据结构和磁盘数据库间的中间件。

#### trie/encoding.go

`encoding`主要处理编码格式的相互转换。这三种编码分别是：

* **KEYBYTES编码：**包含原生key的字节数组，多用于API的输入；
* **HEX编码：**包含key的半字节数组和可选尾终止符，多用于节点在内存中的表示；
* **COMPACT编码：**即Hex-prefix编码，包含key的半字节数组和flag，用于磁盘存储。

#### trie/errors.go

在`errors`中，如果MPT节点不存在，`TryGet`、`TryUpdate`和`TryDelete`函数返回`MissingNodeError`。

#### trie/hasher.go

`hash`函数主要:
1. 计算原有树形结构的哈希值，**体现哈希树的性质**；
2. 保留原有树形结构，**体现压缩前缀树的性质**。

计算原有树形结构的哈希值调用`hashChildren`计算所有子节点的哈希值，将原有的子节点替换成子节点的哈希值。

```go
func (h *hasher) hash(n node, db *Database, force bool) (node, node, error) {
	if hash, dirty := n.cache(); hash != nil {
		if db == nil {
			return hash, n, nil
		}
		if n.canUnload(h.cachegen, h.cachelimit) {
			cacheUnloadCounter.Inc(1)
			return hash, hash, nil
		}
		if !dirty {
			return hash, n, nil
		}
	}
	collapsed, cached, err := h.hashChildren(n, db)
	if err != nil {
		return hashNode{}, n, err
	}
	hashed, err := h.store(collapsed, db, force)
	if err != nil {
		return hashNode{}, n, err
	}
	cachedHash, _ := hashed.(hashNode)
	switch cn := cached.(type) {
	case *shortNode:
		cn.flags.hash = cachedHash
		if db != nil {
			cn.flags.dirty = false
		}
	case *fullNode:
		cn.flags.hash = cachedHash
		if db != nil {
			cn.flags.dirty = false
		}
	}
	return hashed, cached, nil
}
```

`hashChildren`函数递归地从叶子节点向上计算到根节点。

```go
func (h *hasher) hashChildren(original node, db *Database) (node, node, error) {
	var err error

	switch n := original.(type) {
	case *shortNode:
		collapsed, cached := n.copy(), n.copy()
		collapsed.Key = hexToCompact(n.Key)
		cached.Key = common.CopyBytes(n.Key)

		if _, ok := n.Val.(valueNode); !ok {
			collapsed.Val, cached.Val, err = h.hash(n.Val, db, false)
			if err != nil {
				return original, original, err
			}
		}
		if collapsed.Val == nil {
			collapsed.Val = valueNode(nil)
        }
		return collapsed, cached, nil

	case *fullNode:
		collapsed, cached := n.copy(), n.copy()

		for i := 0; i < 16; i++ {
			if n.Children[i] != nil {
				collapsed.Children[i], cached.Children[i], err = h.hash(n.Children[i], db, false)
				if err != nil {
					return original, original, err
				}
			} else {
				collapsed.Children[i] = valueNode(nil)
			}
		}
		cached.Children[16] = n.Children[16]
		if collapsed.Children[16] == nil {
			collapsed.Children[16] = valueNode(nil)
		}
		return collapsed, cached, nil

	default:
		return n, original, nil
	}
}
```

#### trie/iterator.go

`Iterator`是遍历MPT的key-value迭代器。

#### trie/node.go

在`node`中，`fullNode`是分支节点，`shortNode`根据`Val`分为叶子节点和扩展节点。

```go
type (
    fullNode struct {
        Children [17]node
        flags    nodeFlag
    }
    shortNode struct {
        Key   []byte
        Val   node
        flags nodeFlag
   }
    hashNode  []byte
    valueNode []byte
)
```

#### trie/proof.go

`Prove`为指定的key构建默克尔证明，包含遍历路径上所有节点的哈希值。

`VerifyProof`验证默克尔证明。

#### trie/secure_trie.go

`secure_trie`中所有的访问操作的key使用Keccak256的哈希值，避免增加访问时间的节点的长链。

#### trie/sync.go

`TrieSync`是MPT同步调度器。

#### trie/trie.go

在`trie`中，`db`指向数据库存储，`root`包含了当前的根节点，`originalRoot`记录了启动加载时的哈希值。

```go
type Trie struct {
    db           *Database
    root         node
    originalRoot common.Hash
    
    cachegen, cachelimit uint16
}
```

`New`函数构造MPT。如果`root`不为空，就从数据库中加载MPT；否则，新建MPT。

```go
func New(root common.Hash, db *Database) (*Trie, error) {
    if db == nil {
       panic("trie.New called without a database")
    }
    trie := &Trie{
        db:           db,
        originalRoot: root,
    }
    if (root != common.Hash{}) && root != emptyRoot {
        rootnode, err := trie.resolveHash(root[:], nil)
        if err != nil {
            return nil, err
        }
        trie.root = rootnode
    }
    return trie, nil
}
```

`insert`函数向MPT递归插入节点。

**输入：**`n`是待插入的节点，`prefix`是已处理的前缀，`key`是剩余未处理的前缀，`value`是待插入的节点的值。

**输出：**`bool`标识着操作是否改变了MPT，`node`是插入完成后的子树的根节点，`error`是错误信息。

* 如果`n`是`shortNode`，
    * 计算公共前缀，
    * 如果全key匹配
        * 如果value相同，即`dirty == false`
            * 返回错误
        * 更新`shortNode`的value
    * 公共前缀不完全匹配
        * 插入扩展节点和分支节点
        * 递归调用`insert`
* 如果`n`是`fullNode`
    * 递归调用`insert`
* 如果`n`是`nil`，即一棵新的MPT
    * 返回`shortNode`
* 如果`n`是`hashNode`，即节点尚未载入内存
    * 调用`resolveHash`将节点加载到内存
    * 递归调用`insert`

```go
func (t *Trie) insert(n node, prefix, key []byte, value node) (bool, node, error) {
    if len(key) == 0 {
        if v, ok := n.(valueNode); ok {
            return !bytes.Equal(v, value.(valueNode)), value, nil
        }
        return true, value, nil
    }
    switch n := n.(type) {
    case *shortNode:
        matchlen := prefixLen(key, n.Key)
        if matchlen == len(n.Key) {
            dirty, nn, err := t.insert(n.Val, append(prefix, key[:matchlen]...), key[matchlen:], value)
            if !dirty || err != nil {
                return false, n, err
            }
            return true, &shortNode{n.Key, nn, t.newFlag()}, nil
        }
        branch := &fullNode{flags: t.newFlag()}
        var err error
        _, branch.Children[n.Key[matchlen]], err = t.insert(nil, append(prefix, n.Key[:matchlen+1]...), n.Key[matchlen+1:], n.Val)
        if err != nil {
            return false, nil, err
        }
        _, branch.Children[key[matchlen]], err = t.insert(nil, append(prefix, key[:matchlen+1]...), key[matchlen+1:], value)
        if err != nil {
            return false, nil, err
        }
        if matchlen == 0 {
            return true, branch, nil
        }
        return true, &shortNode{key[:matchlen], branch, t.newFlag()}, nil

    case *fullNode:
        dirty, nn, err := t.insert(n.Children[key[0]], append(prefix, key[0]), key[1:], value)
        if !dirty || err != nil {
            return false, n, err
        }
        n = n.copy()
        n.flags = t.newFlag()
        n.Children[key[0]] = nn
        return true, n, nil

    case nil:
        return true, &shortNode{key, value, t.newFlag()}, nil

    case hashNode:
        rn, err := t.resolveHash(n, prefix)
        if err != nil {
            return false, nil, err
        }
        dirty, nn, err := t.insert(rn, prefix, key, value)
        if !dirty || err != nil {
            return false, rn, err
        }
        return true, nn, nil

    default:
        panic(fmt.Sprintf("%T: invalid node: %v", n, n))
    }
}
```

`TryGet`函数根据MPT的key获取value。

`TryUpdate`函数更新或者删除MPT的节点。

`Delete`函数删除MPT的节点。

## 数据的物理存储

![Physical View](img/physicalView_overview.png)

### Block Header的存储

|  | hash to number mapping |
| :-: | :-- |
| Key | "H" + hash(header)  |
| Value | Block number |

![hash to number mapping](img/physicalView_hash2num.png)

![hash to number mapping](img/physicalView_hash2num_rlp.png)

|  | header |
| :-: | :-- |
| Key | "h" + number + hash(header) |
| Value | header |

```go
# core/types/block.go

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
	Nonce       BlockNonce     `json:"nonce"            gencodec:"required"` }
```

![header](img/physicalView_header.png)

![header](img/physicalView_header_rlp.png)

### Block Body的存储

|  | body |
| :-: | :-- |
| Key | "b" + number + hash(header) |
| Value | body |

```go
# core/types/block.go

type Body struct {
    Transactions []*Transaction
    Uncles       []*Header
}
```

```go
# core/types/transaction.go

type Transaction struct {
    data txdata
    
    hash atomic.Value
    size atomic.Value
    from atomic.Value
}

type txdata struct {
    AccountNonce uint64          `json:"nonce"    gencodec:"required"`
    Price        *big.Int        `json:"gasPrice" gencodec:"required"`
    GasLimit     uint64          `json:"gas"      gencodec:"required"`
    Recipient    *common.Address `json:"to"       rlp:"nil"`
    Amount       *big.Int        `json:"value"    gencodec:"required"`
    Payload      []byte          `json:"input"    gencodec:"required"`
    
    V *big.Int `json:"v" gencodec:"required"`
    R *big.Int `json:"r" gencodec:"required"`
    S *big.Int `json:"s" gencodec:"required"`
    
    Hash *common.Hash `json:"hash" rlp:"-"`
}
```

|  | transaction Lookup Entries |
| :-: | :-- |
| Key | "l" + hash(tx) |
| Value | Entry |
|  |  |
| Key | 6c91cae53fa2176b5a855a1c654d5f9668a34dfece29a55aecb66e6ec4512fb053 |
|  | "l" + 交易的哈希值 |
| Value | e3a0422ae9ec19018be6dc72a696074e79dcf7594cf140abf729098092ddd3f106ac0a80 |
|  | [<br>	422ae9ec19018be6dc72a696074e79dcf7594cf140abf729098092ddd3f106ac, (hash(header))<br>	0a, (block number)<br>	"", (transaction index)<br>	] |

![body](img/physicalView_body.png)

### 其他存储

|  | number to hash mapping |
| :-: | :-- |
| Key | "h" + number + "n" |
| Value | hash(header) |
|  |  |
| Key | 68000000000000000a6e |
|  | "h" + 第10个区块 + "n" |
| Value | 422ae9ec19018be6dc72a696074e79dcf7594cf140abf729098092ddd3f106ac |
|  | 第10个区块的哈希值 |

|  | block total difficulty |
| :-: | :-- |
| Key | "h" + number + hash(header) + "t" |
| Value | td |
|  |  |
| Key | 68000000000000000a422ae9ec19018be6dc72a696074e79dcf7594cf140abf729098092ddd3f106ac74 |
|  | "h" + 第10个区块 + 第10个区块的哈希值 + "t" |
| Value | 83150900 |
|  | RLP(总难度) |

|  | block receipts |
| :-: | :-- |
| Key | "r" + number + hash(header) |
| Value | Receipt |

|  | Last header's hash |
| :-: | :-- |
| Key | "LastHeader" |
| Value | hash(header) |

|  | ... |
| :-: | :-- |

### 附录

```go
# core/database_util.go

var (
	headHeaderKey = []byte("LastHeader")
	headBlockKey  = []byte("LastBlock")
	headFastKey   = []byte("LastFast")
	trieSyncKey   = []byte("TrieSync")

	headerPrefix        = []byte("h")
	tdSuffix            = []byte("t")
	numSuffix           = []byte("n")
	blockHashPrefix     = []byte("H")
	bodyPrefix          = []byte("b")
	blockReceiptsPrefix = []byte("r")
	lookupPrefix        = []byte("l")
	bloomBitsPrefix     = []byte("B")

	preimagePrefix = "secure-key-"
	configPrefix   = []byte("ethereum-config-")

	BloomBitsIndexPrefix = []byte("iB")

	oldReceiptsPrefix = []byte("receipts-")
	oldTxMetaSuffix   = []byte{0x01}

	ErrChainConfigNotFound = errors.New("ChainConfig not found")

	preimageCounter    = metrics.NewRegisteredCounter("db/preimage/total", nil)
	preimageHitCounter = metrics.NewRegisteredCounter("db/preimage/hits", nil)
)
```

## 查询任务实战

《以太坊技术详解与实战》第2章 以太坊架构和组成 2.4 数据结构与存储 2.4.1 数据组织形式 3. Merkle Patricia树
[Page 27-28]
利用存储的三颗MPT树，客户端可以轻松地查询以下内容。

##### 第一种类型的查询

* 某笔交易是否被包含在特定的区块中。

第一种类型使用交易树处理。

**Input:** TxHash

**Output:** Number, BlockHash

![](img/dataStructureStorage_1.png)

```python
# Transaction Lookup Entries
# DB.get(l,49f65849655317172a3f3f499303dc74ad30e7859b9b2ac7e9b04e02e30c7b49)
>>> DB.get('6c49f65849655317172a3f3f499303dc74ad30e7859b9b2ac7e9b04e02e30c7b49')
'e3a05e33fee0dd7a6a2d6c99a1879f9b3819572c9e18019a4de785a06a3c039d1fca1880'
[
  5e33fee0dd7a6a2d6c99a1879f9b3819572c9e18019a4de785a06a3c039d1fca,
  18,
  "",
]
```

##### 第二种类型的查询

* 查询某个地址在过去的30天中发出某种类型事件的所有实例（例如，一个众筹合约完成了它的目标）。

第二种类型使用收据树。

> Docs » web3.eth » getPastLogs

**getPastLogs**

```
    web3.eth.getPastLogs(options [, callback])
```

Gets past logs, matching the given options.

**Parameters**

1. ``Object`` - The filter options as follows:
  - ``fromBlock`` - ``Number|String``: The number of the earliest block (``"latest"`` may be given to mean the most recent and ``"pending"`` currently mining, block). By default ``"latest"``.
  - ``toBlock`` -  ``Number|String``: The number of the latest block (``"latest"`` may be given to mean the most recent and ``"pending"`` currently mining, block). By default ``"latest"``.
  - ``address`` -  ``String|Array``: An address or a list of addresses to only get logs from particular account(s).
  - ``topics`` - ``Array``: An array of values which must each appear in the log entries. The order is important, if you want to leave topics out use ``null``, e.g. ``[null, '0x12...']``. You can also pass an array for each topic with options for that topic e.g. ``[null, ['option1', 'option2']]``


.. _eth-getpastlogs-return:

**Returns**

``Promise`` returns ``Array`` - Array of log objects.

The structure of the returned event ``Object`` in the ``Array`` looks as follows:

- ``address`` - ``String``: From which this event originated from.
- ``data`` - ``String``: The data containing non-indexed log parameter.
- ``topics`` - ``Array``: An array with max 4 32 Byte topics, topic 1-3 contains indexed parameters of the log.
- ``logIndex`` - ``Number``: Integer of the event index position in the block.
- ``transactionIndex`` - ``Number``: Integer of the transaction's index position, the event was created in.
- ``transactionHash`` 32 Bytes - ``String``: Hash of the transaction this event was created in.
- ``blockHash`` 32 Bytes - ``String``: Hash of the block where this event was created in. ``null`` when its still pending.
- ``blockNumber`` - ``Number``: The block number where this log was created in. ``null`` when still pending.

**Example**

```
    web3.eth.getPastLogs({
        address: "0x11f4d0A3c12e86B4b5F39B213F7E19D048276DAe",
        topics: ["0x033456732123ffff2342342dd12342434324234234fd234fd23fd4f23d4234"]
    })
    .then(console.log);

    > [{
        data: '0x7f9fade1c0d57a7af66ab4ead79fade1c0d57a7af66ab4ead7c2c2eb7b11a91385',
        topics: ['0xfd43ade1c09fade1c0d57a7af66ab4ead7c2c2eb7b11a91ffdd57a7af66ab4ead7', '0x7f9fade1c0d57a7af66ab4ead79fade1c0d57a7af66ab4ead7c2c2eb7b11a91385']
        logIndex: 0,
        transactionIndex: 0,
        transactionHash: '0x7f9fade1c0d57a7af66ab4ead79fade1c0d57a7af66ab4ead7c2c2eb7b11a91385',
        blockHash: '0xfd43ade1c09fade1c0d57a7af66ab4ead7c2c2eb7b11a91ffdd57a7af66ab4ead7',
        blockNumber: 1234,
        address: '0xde0B295669a9FD93d5F28D9Ec85E40f4cb697BAe'
    },{...}]
```

![](img/dataStructureStorage_2_1.png)
![](img/dataStructureStorage_2_2.png)

```python
# Block receipts
# DB.get(r,4,92a635428702d188fda19e9d1aebcee1ca6c1ff624f9fa5ed873c0a7e4062327)
>>> DB.get('72000000000000000492a635428702d188fda19e9d1aebcee1ca6c1ff624f9fa5ed873c0a7e4062327')
'f902c8f90161a0b4708e4f774cb254dafa538df357fb403b76ae473a64a553156e0074ab926495825208b9010000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a0e2e69f1603bce3cad238f61b0802a39330782bedf5f0770561b81d71dc01f2a5940000000000000000000000000000000000000000c0825208f90161a08a5030750b857e30b4209d0ea68dd53c7c7e1807149a5c3581906f8f9e00f95882a410b9010000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a0cf1245bdfaaa8d071802e2c10bd4a5fb2ba55dccbb6ffe843e1d576fa7bbdb80940000000000000000000000000000000000000000c0825208'
[
  [
    b4708e4f774cb254dafa538df357fb403b76ae473a64a553156e0074ab926495,
    5208,
    00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000,
    e2e69f1603bce3cad238f61b0802a39330782bedf5f0770561b81d71dc01f2a5,
    0000000000000000000000000000000000000000,
    [],
    5208,
  ],
  [
    8a5030750b857e30b4209d0ea68dd53c7c7e1807149a5c3581906f8f9e00f958,
    a410,
    00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000,
    cf1245bdfaaa8d071802e2c10bd4a5fb2ba55dccbb6ffe843e1d576fa7bbdb80,
    0000000000000000000000000000000000000000,
    [],
    5208,
  ],
]

```

##### 第三种类型的查询

* 目前某个账户的余额。

##### 第四种类型的查询

* 一个账户是否存在。

第三和第四种则是由状态树负责处理的。

详见[从世界状态中获取账户余额](./docs/accountBalance.md)。

##### 第五种类型的查询

* 假如在某个合约中进行一笔交易，交易的输出是什么。

计算前4个查询任务相对简单，只需要服务器根据查询需求找到对象，获得对应的Merkle树分支，即可得到结果并返回给客户端。

第五种查询任务则相对复杂一些，它也是由状态树来处理的。如果在根为`S`的状态树上执行一笔交易`T`，其结果状态树将是根为`S'`，输出为`O''`（“输出”是以太坊中的一种概念，每笔交易都是一个函数调用）。服务器会在本地创建一个假的区块，将其状态设为`S`，并假装是一个轻客户端，请求执行这笔交易并将执行结果返回给客户端。如果在请求这笔交易的过程中需要客户端确定一个账户的余额，这个轻客户端将会发出一个余额查询。如果这个轻客户端需要检查存储在一个特定合约的项目，该轻客户端则对此进行查询。

详见[智能合约的调用方式](./docs/evmInterpreter.md#%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6%E8%B0%83%E7%94%A8%E7%9A%84%E6%96%B9%E5%BC%8F)。

