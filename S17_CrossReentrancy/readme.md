---
title: S17. "跨服"重入攻击
tags:
  - solidity
  - security
  - fallback
  - modifier
  - ERC721
  - ERC777
---

# WTF Solidity 合约安全: S17. "跨服"重入攻击

我最近在重新学 solidity，巩固一下细节，也写一个“WTF Solidity 极简入门”，供小白们使用（编程大佬可以另找教程），每周更新 1-3 讲。

推特：[@0xAA_Science](https://twitter.com/0xAA_Science)

社区：[Discord](https://discord.gg/5akcruXrsk)｜[微信群](https://docs.google.com/forms/d/e/1FAIpQLSe4KGT8Sh6sJ7hedQRuIYirOoZK_85miz3dw7vA1-YjodgJ-A/viewform?usp=sf_link)｜[官网 wtf.academy](https://wtf.academy)

所有代码和教程开源在 github: [github.com/AmazingAng/WTFSolidity](https://github.com/AmazingAng/WTFSolidity)

---

在智能合约安全领域，重入攻击永远是一个备受关注的话题。在[重入攻击](../S01_ReentrancyAttack/readme.md)这一讲中，`0xAA`生动展示了教科书级经典的重入攻击思路；而在生产环境中，常常有一些更加安排巧妙，复杂的实例一直在以各种新瓶装旧酒的面目不断地出现，并且成功地对很多项目造成了破坏。这些实例展示了攻击者如何利用智能合约中的漏洞来搭配组合出精心策划的攻击。这一讲，我们将介绍一些生产环境中真实发生的具有“跨服”属性的重入攻击案例。所谓“跨服”，是对这一类型的攻击目标的生动概括，因为它们共同的手段是从某一个函数开始入手，但是攻击对象却是其他函数/合约/项目等等。在本讲中我会带领大家简化并提炼其操作，探讨攻击者的思路、利用的漏洞以及对应的防御措施。通过了解这些实例，我们可以更好地理解重入攻击的本质，并且提高我们编写安全智能合约的技能和意识。

注：以下所展示的代码示例均为简化过的`pseudo-code`, 主要以阐释攻击思路为目的。内容源自众多`Web3 Security Researchers`所分享的审计案例,感谢他们的贡献！


## 1. 跨函数重入攻击

*“那一年，我戴了重入锁，不知对手为何物。直到那天，那个男人从天而降，还是卷走了我的银钱...” -- 戴锁婆婆*

请看如下代码示例：
```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

contract VulnerableBank {
  mapping(address => uint256) public balances;

  uint256 private _status; // 重入锁

  // 重入锁
  modifier nonReentrant() {
      // 在第一次调用 nonReentrant 时，_status 将是 0
       require(_status == 0, "ReentrancyGuard: reentrant call");
      // 在此之后对 nonReentrant 的任何调用都将失败
      _status = 1;
      _;
      // 调用结束，将 _status 恢复为0
      _status = 0;
  }

  function deposit() external payable {
    require(msg.value > 0, "Deposit amount must ba greater than 0");
    balances[msg.sender] += msg.value;
  }

  function withdraw(uint256 _amount) external nonReentrant {
    uint256 balance = balances[msg.sender];
    require(balance >= _amount, "Insufficient balance");

    (bool success, ) = msg.sender.call{value: _amount}("");
    require(success, "Withdraw failed");

    balances[msg.sender] = balance - _amount;
  }

  function transfer(address _to, uint256 _amount) external {
    uint256 balance = balances[msg.sender];
    require(balance >= _amount, "Insufficient balance");

    balances[msg.sender] -= _amount;
    balances[_to] += _amount;
  }
}
```

在上面的`VulnerableBank`合约中，可以看到转账`ETH`的步骤仅存在于`withdraw`这一个函数之内，而此函数已经使用了重入锁`nonReentrant`。那么，还有什么方法来对这个合约进行重入攻击呢？

请看如下攻击者合约示例：

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "../IVault.sol";

contract Attack2Contract {
    address victim;
    address owner;

    constructor(address _victim, address _owner) {
        victim = _victim;
        owner = _owner;
    }

    function deposit() external payable {
        IVault(victim).deposit{value: msg.value}("");
    }

    function withdraw() external {
        Ivault(victim).withdraw();
    }

    receive() external payable {
        uint256 balance = Ivault(victim).balances[address(this)];
        Ivault(victim).transfer(owner, balance);
    }
}
```

如上所示，攻击者重入的不再是`withdraw`函数，而是转头去重入没有戴锁的`transfer`函数。`VulnerableBank`合约的设计者的固有思路认为`transfer`函数中只是更改 `balances mapping`而没有转账`ETH`的步骤，所以应该不是重入攻击的对象，所以没有给它加上锁。而攻击者利用`withdraw`先将`ETH`转账，转账完成的时候`balances`没有立即更新，而随机调用了`transfer`函数将自己原本已不存在的余额成功转移给了另一个地址`owner`，而此地址完全可以是攻击者的一个小号而已。由于`transfer`函数没有转账`ETH`所以不会持续将执行权交出，所以这个重入只是攻击了额外一次便结束。结果是攻击者“无中生有”出了这一部分钱，实现了“双花”的功效。

那么问题来了：

*如果改进一下， 将合约中的所有跟资产转移沾边的函数都加上重入锁，那是不是就安全了呢？？？*

请看下面的进阶案例...


## 2. 跨合约重入攻击

我们的第二位受害者是一个多合约组合系统，它是一个去中心化合约交易平台，我们只看问题发生的关键处，是跟两个合约有关。第一个合约是`TwoStepSwapManager`, 它是面向用户的合约，里面包含有允许用户直接发起的提交一个swap交易的函数，还有同样是可由用户发起的，用来取消正在等待执行但尚未执行的swap交易的函数；第二个合约是`TwoStepSwapExecutor`, 它是只能由管理的角色来发起的交易，用于执行某个处于等待中的swap交易。这两个合约的 *部分* 示例代码如下：

```
// Contracts to create and manage swap "requests"

contract TwoStepSwapManager {
    struct Swap {
        address user;
        uint256 amount;
        address[] swapPath;
        bool unwrapnativeToken;
    }

    uint256 swapNonce;
    mapping(uint256 => Swap) pendingSwaps;

    uint256 private _status; // 重入锁

    // 重入锁
    modifier nonReentrant() {
      // 在第一次调用 nonReentrant 时，_status 将是 0
        require(_status == 0, "ReentrancyGuard: reentrant call");
      // 在此之后对 nonReentrant 的任何调用都将失败
        _status = 1;
        _;
      // 调用结束，将 _status 恢复为0
        _status = 0;
     }

    function createSwap(uint256 _amount, address[] _swapPath, bool _unwrapnativeToken) external nonReentrant {
        IERC20(swapPath[0]).safeTransferFrom(msg.sender, _amount);
        pendingSwaps[++swapNounce] = Swap({
            user: msg.sender,
            amount: _amount,
            swapPath: _swapPath,
            unwrapNativeToken: _unwrapNativeToken
        });
    }

    function cancelSwap(uint256 _id) external nonReentrant {
        Swap memory swap = pendingSwaps[_id];
        require(swap.user == msg.sender);
        delete pendingSwaps[_id];

        IERC20(swapPath[0]).safeTransfer(swap.user, swap.amount);
    }
}
```

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

// Contract to exeute swaps

contract TwoStepSwapExecutor {


    /* 
        Logic to set prices etc... 
    */


    uint256 private _status; // 重入锁

    // 重入锁
    modifier nonReentrant() {
      // 在第一次调用 nonReentrant 时，_status 将是 0
        require(_status == 0, "ReentrancyGuard: reentrant call");
      // 在此之后对 nonReentrant 的任何调用都将失败
        _status = 1;
        _;
      // 调用结束，将 _status 恢复为0
        _status = 0;
    }

    function executeSwap(uint256 _id) external onlySwapExecutor nonReentrant {
        Swap memory swap = ISwapManager(swapManager).pendingSwaps(_id);

        // If a swapPath ends in WETH and unwrapNativeToken == true, send ether to the user
        ISwapManager(swapManager).swap(swap.user, swap.amount, swap.swapPath, swap.unwrapNativeToken);

        ISwapManager(swapManager).delete(pendingSwaps[_id]);
    }
}
```

从上面两个合约的示例代码可以看出，所有相关的函数均使用了重入锁。然而，那个男人还是成功地对戴锁婆婆施展了重入魔法，再再再一次卷走了原本不属于他的钱财。这一次，他又是如何做到的呢？

俗话说得好， *“灯下黑“* ，答案就在最表面上反而容易被忽视 --- 因为这是 两 个 合 约...锁的状态是不互通的！ 管理员调用了`executeSwap`来执行了那个攻击者提交的swap，此合约的重入锁开始生效变成`1`。当运行到中间那步`swap（）`的时候，发起了`ETH`转账，将执行权交给了攻击者的恶意合约的`fallback`函数，在那里被设置了对`TwoStepSwapManager`合约的`cancelSwap`函数的调用，而此时这个合约的重入锁还是`0`，所以`cancelSwap`开始执行，此合约的重入锁开始生效变成`1`，然而为时已晚。。。 攻击者收到了`executeSwap`发送给他的swap过来的`ETH`，同时还收到了`cancelSwap`退给他的当初送出去用来swap的本金代币。他他他又一次“无中生有”了！


### 全局重入锁

若想要防范这种跨合约重入攻击，我这里送同学们一个重入锁的升级版 -- 全局重入锁。适合用于同学们以后架构多合约系统。请看以下简易代码思路：

```
pragma solidity ^0.8.0;

import "../data/Keys.sol";
import "../data/DataStore.sol";

abstract contract GlobalReentrancyGuard{
    uint256 private constant NOT_ENTERED = 0;
    uint256 private constant ENTERED = 1;

    DataStore public immutable dataStore;

    constructor(DataStore _datastore) {
        dataStore = _dataStore;
    }

    modifier globalNonReentrant() {
        _nonReentrantBefore();
        _;
        _nonReentrantAfter();
    }

    function _nonReentrantBefore() private {
        uint256 status = dataStore.getUint(Keys.REENTRANCY_GUARD_STATUS);

        require(status == NOT_ENTERED, "ReentrancyGuard: reentrant call");

        dataStore.setUint(Keys.REENTRANCY_GUARD_STATUS, ENTERED);
    }

    function _nonReentrantAfter() private {
        dataStore.setUint(Keys.REENTRANCY_GUARD_STATUS, NOT_ENTERED);
    }
}
```

一句话概括这个全局重入锁的核心就是，建立一个单独的合约用来储存重入状态，然后，在你的系统里的任何合约里相关的函数在执行的时候，都要来这同一个地方来查看当前的重入状态，这样你的所有合约就都被重入保护起来了。

看似美妙，但还没完... 攻击者还有更新的花招即便是用全局重入锁也无法防范的。接着往下看: ...


## 3. 跨项目重入攻击

越写越大了。。。所谓跨项目的重入攻击，其核心与上面两例其实也是比较类似。本质就是趁某项目合约的某个状态变量在还未来得及更新时，就利用接手的执行权来发起外部函数调用。如果有第三方合作项目的合约是依赖于前面提到的项目合约里这个状态变量的值来做某些决策的，那么攻击者就可以去攻击这个合作项目的合约，因为在此刻它读到的是一个过期的状态值，会导致它执行一些错误的行为令攻击者获利。 通常，合作项目的合约通过一些`getter`函数或其他公开的只读函数的调用来传递信息，所以这类攻击也通常体现为`只读重入攻击 Read-Only Reentrancy`。

请看如下示例代码：

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

contract VulnerableBank {
  mapping(address => uint256) public balances;

  uint256 private _status; // 重入锁

  // 重入锁
  modifier nonReentrant() {
      // 在第一次调用 nonReentrant 时，_status 将是 0
       require(_status == 0, "ReentrancyGuard: reentrant call");
      // 在此之后对 nonReentrant 的任何调用都将失败
      _status = 1;
      _;
      // 调用结束，将 _status 恢复为0
      _status = 0;
  }

  function deposit() external payable {
    require(msg.value > 0, "Deposit amount must ba greater than 0");
    balances[msg.sender] += msg.value;
  }

  function withdraw(uint256 _amount) external nonReentrant {
    require(_amount > 0, "Withdrawal amount must be greater than 0");
    require(isAllowedToWithdraw(msg.sender, _amount), "Insufficient balance");

    (bool success, ) = msg.sender.call{value: _amount}("");
    require(success, "Withdraw failed");

    balances[msg.sender] -= _amount;
  }

  function isAllowedToWithdraw(address _user, uint256 _amount) public view returns(bool) {
    return balances[_user] >= _amount;
  }
}
```

如代码所示，在这个合约中，已经没有攻击者发挥重入的空间了。然而，这里没有，不代表别处没有。。。 我们可以看到合约里有一个公开的只读函数`isAllowedToWithdraw`，这类函数就是用来以提供信息为目的的。很多项目的合约里都或多或少有一些这类函数，而这类函数又常被其他项目的合约来调用获取信息，最终完成Defi世界里的一个乐高积木。可以看到这个重要的`withdraw`函数已经被上了锁，不可以重入攻击，但是在他的执行过程中的`ETH`转账那一步，`ETH`刚刚转出，假设攻击者想要此刻调用`isAllowedToWithdraw`函数，可以预见即便是`_amount`数值很大，攻击者的存款实际已被掏空，但返回值仍然是`true`因为账本在此刻还没有更新。那么，攻击者就可以在他的恶意合约里的`fallback`函数中设置外部函数调用,去攻击他已知的其他项目的依据`isAllowedToWithdraw`函数返回结果来制定操作的那些合约。

上面这个合约本身不遭攻击，而合作伙伴的合约遭到攻击。。。典型的：

*“我不杀伯仁，伯仁却因我而死...” -- 戴锁婆婆*


## 4. ERC721 & ERC777 Reentrancy

这两种代币标准都各自规定了一个回调函数：

ERC721: `function onERC721Received(address _operator, address _from, uint256 _tokenId, bytes memory _data) public returns(bytes4);`

ERC777: `function tokensReceived(address _operator, address _from, address _to, uint256 _amount, bytes calldata _userData, bytes calldata _operatorData) external;`

有回调函数的存在就有接手代码执行权的机会，同时也会营造出重入攻击的可能性。对这一情况就不展示代码示例了，因为结合上述几个案例，这一条现在应该很容易理解了。并且，实在是能够玩出无穷花样。


## 总结

至此，我们审阅了几个实际发生的，各种花样的重入攻击的逻辑本体和它们的简易代码，相信各位同学应该不难看出，这些合约被攻击，是由于它们都共有一个缺陷。那就是这些合约的设计对于重入攻击的防范，都太过于依赖一个直截了当的工具（重入锁）的保护，而没有贯彻另一条良好的设计习惯 *检查-影响-交互模式* 。 简单的工具永远不会是完美防御，贯彻的方法论才是你永远的后盾 *（报告首长，本节代码课的思政任务已传达，请验收）*

所以，对于使用小工具，还是使用方法论的取舍，我们作为solidity devs，答案我想应该是：既要...又要...！从跨函数的攻击，再到跨合约，跨项目的攻击，若是要求devs和auditors记住这越来越庞大的乐高之间的千丝万缕的联系，实在是有些强人所难。于是，在构造过程中的每一步，都标准地布置多道不同防御机制，便能省心地获得更好的结果。

![](./img/S17-1.png)