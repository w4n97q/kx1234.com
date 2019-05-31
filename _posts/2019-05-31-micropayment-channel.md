---
layout: post
title: 微支付通道
categories: bitcoin 闪电网络 微支付通道
tag: 
date: 2019-05-31
---

# 微支付通道

注： 未完成，待修改，有部分错误

单一方向的多次支付，通过减少链上操作，降低交易花费

## 步骤

1. Create a public key (K1) for client. Request a public key from the server (K2).

> 为客户端创建一个公钥K1，从服务器获取服务器公钥K2

2. Create and **sign** but do **not broadcast** a transaction (T1) that sets up a payment of (for example) 10 BTC to an output requiring both the server's private key and one of your own to be used. A good way to do this is use OP_CHECKMULTISIG. The value to be used is chosen as an efficiency tradeoff.

> 首先创建一个客户端与服务器双重签名的P2SH，命名为`CS`，然后客户端创建一个交易T1，只签名但不广播此交易。在T1交易中，客户端支付10BTC到`CS`

3. Create a refund transaction (T2) that is connected to the output of T1 which sends all the money back to yourself. It has a **timelock** set for some time in the future, for instance a few hours. Don't sign it, and provide the unsigned transaction to the server. By convention, the output script is `2 K1 K2 2 CHECKMULTISIG`

> 客户端创建一个赎回交易T2,此交易以T1为输入，把全部金额返回给自己。这笔交易有一个timelock，以保证它可以在未来的一个时间点执行，比如几个小时以后。客户端并不为T2签名，而是将未签名的交易发给服务器。通常情况下，这个输出脚本是 `2 K1 K2 2 CHECKMULTISIG`

4. The server signs T2 using its private key associated with K2 and returns the signature to the client. Note that it has not seen T1 at this point, just the hash (which is in the unsigned T2).

> 服务器用自己的私钥为T2签名，并将签名发给客户端。注意：此时只是使用了T1的hash，但T1还没有被广播到网络。

5. Client verifies the server\'s signature is correct and aborts if not.

> 客户端验证服务器的签名

6. The client signs T1 and passes the signature to the server, which now broadcasts the transaction (either party can do this if they both have connectivity). This locks in the money.

> 客户端签名T1,并将签名发给服务器，并广播此交易（双方都可以）。待交易被确认，押金就被锁定了。

7. The client then creates a new transaction, T3, which connects to T1 like the refund transaction does and has two outputs. One goes to K1 and the other goes to K2. It starts out with all value allocated to the first output (K1), ie, it does the same thing as the refund transaction but is not time locked. The client signs T3 and provides the transaction and signature to the server.

> 就像创建T2一样，再创建一条交易T3, 同样使用T1作为输入。T3有两个输出，一个是给客户端K1,一个是给服务器K2。此交易将金额重新分配，但不像T2,它是没有timelock的，只要广播出去就可以马上被打包到区块。客户端签名T3，并发给服务器。

8. The server verifies the output to itself is of the expected size and verifies the client's provided signature is correct.

> 服务器验证交易的输出和签名

9. When the client wishes to pay the server, it adjusts its copy of T3 to allocate more value to the server's output and less to its own. It then re-signs the new T3 and sends the signature to the server. It does not have to send the whole transaction, just the signature and the amount to increment by is sufficient. The server adjusts its copy of T3 to match the new amounts, verifies the signature and continues.

> 当客户端想要支付给服务器，它只需要复制T3，并修改输出金额，然后重新给T3签名，并发送到服务器。客户端不需要发送全部的交易，只需要发送签名和增加金额就足够了。服务器调整T3的拷贝，验证签名，然后继续。

### 总结

客户端持有T2，有双方签名，但有timelock。

服务器持有T3，有双方签名，没有timelock。

如果服务器违约，客户端可以广播T2, 就可以在timelock之后拿回自己的资产。

如果客户端违约，服务器可以在RefundTx被执行前，广播最新的T3，由于T3没有timelock，会被立即打包，所以客户端持有的RefundTx就失效了。

### 扩展

这个实现是微支付通道在wiki里的说明，但实际上这里的T3,可以让双方都持有，但需要增加一个逐渐收敛的timelock，这样可以让双方都能作废历史交易，并且可以中止交易。

如果双方均持有一组交易
{locktime: 100, {c:100, s:0}}
{locktime: 99, {c:99, s:1}}
{locktime: 98, {c:98, s:2}}
{locktime: 97, {c:97, s:3}}
...
{locktime: 90, {c:90, s:10}}

Client收到5批商品后，广播了交易A{locktime: 99, {c:99, s:1}}，被Server发现后，server可以广播最新的交易B{locktime: 90, {c:90, s:10}}
因为B.locktime < A.locktime, 所以B交易会优先被区块链打包确认，这样就可以做到作废A交易了。

### 思考
如果locktime离的太近，被高fee的交易堵塞，一直没有被打包，然后A用较高的fee广播了refundTx，被执行了，怎么办？

> 交易池会替换交易