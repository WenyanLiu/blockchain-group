# 虚拟机（EVM）的基本数据结构
根据以太坊黄皮书的介绍，交易执行是以太坊协议中最复杂的部分，因为它的核心在于定义了状态转换函数。根据我们之前的学习中，我们
追踪命令行调用交易的命令，从交易的基本数据结构，到交易创建、入池验证、打包验证，到最后创建虚拟机执行交易，还原了整个虚拟机
外交易的执行流程。从本次学习开始，我们深入到以太坊虚拟机内部，从虚拟机的基本结构开始，继续追踪交易与合约的执行。

首先，在go语言代码中，虚拟机的基本结构如下(core/vm/evm.go)：

    type EVM struct {
    	// Context provides auxiliary blockchain related information 提供区块链中的相关信息
    	Context
    	// StateDB gives access to the underlying state 获取当前状态的信息
    	StateDB StateDB
    	// Depth is the current call stack 当前的堆栈
    	depth int
    
    	// chainConfig contains information about the current chain 当前链的信息
    	chainConfig *params.ChainConfig
    	// chain rules contains the chain rules for the current epoch 当前时间的 chain rule
    	chainRules params.Rules
    	// virtual machine configuration options used to initialise the
    	// evm. 初始化新建配置选项
    	vmConfig Config
    	// global (to this context) ethereum virtual machine
    	// used throughout the execution of the tx. 解释器，用于执行EVM中的合约代码
    	interpreter *Interpreter
    	// abort is used to abort the EVM calling operations
    	// NOTE: must be set atomically 终止虚拟机操作命令
    	abort int32
    	// callGasTemp holds the gas available for the current call. This is needed because the
    	// available gas is calculated in gasCall* according to the 63/64 rule and later
    	// applied in opCall*.  当前操作的gas
    	callGasTemp uint64
    }

在以太坊的黄皮书中，定义的EVM是一个虚拟状态机，具有基于堆栈的架构，我们先从结构体来进行理解。先来看ApplyTransaction()中，新建虚拟机的代码(core/state_process.go)：

    // Create a new context to be used in the EVM environment
    context := NewEVMContext(msg, header, bc, author)
    // Create a new environment which holds all relevant information
    // about the transaction and calling mechanisms.
    vmenv := vm.NewEVM(context, statedb, config, cfg)

首先是需要新建Context结构，其对应的结构体如下(core/vm/evm.go)：

    type Context struct {
    	// CanTransfer returns whether the account contains
    	// sufficient ether to transfer the value
    	CanTransfer CanTransferFunc
    	// Transfer transfers ether from one account to the other
    	Transfer TransferFunc
    	// GetHash returns the hash corresponding to n
    	GetHash GetHashFunc
    
    	// Message information
    	Origin   common.Address // Provides information for ORIGIN
    	GasPrice *big.Int       // Provides information for GASPRICE
    
    	// Block information
    	Coinbase    common.Address // Provides information for COINBASE
    	GasLimit    uint64         // Provides information for GASLIMIT
    	BlockNumber *big.Int       // Provides information for NUMBER
    	Time        *big.Int       // Provides information for TIME
    	Difficulty  *big.Int       // Provides information for DIFFICULTY
    }

可以看出，该结构体携带有交易的信息(GasPrice, GasLimit)(Origin为交易发送方)，Block的信息(Number, Difficulty)(Coinbase为矿工)，以及转帐函数等，提供给EVM；
TranferFunc定义了具体的转账函数操作;StateDB为EVM提供statedb的相关操作；Interpreter结构体作为解释器，用来解释执行EVM中合约(Contract)的指令(Code)。

