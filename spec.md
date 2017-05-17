# Extension Blocks 扩展区块

```
层级: Consensus (soft-fork) 共识层（软分叉）
标题: Extension Blocks 扩展区块
作者: Christopher Jeffrey <chjj@purse.io>
        Joseph Poon <joseph@lightning.network>
        Fedor Indutny <fedor@indutny.com>
        Stephen Pair <stephen@bitpay.com>
状态: Draft 草案
起草于: 2017-03-17
许可: Public Domain 公有领域
```

译文摘自巴比特：<http://www.8btc.com/extension-blocks>，有改动。

## Abstract 摘要

This specification defines a method of increasing bitcoin transaction
throughput without altering any existing consensus rules.

本规范定义了一种可增加比特币的交易吞吐量同时又不改变现行共识规则的方法。

## Motivation 目的

The bitcoin network's transaction throughput is correlated with its consensus
rules regarding retargetting and denial-of-service limits.

比特币的交易吞吐量与它的共识规则有关，这一规则涉及难度调整策略（retargetting）和DoS（denial-of-service，拒绝服务攻击）限制策略。
（译者注：根据下面那段的描述，这里说的retargetting应该是指：每隔2016个区块，比特币网络自动进行挖矿难度调整。）

Bitcoin retargetting ensures that the time in between mined blocks will be
roughly 10 minutes. It is not possible to change this rule. There has been
debate regarding other ways of greatly increasing transaction throughput, with
no proposed consensus-layer solutions that have proven themselves to be
particularly safe.

比特币的难度调整策略（retargetting）确保了挖出两个区块之间的时间间隔大约为十分钟，改变这规则是不可能的。已经有多种显著提高交易吞吐量的方法，但这些方法却没有提出共识层解决方案来证明自己足够安全。

## History 历史

_Auxiliary blocks_, as first proposed by [Johnson Lau in 2013][aux], outlined a
way of having funds enter and exit an additional block by using special
opcodes.

**附加区块**，在2013年首次由Johnson Lau提出，该方案提出了通过特殊操作码让资金进入和退出附加区块的方法。

_Extension blocks_ were a later proposal, also [from Lau in 2017][ext], which
revised many of the ideas of the original proposal.

**扩展区块**是另一个改进提案，在2017年同样由Johnson Lau提出，新方案改变了原来方案的很多想法。

## Specification 规范

Extension blocks devise a second layer on top of canonical bitcoin blocks in
which a miner will commit to the merkle root of an additional block of
transactions.

扩展区块，是在比特币主区块之上构建了一个亚层，矿工将这个附加区块里的交易合并到主区块的梅克尔树根（merkle root）。

Extension blocks leverage several features of BIP141, BIP143, and BIP144 for
transaction opt-in, serialization, verification, and network services. This
specification should be considered an extension and modification to these BIPs.
Extension blocks are _not_ compatible with BIP141 in its current form, and will
require a few minor additional rules.

扩展区块方案使用了若干来自BIP141、BIP143和BIP144的特征来实现交易的选入（opt-in）、序列化（serialization）、验证（verification）和网络服务（network services）。扩展区块方案应该视为对这几个改进协议（BIPs）的扩展和修改。扩展区块目前的版本和BIP141不兼容，需要附加少量额外的规则。

Extension blocks maintain their own UTXO set in order to avoid interference
with the existing UTXO set of non-upgraded nodes. This is a tricky endeavor,
requiring a `resolution` transaction to be present at the end of every
canonical block.

为了避免和未升级的节点的UTXO集相冲突，扩展区块使用自己独立的UTXO集。这是非常巧妙的努力，需要在每个主区块的最末端加入一笔解析交易（resolution transaction）。

This specification prescribes a way of fooling non-upgraded nodes into
believing the existing UTXO set is still behaving as they would expect.
Unfortunately, this requires a bit of extra book-keeping on the upgraded node's
side.

这份规范提供了一种方法，来欺骗未升级的节点相信这些存在的UTXO集依然遵行它们期望的规则。不幸的是，这需要在升级的节点这一侧，增加额外的记账（book-keeping）。

### Commitment 承诺

An upgraded miner willing to include extension block is to include an extra
coinbase output of 0 value. The output script exists as such:

一个升级过的矿工如果想打包扩展区块，它会在自己打包的主区块中包含一个额外的coinbase输出，这个输出发送0个比特币，其脚本如下：

```
OP_RETURN 0x24 0xaa21a9ef[32-byte-merkle-root]
```

The commitment serialization and discovery rules follows the same rules defined
in BIP141.

特征（译者注：这里的特征原文是commitment，经过阅读BIP141 这个词是个加密学术语，就是通过一个值来保证和另一串数据的一一对应关系，是通过计算哈希得出。BIP141的里定义：The commitment is recorded in a scriptPubKey of the coinbase transaction. 即，特征被记录在coinbase交易的公钥脚本中）的序列化和搜索规则遵从BIP 141的规则。

The merkle root is to be calculated as a merkle tree with all extension and
canonical block txids and wtxids as the leaves.

