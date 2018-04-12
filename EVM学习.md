# 以太坊虚拟机（EVM）的基本设计概念
## 虚拟机基本概念
程序虚拟机（process virtual machines or application virtual machines)是通过对底层硬件或者操作系统的抽象，为应用程序的运行
提供一个与平台无关的运行环境，使得程序可以在任何平台上以相同的方式执行。一般的程序虚拟机是为单个进程的运行提供虚拟资源。

通常而言，虚拟机的基础架构为：

![虚拟机架构.png](./img/2018410/虚拟机架构.PNG)

- loader 加载虚拟运行程序，将guest程序的代码和数据写入到内存。
- initialization block 为程序数据结构分配内存空间，建立和OS的通信信号
- emulation engine  使用解释器或者二进制仿真器来模拟Guest操作
- code cach and code cache manager 缓存与管理预编译或者编译好的相关代码 
- profile database 包含动态收集的相关信息用以指导优化执行
- OS call emulator 操作系统调用的仿真器（Translates OS calls from the guest into an appropriate calls to host OS)
- exception emulator 处理可能发生的陷阱或中断

可以看出来虚拟机实现的核心在于仿真器，它的主要作用是将guest程序的操作转变为真正的系统操作。
## 以太坊虚拟机



以太坊虚拟机（EVM）是图灵完备虚拟机器，它的唯一限制就是EVM本质上是被gas束缚，可以完成的计算总量是被提供的gas总量限制的。

EVM具有基于堆栈的架构。堆栈机器 就是使用后进先出来保存临时值的计算机。它的每个堆栈项的大小为256位，堆栈有一个最大的大小，为1024位。

EVM有内存，项目按照可寻址字节数组来存储。内存是易失性的，也就是数据是不持久的。

_EVM也有一个存储器。不像内存，存储器是非易失性的，并作为系统状态的一部分进行维护。EVM分开保存程序代码，在虚拟ROM 中只能通过特殊指令来访问。这样的话，EVM就与典型的冯·诺依曼架构 不同，此架构将程序的代码存储在内存或存储器中。_

![EVM.png](./img/2018410/EVM.png)

EVM同样有属于它自己的语言：“EVM字节码”，当一个程序员比如你或我写一个在以太坊上运行的智能合约时，我们通常都是用高级语言例如Solidity来编写代码。然后我们可以将它编译成EVM可以理解的EVM字节码。

前面我们已经说过，每条指令都会存储对应的处理逻辑、gas消耗，堆栈验证方法以及内存使用大小，我们来详细的看以太坊的一个指令,
```
    ADD: 
    {
            execute:       opAdd,
            gasCost:       constGasFunc(GasFastestStep),
            validateStack: makeStackFunc(2, 1),
            valid:         true,
    },
```
其中，执行的逻辑为：
```
func opAdd(pc *uint64, evm *EVM, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
	x, y := stack.pop(), stack.pop()
	stack.push(math.U256(x.Add(x, y)))

	evm.interpreter.intPool.put(y)

	return nil, nil
```
其他的操作的形式都是如此，但是ADD操作不涉及到内存大小的分配（为什么？），没有对memorySize进行定义 
从这里来理解，我们发现在将栈中数据计算完成后，会将结果值保存到intPool中，这个是在解释器中创建的变量，也是一个256*1024的栈结构


