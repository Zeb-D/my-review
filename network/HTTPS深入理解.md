# HTTPS深入学习

说到Https 大家想到得是Http升级版，保证http以加密方式进行传输。大家需要[深入学习HTTP](./HTTP深入学习.md)

<br>

## 加密方式

- 对称加密：
  - 加密和解密都是用同一个密钥
- 非对称加密：
  - 加密用公开的密钥，解密用私钥
  - (私钥只有自己知道，公开的密钥大家都知道)
- 数字签名：
  - 验证传输的内容**是对方发送的数据**
  - 发送的数据**没有被篡改过**
- 数字证书（Certificate Authority）简称CA
  - 认证机构证明是**真实的服务器发送的数据**。

<br>



## 加密算法

说到了加密方式，肯定得了解下相对应的加密方式会使用哪些常见的加密算法，也会主要比较分析算法：

常见的加密算法 可以分成**三类**，对称加密算法，非对称加密算法和Hash算法。

###常见的对称加密算法：

DES、3DES、DESX、Blowfish、IDEA、RC4、RC5、RC6和AES

DES（Data Encryption Standard）：数据加密标准，速度较快，适用于加密大量数据的场合。

3DES（Triple DES）：是基于DES，对一块数据用三个不同的密钥进行三次加密，强度更高。

AES（Advanced Encryption Standard）：高级加密标准，是下一代的加密算法标准，速度快，安全级别高；

AES与3DES的比较

| 算法名称 | 算法类型        | 密钥长度         | 速度   | 解密时间（建设机器每秒尝试255个密钥） | 资源消耗 |
| ---- | ----------- | ------------ | ---- | -------------------- | ---- |
| AES  | 对称block密码   | 128、192、256位 | 高    | 1490000亿年            | 低    |
| 3DES | 对称feistel密码 | 112位或168位    | 低    | 46亿年                 | 中    |

###常见的非对称加密算法：

RSA、ECC（移动设备用）、Diffie-Hellman、El Gamal、DSA（数字签名用）

RSA：由 RSA 公司发明，是一个支持变长密钥的公共密钥算法，需要加密的文件块的长度也是可变的；

DSA（Digital Signature Algorithm）：数字签名算法，是一种标准的 DSS（数字签名标准）；

ECC（Elliptic Curves Cryptography）：椭圆曲线密码编码学。

RSA和ECC的安全性和速度的比较。

| 攻破时间(MIPS年) | RSA/DSA(密钥长度) | ECC密钥长度 | RSA/ECC密钥长度比 |
| ----------- | ------------- | ------- | ------------ |
| 104         | 512           | 106     | 5：1          |
| 108         | 768           | 132     | 6：1          |
| 1011        | 1024          | 160     | 7：1          |
| 1020        | 2048          | 210     | 10：1         |
| 1078        | 21000         | 600     | 35：1         |

RSA和ECC安全模长得比较

| 功能                 | Security Builder 1.2 | BSAFE 3.0 |
| ------------------ | -------------------- | --------- |
| 163位ECC(ms)        | 1,023位RSA(ms)        |           |
| 密钥对生成              | 3.8                  | 4,708.3   |
| 签名                 | 2.1(ECNRA)           | 228.4     |
| 3.0(ECDSA)         |                      |           |
| 认证                 | 9.9(ECNRA)           | 12.7      |
| 10.7(ECDSA)        |                      |           |
| Diffie—Hellman密钥交换 | 7.3                  | 1,654.0   |

总之，由上面表格可以看出：

ECC和RSA相比，在许多方面都有对绝对的优势：

抗攻击性强。相同的密钥长度，其抗攻击性要强很多倍。

计算量小，处理速度快。ECC总的速度比RSA、DSA要快得多。

存储空间占用小。ECC的密钥尺寸和系统参数与RSA、DSA相比要小得多，意味着它所占的存贮空间要小得多。这对于加密算法在IC卡上的应用具有特别重要的意义。

带宽要求低。当对长消息进行加解密时，三类密码系统有相同的带宽要求，但应用于短消息时ECC带宽要求却低得多。带宽要求低使ECC在无线网络领域具有广泛的应用前景。<br>



###常见的Hash算法：

MD2、MD4、MD5、HAVAL、SHA、SHA-1、HMAC、HMAC-MD5、HMAC-SHA1

散列是信息的提炼，通常其长度要比信息小得多，且为一个固定长度。加密性强的散列一定是不可逆的，这就意味着通过散列结果，无法推出任何部分的原始信息。任何输入信息的变化，哪怕仅一位，都将导致散列结果的明显变化，这称之为雪崩效应。散列还应该是防冲突的，即找不出具有相同散列结果的两条信息。具有这些特性的散列结果就可以用于验证信息是否被修改。

单向散列函数一般用于产生消息摘要，密钥加密等，常见的有：

-  MD5（Message Digest Algorithm 5）：是RSA数据安全公司开发的一种单向散列算法，非可逆，相同的明文产生相同的密文。
-  SHA（Secure Hash Algorithm）：可以对任意长度的数据运算生成一个160位的数值；