被用于计算该梅克尔根的梅克尔树会将所有扩展区块和主区块中的txids和wtxids作为叶子计算进去。

Any block containing an extension block MUST include an extension commitment
output.

任何包含扩展区块的主区块必须包含扩展特征（extension commitment）输出。

### Extension block opt-in 交易进出扩展区块

（译者注：opt-in在这里表示交易进入扩展区块；对应的反意词是opt-out，表示交易退出扩展区块。）

Outputs can signal to enter the extension block by using witness program
scripts (specified in BIP141). Outputs signal to exit the extension block if
the contained script is either a minimally encoded P2PKH or P2SH script.

主区块内的交易输出可以使用见证程序脚本（witness program scripts）（见证程序脚本由BIP141定义）发出信号来进入扩展区块。
扩展区块内交易的输出脚本若为最小编码的P2PKH或P2SH脚本，则表示该输出退出扩展区块进入主区块。

Output script code aside from witness programs, p2pkh or p2sh is considered
invalid in extension blocks.

在扩展区块中，输出脚本除见证程序、P2PKH或P2SH之外，都被认为无效。

### Resolution 解析

Resolution transactions are a book-keeping mechanism which exist to maintain
the total value of the extension block UTXO set. They exist as a consecutively
redeemed OP_TRUE output. They handle both entrance into and exits from the
extension block UTXO set.

解析交易是一种记账机制,它维持了扩展区块上面UTXO集的所有价值。它们以连续的OP_TRUE输出的形式存在，处理进入和离开扩展区块的UTXO集。

Every block containing entering or exiting outputs MUST contain a final
`resolution` transaction. This resolution transaction spends all outputs that
intend to enter the extension block. The resolution transaction MUST appear as
the last transaction in the block (in order to properly sweep all of the newly
created outputs). The funds are to be sent to a single anyone-can-spend
(OP_TRUE) output. This output is forbidden by consensus rules to be spent by
any transaction aside from another `resolution` transaction.

如果一个主区块中包含进入或离开扩展区块的交易输出，则它**必须**在区块最后包含一个解析交易。这个解析交易会花费掉所有打算进入扩展区块的输出。解析交易**必须**作为区块的最后一笔交易出现(这样才能正确地抹去所有新创建的输出值)。资金将会被发送到一个单一的“任何人都可花费”（OP_TRUE）的输出。根据共识规则，除非有另一个解析交易，否则这一输出是禁止花费的。

The resolution transaction MUST contain additional outputs for outputs that
intend to exit the extension block.

解析交易**必须**包含附加的输出，以作为离开扩展区块的输出。

Coinbase outputs MUST NOT contain witness programs, as they cannot be sweeped
by the resolution transaction due to previously existing consensus rules.

Coinbase输出**必须不**包含见证程序，这是因为在现存的共识规则下，它们无法被解析交易抹去。

The first input of a resolution transaction MUST reference the first output of
the previous resolution transaction.

解析交易的第一个输入**必须**引用上一个解析交易的输出。

Fees are to propagate up from the extension block into the resolution
transaction. In other words, the resolution transaction's fee should should be
equal to the amount of total fees collected in the extension block.

手续费是从扩展区块向上传出进入解析交易的。换句话说，解析交易的手续费，必须等于扩展区块的所有交易手续费之和。

#### Bootstrapping 引导程序

In order to bootstrap the activation of extension blocks, a "genesis"
resolution transaction MUST be mined in the first block to include an extension
block commitment along with any entering outputs (the `activation` block). This
is the only resolution transaction in existence that does not require a
reference to a previous resolution transaction.

为了引导扩展区块的激活，一个创世解析交易**必须**首先被挖出，它存在于第一个包含了扩展区块特征的主区块内，记录了任何首次进入扩展区块的输出（该主区块称为激活块）。这是唯一一个，不需要引用上一个解析交易输出的解析交易。

The genesis resolution transaction MAY also include a 1-100 byte script in the
first input, containing a single push-only opcode. This allows the miner of the
genesis resolution to add a special message. The input script MUST execute
without failure (no malformed pushdatas, no OP_RESERVED).

创世解析交易也**可以**在第一个输入脚本中包含一个1到100字节的推送数据（pushdata），从而允许矿工在创世解析交易中添加一个特殊信息。推送数据必须能够转换成一个真值布尔数。

#### Resolution Rules 解析规则

The resolution transaction's first output MUST have a value equal to:

解析交易的第一个输出金额**必须**等于：

`(previous-resolution-value + entering-value - exiting-value) - ext-block-fees`

`(前一个解析值金额 + 进入金额 - 离开金额) - 扩展区块手续费`

The following output scripts and values must replicate the extension block's
exiting outputs _exactly_ (in the same order they appear in the ext. block).

接下来的输出脚本和金额必须精确复制扩展区块的现有输出（包括在扩展区块出现的顺序也要相同）。

The resolution transaction's version MUST be set to the uint32 max (`2^32 - 1`).
After activation, this version is forbidden by consensus rules to be used with
any other transaction on the canonical chain or extension chain. This is
required for easy non-contextual identification of a resolution transaction.

