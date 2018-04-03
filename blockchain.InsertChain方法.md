# 区块插入具体过程

以太坊通过fetcher和downloader将邻居节点传送的区块进行验证，并插入本地链中。

#### fetcher的方式

- fetcher用来处理单个block的消息的添加，只能添加本地当前chainHeight范围（chainHeight-6 至chainHeight+32）的区块。

- handler.go会执行loop循环会处理邻居发送的NewBlockMsg消息，其中有新的block信息，交给fetcher的插入队列f.queue
- fetcher会对block的number进行验证后调用自身的insert方法
- insert方法中会调用f.verifyHeader验证block头部，验证通过后会执行f.insertChain方法，f.insertChain方法会调用blockchain.InsertChain方法插入区块

#### downloader的方式
- downloader方法用来和邻居节点进行区块同步，在有新邻居加入或每10秒回强制同步一次

- handler.go会执行loop循环会处理邻居发送的NewBlockMsg消息，如果调用sync.go的synchronise方法 
- sync.go的synchronise方法会设置downloader的模式，FullSync还是FastSync，调用downloader.Synchronise方法
- downloader.Synchronise方法会调用d.syncWithPeer方法
- d.syncWithPeer会创建fetchers函数数组，调用d.spawnSync(fetchers)方法
- d.spawnSync方法会执行fetchers中的函数：
- fetchHeaders，fetchBodies，fetchReceipts，processHeaders，processFullSyncContent或者processFastSyncContent
- 获取到block之后会执行blockchain.InsertChain方法插入区块

#### 

![](.\img\newBlockInsert.jpg)



#### chainHeadEvent通知

当节点挖掘或者收到邻居的block并插入到链上时，会发送chainHeadEvent通知。

tx_pool会执行reset方法，更新交易池中的交易

```
tx_pool.go
func (pool *TxPool) loop() {
for {
		select {
		// Handle ChainHeadEvent
		// 处理新链头事件
		case ev := <-pool.chainHeadCh:
			if ev.Block != nil {
				pool.mu.Lock()
				// 家园阶段
				if pool.chainconfig.IsHomestead(ev.Block.Number()) {
					pool.homestead = true
				}

				// 重置当前交易池，确保其中交易有效
				pool.reset(head.Header(), ev.Block.Header())
				head = ev.Block

				pool.mu.Unlock()
			}
    }
}
```

txpool会执行setNewHead()

```
//txpool.go
func (pool *TxPool) eventLoop() {
	for {
		select {
		case ev := <-pool.chainHeadCh:

			// 设置新的head
			pool.setNewHead(ev.Block.Header())
			// hack in order to avoid hogging the lock; this part will
			// be replaced by a subsequent PR.
			//
			time.Sleep(time.Millisecond)

		// System stopped
		case <-pool.chainHeadSub.Err():
			return
		}
	}
}
```



worker会执行commitNewWork方法，组装新的block

```
// wroker.go  注册监听
func (self *worker) update() {
	defer self.txSub.Unsubscribe()
	defer self.chainHeadSub.Unsubscribe()
	defer self.chainSideSub.Unsubscribe()

	for {
		// A real event arrived, process interesting content
		select {
		// Handle ChainHeadEvent
		// 得知已经有新的链头 就组装下一个块
		case <-self.chainHeadCh:
			self.commitNewWork()
	}
}
```





#### blockchain.insertChain()


