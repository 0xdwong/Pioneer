# 二层网络上的以太坊智能合约: **Optimistic Rollup**





这篇文章概述了optimistic rollup：一种使用[OVM](https://medium.com/plasma-group/introducing-the-ovm-db253287af50)在二层网网络上启用智能合约的结构。 该构造大量借鉴了plasma和zkRollup设计，并以Vitalik所描述的 [shadow chains](https://blog.ethereum.org/2014/09/17/scalability-part-1-building-top/) 为基础 。 **此结构类似于[Plasma](https://learnblockchain.cn/tags/Plasma)，但放弃了一些扩展性，以便在二层网络中运行完全通用的智能合约（例如Solidity），同时还享有和一层网络相同的安全性**。optimistic rollup的可扩展性与一层网络数据可用带宽成正比，一层网络可以包括Eth1，Eth2， 甚至[Bitcoin现金或以太坊经典](https://ethresear.ch/t/bitcoin-cash-a-short-term-data-availability-layer-for-ethereum/5735)，optimistic rollup都可以在二层网络上提供类EVM的链。

> 备注：下文中二层网络将使用简写 L2 ，相应的以太坊主网（或其他网络）称为 L1



![1_qk6yWTozTxMfZXZILtvpAQ](https://img.learnblockchain.cn/pics/20200727225147.png)



# 快速概述

让我们先从一些直觉开始，了解如何在以太坊主网上进行 optimistic rollup，然后再深入研究。



以下是 optimistic rollup智能合约（名为Fred）的生命历程：

1. 开发人员编写了一个名为Fred的Solidity合约。

2. 开发人员将交易在链下发送到绑定的**聚合商(aggregator)**（L2的区块生产者），聚合商负责部署合约。
   — 任何支付了保证金的人都可以称为聚合商(aggregator)。
   — 同一条链上有多个聚合商。
   — 可以支付费用，但聚合商需要（帐户抽象/[元交易](https://learnblockchain.cn/article/584)）。
   — 开发人员可以即时确认交易，否则聚合商将损失保证金。

3. 聚合商在本地处理交易并计算新的状态根。

4. 聚合商提交以太坊交易(支付gas费用)，其中包含交易和状态根（一个optimistic rollup区块）。

5. 如果**任何人**下载了该块并发现其无效，则可以使用`verify_state_transition(prev_state, block, witness)`来证明其无效， 将：

   — 删除恶意聚合的区块以及在无效块之上构建的任何聚合区块。
   — 用聚合商的部分保证金奖励证明者。

6. Fred 知道自己的部署交易现在已成为每个未来有效的optimistic rollup状态的一部分，因此有安全保障。可以将Fred的主网ERC20发送到L2中！ 好极了！

   

仅此而已！ 用户和智能合约的行为应该与我们今天在以太坊主网上看到的非常相似，只是扩容了！ 现在，让我们探讨一下这整个过程的可能性。



# 深入Optimistic Rollup

首先，让我们定义创建像以太坊这样的无许可智能合约平台的含义。 构建这些可爱的状态机之必须满足三个属性：

1. **可用的（availabe）总体状态** — 任何相关方都可以下载当前的总体状态。
2. **有效的（valid）总体状态** — 总体状态是有效的 (例如：没有无效的状态转换)。
3. **活跃的（live）总体状态** — 任何感兴趣的一方都可以提交转换总体状态的交易。

>  译者注：总体状态对应的原文是 head state， 直译或许是头部状态， 不过中文里貌似没有这个说法。



您会注意到以太坊L1满足了这三个属性，因为我们相信：

1. 矿工不会在不可用的区块上进行挖矿;

2. 矿工不会在无效的区块上进行挖矿[*](https://eprint.iacr.org/2015/702.pdf)；

3. 并非所有矿工都会审查交易。

   

但是，以太坊目前无法扩容。另一方面，在一些类似的安全假设下，optimistic rollup 可以大规模扩容下，提供这三个保证。 为了了解构造和安全性的假设，将逐一检查我们希望分别确保的每个属性。



# #1: 可用的总体状态



Optimistic rollup 使用经典 rollup技术（[此处概述](https://ethresear.ch/t/on-chain-scaling-to-potentially-500-tx-sec-through-mass-tx-validation/3477)）来确保当前状态数据的可用性。该技术很简单-区块生产者（称为聚合商）通过以太坊主网上的calldata（调用以太坊函数的输入参数）传递所有区块的交易和状态根。calldata数据将默克尔化来并存储一个32字节的状态根。提示：calldata为每32字节2,000 gas，而存储成本为20,000 gas。此外，在[伊斯坦布尔硬分叉](https://learnblockchain.cn/2019/11/21/istanbul-update)中，calldata的gas成本降低了近5倍。

值得注意的是，我们也可以使用除以太坊主网之外的其他网络作为L1提供数据可用性，包括 [Bitcoin Cash](https://ethresear.ch/t/bitcoin-cash-a-short-term-data-availability-layer-for-ethereum/5735) 和[Eth2]([https://learnblockchain.cn/tags/%E4%BB%A5%E5%A4%AA%E5%9D%8A2.0](https://learnblockchain.cn/tags/以太坊2.0))。在Eth2阶段1中，所有分片都可以作为L1提供数据可用性，以分片数量线性地扩容TPS。这样的吞吐量是足够的，直到遇到其他可扩展性瓶颈（例如状态计算）。



## 安全假设

在这里，我们假设以太坊主网上多数是诚实的。 另外，如果我们使用Eth2、ETC或Bitcoin Cash，我们也将类似假设。



> 在这些假设下，使用可信的L1来发布所有交易，我们可以确保任何人都可以计算当前的总体状态，从而满足属性1。

# #2: 有效的总体状态

我们需要确保的第二个属性是有效的总体状态。 在zkRollup中，使用零知识证明来确保有效性。 从长远来看，这是一个不错的解决方案，但目前无法为任意状态转换创建有效的zkProofs。 但是，仍然有希望使用通用的EVM型状态机！ 我们可以使用plasma/[truebit](https://learnblockchain.cn/tags/TrueBit)类似的加密经济学进行有效性博弈。

## 密码经济学有效性博弈

从较高层次来看，区块提交和有效性博弈规则如下：

1. 聚合商提交保证金才能开始生产区块。
2. 每个区块包含 `[access_list, transactions, post_state_root]`.
3. 有保证金的聚合商将按照“先到先得”的原则（或根据需要使用轮流方式）将所有区块提交到`ROLLUP_CHAIN`合约。
4. **任何人**都可能证明该区块无效，**并赢得了聚合商的部分保证金**。



要证明一个块无效，必须证明以下三个属性之一：


1. INVALID_BLOCK: 提交的块是无效。
   这是通过`is_valid_transition(prev_state, block, witness) => boolean`计算的。
2. SKIPPED_VALID_BLOCK: 提交的块“忽略”了一个有效块。
3. INVALID_PARENT: 提交的块的父块是无效的。



这三个状态转换有效性条件如图所示：

![1_cv_RR7vxY0BW89QKr7mRsA](https://img.learnblockchain.cn/pics/20200727225247.png)



该状态有效性博弈具有一些有趣的属性：



1. **可插拔的有效性检查器**: 我们可以为`is_valid_transition(…)`定义不同的有效性检查器，从而允许我们使用不同的VM（包括EVM和WASM在内）来运行智能合约。
2. **仅有一条有效链**: 提交到以太坊的区块，可以确定交易和区块的总顺序。 这使我们能够确定性地确定“头(head)”块，从而要求聚合商在提交新块之前删减无效的块。
3. **部分验证**: 此有效性博弈可以以单个UTXO为基础进行。方案使用部分无效取代全部区块整体无效 -- 这类似于Plasma Cash。 请注意，并不需要证明一个块的所有无效转换。 部分区块无效验证意味着我们可以只验证我们关心的状态， 以确保安全。 要了解有关UTXO如何实现并行性请查看[Cryptoeconomics.study视频](https://www.youtube.com/watch?v=-xoCoZGJ9AQ)！

## 有关瞭望塔

>  译者注：瞭望塔是一个帮助用户监视交易欺诈的实体，因为普通用户很难实时在线，瞭望塔则可以充当用户的代理。



采用L2的挑战之一是[瞭望塔](https://blockonomi.com/watchtowers-bitcoin-lightning-network/)的复杂性。 与用户签约的瞭望塔，是另一个加入的来管理已经复杂的系统实体。 幸运的是，optimistic rollup 加密经济有效性博弈自然地激励了瞭望塔！ 所有数据都是可用的，因此任何人都可以运行全节点验证无效交易以获得“所有”构建无效链的聚合商的保证金。 这种风险激励将促使聚合商成为瞭望塔，从而验证他们建立的链，从而减轻了验证者的困境。



## 有关Plasma

许多plasma构造也依赖于加密经济学的有效性博弈。 但是，在没有zkProofs或数据劫持攻击的[渔夫博弈](https://github.com/ethereum/research/wiki/A-note-on-data-availability-and-erasure-coding#what-is-the-data-availability-problem)（数据可用性问题）情况下，在plasma中执行合约自治是不可能的。 值得庆幸的是，rollup 通过发布计算链上状态转换所需的最少信息来解决数据可用性问题。 不过，如果我们想扩容到每秒数十万（甚至更多）的交易 plasma 仍然是至关重要，但是这是远期的需要，但对于许多智能合约而言，从中期来看并不是必需的。



## 安全假设

1. 加密经济学有效性博弈是基于**诚实甚至理性的验证者**假设下工作的。 我们可以说他是“理性”的验证者，而不是“诚实”的验证者，因为挑战博弈可能会从经济上激励他们。

2. 另外，我们假设主网是**活跃的**，这意味着它不会审查所有试图证明无效的交易（译者注：**审查意味着干预**，这里的意思是指：证明无效的交易并不会选择性的干预，例如如丢弃该交易）。 请注意，从某种意义上说，聚合商的解除保证金退出期限是主网上的活跃性假设（例如，如果我们需要1个月的保证金退出期限，需要在该月内证明提交无效性证明，才能罚没保证金）。

   

> 在这些假设下，所有无效块或状态转换都将被丢弃，从而使我们获得一个有效总体状态，并满足属性 #2。

# #3: 活跃的总体状态

我们必须满足的最终属性是活跃，通常被称为抗审查能力。 确保这一点的关键是：



1. 保证金大小超过`MINIMUM_BOND_SIZE`的任何人都可能成为同一rollup链的聚合商。

2. 由于诚实的聚合商可能会移除无效的块，因此即便产生了无效区块，链也**不会停止**。

   

有了这两个属性，我们就可以保持活跃！诚实的聚合商可能总是避免提交无效区块，因此，即使只有一个非审查性聚合商，你的交易最终也会得到确定，这与主网类似。



## 有关活跃与即时确认



我们真正想要的一个属性是即时确认。 这样就可以为用户提供亚秒级的反馈，表明他们的交易将被处理。 我们可以通过在块上指定聚合器短时的垄断出块来实现这一目标。 缺点是，它降低了抗审查能力，因为现在一个参与方一段时间内具备了审查。 很想听听有关此权衡的任何研究！



## 安全假设

保持活跃基于两个安全性假设：

1. 存在一个非审查聚合商。
2. 以太坊主网是非审查的

> 在这些假设下，optimistic rollup 链将能够基于任何有效的用户交易来进化和变更总体状态 （满足属性#3）。



现在，这三个属性均已满足，并且我们在以太坊L2中提供了一个无许可的智能合约平台！



# 可扩展性指标

以下估算是**完全基于数据可用性 **。 实际上，可能会遇到其他瓶颈，其中一个是状态计算。 但是，这确实提供了可用的上限。



## 在ETH1数据可用性下的ERC20 转账

计算基于 [一个简单的 call-data 计算 python 脚本](https://gist.github.com/karlfloersch/1bf6ab7871f41e3a5a921c0a007ad5c6).



请注意，这些 ERC20 转账对calldata 进行了优化。 另外请注意，Optimistic Rollup 的好处是我们不仅限于ERC20传输！



**ECDSA 签名**
~100 TPS： 未采用 EIP 2028 时
~450 TPS： 采用 EIP 2028 后 ( 2019年 10 月已经很到主网)

**BLS 签名 / SNARK-ed 签名**
~400 TPS ： 未采用 EIP 2028 时
~2000 TPS ： 采用 EIP 2028 后 ( 2019年 10 月已经很到主网)

## 在其他的数据可用性下（ 如 ETH2, Bitcoin Cash)



**与L1可以处理的吞吐量的数量成线性关系**。

超过2000 TPS！



# Optimistic Rollup 与 Plasma

Optimistic Rollup与Plasma有很多共同点。 两者都使用聚合商通过加密经济有效性博弈来确保安全性，从而在主网上提交区块。 唯一的区别是是否具有确保区块可用的可用性收据。



![1_XM9jBBbYE20kFC7PIngipA](https://img.learnblockchain.cn/pics/20200727225426.png)



两种解决方案之间的相似之处使得两者之间可以共享许多基础架构和代码。 在成熟的L2生态系统中，我们很可能会看到rollup，plasma和[状态通道](https://learnblockchain.cn/tags/%E7%8A%B6%E6%80%81%E9%80%9A%E9%81%93)在同一个客户端（智能钱包）中一起工作。 哦，我是否提到过[OVM]（https://medium.com/plasma-group/introducing-the-ovm-db253287af50）？ 😁



# Yay Optimistic Rollup 🎉



Optimistic Rollup会在L2中占据一席之地。 它在通用智能合约平台，简单性，安全性和扩展性之间做了一些权衡。 再加上其能够安全的运行智能合约，意味着它甚至可以用于裁定其他第二层解决方案，例如Plasma和状态通道！



> 就把它称为二代 L2 中的 L1 吧。



无论如何，研究已经够了，是时候实施强大，全面且用户友好的以太坊第2层了！ 😍



---

*特别感谢Vitalik Buterin与我一起解决这些想法并提出了很多建议*。

此外，还要感谢Ben Jones所做的大部分工作以及Jinglan Wang，Kevin Ho 和Jesse Walden 的编辑。

**更新**：John Adler 在他的《合并共识》中的文章与optimistic rollups 进行了对比分析，参考[此处](https://ethresear.ch/t/minimal-viable-merged-consensus/5617),另外，[此提案](https://ethresear.ch/t/multi-threaded-data-availability-on-eth-1/5899)可以提高 ETH1 的数据可用性 - 更多的 TPS！

[Plasma Group Blog]（https://medium.com/plasma-group?source=post_sidebar--------------------------post_sidebar-）



原文: https://medium.com/plasma-group/ethereum-smart-contracts-in-l2-optimistic-rollup-2c1cef2ec537

作者: [Karl Floersch](https://medium.com/@karl_dot_tech?source=post_page-----2c1cef2ec537----------------------)