解析交易的版本**必须**设置到uint32最大（2^32-1）。在激活之后，这一版本号会被共识规则在主区块链（1M链）或其他扩展链上的交易禁止使用。这一要求是为了解析交易在非上下文状态（non-contextual）下被识别。

### Entering the extension block 进入扩展区块

Any witness program output is considered an opt-in to fund a keyhash or
scripthash within the extension block's UTXO set.

任何见证程序输出，如果存在属于扩展区块UTXO集的keyhash或scripthash输出，都视为一个选入（opt-in，即进入扩展区块）。

Example block:
区块示例：

```
交易#1 (coinbase):

输出#0：
  - 脚本: P2PKH
  - 金额: 12.5

输出#1:
  - 脚本: OP_RETURN 0xaa21a9ef[merkle-root]
  - 金额: 0
```

```
交易#2 (扩展块入账交易):

输出#0:
  - 脚本: P2WPKH (进入扩展块UTXO集)
  - 金额: 5.0

输出#1:
  - 脚本: P2PKH (留在主块UTXO集内)
  - 金额: 2.5
```

The resolution transaction shall redeem any witness program outputs:
解析交易将会赎回任何见证程序输出：

```
交易#3 (解析交易):

输入#0:
  - 前向输出（Outpoint）:
    - 哈希: 前向解析交易的txid
    - 序号: 0

输入#1:
  - 前向输出（Outpoint）:
    - 哈希: 交易#2的TXID
    - 序号: 0

输出#0:
  - 脚本: OP_TRUE
  - 金额: 5.0
```

In the case that a spender wants to redeem tx2/output0, the corresponding
extension block may exist as such:
想在扩展块内赎回tx2/output0，则该扩展块可有如下交易：

```
交易#4 (赎回交易#2):

输入#0:
  - 前向输出（Outpoint）:
    - 哈希: 交易#2的TXID
    - 序号: 0

输出#0:
  - 脚本: P2WPKH (该输出留在扩展块内)
  - 金额: 5.0
```

### Exiting the extension block 离开扩展区块

In order to ensure a 1-to-1 value between the existing blockchain and the
extension block, an exit must be provided for outputs that exist in the
extension block.

为了确保现存的区块链和扩展区块里的币汇率为1：1，一笔离开扩展区块的金额必须提供在扩展区块里存在的输出。

In a later transaction, the spender of Transaction #2's output may want an
output to exit with some of their money.
接着上面的例子，随后交易#2输出的花费者可能想要把部分资金从扩展区块转回主区块。

主块：
```
交易#5 (coinbase):

输出#0:
  - 脚本: P2PKH
  - 金额: 12.5

输出#1:
  - 脚本: OP_RETURN 0xaa21a9ef[merkle-root]
  - 金额: 0
```

```
交易#6 (解析交易):

输入#0:
  - 前向输出（Outpoint）:
    - 哈希: 前向解析交易的txid
    - 序号: 0

输出#0:
  - 脚本: OP_TRUE
  - 金额: 2.5

输出#1:
  - 脚本: P2PKH (重复下面扩展块内的退出输出)
  - 金额: 2.5
```

Extension block:
扩展块：

```
交易#7:

输入#0:
  - 前向输出（Outpoint）:
    - 哈希: 交易#4的TXID
    - 序号: 0

输出#0:
  - 脚本: P2WPKH (该输出留着扩展区块内)
  - 金额: 2.5

输出#1:
  - 脚本: P2PKH (该输出退出扩展区块)
  - 金额: 2.5
```

#### Exit Redemption 赎回退出扩展区块的输出

As described above, outputs exit from the extension block onto the main chain
by way of the resolution transaction. The outpoint created in the extension
block MUST not to be spent on either chain. Instead, exiting outputs must be
spent from outpoints created on the resolution transaction.

根据上面的描述，扩展区块里的输出是通过解析交易来离开扩展区块回到主链上的。在扩展区块里该交易的前向输出（outpoint）**必须没有**被任何一边花费。此外，离开扩展区块的交易输出必须做为解析交易的前向输出（outpoints）被花费。

#### Exit Maturity Requirement 退出成熟期要求

Similar to coinbase transactions, resolution transactions can also be
permanently un-mined in the case of a reorganization. This potentially
invalidates all spends from any exiting outputs it may have contained,
rendering the spending transactions not relayable and no longer mineable on the
best chain. An exit maturity requirement is required for this reason.

类似于coinbase交易，在区块链分叉并重构（reorganization）之后，解析交易就有可能永远不会被挖到。这会使得所有离开扩展区块的交易输出都无效，所有与之关联的花费交易都不可靠，并且使矿工无法进行最佳链挖矿。基于以上原因，离开扩展区块的交易需要等待一个退出成熟期。

详见：https://github.com/tothemoon-org/extension-blocks/issues/9

### Fees 手续费

Fees collected from inside the extension block propagate up to the
corresponding resolution transaction. The resolution transaction's fee MUST be
equal to the cumulative amount of fees collected inside the extension block.

