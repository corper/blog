# 支付通道基本原理

支付通道是指双方或多方无需信任地交换和更新tx的机制。在支付通道技术里，tx会在多方之间多次更新，除了最后一次更新需要上链，其他的更新都可以链下进行。这个特点特别适合需要快速更新tx的场景，比如频繁快速的支付等。相比每次支付都上链，支付通道的优势之一是可以节省手续费。

## 必备知识：nSequence和nLockTime

在了解支付通道之前，需要先了解比特币的两个技术点：[nSequence和nLockTime](https://wiki.bitcoinsv.io/index.php/NLocktime_and_nSequence)。

### nSequence约束tx更新

每个tx的每个input都有一个nSequence字段，可以把该字段理解为input的版本号，版本号越大表示版本越新。该字段为最大值0xFFFFFFFF时，表示已经是最新版本了，不会再更新了。

矿池内存池可以接受nSequence大的tx替换nSequence小的tx，但反过来不行。

### nLockTime和nSequence联合约束tx的打包时间

每个tx都有一个nLockTime字段，该字段的数值表示一个时间，含义为：在这个时间之前，该tx可能会更新。 

一个tx中，如果至少有一个input的nSequence值不是最大值，那么就表示说：这个tx不是最终版本，可能还会更新，先别打包进区块。什么时候打包呢？等当前时间超过nLockTime时，无论是不是最终版本，都可以打包进区块了。也就是说，如果nLockTime是未来的某个时间，那么该tx就只能在内存池里等待被更新，一直到当前时间超过nLockTime。

## 案例分析：微支付

有了nSequence和nLockTime，就可以实现支付通道了，我们设想如下场景：

Alice是一个论坛的管理员，她的老板你不用猜就知道，叫Bob。每当有人在论坛上发帖时，Alice都会审核帖子是否符合论坛规定。Alice领的是计件工资，审核一个帖子获得0.0001BSV，一天结算一次。但Bob老拖欠工资，Alice就说我每审核完一个帖子，你就得给我付一个帖子的钱，你不给钱，后面的帖子我就不审了。Bob说行是行，但这手续费占比可就太高了，不划算啊。要不这样，咱用支付通道吧，可省手续费了。

![支付通道流程](/Users/edward/workspace/blog/payment-channel-fundamental/en-micropayment-channel.svg)

1. 创建“抵押tx”

   * Bob：先转了0.1BSV到一个Alice和Bob 2/2签名的output里，作为抵押款。这个tx他自己留着没公开，也没给Alice。

2. 创建“退款tx”

   * Bob：又创建了一个tx，把抵押款作为输入，把所有抵押款（除去手续费）转到自己的地址。但该tx的nLockTime是24小时后，并且input的nSequence设置为1。这个tx的意思是说：如果24小时时间里，Alice啥也没干，那就全额退款给Bob。并且，这个tx后续可能会有更新。但这个tx要花抵押款，只有Bob一个人签名是做不到的，所以他签名后把该tx发给Alice签名。

   * Alice：检查是24小时后退款才会生效，就放心签了名给了Bob。然后，要求Bob把抵押tx发给自己。

3. 广播“抵押tx”

   * Bob：应Alice要求，把抵押tx给了Alice。

   * Alice：确定抵押tx的output里的确有2/2签名的0.1BSV抵押款，并且退款tx也的确是花的抵押tx里的钱。于是，Alice把抵押tx提交给矿池。

4. 更新“退款tx”，以支付工资

   * Alice：审核了一个帖子，需要Bob付款了。所以她更新了“退款tx”，增加了一个新的output，放了0.0001到这个output，该output Alice自己可以独立花费。并把input的nSequence加1。现在，新版本的“退款tx”的功能已经不仅仅是退款，而且还有支付给Alice工钱的功能。Alice先对新版本的tx签名，然后发给Bob也进行签名。

   * Bob：收到新版本的退款tx后，检查确实只有0.0001BSV转给Alice，剩下的还是转给自己，就放心签名并发给Alice。

Alice和Bob可以一直重复第4步，每次都给Alice的output多加0.0001BSV，从Bob的output减少0.0001BSV，直到一天工作结束。任何一个人都可以把最新的tx提交给矿池，等待24小时到期后各得其所。

或者中间有一方不满意了，直接中断了合作，把最新tx提交给矿池，此时每个人仍然得到结束前最新的那次协商的资金。因为每次协商只有0.0001BSV的差距，所以双方也不会有大损失。

## 总结

通过支付通道，可以多方无需信任地快速更新tx，并节省上链手续费。

本文只举了一个基本的例子来介绍原理。用支付通道还可以实现更复杂的功能，后续文章会继续介绍其他复杂的应用场景。

----

参考资料：

1. [Micropayment Channel](https://developer.bitcoin.org/devguide/contracts.html?highlight=channel#micropayment-channel)
2. [Payment Channels](https://wiki.bitcoinsv.io/index.php/Payment_Channels)
3. [nLocktime and nSequence](https://wiki.bitcoinsv.io/index.php/NLocktime_and_nSequence)