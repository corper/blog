# 基于BSV智能合约的棋牌类游戏设想

有了[BSV的智能合约](https://blog.csdn.net/Edward_sv/article/details/106688515)相关技术，实现棋牌类的智能合约应该是可行的。我们以中国象棋为例，设想是否可以实现这样的一个合约：

1. 合约初始化时确定对弈双方和行走顺序

2. 只有对弈双方才可以走棋

   只有通过对弈双方的才有权限花费合约UTXO，触发合约迁移到下一个状态。其他人无法花费合约UTXO。

3. 合约约束回合制规则

   每人每回合只走一步，交替走棋。不能多走，也不能破坏顺序。

5. 合约约束行走规则

   “马走日，象走田”等规则均由合约来约束，不符合象棋规则的走棋无法触发合约状态变迁。

6. 合约可以判断胜负

   一方的“将”被杀时，合约终止，并记录胜负。



我们分析每一个需求用什么方式来实现。如下的方案顺序与上述需求顺序一一对应。

1. 两个玩家的公钥

   初始化合约时，脚本中先后存储两个玩家的公钥P1和P2，约定P1先走。

2. 签名校验

   每次合约变迁，校验P1或P2的签名，只有签名校验通过才可以花费UTXO，触发合约状态变迁。

3. 步数计数器

   在合约状态有一个步数计数器，初始化为1，每次变迁计数器加1且只加1。计数器变迁为单数时校验P1签名，变迁为双数时校验P2签名。

4. 棋盘状态、行走指示、规则逻辑

   * 棋盘状态。在合约的数据部分有一块空间用于存储棋盘状态：棋盘的每个坐标上有哪一方的什么棋子（或没有棋子）。

   * 行走指示。玩家确定本次要行走的起始坐标和目的坐标，因为棋盘状态已经记录了坐标上有什么棋子，所以就不必再指定棋子。把两个坐标作为解锁脚本参数传入，告诉合约准备如何走棋。
   * 规则逻辑。行棋规则的合法性判断，把“棋盘状态”和“行走指示”这两个数据作为输入，用脚本实现象棋规则逻辑判断，计算出这一步棋是否合法。
   * 规则检验。合约可以通过OP_PUSH_TX技术，在scriptCode中获得上一步棋盘状态，“行走指示”从输入脚本参数获得。用这两个信息作为输入调用上述规则逻辑进行规则校验，如果校验成功，则更新棋盘状态。

5. 胜负判决

   用脚本判断上一步更新后的棋盘状态，一方的“将”是否已被杀，如果被杀，则棋局终结，不再允许后续触发的合约状态更改。“棋盘状态”数据本身已经可以明确胜负状态，不必再增加其他数据记录。

   

​     