从扩展区块里收集到的手续费向上传递到对应的解析交易。解析交易的手续费**必须**等于扩展区块里所有交易的手续费之和。

On the policy layer, transaction fees may be calculated by transaction cost as
well as additional size/legacy-sigops added to the canonical block due to
entering or exiting outputs.

在费用策略制定层面,交易费的计算方法可以考虑交易成本，同时考虑进入和离开的输出导致主区块额外增长的`交易尺寸/传统签名操作次数`（size/legacy-sigops）。

In the previous example, the spender of Transaction #2's output could have also
added a fee.

在之前的例子中，交易#2的输出可以包含一笔手续费：

```
交易#5 (coinbase):

输出#0:
  - 脚本: P2PKH
  - 金额: 12.501 (块奖励 + 手续费)

输出#1:
  - 脚本: OP_RETURN 0xaa21a9ef[merkle-root]
  - 金额: 0
```

```
交易#6 (解析交易):

输入#0:
  - 前向输出:
    - 哈希: 前向解析交易的txid
    - 序号: 0

输出#0:
  - 脚本: OP_TRUE
  - 金额: 2.499 (减去了0.001 BTC的手续费)

输出#1:
  - 脚本: P2PKH (重复下方扩展块内的退出交易)
  - 金额: 2.5
```

Extension block:
扩展块：

```
交易#7:

输入#0:
  - 前向输出:
    - 哈希: 交易#4的TXID
    - 序号: 0

输出#0:
  - 脚本: P2WPKH (该输出留在扩展块内)
  - Value: 2.499 (手续费被减去，并传播至主区块)

输出#1:
  - 脚本: P2PKH (该输出退出扩展区块)
  - 金额: 2.5
```

### Verification 验证

Verification of transactions within the extension block shall enforce all
currently deployed softforks, along with an extra BIP141-like ruleset.

扩展块内的交易验证应强制执行所有当前部署的软分叉，以及一个额外的类BIP141规则集。

Transactions within the extended transaction vector MAY include a witness
vector using BIP141 transaction serialization.

扩展交易矢量（extended transaction vector）中的交易**可以**包含一个使用BIP141交易序列化表示的见证矢量（witness vector）。

Verification shall be performed on extended transactions with `VERIFY_WITNESS`
rules.

扩展交易的验证工作将会使用`VERIFY_WITNESS`规则。

Extended transactions MUST NOT have any access to the canonical UTXO set.

扩展交易**不可以**使用任何来自主区块的UTXO集。

If an extension block fails any consensus check, the upgraded node MUST
consider the entire block invalid.

只要扩展区块在通过任何一项共识检查时失败，则已经升级的节点就必须认为整个区块无效（包括主区块和扩展区块）。

### BIP141 Rule Changes BIP141规则改变

- Aside from the resolution transaction, witness program outputs are only
  redeemable inside of an extension block.
- Witness transactions may _only_ contain witness program inputs.
- BIP141's nested P2SH feature is no longer available, and no longer a
  consensus rule.
- The concepts of `block weight` and `transaction weight` have been removed.
- The concept of `sigops cost` remains present for future soft-forkable and
  upgradeable DoS limits.

译文：
- 除了解析交易外，见证程序的输出必须只在扩展区块内被赎回。
- 见证交易可以只包含见证程序输入。
- BIP141的P2SH嵌套特性不再可用，该特性不再作为共识规则。
- 块权重（`block weight`）和交易权重（`transaction weight`）的概念被移除。
- 签名操作成本（`sigops cost`）的概念保留，以备将来的软分叉或DoS限制策略升级使用。

### DoS Limits DoS限制

DoS limits shall be enforced by extension block size as well as the newly
defined metrics of inputs and outputs cost. Be aware that exiting outputs
inside the extension block affect DoS limits in the canonical block, as they
add size and legacy sigops.

DoS限制应通过扩展块大小以及新定义的输入和输出成本标准来实施。请留意退出扩展块的输出对主块中DoS限制的影响，因为它们增加了解析交易的大小和传统签名操作的次数。

```
MAX_BLOCK_SIZE 最大主块大小: 1000000 (未改变)
MAX_BLOCK_SIGOPS 最大主块签名次数: 20000 (未改变)
MAX_EXTENSION_SIZE 最大扩展块大小: 待定
MAX_EXTENSION_COST 最大扩展块成本: 待定
```

The maximum extension size should be intentionally high. The average case block
is truly limited by inputs (sigops) and outputs cost.

应该刻意设置一个较高的最大扩展块大小。块的平均大小真正受限于输入（签名操作）和输出的成本。

Future size and computational scalability can be soft-forked in with the
addition of new witness programs. On non-upgraded nodes, unknown witness
programs count as 1 inputs/outputs cost. Lesser cost can be implemented for
newer witness programs in order to allow future soft-forked dos limit changes.

