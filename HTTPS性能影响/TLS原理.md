# TLS原理

Transport Layer Security (TLS) 即传输层安全，用以实现互联网上的安全通信。其处于传输层协议和应用层协议之间，可以用于身份验证和加密传输等。

## 设计目标

* **加密** 混淆数据的机制
* **身份验证** 验证身份标识有效性的机制
* **完整性** 检测消息是否被篡改或伪造的机制

对于互联网上的通信，私密性指确保没有除通信双方以外的第三者能读懂通信内容，TLS协议中通过`symmetric encryption`(对称加密算法如`AES`)。

身份验证指确保通信对方的身份与其标识的身份一致，在TLS中通常使用`asymmetric encryption`(非对称加密算法)实现。网站使用`证书`和`公钥加密算法`向用户代理证明身份。用户代理需要从两方面来信任服务器证书：一是确保服务器是该证书的拥有者；二是确保该证书是受信任的。

一个网站的证书中包含了该网站服务器的公钥，网站使用私钥进行签名，用户代理通过证书中的公钥就能验证该证书来自特地网站；如果一个证书由可信任的证书颁发机构颁发，且包含正确的主机名，用户代理认为它是值得信任的（涉及到证书链认证）。

使用 `MAC` (Message Authentication Code，消息认证码)签署每一条消息。`MAC `算法是一个单 向加密的散列函数(本质上是一个校验和)，密钥由连接双方协商确定。只要发送 TLS 记录，就会生成一个 MAC 值并附加到该消息中。接收端通过计算和验证这个`MAC`值来判断消息的完整性和可靠性。

Web上，通过`TLS握手`来实现上面的过程。

关于基础密码学概念如：对称加密、非对称加密、数字签名等请查阅相关资料或者书籍《深入浅出密码学》。

关于数字签名还发现一个比较好的图文解释：[http://www.youdzone.com/signature.html](http://www.youdzone.com/signature.html)

## TLS握手

主要有两种不同的握手协议：一种基于`RSA`，一种基于`Diffie-Hellman`。它们在对称密钥交换和身份验证方式上稍有差别。两种方案都有自己的优缺点。如果证书是`RSA`证书，则RSA握手的计算更快。通常，非对称加密算法比对称加密算法慢得多，也会消耗更多的CPU。因此非对称加密算法通常只用于数字签名；对于具有应用数据，还是使用对称加密更好。

DH握手虽然需要使用两种不同的算法，但是它具有前向保密性，每次对称密钥的生成独立于服务器私钥，因而就算服务器私钥泄漏，之前的通信流量也不会全部暴露。另外，对于非RSA证书使用DH握手性能更好。

### 名词解释

1. `Session Key`

握手的产出结果，用于握手后通信双方对称加密通信数据。

2. `Client random`

32字节的序列，每次连接都不一样，通常由4字节的时间戳加上28字节的随机数组成。

3. `Server random`

与`Client random`类似，由服务端产生。

4. `Pre-master secret`
预主密钥。用于和上述的`Client random`和`Server random`通过`“pseudorandom function” (PRF)`共同生成`Session key`。

5. `Cipher suite`
密码套件，由客户端和服务端共同协商决定使用的密码套件。涉及到：

* 身份验证方法
* 密钥交换方法
* 加密算法 4 
* 加密蜜钥大小
* 密码模式(可应用时) 5  MAC算法(可应用时)
* PRF(只有TLS 1.2一定使用，其他版本取决于各自协议)
* 用于Finished消息的散列函数(TLS 1.2)
* verify_data结构的长度(TLS 1.2)

### RSA握手

#### Message 1: “Client Hello”

`client hello`包括了期望的TLS协议版本、`client random`和支持的密码套件清单等。

#### Message 2: “Server Hello”

服务端收到`client hello`后，选择合适的密码套件，`server hello`包括`server random`、选择的密码套件、服务器证书。该证书中包含了服务器的公钥和主机名。

#### Message 3: “Client Key Exchange”

客户端验证证书是由信任的证书颁发机构颁发，客户端创建随机的`pre-master secret`，生成的`secret`会使用服务器的公钥进行加密，并发送给服务端。

服务器收到该`secret`后，使用自己的私钥解密得到`pre-master secret`，这样通信双方都有了`pre-master secret`，且都有了`client random`和`server random`，这样它们就能够生成同样的`session key`。然后双方会交换一个短的信息用于表示接下来的通信将使用这个`session key`用于对称加密。

当双方交换`Finished`信息后，握手完成。分为`client finished`和`server finished`，它们都由`session key`加密，之后，通过对称加密通信。

此种方式，实现优雅，将身份验证和密钥交换集成到同一个步骤。

不足的是，该方式没有前向保密性。假设某个第三者记录了通信双方的握手和通信过程，当第三者获取了服务端的私钥，就可以解密得到每次握手的`session key`，从而获取所有通信内容。即此种方式的安全性依赖于服务端的私钥安全性。


### Ephemeral Diffie-Hellman handshake

`Diffie-Hellman`握手使用两种机制：一个用于密钥交换，一个用于验证服务器。其主要的特征是使用`Diffie-Hellman`密钥交换算法。

具体握手过程这里不再详述。请参考具体规范或其他文章。