## 交易执行函数 evm.call(core/vm/evm.go)
1. 检查堆栈深度（不超过1024），以及账户是否能够转账（当前账户余额是否充足）
2. 执行Transfer(),本质上就是一个转入与转出操作
```
    // Transfer subtracts amount from sender and adds amount to recipient using the given Db
    func Transfer(db vm.StateDB, sender, recipient common.Address, amount *big.Int) {
    	db.SubBalance(sender, amount)
    	db.AddBalance(recipient, amount)
    }
```
所有关于状态信息的操作，均在StateDB中完成，这个对象是在work执行交易时传入,目前为止，相关的改变并没有提交到写到链上！
3. 创建合约并赋值合约code
```
    // Initialise a new contract and set the code that is to be used by the EVM.
	// The contract is a scoped environment for this execution context only.
	contract := NewContract(caller, to, value, gas)
	contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))
```
合约是EVM用来执行虚拟机指令的结构体 **这一步让我比较困惑，因为智能合约的部署和调用都是tx的形式，
而所有的tx执行都会创建一个合约的结构体来执行所携带的指令操作，这一步逻辑有待进一步理解**
```
    // Contract represents an ethereum contract in the state database. It contains
    // the the contract code, calling arguments. Contract implements ContractRef
    type Contract struct {
    	// CallerAddress is the result of the caller which initialised this
    	// contract. However when the "call method" is delegated this value
    	// needs to be initialised to that of the caller's caller.
    	CallerAddress common.Address
    	caller        ContractRef            \\转出方地址
    	self          ContractRef             \\转入方地址
    
    	jumpdests destinations // result of JUMPDEST analysis.
    
    	Code     []byte  \\ 指令数组
    	CodeHash common.Hash
    	CodeAddr *common.Address
    	Input    []byte   \\数据数组
    
    	Gas   uint64
    	value *big.Int
    
    	Args []byte
    
    	DelegateCall bool
    }
```
这里会引申出很多问题，首先，有转出转入放地址后，最上面的一个callerAddress是什么的地址呢？从注释来看，应该是初始化合约的地址
那么，如果是调用合约的话，这一步就好理解了，因为需要通过找到合约的地址，执行相关的合约代码，但是如果只是转账操作，这一步又会
如何执行呢？

4. 调用该合约指令（core/vm/evm.go)
```
    ret, err = run(evm, contract, input)
```
在run函数中，首先会判断contract待执行的指令是否是一组预编译的合约集合，否则调用解释器进行执行预编译的合约主要是 SHA-3和一些加密算法
最后，将会调用Interpreter来执行code。我们看看Interpreter的结果：
```
// Interpreter is used to run Ethereum based contracts and will utilise the
// passed evmironment to query external sources for state information.
// The Interpreter will run the byte code VM or JIT VM based on the passed
// configuration.
type Interpreter struct {
	evm      *EVM
	cfg      Config
	gasTable params.GasTable //表示很多操作的Gas价格
	intPool  *intPool

	readOnly   bool   // Whether to throw on stateful modifications
	returnData []byte // Last CALL's return data for subsequent reuse
}

```

Interpreter结构体通过一个**Config类型的成员变量，间接持有一个包括256个operation对象在内的数组JumpTable**。而Interpreter的Run()函数就很好理解了，其核心流程就是逐个byte遍历入参Contract对象的Code变量，将其解释为一个已知的operation，然后依次调用该operation对象中的函数，
我们看看Operation的数据结构：
```
type operation struct {
	// op is the operation function
	execute executionFunc
	// gasCost is the gas function and returns the gas required for execution
	gasCost gasFunc
	// validateStack validates the stack (size) for the operation
	validateStack stackValidationFunc
	// memorySize returns the memory size required for the operation
	memorySize memorySizeFunc

	halts   bool // indicates whether the operation should halt further execution
	jumps   bool // indicates whether the program counter should not increment
	writes  bool // determines whether this a state modifying operation
	valid   bool // indication whether the retrieved operation is valid and known
	reverts bool // determines whether the operation reverts state (implicitly halts)
	returns bool // determines whether the operations sets the return data content
}
```
核心内容是4个函数操作，分别对应执行、gas消耗，堆栈检查和内存信息，而已定义的operation，种类很丰富，包括算术、逻辑、业务（SHA3）等操作
 
 相较而言，evm.create()方法中会常见新的地址作为转入方地址，对应的就是新合约地址。
call 函数中 在进入evm.call()之前就已经完成了nonce+=1，而create()会在方法中完成