未来的区块尺寸和计算的可扩展性可以通过软分叉附加新的见证程序来实现。在未升级的节点中，未知见证程序的计算量开销被当成是 1 inputs/outputs 来计算。对新的见证程序设置更低的计算量开销可以使未来通过软分叉实现DoS限制策略的变更。

##### Extended Transaction Cost 扩展交易成本

Extension blocks leverage BIP141's upgradeable script behavior to also allow
for upgradeable DoS limits.

扩展区块使用BIP141的可升级脚本，也允许可升级的DoS限制。

###### Calculating Inputs Cost 输入成本的计算

Witness key hash v0 shall be worth 1 point, multiplied by a factor of 8.

见证公钥哈希v0的成本按1点算，乘上一个系数8。

Witness script hash v0 shall be worth the number of accurately counted sigops
in the redeem script, multiplied by a factor of 8.

见证脚本哈希v0的成本为在赎回脚本中确切的签名次数（sigops）乘上一个系数8。

Unknown witness programs shall be worth 1 point, multiplied by a factor of 1.

未知见证程序的成本按1点算，乘上系数1。

To reduce the chance of having redeem scripts which simply allow for garbage
data in the witness vector, every 73 bytes in the serialized witness vector is
worth 1 additional point.

由于赎回脚本允许在见证矢量（witness vector）中产生垃圾数据，因此为了减少赎回脚本的使用几率，每73字节的序列化见证矢量（serialized witness vector）计为额外的1点。

This leaves room for 7 future soft-fork upgrades to relax DoS limits.

这样给未来留下7点的空间，用于软分叉升级DoS限制。

###### Calculating Outputs Cost 输出成本的计算

Currently defined witness programs (v0) are each worth 8 points. Unknown
witness program outputs are worth 1 point. Any exiting output is always worth
8 points.

当前定义的见证程序（v0）按8点计。未知的见证程序按1点计。任何现有的输出总是计为8点。

This leaves room for 7 future soft-fork upgrades to relax DoS limits.

这样给未来留下7点的空间，用于软分叉升级DoS限制。

#### Dust Threshold 垃圾交易阈值

A consensus dust threshold is now enforced within the extension block.

扩展块内目前执行一个共识的垃圾交易阙值。

Outputs containing less than 500 satoshis of value are _invalid_ within an
extension block. This _includes_ entering outputs as well, but not exiting
outputs.

在扩展区块内，任何包含500聪以下输出的交易直接判定为垃圾交易。这种判断法适用于进入扩展区块的交易，但不适用于离开扩展区块的交易。

### Rules for extra Lightning security 额外的闪电网络安全规则

（译者注：这一部分大体意思是关于外挂在扩展区块的闪电网络关闭通道，将通道状态广播到扩展区块时的成本预估，主要是通过收紧对广播交易的区块空间来惩罚非法广播，让攻击的手续费变的非常高）

Lightning Network faces certain kinds of systemic attacks as defined in the
Lightning Network whitepaper risks (mass-closeout of old states).

在闪电网络白皮书中，已经定义了闪电网络所面临的系统性攻击(旧状态的大规模关闭)。

If the second highest transaction version bit (30th bit) is set to to `1`
within an extension block transaction, an extra 700-bytes is reserved on the
transaction space used up in the block. [NOTE: Transaction space and sigops
cost not yet defined]

如果扩展区块版本位的第二高位（第30位）设置为1，则区块的交易空间中应保留额外的700字节。（注意：交易空间和签名操作成本尚未定义。）

This space per transaction is pre-allocated and can be consumed in the same
block by two transactions (of a maximum size of 350 bytes each), which fulfill
specific constraints as defined below.

这一空间中共有两个预分配的位置，可被该区块内的两笔交易所使用（每笔最大350字节），且填充时满足以下约束条件：

The first allocation may only be consumed by a transaction which directly
spends from the first output of the transaction with the 30th bit set.

第一个位置只能被满足以下条件的交易使用：该交易直接花费了一个版本号第30位为1的交易的第一个输出。

The second allocation may only consume the first output of ANY transaction
within the past 2016 blocks which have the 30th bit set.

第二个位置只能被满足以下条件的交易使用：该交易花费了最近2016个区块内的**任何**一个版本号第30位为1的交易的第一个输出。

If the allocation is not used by other transactions, the transaction consumes
that extra space, reducing the blocksize by up to 700 bytes in available space.

如果这两个位置没有被其他交易使用，则满足这些条件的交易使用这两个位置，这让区块的可用空间减少了最多700字节。
（译者注：也许这一句的翻译是错误的。）

This is a consensus rule within the extension block and does not apply to the
main-chain block.

这一共识规则只适用于扩展区块，不适用于主链区块。

The purpose of this is to ensure that without miner coordination, the costs
will be unusually high for systemic attacks, since blockspace is preallocated
for miners to include penalty Lightning Network transactions, and this is
enforced since both parties must agree to the version bit in Commitment
Transaction in Lightning. This is an opt-in feature in Lightning, and trades
higher transaction fees for greater availability for penalties. The assumption
is that in the majority of cases of an incorrect broadcast, the penalty will be
included in the same block via the second allocation, and give room for other
transactions in the first allocation.

