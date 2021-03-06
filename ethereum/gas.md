# 以太坊中Gas的设计哲学	

## 前言

以太坊是一个去中心化的、可信任的全球化区块链平台，基于以太坊开发的应用可以天然地拥有**无故障时间**、**应用强信任**、**数据不可更改**等特点。

开发者通过编写部署在以太坊上的智能合约来描述必要的应用逻辑，使得以太坊能够作为一个**世界级的计算机**对外提供服务。

以太坊这个**计算机**每次状态的变更是通过执行一笔交易完成的，换一句话说，以太坊的一笔交易可以视作是对这个计算机进行操作的一个**会话（session）**，若一笔交易执行成功，则在该会话周期内所有的状态改动都会被执行；反之所有的状态改动都会被回滚。从数据库的视角来看，以太坊拥有**原子性**和**一致性**的特点。

在一笔交易中包含了**经过用户签名**的对以太坊进行状态变更的所有所需信息。一笔交易大致包含以下内容：

1. 交易的发起者；
2. 交易发起者利用私钥文件对本笔交易的签名，网络中其余的参与者可以通过这个签名数据确保这笔交易的字段**未被恶意攻击者篡改**；
3. 转账的金额；
4. 对智能合约的**调用信息**；
5. 这笔交易被赋予的**gas数量**，被赋予的gas数量决定了在执行本笔交易这个会话周期内被允许使用的**系统资源的上限**；
6. 这笔交易被指定的**gas价格**；

其中，交易中的合约调用信息指明了需要调用的合约函数以及调用参数等信息，以此方式实现（1）合约代码部署（2）合约代码调用等功能。

那么，以太坊作为一个对公众开放的，全球性的**计算机**，势必需要考虑到维护这个计算机安全性的一些机制。众所周知，区块链上的智能合约有以下两种性质：（1）**确定性**及（2）**可终止性**。区块链是一个通过多方存储、多方计算的方式来实现数据不可篡改、计算结果可信的分布式系统。智能合约会在区块链网络的多个节点中运行。如果一个智能合约是非确定性的，那么不同节点运行的结果就可能不一致，从而导致共识无法达成，网络陷入停滞；如果智能合约是永不停止的，那么节点将耗费无穷多的时间和资源去运行合约，同样导致网络进入停滞状态。

而**本文着重介绍的Gas机制就为以太坊提供了虚拟机可终止性的重要特征**，接下来也将从Gas的定义、作用等方面进行展开。

## Gas的定义

**Gas是用来计量以太坊系统资源使用情况的最小计量单位**。

以太坊作为一个对全球开放的区块链平台，任何人都有权利去部署属于自己的应用，抑或是去使用基于区块链的应用程序。然而这些应用程序都会产生一些属于自己的数据，故而需要使用区块链上的存储资源。其次这些应用程序的执行是需要消耗计算资源。综合来说，以太坊需要消耗系统资源来支撑应用程序的正确运行。

而这些系统资源是有限且非常宝贵的，因此用户在使用过程中必须**支付一定量的使用费来购买这些系统资源**，这个使用费也就是我们所熟知的手续费。

以太坊为这些系统资源制定了一个**定价表**，将系统资源量化成gas的数量，例如执行一个`ADD`操作对应3个Gas，一个`SStore`操作对应20000个Gas。因此从这个角度来看，gas是计量以太坊系统资源的最小单位。

**Gas是以太坊虚拟机实现可终止性的一种机制**。

以太坊是一个全冗余的系统，即整个网络有n个节点，那么同样一笔交易会被所有的节点都执行一遍。同时为了确保不同的节点执行同样一笔交易的结果是严格一致的，以太坊所有的交易都串行执行。

假设恶意攻击者部署了一个恶意合约，内含一个死循环的合约函数，那么恶意攻击者可以通过调用该死循环函数轻易地使整个网络都陷入不可用的状态。

为了解决这个极其严重的安全漏洞，以太坊规定用户在发起交易时，必须指定这一笔交易能够使用的Gas的上限数量。

倘若某节点在执行这笔交易时，所使用系统资源而累积消耗的Gas数量超过了用户指定的Gas上限时，便会抛出一个`Out of Gas`的错误信息，表明这笔交易执行失败，在此会话期间所有的状态改动都会被回滚。

而这个Gas的上限数量也不是可以随便指定的，**用户是需要用自己的以太币进行预付的**。

交易执行之前，矿工会首先检查发起者账户中的余额是否足够来购买指定的Gas，若余额不足直接拒绝这笔交易；交易运行过程中，每使用一种系统资源，便会扣除一定量的Gas；最终执行结束，剩余的Gas换算成对应的以太币返还给用户，而这部分消耗的Gas会换算成以太币奖励给矿工，作为其经济收入的一部分。

综合起来，用户需要支付一定量的以太币，来预购一定量的Gas供合约调用使用。因此面对这种恶意的不可终止的攻击手段，攻击者可能需要付出极高的经济代价，**所以通过这种经济制裁的手段，以太坊的虚拟机拥有可终止的基本特征**。

### Gas与Fee

人们经常容易混淆gas和fee两个完全不同的概念，故在这里做一下简单的解释。

在介绍gas与fee之前我们先简述一下在加油站加油的场景：通常你可能会要求工作人员帮你加10L的汽油，不同的汽油种类有不同的价格，例如89#汽油的价格是6.67元/升，92#汽油的价格是7.13元/升。最终你选择加10升89号汽油，付给工作人员66.7元。

