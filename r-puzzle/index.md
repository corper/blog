# R-Puzzle基本原理

## 长话短说

普通的ECDSA签名校验是校验整个签名与预期是否一致，而R-Puzzle仅校验ECDSA签名中的r是否与预期的一样。

## ECDSA签名

ECDSA签名需要用到的信息：

* 基点：G
* G的阶：n

* 签名者的私钥：d，从私钥可计算出公钥Q = d * G
* 待签名的信息：m

签名过程：

1. **生成一个新的随机私钥k，从私钥可计算出其公钥R = k * G。R的横坐标标记为xR，纵坐标标记为yR。**
2. **令 r = xR mod n**
3. 计算 H = Hash(m)
4. 按照数据类型转换规则，将H转化为一个大端的整数e
5. 计算 s = k^-1(e + r*d) mod n
6. 签名 = (r, s)

最终，签名包含r和s两部分。其中**r**就是R-Puzzles中的**R**。

通过上述签名计算过程可以知道，r只需要通过随机私钥k就可以生成，并不需要签名者的私钥d。这是R-Puzzle的关键技术原理。

## P2RPH脚本模板

P2RPH是Pay to R-Puzzle Hash脚本模板的缩写，这个输出脚本是这样的：

`OP_OVER OP_3 OP_SPLIT OP_NIP OP_1 OP_SPLIT OP_SWAP OP_SPLIT OP_DROP OP_HASH160 <Hash(r)> OP_EQUALVERIFY OP_CHECKSIG`

传入正确的**公钥**和**签名**：`<pubkey> <sig>`，就可以解锁该脚本。

P2RPH脚本主要分三部分：

1. 从签名中解析出r：`OP_OVER OP_3 OP_SPLIT OP_NIP OP_1 OP_SPLIT OP_SWAP OP_SPLIT OP_DROP`
2. 判断r是否与预期一致：`OP_HASH160 <Hash(r)> OP_EQUALVERIFY`
3. 校验签名：`OP_CHECKSIG`

## P2RPH的特点

* **先生成transaction，再决定谁可以花**

  P2PKH是需要先知道谁可以花费（公钥Hash）这个output，然后再生成transaction。而P2RPH可以先生成transaction，再确定让谁来花。

  因为只要知道k，就可以计算出正确的r。OP_CHECKSIG只检查签名是否正确，不在乎是用什么公私钥对进行的签名。所以就可以先生成P2RPH的transaction，如果想让谁花这笔钱，就在链下把k值给他，而不暴露私钥。

* **k值不会被公开**

  这里的公开是指不会被放到链上。

  P2RPH需要证明“知道k值”才可以被花费，而证明“知道k值”并不需要公开k。这一点跟简单的Hash-Puzzle不一样，Hash-Puzzle是需要公开要知道的知识才可以花费。

## 安全性

### 不要用相同私钥生成两个k值一样的签名

如果用同样的私钥生成两个k值一样的签名，那么知道k值的人会通过这两个签名反推出私钥。

### 避免签名可塑性

在花费R-Puzzle时，公钥P、签名`<r,s>`、签名信息m都会被暴露。可以通过创建一个新的信息m'，并结合这些已被暴露的信息，重新计算出一个合法的公钥P'。方法是： `P' = P + [ r^-1 [ H(m) - H(m') ] ] * G`。这样，就可以用新的公钥P'来花费P2RPH了。

这当然是个非常值得关注的安全问题。一种解决办法就是在脚本中再增加一个签名，这个签名的私钥就是公钥P对应的私钥，但签名用了不用于k的随机私钥k'，生成的是不同于r的r'，即新增签名为<r', s'>。这样，解锁参数由原来的两个变成了三个：

`<sig'> <pubkey> <sig(r-puzzle)>`

当然，对应的P2RPH脚本模板也要增加对新签名的校验：

`OP_DUP OP_3 OP_SPLIT OP_NIP OP_1 OP_SPLIT OP_SWAP OP_SPLIT OP_DROP OP_HASH160 <Hash(r)> OP_EQUALVERIFY OP_OVER OP_CHECKSIGVERIFY OP_CHECKSIG`

## 参考资料

* [Bitcoin SV wiki](https://wiki.bitcoinsv.io/index.php/R-Puzzles)

* [R-Puzzles](https://www.youtube.com/watch?v=CqqTCsLzbEA)

* [CoinGeek Toronto Conference 2019: Dr. Craig Wright on R-PUZZLEs, the METANET, and ATTESTATION SYSTEM](https://www.youtube.com/watch?v=9EHKvNuRc0A)

* [比特币交易中的签名与验证](https://www.jianshu.com/p/a21b7d72532f)

* [比特币(BSV)应用开发学习系列-bsv js库介绍(2)](https://zhuanlan.zhihu.com/p/84294136)

  