以上规则是为了确保：在无需矿工互相协同的情况下，攻击这一系统所要支付的手续费会异常的高，因为闪电网络交易是预订了区块空间的，并且闪电网络的交易双方都必须同意位于Commitment Transaction中的交易版本位。这是一个在闪电网络里的opt-in（自愿接受）特性，通过收取更高的手续费来提升处罚的效力。这是假定在大多数非法广播的场景中，同一区块内的处罚将位于第二个分配位置，并把第一分配位置留给其他交易。
（译者注：这一段的翻译可能不太准确。）

### Migration and adoption concerns 迁移与采纳的影响

Most of the bitcoin ecosystem is currently well-equipped to handle an upgrade
to BIP141.

大头数比特币生态系统现在已经做好了升级到BIP141的准备。

For wallets currently supporting BIP141, the migration should be trivial.

对于现在支持BIP141的钱包，所需的迁移成本是很微小的。

For fullnode-based services, APIs may be altered to transparently serve
extension block transactions to clients as if they appeared in the canonical
block. This, of course, would not include any miner API.

对全节点来说，API可以修改为透明的为扩展区块交易服务，就好像这些都是传统交易一样。当然这不包含矿工用的API（就是说矿工必须能够区分扩展和非扩展区块，而外接的程序则不一定需要区分）。

### Wallet concerns and migration 钱包的影响与迁移

Wallets currently supporting BIP141 must be modified in a few key ways in order
to achieve compatibility with extension blocks.

现在支持BIP141的钱包，必须做如下关键调整保证和扩展区块兼容：

1. Wallets must pick a chain to spend from when creating a transaction (either
   the canonical block or the extension block, but not both). In other words,
   transactions must have all witness program inputs, or all non-witness program
   inputs. For wallets that support both chains, the coin selector can
   automatically pick which chain to use if the user does not specify.

2. Wallets supporting extension blocks must ignore inputs on
   resolution transactions if seen. This should be a simple check of transaction
   version number, and similar to how wallets already ignore a coinbase's input.
   This is necessary to prevent wallets from mistakenly seeing a double spend.

3. Wallets supporting both canonical block and extension block funds
   must ignore exiting outputs within the extension block. This is necessary to
   prevent wallets from mistakenly indexing the same output twice.

译文：

1. 当创建一笔交易时，钱包必须选择一条链来开始花费（要么选择主区块，要么选择扩展区块，但不能同时选两者）。
   换句话说，交易的输入必须要么全为见证程序输入，要么没有任何见证程序输入。
   对于支持两条链的钱包来说，如果用户没有定义使用哪条链的话，币选择器会自动选择一条链。
   
2. 支持扩展区块的钱包，当看到解析交易的输入时，必须忽略掉。
   这可以非常简单的通过交易版本号来实现，就和钱包已经实现的忽略Coinbase交易的输入那样。
   这一点对防止钱包错误地判断双花很重要。
   
3. 同时支持主区块和扩展区块资金的钱包必须忽略掉从扩展区块里离开的交易的输出。
   这是为了防止钱包错误地将同样的输入索引两次。

The latter two points only apply to wallets with operate via direct blockchain
monitoring. Monitoring wallets typically watch the blockchain and index their
own transactions and outputs.

后两点只适合这样的钱包：这些钱包通过直接监控区块链来进行操作。监控类钱包通常关注区块链，并且索引他们自己的交易和输出。

### Mempool Concerns 对内存池的影响

Changes to mempool implementations are surprisingly minimal. Although details
may differ across implementations, a conforming mempool must disallow
cross-chain spends (mixed inputs on both chains), as well as track exiting
output's outpoints. These outpoints may not be spent inside the mempool (they
must be redeemed from the next resolution txid in reality).

内存池实现所需的改动是非常小的。尽管实现细节上可能略有不同，但是一个合规的内存池必须在追踪确认某输入的前向输出（outpoints）时拒绝跨链花费（同时包含主块和扩展块上的输入）。这样的前向输出是不能在内存池被下一笔交易所花费的（因为它们只有在下一个解析交易的txid真实生成时才能被赎回）。

### Mining Concerns 对挖矿的影响

#### Additional size and sigops calculation 附加大小和签名操作计算

Nodes creating block templates internally will have to take into account
entering and exiting outputs when reserving size for the canonical block. A
transaction with entering outputs or exiting outputs adds size to the canonical
block.

节点在内部创建区块模板时，在为主区块留出保留空间时应考虑到进出扩展区块的交易输出的大小。包含了进出扩展区块的输出的交易会增加主区块的大小。

Entering outputs add size in the form of inputs in the resolution transaction.

进入扩展区块的交易输出将会做为解析交易的输入增加解析交易的尺寸。

Exiting outputs add both size and legacy sigops in the form of duplicated
outputs on the resolution transaction.

离开扩展区块的交易输出会在解析交易中以重复输出的形式增加大小和传统签名次数（sigops）。

Transaction sorting and selection algorithms must take care to account for
this.

