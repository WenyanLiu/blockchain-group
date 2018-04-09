# 新区块插入
获得新区块有两个途径
>1. downloader，主要负责区块链最开始的同步工作，不断下载区块到达和主链同步
>2. fetcher，收到邻居消息后获取block




### fetcher
>fetcher包含基于块通知的同步。当我们接收到NewBlockHashesMsg或者NewBlockMsg消息时，根据消息传递的内容得到要同步的区块，然后更新本地区块链。


fetcher 的数据结构
```
type Fetcher struct {
	// Various event channels
	notify chan *announce	//announce的通道，
	inject chan *inject		//inject的通道

	blockFilter  chan chan []*types.Block	 //通道的通道？
	headerFilter chan chan *headerFilterTask 
	bodyFilter   chan chan *bodyFilterTask

	done chan common.Hash
	quit chan struct{}

	// Announce states
	announces  map[string]int              //  key是peer的名字， value是announce的count， 为了避免内存占用太大。
	announced  map[common.Hash][]*announce //  等待调度fetching的announce ，announce是一个hash通知，表示网络上有合适的新区块出现。  

	fetching   map[common.Hash]*announce   // 正在fetching的announce
	fetched    map[common.Hash][]*announce // 已经获取区块头的，等待获取区块body
	completing map[common.Hash]*announce   //头和体都已经获取完成的announce

	// Block cache
	queue  *prque.Prque             //包含了import操作的队列(按照区块号排列)
	queues map[string]int          // 每个邻居的block数 key是peer，value是block数量。 避免内存消耗太多。
	queued map[common.Hash]*inject // 已经放入队列的区块。 为了去重。

	// Callbacks  依赖了一些回调函数。
	getBlock       blockRetrievalFn   // Retrieves a block from the local chain
	verifyHeader   headerVerifierFn   // 验证header
	broadcastBlock blockBroadcasterFn // 广播 block 到链接的节点
	chainHeight    chainHeightFn      // Retrieves the current chain's height
	insertChain    chainInsertFn      // 插入一组block到链上
	dropPeer       peerDropFn         // Drops a peer for misbehaving

	// Testing hooks
	// 测试用
	announceChangeHook func(common.Hash, bool) // Method to call upon adding or deleting a hash from the announce list
	queueChangeHook    func(common.Hash, bool) // Method to call upon adding or deleting a block from the import queue
	fetchingHook       func([]common.Hash)     // Method to call upon starting a block (eth/61) or header (eth/62) fetch
	completingHook     func([]common.Hash)     // Method to call upon starting a block body fetch (eth/62)
	importedHook       func(*types.Block)      // Method to call upon successful block import (both eth/61 and eth/62)
}
```

fetcher会运行一个loop函数，处理队列中的区块，循环等待其他节点发来的消息处理事件，主要处理六个事件

1.从f.queue中取出inject结构体，调用insert方法插入block到链上
```
type inject struct {
	origin string // 邻居peer的hash
	block  *types.Block
}
op := f.queue.PopItem().(*inject)
    // 插入区块
    f.insert(op.origin, op.block)
```

2.在收到NewBlockHashesMsg消息的时候，handler.go会将blockHash加入fetcher的notify通道，loop()中接收f.notify通道的notification，放入f.announced

```
case notification := <-f.notify:
    f.announced[notification.hash] = append(f.announced[notification.hash], notification)

```

3.在收到NewBlockMsg消息的时候，handler.go的handle()方法将接收到的block加入f.inject通道中。oop()中接收f.inject通道的inject结构体，放入f.queue中

```
case op := <-f.inject:
	f.enqueue(op.origin, op.block)
```

4.fetcherTimer到时间，调用fetchHeader返回得到fetchHeader

```
case <-fetchTimer.C:
// 调用fetchHeader返回得到fetchHeader
	fetchHeader, hashes := f.fetching[hashes[0]].fetchHeader, hashes

```
5.completeTimer到时间，从fetched中取出hash和announces，加入request和f.completing，遍历request，得到hashes，调用fetchBodies请求body
```
case <-fetchTimer.C:
// 调用fetchBodies请求body
	go f.completing[hashes[0]].fetchBodies(hashes)
```

6.headerFilter，
- handle.go接收到BlockHeadersMsg消息，会调用fetcher.FilterHeaders方法将header放入f.headerFilter通道
- 从f.headerFilter通道中取出filter，filter通道中取出task，遍历task.headers得到header，
- 验证fetching中有对应header.hash的announce、fetched、completing 和 queue队列中没有对应hash，验证announce的number和header对应，验证链上没有对应的hash的block。
- 根据区块头查看，如果这个区块不包含任何交易或者是Uncle区块。那么我们就不用获取区块的body了。 那么直接将没有body的block插入complete，announce放入completing中。
- 遍历complete，**取出block（无body）放入f.queue中**

7.bodyFilter
- handle.go接收到BlockBodiesMsg消息，会调用fetcher.FilterBodies方法将trasactions, uncles放入f.bodyFilter通道
- 从bodyFilter通道中取出filter通道，从filter中取出task
- 遍历task中的transactions和uncles
- 遍历completing的每个hash和announce
- 如果hash不在f.queued（已经加入完成的队列），比较announce.header与types计算的TxHash和UncleHash，匹配成功标记marked为true
- 如果链上没有对应hash的block，使用announce.header和task.transaction[i]和task.uncles[i]组装一个block，**将block放入f.queue中**




**fetcher中实现的insert方法**  
首先使用verifyHeader验证block.header，验证通过后会给邻居发送NewBlockMsg通知此block消息  

然后会调用inserChain方法将区块插入主链，如果成功，会给邻居发送NewBlockHashesMsg通知此block的hash消息  

成功插入后，会将block的hash放入f.done 通道，通知此block以成功加入

```
// 验证区块header
switch err := f.verifyHeader(block.Header()); err {
		case nil:
			// All ok, quickly propagate to our peers
			propBroadcastOutTimer.UpdateSince(block.ReceivedAt)
			go f.broadcastBlock(block, true)
// 插入block
if _, err := f.insertChain(types.Blocks{block}); err != nil {
			log.Debug("Propagated block import failed", "peer", peer, "number", block.Number(), "hash", hash, "err", err)
			return
		}
		// If import succeeded, broadcast the block
		// 如果插入成功， 那么广播区块， 第二个参数为false。那么只会对区块的hash进行广播。NewBlockHashesMsg
		propAnnounceOutTimer.UpdateSince(block.ReceivedAt)
		go f.broadcastBlock(block, false)			
```



### downloader(待补充)

>downloader主要负责区块链最开始的同步工作，当前的同步有两种模式:

>一种是传统的fullmode,这种模式通过下载区块header，和区块body来构建区块链，同步的过程就和普通的区块插入的过程一样，包括区块头的验证，交易的验证，交易执行，账户状态的改变等操作，这是一个比较消耗CPU和磁盘的一个过程。

>另一种模式就是 快速同步的fast sync模式。简单的说 fast sync的模式会下载区块头，区块体和收据，插入的过程不会执行交易，然后在一个区块高度(最高的区块高度 - 1024)的时候同步所有的账户状态，后面的1024个区块会采用fullmode的方式来构建。 这种模式会加快区块的插入时间，同时不会产生大量的历史的账户信息。会相对节约磁盘， 但是对于网络的消耗会更高。 因为需要下载收据和状态。