**SHA-1与MD5的比较**

因为二者均由MD4导出，SHA-1和MD5彼此很相似。相应的，他们的强度和其他特性也是相似，但还有以下几点不同：

1.  对强行供给的安全性：最显著和最重要的区别是SHA-1摘要比MD5摘要长32 位。使用强行技术，产生任何一个报文使其摘要等于给定报摘要的难度对MD5是2128数量级的操作，而对SHA-1则是2160数量级的操作。这样，SHA-1对强行攻击有更大的强度。
2.  对密码分析的安全性：由于MD5的设计，易受密码分析的攻击，SHA-1显得不易受这样的攻击。
3.  速度：在相同的硬件上，SHA-1的运行速度比MD5慢。

<br>

## 算法与加密选择

###对称与非对称算法比较

上面综述了两种加密方法的原理，也比较部分加密算法优劣。这里讲展开 公钥 与 私钥 选择加密方式与加密算法。

虽然开始本文第一节 进行了加密方式 名词解释，但 这里作个 对称与非对称算法 比较：

- 在管理方面：公钥密码算法只需要较少的资源就可以实现目的，在密钥的分配上，两者之间相差一个指数级别（一个是n一个是n2）。所以私钥密码算法不适应广域网的使用，而且更重要的一点是它不支持数字签名。
- 在安全方面：由于公钥密码算法基于未解决的数学难题，在破解上几乎不可能。对于私钥密码算法，到了AES虽说从理论来说是不可能破解的，但从计算机的发展角度来看。公钥更具有优越性。
- 从速度上来看：AES的软件实现速度已经达到了每秒数兆或数十兆比特。是公钥的100倍，如果用硬件来实现的话这个比值将扩大到1000倍。

总之，由于非对称加密算法的运行速度比对称加密算法的速度慢很多，当我们需要加密大量的数据时，建议采用对称加密算法，提高加解密速度。

对称加密算法不能实现签名，因此签名只能非对称算法。

由于对称加密算法的密钥管理是一个复杂的过程，密钥的管理直接决定着他的安全性，因此当数据量很小时，我们可以考虑采用非对称加密算法。

在实际的操作过程中，我们**通常采用的方式**是：采用非对称加密算法管理对称算法的密钥，然后用对称加密算法加密数据，这样我们就集成了两类加密算法的优点，既实现了加密速度快的优点，又实现了安全方便管理密钥的优点。

那采用多少位的密钥呢？ RSA建议采用1024位的数字，ECC建议采用160位，AES采用128为即可。

<br>

##通讯之路演化

好了，既然说了加密方式、加密算法，那我们来看看HTTP在加密传输走了哪些路：

- HTTP时代：双方（服务端、客户端）传输数据之间没有任何的加密，直接传输
  - 内容被看得一清二楚，毫无隐私可言
- HTTP+ 对称加密 时代：使用对称加密的方式来保证传输的数据只有两个人知道
  - 此时有个问题：**密钥不能通过网络传输**(因为没有加密之前，都是不安全的)，所以双方先告诉对方密码是多少，再数据传输。
- HTTP+ 非对称加密 时代：服务端需要对接各种客户端如网页、app等(同样不想泄漏了自己的通讯信息)。那有那么多终端，难道每一次都要先告诉对方密码？(说明维护多个对称密钥是麻烦的！)--->所以用到了非对称加密
  - 服务端自己保留一份密码，独一无二的(私钥)。告诉其他网页、app、第三方平台等一份密码(这份密码是公开的，谁都可以拿--->公钥)。让他们给我发消息之前，先用那份服务端告诉他们的密码加密一下，再发送给我。我收到信息之后，用自己独一无二的私钥解密就可以了！
- HTTP+ 非对称加密 出现的问题：虽然客户端不知道私钥是什么，拿不到你原始传输的数据，但是可以拿到加密后的数据，他们可以改掉某部分的数据再发送给服务器，这样服务器拿到的数据就不是完整的了。
  - 比如我女朋友给我发了一条信息”东，我喜欢你“，然后用我给的公钥加密，发给我了。此时不怀好意的人截取到这条加密的信息，他**破解不了原信息**。但是他可以**修改加密后的数据**再传给我。可能我拿到收到的数据就是”哼，你今晚跪键盘吧“，这是不是要凉凉了！！！
- HTTP + 数字签名：拿到的数据可能被篡改了，我们可以使用数字签名来解决被篡改的问题。数字签名其实也可以看做是**非对称加密的手段一种**，具体是这样的：得到原信息hash值，用**私钥**对hash值加密，**另一端**用**公钥**解密，最后比对hash值是否变了。如果变了就说明被篡改了。(一端用私钥加密，另一端用公钥解密，也确保了来源)
- HTTP + 数字证书：好像使用了数字签名就万无一失了，其实还有问题。我们使用非对称加密的时候，是使用**公钥进行加密的**。如果**公钥被伪造了**，后面的数字签名其实就毫无意义了。讲到底：**还是可能会被中间人攻击**~此时我们就有了**CA认证机构来确认公钥的真实性**！