交易的排序和选择算法必须仔细考虑这些特点。

#### Extensions to `getblocktemplate` (BIP22) 对`getblocktemplate`（BIP22）的扩展

In addition to the `transactions` array, conforming implementations MUST
include an `extension` array. The `extension` array contains transactions as
defined in BIP22 as well as the BIP145 extensions to `getblocktemplate`.

除`transactions`数组外，兼容本协议的实现还必需包含一个`extension`数组。该`extension`数组包括了按照BIP22定义的交易，以及按BIP145定义的`getblocktemplate`的扩展区块。

Conforming implementations MUST include the resolution transaction as part of
the `transactions` array. The resolution transaction must have an extra
property called `resolution`, which is a boolean value set to `true`. Including
the resolution TX in the `transactions` array directly is done for backward
compatibility with existing mining clients.

兼容的实现**必须**在其`transactions`数组中包含解析交易。该解析交易必须有一个额外的属性`resolution`，设置为布尔值`true`。在`transactions`数组中包含解析交易使其可以兼容现有的挖矿客户端。

`default_witness_commitment` has been renamed to `default_extension_commitment`
and includes the extension block commitment script.

`default_witness_commitment`在扩展区块中被重命名为`default_extension_commitment`，并包含在扩展区块的commitment脚本里。

An extra `mutable` field is defined for `"resolution"`. If `transactions` is
present in the mutable array, `resolution` allows extension block mutation,
provided the client is able to properly update the resolution transaction.

一个额外的`mutable`字段被定义在`"resolution"`数组中。如果`transactions`现在是可变数组，`resolution`允许扩展区块可变，这让客户端可以正确更新解析交易。

Along with the `resolution` mutable field, a `resolution` `capabilities` field
is also defined in order for the client to state that it is capable of updating
the resolution transaction.

与`resolution`字段一起，新定义的`resolution` `capabilities`字段是为了让客户端能够申明其更新解析交易的能力。

For nodes dealing with out of date mining clients, storing the extension block
locally in a map of `commitment-hash->ext. block` may be required (along with
some reserialization during a `submitblock` call).

对于与老旧挖矿客户端交互的节点来说，可能需要在本地使用键值对为`commitment-hash->ext. block`的map容器来存储扩展区块（并且还要在调用submitblock期间，完成一些重新序列化的工作）。

### Data Migration Concerns 数据迁移的影响

It is likely that implementations will need to include an extra bit on every
stored UTXO in order to mark which chain they belong to. Depending on the
implementation's UTXO serialization/compression format, this may require a
database migration.

每一个实现（implenmentations）都需要一个额外的位（bit）来存储一个UTXO位于哪一条链。根据实现的UTXO序列化/压缩格式的不同，这可能需要一次数据库迁移。

### Activation 激活

- 版本位: 2
- 部署名称: `extblk` (在GBT中记作`!extblk`)
- 开始时间: 待定
- 超时时间: 待定

### Deactivation 失活

Miners may vote on deactivation of the extension block via the 28th BIP9
softfork bit. The "deactivation" deployment's start time shall begin 3 years
after the start time of extension blocks, with a miner activation threshold of
95%. The minimum locked-in period must be at least 26 retarget intervals (1
year).

扩展区块激活后，矿工可以通过第28个BIP9软分叉位来实现让扩展区块失活。这个“失活”部署的开始时间应该在扩展区块激活后3年才可以，使“失活”生效的算力阈值为95%。最小的锁定周期必须至少为26个难度调整周期（1年）。

By this point, a future extension block ruleset will likely have been
developed, which is superior in terms of feature-set and scalability (see also:
[Rootstock][rsk] and/or [Mimblewimble][mw]). This enables updates for long-term
scalability solutions with minimal baggage of supporting deprecated chains.

通过这一点，将来扩展区块规则集可以得到发展，它在功能集和可扩展性方面是非常优越的（另请阅读：[Rootstock][rsk]和[Mimblewimble][mw]）。这使得长期扩展解决方案在支持将要丢弃的链上升级时，代价和包袱很小。

After deactivation, it is expected by the community that exits from the
extension block will still still be possible and secure according to the terms
of the yet-to-be-designed soft-fork, but entrance into and transfer of funds
within the extension block could be no longer permitted.

在失活后，社区肯定希望离开扩展区块的交易将以可选择的和安全的被设计好的软分叉进行，但进入扩展区块的交易和转移资金到扩展区块需要被禁止。

We propose two possible solutions for deactivation, one of which will be
selected in the final version of this document.

我们提出两个失活解决方案，本文档将选择其中之一作为最终方案。

#### Deactivation Option #1 失活方案1

Upon activation of the 28th bit, the resolution output will return to being an
output which anyone-can-spend as a consensus rule today. This 28th BIP9 bit (or
another BIP9 bit in conjunction) can be overloaded to enable soft-fork
activation to prevent this from actually being an anyone-can-spend in the
future. This allows for enabling future extension block features without
hard-forking the code.

