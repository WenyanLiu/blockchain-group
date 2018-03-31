# 虚拟机（EVM）的基本数据结构
根据以太坊黄皮书的介绍，交易执行是以太坊协议中最复杂的部分，因为它的核心在于定义了状态转换函数。根据我们之前的学习中，我们
追踪命令行调用交易的命令，从交易的基本数据结构，到交易创建、入池验证、打包验证，到最后创建虚拟机执行交易，还原了整个虚拟机
外交易的执行流程。从本次学习开始，我们深入到以太坊虚拟机内部，从虚拟机的基本结构开始，继续追踪交易与合约的执行。

首先，在go语言代码中，虚拟机的基本结构如下：

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

在以太坊的黄皮书中，定义的EVM是一个虚拟状态机，具有基于堆栈的架构，我们先从结构体来进行理解。先来看ApplyTransaction()中，新建虚拟机的代码：

    // Create a new context to be used in the EVM environment
    context := NewEVMContext(msg, header, bc, author)
    // Create a new environment which holds all relevant information
    // about the transaction and calling mechanisms.
    vmenv := vm.NewEVM(context, statedb, config, cfg)

首先是需要新建Context结构，其对应的结构体如下：

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

可以看出，该结构体携带有交易的信息(GasPrice, GasLimit)，Block的信息(Number, Difficulty)，以及转帐函数等，提供给EVM；

StateDB 接口是针对
state.StateDB 结构体设计的本地行为接口，可为EVM提供statedb的相关操作； Interpreter结构体作为解释器，用来解释执行EVM中合约
(Contract)的指令(Code)。

虚拟机中有30个文件