大家可能对数字签名、数字证书有点小感冒，其实这是加密方法中的双钥加密。

首先 我们得认识下，信息安全中三个比较重要的特性：

1. 保密性(Confidentiality)：信息在传输时不被泄露
2. 完整性（Integrity）：信息在传输时不被篡改
3. 有效性（Availability）：信息的使用者是合法的

这三要素统称为CIA Triad。

公钥密码解决保密性问题，数字签名解决完整性问题和有效性问题

<br>

## 数字签名、数字证书解读

**双钥加密 原理**：

a) 公钥和私钥是一一对应的关系，有一把公钥就必然有一把与之对应的、独一无二的私钥，反之亦成立。

b) 所有的（公钥, 私钥）对都是不同的。

c) 用公钥可以解开私钥加密的信息，反之亦成立。

d) 同时生成公钥和私钥应该相对比较容易，但是从公钥推算出私钥，应该是很困难或者是不可能的。

在双钥体系中，**公钥用来加密信息，私钥用来数字签名**。

那么**数字签名 流程**：

1）客户端拿着公钥进行数据加密，服务端用私钥进行数据解密。

2）服务端给客户端发信息时，先用Hash函数，生成信息的摘要，然后服务端对这个摘要进行私钥加密，生成“数字签名 signature” ，然后将这个签名 放到消息 的后面， 客户端接受到消息后， 拿到 数字签名，用公钥解密，得到 信息的摘要。由此证明信息是服务端 发出的。

3）客户端 再对信息本身 使用Hash 函数，得到的结果，与服务端传过来的 数字签名 用 公钥解密后的 摘要进行对比，证明 信息是否被 修改过。

**数字签名 方式 并不能完全保证 信息是足够的安全的**，比如：

因为这个 公钥、私钥 虽然是以一对 存在的，但是 只要用一对假的公钥、私钥 分别冒充，将假公钥给客户端，私钥自己伪造一个服务器进行 与客户端进行通信。

这个 数字证书 就出现来解决这 信息安全 问题：如何保证 客户端手上的公钥就是真正的服务端给的？

那么这**数字证书生成过程**：服务端首先会去找 “证书中心” （certificate authority，简称CA），证书中心用自己的私钥 对服务端的公钥和一些相关信息 一起加密，生成 "数字证书"（Digital Certificate）。

这个时候，客户端收到信息后，用CA公钥解开数字证书，就可以拿到 服务端的真实的公钥了，然后就能证明服务端 数字签名 是否是 服务端 签的。

数字证书 是有效期的，也可以进行作废的。当用户私钥丢失、被盗时，认证机构需要对证书进行**作废**(revoke)。要作废证书，认证机构需要制作一张证书作废清单(Certificate Revocation List)，简称**CRL**，因此验证服务端的 数字证书 是否有效 还需要**查询认证机构最新的CRL，并确认该证书是否有效**。

<br>

## Https 请求过程

1、首先，客户端向服务端发出加密（安全连接）请求

2、服务端用自己的私钥加密网页后，连同本身的数字证书，一起发送给客户端

3、客户端（浏览器）的证书管理器，有"受信任的根证书颁发机构"列表。客户端会根据这张列表，查看解开数字证书的公钥是否在列表之内。如果数字证书记载的网址，与你正在浏览的网址不一致，就说明这张证书可能被冒用，浏览器会发出警告。如果这张数字证书不是由受信任的机构颁发的，浏览器会发出另一种警告。

4、如果数字证书是可靠的，客户端就可以使用证书中的服务器公钥，对信息进行加密，然后与服务器交换加密信息。

<br>

## HTTPS 网络解读

说了这么多，HTTPS其实就是在HTTP协议下多加了一层SSL(Secure Sockets Layer 安全套接层)协议（ps:现在都用TLS 传输层安全[Transport Layer Security]协议)，它的网络模型流转如下：

![network-https-connect-flow.png](../image/network-https-connect-flow.png)

可以看出，是在应用层 与 传输层 加了层 抽象层（这在五/七层网络模型 中是不存在的）。

SSL协议可分为两层： 

SSL记录协议（SSL Record Protocol）：它建立在可靠的传输协议（如TCP）之上，为高层协议提供数据封装、压缩、加密等基本功能的支持。 

SSL握手协议（SSL Handshake Protocol）：它建立在SSL记录协议之上，用于在实际的数据传输开始前，通讯双方进行身份认证、协商加密算法、交换加密密钥等。

<br>

## 对比HTTP

HTTPS 传输更加安全

-  所有信息都是加密传播，黑客无法窃听。
-  具有校验机制，一旦被篡改，通信双方会立刻发现。
-  配备身份证书，防止身份被冒充。

<br>

本文章纯手动，请转载注明：https://github.com/Zeb-D/my-review