下面详细解释blockchain的insertChain方法
```
func (bc *BlockChain) insertChain(chain types.Blocks) (int, []interface{}, []*types.Log, error) {
	// Do a sanity check that the provided chain is actually ordered and linked
	// 检查链有序
	for i := 1; i < len(chain); i++ {
		if chain[i].NumberU64() != chain[i-1].NumberU64()+1 || chain[i].ParentHash() != chain[i-1].Hash() {
			// Chain broke ancestry, log a messge (programming error) and skip insertion
			log.Error("Non contiguous block insert", "number", chain[i].Number(), "hash", chain[i].Hash(),
				"parent", chain[i].ParentHash(), "prevnumber", chain[i-1].Number(), "prevhash", chain[i-1].Hash())

			return 0, nil, nil, fmt.Errorf("non contiguous insert: item %d is #%d [%x…], item %d is #%d [%x…] (parent [%x…])", i-1, chain[i-1].NumberU64(),
				chain[i-1].Hash().Bytes()[:4], i, chain[i].NumberU64(), chain[i].Hash().Bytes()[:4], chain[i].ParentHash().Bytes()[:4])
		}
	}
	// Pre-checks passed, start the full block imports
	// 预检查通过，开始全block import
	bc.wg.Add(1)
	defer bc.wg.Done()

	bc.chainmu.Lock()
	defer bc.chainmu.Unlock()

	// A queued approach to delivering events. This is generally
	// faster than direct delivery and requires much less mutex
	// acquiring.
	// 一个发送事件队列
	var (
		stats         = insertStats{startTime: mclock.Now()}
		events        = make([]interface{}, 0, len(chain))
		lastCanon     *types.Block
		coalescedLogs []*types.Log
	)
	// Start the parallel header verifier
	// 开始并行header验证
	headers := make([]*types.Header, len(chain))
	seals := make([]bool, len(chain))

	for i, block := range chain {
		headers[i] = block.Header()
		seals[i] = true
	}

	// 验证头部
	abort, results := bc.engine.VerifyHeaders(bc, headers, seals)

	defer close(abort)

	// Iterate over the blocks and insert when the verifier permits
	// 遍历所有的blocks 在验证通过后插入
	for i, block := range chain {
		// If the chain is terminating, stop processing blocks
		// 一个链终止，停止产生block
		if atomic.LoadInt32(&bc.procInterrupt) == 1 {
			log.Debug("Premature abort during blocks processing")
			break
		}
		// If the header is a banned one, straight out abort
		// header是被禁止的
		if BadHashes[block.Hash()] {
			bc.reportBlock(block, nil, ErrBlacklistedHash)
			return i, events, coalescedLogs, ErrBlacklistedHash
		}
		// Wait for the block's verification to complete
		// 等待block的验证完成
		// TODO : 时间的作用
		bstart := time.Now()

		err := <-results
		if err == nil {
			// 验证body有效性
			err = bc.Validator().ValidateBody(block)
		}
		switch {
		case err == ErrKnownBlock:
			// Block and state both already known. However if the current block is below
			// this number we did a rollback and we should reimport it nonetheless.
			// block和state已经知道
			// 但是如果当前block低于这个数字我们做了回滚我们应该重新导入它
			if bc.CurrentBlock().NumberU64() >= block.NumberU64() {
				stats.ignored++
				continue
			}

		case err == consensus.ErrFutureBlock:
			// Allow up to MaxFuture second in the future blocks. If this limit is exceeded
			// the chain is discarded and processed at a later time if given.
			// 过于超前
			max := big.NewInt(time.Now().Unix() + maxTimeFutureBlocks)
			if block.Time().Cmp(max) > 0 {
				return i, events, coalescedLogs, fmt.Errorf("future block: %v > %v", block.Time(), max)
			}
			bc.futureBlocks.Add(block.Hash(), block)
			stats.queued++
			continue

		case err == consensus.ErrUnknownAncestor && bc.futureBlocks.Contains(block.ParentHash()):
			bc.futureBlocks.Add(block.Hash(), block)
			stats.queued++
			continue

		//祖先已知，但是状态不可用
		case err == consensus.ErrPrunedAncestor:
			// Block competing with the canonical chain, store in the db, but don't process
			// until the competitor TD goes above the canonical TD
			// 得到当前块
			currentBlock := bc.CurrentBlock()
			// 得到本地当前TD
			localTd := bc.GetTd(currentBlock.Hash(), currentBlock.NumberU64())
			externTd := new(big.Int).Add(bc.GetTd(block.ParentHash(), block.NumberU64()-1), block.Difficulty())

			// 本地td更大
			if localTd.Cmp(externTd) > 0 {
				// 写入blocks 不带state
			if err = bc.WriteBlockWithoutState(block, externTd); err != nil {
					return i, events, coalescedLogs, err
				}
				continue
			}
			// Competitor chain beat canonical, gather all blocks from the common ancestor
			// 竞争链击败主链 从共同ancestor收集所有的block
			var winner []*types.Block

			// 得到 父块
			parent := bc.GetBlock(block.ParentHash(), block.NumberU64()-1)
			// 父块的Root 的state有错误
			for !bc.HasState(parent.Root()) {
				winner = append(winner, parent)
				// 继续找父块
				parent = bc.GetBlock(parent.ParentHash(), parent.NumberU64()-1)
			}
			// 逆序？
			for j := 0; j < len(winner)/2; j++ {
				winner[j], winner[len(winner)-1-j] = winner[len(winner)-1-j], winner[j]
			}
			// Import all the pruned blocks to make the state available
			// 导入所有的修剪的block 是的state可用
			bc.chainmu.Unlock()
			// 插入winner到区块链
			_, evs, logs, err := bc.insertChain(winner)
			bc.chainmu.Lock()
			events, coalescedLogs = evs, logs

			if err != nil {
				return i, events, coalescedLogs, err
			}

		case err != nil:
			bc.reportBlock(block, nil, err)
			return i, events, coalescedLogs, err
		}
		// Create a new statedb using the parent block and report an
		// error if it fails.
		// 使用父块创建新的statedb
		var parent *types.Block
		if i == 0 {
			parent = bc.GetBlock(block.ParentHash(), block.NumberU64()-1)
		} else {
			parent = chain[i-1]
		}
		// 新的state
		state, err := state.New(parent.Root(), bc.stateCache)
		if err != nil {
			return i, events, coalescedLogs, err
		}
		
		// Process block using the parent state as reference point.
		// 使用parent state 执行 block
		// 这里会执行ApplyTransaction
		receipts, logs, usedGas, err := bc.processor.Process(block, state, bc.vmConfig)

		if err != nil {
			//log 记录错误
			bc.reportBlock(block, receipts, err)
			return i, events, coalescedLogs, err
		}
		// Validate the state using the default validator
		// 验证state gas使用，收据
		err = bc.Validator().ValidateState(block, parent, state, receipts, usedGas)

		if err != nil {
			bc.reportBlock(block, receipts, err)
			return i, events, coalescedLogs, err
		}
		proctime := time.Since(bstart)

		// Write the block to the chain and get the status.
		// 写block到链上
		status, err := bc.WriteBlockWithState(block, receipts, state)
		if err != nil {
			return i, events, coalescedLogs, err
		}
		
		switch status {
		// 主链
		case CanonStatTy:
			log.Debug("Inserted new block", "number", block.Number(), "hash", block.Hash(), "uncles", len(block.Uncles()),
				"txs", len(block.Transactions()), "gas", block.GasUsed(), "elapsed", common.PrettyDuration(time.Since(bstart)))

			coalescedLogs = append(coalescedLogs, logs...)
			blockInsertTimer.UpdateSince(bstart)
			// 通知block事件
			events = append(events, ChainEvent{block, block.Hash(), logs})
			lastCanon = block

			// Only count canonical blocks for GC processing time
			bc.gcproc += proctime

		//侧链
		case SideStatTy:
			log.Debug("Inserted forked block", "number", block.Number(), "hash", block.Hash(), "diff", block.Difficulty(), "elapsed",
				common.PrettyDuration(time.Since(bstart)), "txs", len(block.Transactions()), "gas", block.GasUsed(), "uncles", len(block.Uncles()))

			blockInsertTimer.UpdateSince(bstart)
			// 侧链通知
			events = append(events, ChainSideEvent{block})
		}
		stats.processed++
		stats.usedGas += usedGas
		stats.report(chain, i, bc.stateCache.TrieDB().Size())
	}
	// Append a single chain head event if we've progressed the chain
	// 加入ChainHeadEvent事件，本地得到此消息会更新txpool，worker重新commiteNewWork
	if lastCanon != nil && bc.CurrentBlock().Hash() == lastCanon.Hash() {
		events = append(events, ChainHeadEvent{lastCanon})
	}
	return 0, events, coalescedLogs, nil
}
```