当第28个BIP9软分叉位被激活后，解析交易输出将退回到一个被现在共识规则定义好的“任何人都可以花费”的输出。这一BIP9位（或者结合其他BIP9位）可以在将来被重载以阻止这个“任何人都可以花费”的软分叉激活。这可以使得将来扩展区块特性不需要硬分叉。

A social contract is understood whereby the funds in the extension block will
be usable and redeemable in the general deactivation design below. If proper
and safe withdrawals are not activated within the terms, users and exchanges
can refuse to acknolwedge blocks with the bit set as a soft-fork.

社区的共识是，在扩展区块失活后，里面的资金是可以被使用和赎回的。如果失活生效期间没有适当的和安全的取款条款，用户和交易所可以通过一个软分叉位来拒绝执行。

It is possible to do a direct upgrade of the extension block by using the same
output set upon BIP9 activation of the 28th bit in conjunction with new rules
on another bit (e.g. 27th bit activates the new chain conditionally upon the
28th bit also being activated, creating a direct migration into a new extension
block without new transactions on the main-chain).

可以通过一个新的软分叉位来升级扩展区块，使用基于BIP9的激活策略，联合（conjunction）第28个位，并使用新的规则（例如，在第27个位激活新的链，并有条件地激活第28位，创造一个直接迁移到新的在主链上没有交易的扩展区块）。

It is understood that this soft-fork will be overloaded with enforcement of
withdrawal/redemption of funds so that it requires the script terms to be
parsed upon withdrawal.

这个软分叉将强制执行取款/赎回资金，并将被重载，所以这要求脚本从语法上描述取款。

#### Deactivation Option #2 失活方案2

Upon activation of the 28th bit, no further transactions are permitted inside
the extension block. Withdrawals to the main-chain only via merkle proofs are
only permitted.

当第28个软分叉位被激活后，不再允许扩展区块内出现交易。只允许通过梅克尔树证明来提现到主链。

This requires code and specification for merkle-proof withdrawals to be
specified and available today.

该方案要求现在就详细说明并发布梅克尔树证明提现的代码和规范。

#### 从此之后的译文没有校对。TODO: 进行校对
#### Deactivation via Merkle Tree Proofs 通过梅克尔树证明使其失效

Redemption from the old extension block to the main-chain and new extension
block can be migrated by way of merkle proofs to be designed in the future.
Funds may be imported to the new extension block by hard-coding a UTXO merkle
root into the implementation as a consensus rule, and verifying imported funds
against a merkle path.

从旧扩展区块到主链的交易赎回和新扩展区块可以通过以后设计梅克尔树证明的方式来转移。资金可以通过将UTXO梅克尔树根作为一种共识规则实现硬编码转入新扩展区块，并根据梅克尔树路径验证转入资金。

To enable importing, nodes require only a copy of the current 32-byte merkle
root of the deactivated extension block.

为了使资金转入，节点只需要一份失效扩展区块当前32字节梅克尔树根的副本。

This removes the necessity for full nodes to store a copy of the full UTXO set
on disk, with a tradeoff of larger on-chain transactions when redeeming in the
future. An alternative would be for all clients to maintain a record of the
UTXO set and keep the full bitfield in memory.

全节点就没必要在磁盘上存储完整UTXO集合，在将来赎回时使用一个大的链上交易进行平衡，（译者注：暂时没明白这是什么意思his removes the necessity for full nodes to store acopy of the full UTXO set on disk, with a tradeoff of larger on-chaintransactions when redeeming in the future.）替代方案是为了为客户端保存UTXO集合的记录并在内存中保持完整的位字段。

Since the TXO set is static (with only whether it is unspent or spent changing)
as it is deactivated from new outputs, this is simpler than currently proposed
designs for changing UTXOs:
https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2015-October/011638.html

基于新输出是无效的，TXO集是静态的（集只有花费或未花费的变化），这比目前提出的动态变化UTXOs的设计简单得多：https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2015-October/011638.html

#### Motivation 目的

While deactivation may be seen as a drawback, the primary benefit of this
specification is _not_ an extension block with a BIP141-ruleset. Instead, it is
that the ecosystem will have laid the groundwork for handling extension blocks.
The future of the bitcoin protocol may include several different extension
blocks with many different rulesets.

虽然失效被看作是一个不利条件，但是本规范的主要优点不是一个使用BIP141规则集的扩展区块。相反，这个生态系统将奠定以操作扩展区块为基础。未来的比特币协议也许会包含几种不同的扩展区块和很多不同的规则集。

## Reference Implementation (WIP) 参考实现（正在开发）

https://github.com/bcoin-org/bcoin-extension-blocks

## Copyright 版权

This document is hereby placed in the public domain.

本文档发布于公有领域。

[aux]: https://bitcointalk.org/index.php?topic=283746.0
[ext]: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-January/013490.html
[rsk]: https://uploads.strikinglycdn.com/files/90847694-70f0-4668-ba7f-dd0c6b0b00a1/RootstockWhitePaperv9-Overview.pdf
[mw]: https://scalingbitcoin.org/papers/mimblewimble.txt