而以太坊中的Gas可以类比于这里的升（从单元大小的角度来看，可能毫升更为合适），那么每Gas需要有一个单价GasPrice，例如这里89#的汽油单价为6.67元／升，通过这个单价因子才能将gas转换成对应的以太币数量。

最终用户的经济支出为 `Fee(wei) = Gas * GasPrice（1 Ether = 10 ** 18 wei）`。

## 区块的GasLimit

### 定义

除了交易体内有GasLimit字段，区块中同样也有属于区块的GasLimit。众所周知，公有链的处理能力是有限的，为了保护系统资源不被过度利用，必须要有一种机制来确保一个区块内能包含的交易数量是有上限的。

由于以太坊不同交易的处理复杂度差异极大，例如最简单的转账交易与复杂的合约调用复杂度相差几个数量级，因此仅仅通过交易数量来限定是远远不够的。

**故在以太坊中，使用GasLimit来限制一个区块中能包含的交易上限，使得能够以资源使用的精度来进行流量控制**。

例如现有三笔交易，其指定的Gaslimit分别为100，200，300。现区块的gas limit为300，那么矿工可以选择将交易1，2打包进入区块，也可以选择选择交易3打包进入区块。当尝试打包更多的交易时，便会得到一个`Transaction exceeds block gas limit`的错误。

### GasLimit的改动

随着区块链技术及生态社区的发展，为了适应系统不同的处理能力，区块的GasLimit必然是会进行变化。以太坊把这个GasLimit改动的权利交由**矿工投票**进行决定，让社区动态地将Gaslimit调整到最合适的值。

具体的**投票方式**是：每个矿工可以通过`targetGasLimit`命令行参数来设置理想的目标GasLimit。

本地矿工节点在新一轮的PoW难题竞赛开始之初，可以根据上一个区块的GasLimit进行动态地微调，调整的范围在正负`1/1024* ParentBlock.GasLimit`的范围内。

由于每一轮出块的矿工以及其GasLimit调整策略都不同，故而可以引申为是由社区通过投票的方式动态地来调整这个GasLimit。

### 经济博弈

一个简单的想法是，矿工可以增大Gaslimit使得区块内包含的交易数量变多，故而其直接的手续费收入会增多。但是Gaslimit的增加会引发两个问题：

1. 交易处理时间变长，导致PoW获胜的概率降低；
2. 区块体积变大，区块传播速度变慢，成为叔区块的概率变高；

以上两种情况都会导致矿工的经济收益下降。

因此，矿工必须选择一个gaslimit的平衡点使得经济收益达到最佳。

## Gas的作用

### 可终止性

前面章节已有详细描述，不再赘述。

### 经济激励

gas除了可以作为计量单元、可终止性保障，更可以视作一种经济激励，引入经济学的原理来使以太坊网络更加健壮。

一般来说，网络中的矿工节点的经济收入来源于两部分：（1）解决PoW难题的固定奖励（2）当前区块中所有交易的手续费。

![](./pic/txflow.jpeg)

如上图所示，用户通过钱包应用签发一笔交易，最终这些交易会汇聚到矿工节点的交易池中等待打包。

由于执行这些交易需要一定的时间开销（例如当前以太坊网络中执行一个区块中所有的交易需要消耗大约200ms的时间），倘若没有经济激励，矿工可以选择**不打包任何的交易**，直接对空的区块进行PoW难题计算赚取那固定的经济奖励。

但是一旦引入了手续费奖励机制，矿工就需要在PoW解体效率与解题的经济回报之间做一个平衡，使得最终的经济收益达到最高，故而促使矿工**尽可能多地打包待处理的交易**，维护了网络的健壮性。

### 交易排序

由于以太坊对公众开放，因此任意用户都可以发起交易，然而本身系统的处理能力是有上限的，因此对于待处理的交易进行一个优先级排序就显得尤为重要。

矿工节点采用了基于GasPrice的排序策略，也就是说在保证**同一个账户交易执行顺序**的前提下，尽可能打包GasPrice高的交易，使得最终自身的经济收益达到最高。

最终以太坊网络中形成一个良性的竞争环境，若想要获得更高的交易处理优先级，就需要付出更多的手续费；反之则需要等到更长的处理时间。

### 流量控制

前面章节已有详细描述，不再赘述。

## 总结

以太坊中的Gas机制有着一举多得的作用，既可以保障最基本的虚拟机可终止性的特点；作为经济激励引导矿工尽可能多地打包交易，提高整体地打包效率；作为度量标准来控制交易处理的速率；更是衍射出了许多经济规则，使得矿工及社区能够良性地竞争，动态有效地改变网络现状。作为以太坊的一个核心机制，在整个系统中扮演了异常重要的作用！

## 参考文献

* [以太坊Gas一览表](https://ethereum.stackexchange.com/q/266/42)
* [https://hudsonjameson.com/2017-06-27-accounts-transactions-gas-ethereum/](https://hudsonjameson.com/2017-06-27-accounts-transactions-gas-ethereum/)
* [https://media.consensys.net/ethereum-gas-fuel-and-fees-3333e17fe1dc?token=I2yNbGS-arJMg6dL](https://media.consensys.net/ethereum-gas-fuel-and-fees-3333e17fe1dc?token=I2yNbGS-arJMg6dL)



