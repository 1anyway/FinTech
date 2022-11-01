
项目简介
起源——copy起家，吸星大法
SuShiSwap是早期Uniswap上的一个分支，最初完全copy的Uniswap代码，通过提供流动性挖矿产生SUSHI将Uniswap锁仓资金吸走起家。
发展——从copy到独立
SuShiSwap通过吸走相当一部分Uniswap的流动性发展起来后，又经历过创始人跑路等系列危机，最终走出了自己的道路。相较于Uniswap，SuShiSwap提供了更广泛的DeFi应用，比如借贷、将流动性挖矿产生的SUSHI用于社区治理等。

项目组成
从SuShiSwap的功能和提供的服务来看，SuShiSwap将其自身分成了四个部分
SuShiSwap Farms
通过从pools获得的SLP token来获得每个新区块的Farm Sushi奖励，在发行的前两周，奖励为每块1000SUSHI，除以池子的比例份额。两周后，奖励设为100Sushi.“Sushi Party”pool例外，有2倍的额外奖励。
如果将SLP发在了farm，那么exchange将不可见。但你仍提供了流动性，也仍将于提取时获得0.25%LP交易费奖励。

SushiSwap Pools
 流动性由LP(流动性提供者)质押的token提供，LP将收到SLP token，代表池中的资产比例份额，用户可随时收回资金。每次用户在SUSHI和ETH之间交易时，收取0.3%的费用，该交易的0.25%返回LP池.
 
SushiSwap Exchange:
允许用户通过自动流动性池将任何ERC20代币交换为任何其他ERC20代币。提供给交易所的流动性来自将其代币投入“ Pools”的流动性提供者（“ LPs”）。 作为回报，他们获得了SLP（Sushiswap流动性提供者）令牌，这些令牌也可以放到“农场”中。 用户在交易所进行交易时，他们需要支付0.3％的交易费。这笔交易费的0.25％会提供给为该集合提供流动性的流动性提供者。 它已添加到池余额中。
为xSushi持有者提供0.05％的费用奖励：
所支付的交易费的剩余部分进入称为SushiBar的池。 SushiBar合同从所有池中收取费用，并在调用奖励分配命令时，然后出售所有费用以将其转化为Sushi（通过SushiSwap）
新的Sushi会在xSushi池中的用户之间分配。 当这些用户撤回其xSushi时，它将比从分发中投入的价值更高。
 
SushiSwap Staking SushiBar(xSushi):
质押Sushi获得xSushi，已质押的xSushi将获得所有交易的%0.05的奖励费。
SushBar使您可以放样Sushi，并获得xSushi作为回报，然后将其放到xSushi池中。
 延伸
Sushi的用途
在SushiSwap Farms上进行流动性挖矿，用于质押来获得交易费提成奖励，用于治理功能的投票，添加为SushiSwap Exchange池上的LP。
投票治理
投票目前在 snapshot.page 管理，基于 SushiPowah(一项投票指标，已锁定在SushiSwap Farms中的每一个SUSHI-ETH SLP token可获得一票)。
总结
SuShipSwap中MasterChef部分的实现非常的经典，许多DeFi项目都从中得到启发并加以借鉴。学习掌握SuShiSwap将对后续DeFi项目的探究、源码的学习大有裨益。


