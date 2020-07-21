# 基于BSV的存储证明

## 场景假设

Alice希望把自己的一个文件存储起来，Bob可以提供存储服务。所以Alice把文件给了Bob，但Alice并不信任Bob真的在帮她存储文件，所以Alice需要让Bob定期证明他正在保存完整的文件。只要Bob证明了他正在存储完整文件，那么Alice就会给Bob一些存储奖励。为了降低成本，Alice和Bob都不希望这个证明过程传输完整的文件。

## 解决方案

### 核心方案

Alice为了让Bob证明他存储了完整的文件，她需要给Bob提出一个问题，这个问题只有拥有完整文件才可以解答出来。所以上述场景的核心方案就是问题的构造：

1. Alice生成一个随机key
2. Alice使用HMAC算法计算HMAC(dataToBeSaved, key)，并记录自己的计算结果，这个结果的数据量非常小，只有几十个字节。
3. Alice把key告诉Bob，请Bob也进行同样的运算
4. 如果Bob跟Alice的计算结果一样，那么说明Bob保存了完整的文件



Alice可以用BSV脚本描述这个问题：

`OP_HASH160 <SHA256(HMAC(dataToBeSaved, key))> OP_EQUALVERIFY OP_RETURN <key>`

1. `OP_RETURN <key>` 是Alice为了把计算HMAC的key暴露给Bob
2. 如果Bob有dataToBeSaved，那么他可以用key计算出HMAC(dataToBeSaved, key)，把结算结果作为解锁信息，就可以让上述脚本执行成功。



### 完整过程

1. Alice提前准备好一系列的随机key
2. 对每个key都进行HMAC运算HMAC(dataToBeSaved, key)，并保存key与运算结果
3. Alice把待保存文件发给Bob
4. Alice定期创建一个tx发给Bob，选择其中一个key生成output脚本（每个key只能用一次）
5. Bob收到tx后，通过key和已保存的文件计算HMAC(dataToBeSaved, key)转走tx里的币，并以此证明他保存了完整文件。

## 引申

这里只是提到了一个非常简单的概念方案，用于启发灵感。真实的应用要比这个概念方案复杂得多，比如：

1. Alice有很多文件要陆续存储和删除
2. 如何方便地存储key和HDMAC计算结果
3. 引入多个存储服务商进行竞争

