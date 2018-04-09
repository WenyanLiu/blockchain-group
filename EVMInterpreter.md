# 以太坊虚拟机

以太坊虚拟机（Ethereum Virtual Machine, EVM）是Account相关的EVM Code执行的关键部分。

## 指令集

详见[以太坊黄皮书](https://ethereum.github.io/yellowpaper/paper.pdf)Appendix H.2. Instruction Set。

![evm_0s](img/evm_0s.png)
![evm_10s20s](img/evm_10s20s.png)
![evm_30s](img/evm_30s.png)
![evm_40s](img/evm_40s.png)
![evm_50s](img/evm_50s.png)
![evm_60s70s](img/evm_60s70s.png)
![evm_80s90s](img/evm_80s90s.png)
![evm_a0s](img/evm_a0s.png)
![evm_f0s](img/evm_f0s.png)

## EVM的汇编代码

合约`C`有状态变量和构造器：

```solidity
pragma solidity ^0.4.21;contract C {    uint256 cbd;    function C() public{        cbd = 109;    }}
```

使用`solc`编译合约`C`：

```solc
======= c1.sol:C =======EVM assembly:... */ "c1.sol":28:114  contract C {  mstore(0x40, 0x60)... */ "c1.sol":64:111  function C() public{  jumpi(tag_1, iszero(callvalue))  0x0  dup1  reverttag_1:    /* "c1.sol":100:103  109 */  0x6d    /* "c1.sol":94:97  cbd */  0x0    /* "c1.sol":94:103  cbd = 109 */  dup2  swap1  sstore  pop... */ "c1.sol":28:114  contract C {  dataSize(sub_0)  dup1  dataOffset(sub_0)  0x0  codecopy  0x0  returnstopsub_0: assembly {... */  /* "c1.sol":28:114  contract C {      mstore(0x40, 0x60)      0x0      dup1      revert    auxdata: 0xa165627a7a723058204c4b7cb2d6601994bbe06eeb297368f48ca11be35e15fe1d8e5aca29313706070029}Binary:60606040523415600e57600080fd5b606d60008190555060358060236000396000f3006060604052600080fd00a165627a7a723058204c4b7cb2d6601994bbe06eeb297368f48ca11be35e15fe1d8e5aca29313706070029
```

其中，`Binary`是EVM运行时的字节码。变量赋值`cbd = 109;`由字节码`606d600081905550`表示，缩进到标签`tag_1`下：

```solc
tag_1:    /* 60 6d */  0x6d    /* 60 00 */  0x0    /* 81 */  dup2
    /* 90 */  swap1
    /* 55 */  sstore
    /* 50 */  pop
```

在汇编代码中，`0x0`是`PUSH(0x0)`的简写，即将数值0压栈。

模拟EVM执行字节序列，同时打印每条指令执行后的机器状态得到：

```solc
    /* 60 6d：压栈6d */  0x6d
    stack: [0x6d]
    store: {}
        /* 60 00：压栈0 */  0x0
    stack: [0x0 0x6d]
    store: {}
        /* 81：复制栈中的第2项 */  dup2
    stack: [0x6d 0x0 0x6d]
    store: {}
    
    /* 90：交换栈中的第1和2项 */  swap1
    stack: [0x0 0x6d 0x6d]
    store: {}
    
    /* 55：存储数值 */  sstore
    stack: [0x6d]
    store: {0x0=>0x6d}
    
    /* 50：出栈 */  pop
    stack: []
    store: {0x0=>0x6d}
```

#### 存疑❓

```diff
- 在汇编语言中，状态变量cbd的名称是如何存储的？
- auxdata里存储的是什么？
```

## :warning: TODO

```go
core/vm.newstack() line 33
core/state.(*StateDB).CreateAccount() line 471
core/state.(*StateDB).createObject() line 447
core/state.(*StateDB).getStateObject() line 396
trie.(*SecureTrie).TryGet() line 79
trie.(*SecureTrie).hashKey() line 180
trie.newHasher() line 46
crypto/sha3.NewKeccak256() line 17
crypto/sha3.(*state).Reset() line 62
crypto/sha3.(*state).Write() line 134
crypto/sha3.(*state).Sum() line 196
crypto/sha3.(*state).clone() line 72
crypto/sha3.(*state).Read() line 170
crypto/sha3.(*state).padAndPermute() line 106
crypto/sha3.(*state).permute() line 86
crypto/sha3.xorInUnaligned() line 15
crypto/sha3.copyOutUnaligned() line 51
trie.returnHasherToPool() line 53
trie.(*Trie).TryGet() line 141
trie.keybytesToHex() line 71
trie.(*Trie).tryGet() line 151
core/state.(*StateDB).setError() line 109
core/state.newObject() line 115
crypto.Keccak256Hash() line 54
crypto/sha3.NewKeccak256() line 17
crypto/sha3.(*state).Write() line 134
crypto/sha3.(*state).Sum() line 196
crypto/sha3.(*state).clone() line 72
crypto/sha3.(*state).Read() line 170
crypto/sha3.(*state).padAndPermute() line 106
crypto/sha3.(*state).permute() line 86
crypto/sha3.xorInUnaligned() line 15
crypto/sha3.copyOutUnaligned() line 51
core/state.(*stateObject).setNonce() line 391
core/state.(*stateObject).Address() line 338
core/state.(*StateDB).MarkStateObjectDirty() line 440
core/state.(*StateDB).setStateObject() line 423
core/state.(*stateObject).Address() line 338
core/state.(*StateDB).SetCode() line 334
core/state.(*StateDB).GetOrNewStateObject() line 429
core/state.(*StateDB).getStateObject() line 396
crypto.Keccak256Hash() line 54
crypto/sha3.NewKeccak256() line 17
crypto/sha3.(*state).Write() line 134
crypto/sha3.(*state).Sum() line 196
crypto/sha3.(*state).clone() line 72
crypto/sha3.(*state).Read() line 170
crypto/sha3.(*state).padAndPermute() line 106
crypto/sha3.(*state).permute() line 86
crypto/sha3.xorInUnaligned() line 15
crypto/sha3.copyOutUnaligned() line 51
core/state.(*stateObject).SetCode() line 360
core/state.(*stateObject).Code() line 344
core/state.(*stateObject).CodeHash() line 400
core/state.(*stateObject).CodeHash() line 400
core/state.(*stateObject).setCode() line 371
common.StringToAddress() line 169
common.BytesToAddress() line 164
common.(*Address).SetBytes() line 234
core/vm.(*EVM).Call() line 144
core/vm.AccountRef.Address() line 42
core.CanTransfer() line 93
core/state.(*StateDB).GetBalance() line 207
core/state.(*StateDB).getStateObject() line 396
trie.(*SecureTrie).TryGet() line 79
trie.(*SecureTrie).hashKey() line 180
trie.newHasher() line 46
crypto/sha3.(*state).Reset() line 62
crypto/sha3.(*state).Write() line 134
crypto/sha3.(*state).Sum() line 196
crypto/sha3.(*state).clone() line 72
crypto/sha3.(*state).Read() line 170
crypto/sha3.(*state).padAndPermute() line 106
crypto/sha3.(*state).permute() line 86
crypto/sha3.xorInUnaligned() line 15
crypto/sha3.copyOutUnaligned() line 51
trie.returnHasherToPool() line 53
trie.(*Trie).TryGet() line 141
trie.keybytesToHex() line 71
trie.(*Trie).tryGet() line 151
core/state.(*StateDB).setError() line 109
core/state.(*StateDB).Snapshot() line 535
core/state.(*StateDB).Exist() line 193
core/state.(*StateDB).getStateObject() line 396
core/vm.AccountRef.Address() line 42
core/vm.AccountRef.Address() line 42
core.Transfer() line 99
core/state.(*StateDB).SubBalance() line 310
core/state.(*StateDB).GetOrNewStateObject() line 429
core/state.(*StateDB).getStateObject() line 396
trie.(*SecureTrie).TryGet() line 79
trie.(*SecureTrie).hashKey() line 180
trie.newHasher() line 46
crypto/sha3.(*state).Reset() line 62
crypto/sha3.(*state).Write() line 134
crypto/sha3.(*state).Sum() line 196
crypto/sha3.(*state).clone() line 72
crypto/sha3.(*state).Read() line 170
crypto/sha3.(*state).padAndPermute() line 106
crypto/sha3.(*state).permute() line 86
crypto/sha3.xorInUnaligned() line 15
crypto/sha3.copyOutUnaligned() line 51
trie.returnHasherToPool() line 53
trie.(*Trie).TryGet() line 141
trie.keybytesToHex() line 71
trie.(*Trie).tryGet() line 151
core/state.(*StateDB).setError() line 109
core/state.(*StateDB).createObject() line 447
core/state.(*StateDB).getStateObject() line 396
trie.(*SecureTrie).TryGet() line 79
trie.(*SecureTrie).hashKey() line 180
trie.newHasher() line 46
crypto/sha3.(*state).Reset() line 62
crypto/sha3.(*state).Write() line 134
crypto/sha3.(*state).Sum() line 196
crypto/sha3.(*state).clone() line 72
crypto/sha3.(*state).Read() line 170
crypto/sha3.(*state).padAndPermute() line 106
crypto/sha3.(*state).permute() line 86
crypto/sha3.xorInUnaligned() line 15
crypto/sha3.copyOutUnaligned() line 51
trie.returnHasherToPool() line 53
trie.(*Trie).TryGet() line 141
trie.keybytesToHex() line 71
trie.(*Trie).tryGet() line 151
core/state.(*StateDB).setError() line 109
core/state.newObject() line 115
crypto.Keccak256Hash() line 54
crypto/sha3.NewKeccak256() line 17
crypto/sha3.(*state).Write() line 134
crypto/sha3.(*state).Sum() line 196
crypto/sha3.(*state).clone() line 72
crypto/sha3.(*state).Read() line 170
crypto/sha3.(*state).padAndPermute() line 106
crypto/sha3.(*state).permute() line 86
crypto/sha3.xorInUnaligned() line 15
crypto/sha3.copyOutUnaligned() line 51
core/state.(*stateObject).setNonce() line 391
core/state.(*stateObject).Address() line 338
core/state.(*StateDB).MarkStateObjectDirty() line 440
core/state.(*StateDB).setStateObject() line 423
core/state.(*stateObject).Address() line 338
core/state.(*stateObject).SubBalance() line 289
core/state.(*StateDB).AddBalance() line 301
core/state.(*StateDB).GetOrNewStateObject() line 429
core/state.(*StateDB).getStateObject() line 396
core/state.(*stateObject).AddBalance() line 273
core/state.(*stateObject).empty() line 100
core/vm.NewContract() line 73
core/vm.AccountRef.Address() line 42
core/state.(*StateDB).GetCodeHash() line 251
core/state.(*StateDB).getStateObject() line 396
core/state.(*stateObject).CodeHash() line 400
common.BytesToHash() line 45
common.(*Hash).SetBytes() line 108
core/state.(*StateDB).GetCode() line 226
core/state.(*StateDB).getStateObject() line 396
core/state.(*stateObject).Code() line 344
core/vm.(*Contract).SetCallCode() line 163
core/vm.run() line 44
core/vm.(*EVM).ChainConfig() line 404
params.(*ChainConfig).IsByzantium() line 202
params.isForked() line 289
core/vm.(*Interpreter).Run() line 110
core/vm.NewMemory() line 31
core/vm.newstack() line 33
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.gasPush() line 477
core/vm.(*Contract).UseGas() line 133
core/vm.(*intPool).get() line 42
core/vm.(*Stack).len() line 62
common.RightPadBytes() line 104
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.gasPush() line 477
core/vm.(*Contract).UseGas() line 133
core/vm.(*intPool).get() line 42
core/vm.(*Stack).len() line 62
common.RightPadBytes() line 104
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.memoryMStore() line 62
core/vm.(*Stack).Back() line 83
core/vm.(*Stack).len() line 62
core/vm.calcMemSize() line 29
core/vm.bigUint64() line 66
core/vm.toWordSize() line 72
common/math.SafeMul() line 95
core/vm.gasMStore() line 281
core/vm.memoryGasCost() line 29
core/vm.toWordSize() line 72
core/vm.(*Memory).Len() line 92
common/math.SafeAdd() line 90
core/vm.(*Contract).UseGas() line 133
core/vm.(*Memory).Resize() line 53
core/vm.(*Memory).Len() line 92
core/vm.(*Memory).Len() line 92
core/vm.opMstore() line 608
core/vm.(*Stack).pop() line 55
core/vm.(*Stack).pop() line 55
common/math.PaddedBigBytes() line 134
common/math.ReadBits() line 176
core/vm.(*Memory).Set() line 37
core/vm.(*intPool).put() line 62
core/vm.(*Stack).push() line 43
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.(*Contract).UseGas() line 133
core/vm.opCallValue() line 441
core/vm.(*intPool).get() line 42
core/vm.(*Stack).len() line 62
core/vm.(*Stack).pop() line 55
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.(*Contract).UseGas() line 133
core/vm.opIszero() line 259
core/vm.(*Stack).peek() line 77
core/vm.(*Stack).len() line 62
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.gasPush() line 477
core/vm.(*Contract).UseGas() line 133
core/vm.(*intPool).get() line 42
core/vm.(*Stack).len() line 62
core/vm.(*Stack).pop() line 55
common.RightPadBytes() line 104
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.(*Contract).UseGas() line 133
core/vm.opJumpi() line 657
core/vm.(*Stack).pop() line 55
core/vm.(*Stack).pop() line 55
core/vm.destinations.has() line 33
core/vm.codeBitmap() line 72
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set8() line 59
core/vm.(*bitvec).set8() line 59
core/vm.(*bitvec).set8() line 59
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).set() line 55
core/vm.(*bitvec).codeSegment() line 66
core/vm.(*intPool).put() line 62
core/vm.(*Stack).push() line 43
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.(*Contract).UseGas() line 133
core/vm.opJumpdest() line 674
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.gasPush() line 477
core/vm.(*Contract).UseGas() line 133
core/vm.(*intPool).get() line 42
core/vm.(*Stack).len() line 62
core/vm.(*Stack).pop() line 55
common.RightPadBytes() line 104
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.gasPush() line 477
core/vm.(*Contract).UseGas() line 133
core/vm.(*intPool).get() line 42
core/vm.(*Stack).len() line 62
core/vm.(*Stack).pop() line 55
common.RightPadBytes() line 104
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.gasDup() line 487
core/vm.(*Contract).UseGas() line 133
core/vm.(*Stack).dup() line 72
core/vm.(*intPool).get() line 42
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.gasSwap() line 482
core/vm.(*Contract).UseGas() line 133
core/vm.(*Stack).swap() line 67
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.gasSStore() line 124
core/vm.(*Stack).Back() line 83
core/vm.(*Stack).len() line 62
core/vm.(*Stack).Back() line 83
core/vm.(*Stack).len() line 62
core/vm.(*Contract).Address() line 143
core/vm.AccountRef.Address() line 42
common.BigToHash() line 52
common.BytesToHash() line 45
common.(*Hash).SetBytes() line 108
core/state.(*StateDB).GetState() line 260
core/state.(*StateDB).getStateObject() line 396
core/state.(*stateObject).GetState() line 185
core/state.(*stateObject).getTrie() line 171
core/state.(*cachingDB).OpenStorageTrie() line 128
trie.NewSecure() line 54
trie.New() line 101
trie.(*Trie).SetCacheLimit() line 84
trie.(*SecureTrie).TryGet() line 79
trie.(*SecureTrie).hashKey() line 180
trie.newHasher() line 46
crypto/sha3.(*state).Reset() line 62
crypto/sha3.(*state).Write() line 134
crypto/sha3.(*state).Sum() line 196
crypto/sha3.(*state).clone() line 72
crypto/sha3.(*state).Read() line 170
crypto/sha3.(*state).padAndPermute() line 106
crypto/sha3.(*state).permute() line 86
crypto/sha3.xorInUnaligned() line 15
crypto/sha3.copyOutUnaligned() line 51
trie.returnHasherToPool() line 53
trie.(*Trie).TryGet() line 141
trie.keybytesToHex() line 71
trie.(*Trie).tryGet() line 151
common.EmptyHash() line 139
common.BigToHash() line 52
common.BytesToHash() line 45
common.(*Hash).SetBytes() line 108
common.EmptyHash() line 139
core/vm.(*Contract).UseGas() line 133
core/vm.opSstore() line 634
core/vm.(*Stack).pop() line 55
common.BigToHash() line 52
common.BytesToHash() line 45
common.(*Hash).SetBytes() line 108
core/vm.(*Stack).pop() line 55
core/vm.(*Contract).Address() line 143
core/vm.AccountRef.Address() line 42
common.BigToHash() line 52
common.BytesToHash() line 45
common.(*Hash).SetBytes() line 108
core/state.(*StateDB).SetState() line 342
core/state.(*StateDB).GetOrNewStateObject() line 429
core/state.(*StateDB).getStateObject() line 396
core/state.(*stateObject).SetState() line 211
core/state.(*stateObject).GetState() line 185
core/state.(*stateObject).getTrie() line 171
trie.(*SecureTrie).TryGet() line 79
trie.(*SecureTrie).hashKey() line 180
trie.newHasher() line 46
crypto/sha3.(*state).Reset() line 62
crypto/sha3.(*state).Write() line 134
crypto/sha3.(*state).Sum() line 196
crypto/sha3.(*state).clone() line 72
crypto/sha3.(*state).Read() line 170
crypto/sha3.(*state).padAndPermute() line 106
crypto/sha3.(*state).permute() line 86
crypto/sha3.xorInUnaligned() line 15
crypto/sha3.copyOutUnaligned() line 51
trie.returnHasherToPool() line 53
trie.(*Trie).TryGet() line 141
trie.keybytesToHex() line 71
trie.(*Trie).tryGet() line 151
core/state.(*stateObject).setState() line 221
core/vm.(*intPool).put() line 62
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.(*Contract).UseGas() line 133
core/vm.opPop() line 592
core/vm.(*Stack).pop() line 55
core/vm.(*intPool).put() line 62
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.gasPush() line 477
core/vm.(*Contract).UseGas() line 133
core/vm.(*intPool).get() line 42
core/vm.(*Stack).len() line 62
core/vm.(*Stack).pop() line 55
common.RightPadBytes() line 104
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.gasDup() line 487
core/vm.(*Contract).UseGas() line 133
core/vm.(*Stack).dup() line 72
core/vm.(*intPool).get() line 42
core/vm.(*Stack).len() line 62
core/vm.(*Stack).pop() line 55
core/vm.(*Stack).len() line 62
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.gasPush() line 477
core/vm.(*Contract).UseGas() line 133
core/vm.(*intPool).get() line 42
core/vm.(*Stack).len() line 62
common.RightPadBytes() line 104
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.gasPush() line 477
core/vm.(*Contract).UseGas() line 133
core/vm.(*intPool).get() line 42
core/vm.(*Stack).len() line 62
common.RightPadBytes() line 104
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.memoryCodeCopy() line 42
core/vm.(*Stack).Back() line 83
core/vm.(*Stack).len() line 62
core/vm.(*Stack).Back() line 83
core/vm.(*Stack).len() line 62
core/vm.calcMemSize() line 29
core/vm.bigUint64() line 66
core/vm.toWordSize() line 72
common/math.SafeMul() line 95
core/vm.gasCodeCopy() line 203
core/vm.memoryGasCost() line 29
core/vm.toWordSize() line 72
core/vm.(*Memory).Len() line 92
common/math.SafeAdd() line 90
core/vm.(*Stack).Back() line 83
core/vm.(*Stack).len() line 62
core/vm.bigUint64() line 66
core/vm.toWordSize() line 72
common/math.SafeMul() line 95
common/math.SafeAdd() line 90
core/vm.(*Contract).UseGas() line 133
core/vm.(*Memory).Resize() line 53
core/vm.(*Memory).Len() line 92
core/vm.opCodeCopy() line 513
core/vm.(*Stack).pop() line 55
core/vm.(*Stack).pop() line 55
core/vm.(*Stack).pop() line 55
core/vm.getDataBig() line 55
common/math.BigMin() line 113
common/math.BigMin() line 113
common.RightPadBytes() line 104
core/vm.(*Memory).Set() line 37
core/vm.(*intPool).put() line 62
core/vm.(*Stack).push() line 43
core/vm.(*Stack).push() line 43
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.gasPush() line 477
core/vm.(*Contract).UseGas() line 133
core/vm.(*intPool).get() line 42
core/vm.(*Stack).len() line 62
core/vm.(*Stack).pop() line 55
common.RightPadBytes() line 104
core/vm.(*Stack).push() line 43
core/vm.(*Contract).GetOp() line 108
core/vm.(*Contract).GetByte() line 114
core/vm.(*Stack).require() line 88
core/vm.(*Stack).len() line 62
core/vm.(*Stack).len() line 62
core/vm.(*Interpreter).enforceRestrictions() line 87
core/vm.memoryReturn() line 103
core/vm.(*Stack).Back() line 83
core/vm.(*Stack).len() line 62
core/vm.(*Stack).Back() line 83
core/vm.(*Stack).len() line 62
core/vm.calcMemSize() line 29
core/vm.bigUint64() line 66
core/vm.toWordSize() line 72
common/math.SafeMul() line 95
core/vm.gasReturn() line 398
core/vm.memoryGasCost() line 29
core/vm.toWordSize() line 72
core/vm.(*Memory).Len() line 92
core/vm.(*Contract).UseGas() line 133
core/vm.(*Memory).Resize() line 53
core/vm.(*Memory).Len() line 92
core/vm.opReturn() line 843
core/vm.(*Stack).pop() line 55
core/vm.(*Stack).pop() line 55
core/vm.(*Memory).GetPtr() line 78
core/vm.(*intPool).put() line 62
core/vm.(*Stack).push() line 43
core/vm.(*Stack).push() line 43
```


