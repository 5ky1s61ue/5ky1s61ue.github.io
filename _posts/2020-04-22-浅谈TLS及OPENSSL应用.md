---
layout: post
author: oy
tag: TLS OpenSSL fingerprint CipherSuite HSTS SNI
---
## TLS

### 为什么需要TLS

> 可能更进一步需要首先回答，为什么需要HTTPS/加密？  
通常来说就是为了防止被嗅探，MITM攻击，或者被IPS等中间媒介篡改请求or响应(广告插入等)。  
高密保性行业(比如金融证券)自不必说，随着网络用户隐私越来越受重视，HTTPS只会越来越普及。  
而TLS安全性优于SSL，TLS免费证书也非常容易获取，所以TLS在某种程度上是不可或缺的。

### TLS握手

> 握手，其实个互相认证双方身份的过程。(从这个角度看，现实生活中的握手也会有认证/认可的意义)
TLS握手的过程主要干这么几件事情： 
>
1. 协商使用哪个TLS协议版本。
2. 决定使用哪一套CipherSuites。
3. 通过验证TLS证书的数字签名来对服务端进行认证。
4. 生成TLS会话密钥(握手完成后进行加密通信使用)

> 而TLS握手的具体方式/步骤 和 密钥交换算法(Key Exchange Algorithm) 紧密相连(Kx作为CipherSuites其中一部分，所以和双方支持的CipherSuites自然也很相关)。
	
主流常用的两种Kx算法是RSA和DH，所以基于这两种Kx算法的TLS握手过程也 具有普遍意义。

#### TLS握手-RSA:
>
1. Client hello: 客户端发起握手，携带客户端支持的TLS version、CipherSuites和随机数(client random) 但如果是会话重用的场景，这时候client hello里会携带session id，server验证后直接重用完成握手进行会话了
2. Server hello: 服务端响应client hello的信息，包含服务端的SSL证书、服务端选择的CipherSuites和随机数(server random)，服务端在设置了会话重用的情况下还会发送session id用于会话重用
3. Authentication: 客户端认证服务端的SSL证书(通过向签发证书的CA确认)
4. The premaster secret: 客户端发送 预主密钥(一个或多个随机字符串构成，根据证书公钥加密生成，因此只能被服务端保存的配对私钥进行解密)
5. Private key used: 服务端使用私钥解密 premaster secret
6. Session keys created: client和server两端根据 client random、server random、premaster secret 这三个因子 和约定好的 CipherSuites中的Enc算法生成session keys，所以得到的keys是一致的，后面的加密通信都用这个keys进行加解密就OK了
7. Client is ready: client发送被session key加密的'finished'信息，告诉server准备会话通信了
8. Server is ready: server同上，告诉client准备会话通信了
9. Secure symmetric encryption achieved: 完成安全对称加密(握手完成)，全程采用session key进行加密通信。 后续client就可以请求它想要的资源了

#### TLS握手-DH:

值得一提的是，要支持DH方式握手那么服务端要指定/配置好 dh parameter

>
1. Client hello: 同RSA
2. Server hello: 同RSA, 但是又多了下面三个步骤  
	* Server的数字签名: Server用私钥加密(client random,server random, 和server上的DH parameter）这些数据生成一个 数字签名，从而确定Server有与Cert中公钥匹配的私钥
	* 数字签名确认: Client用公钥解密数字签名，从而确认Server掌控私钥。同时把Client的DH parameter发送给Server。
	* 两端计算预主密钥: 与RSA中客户端生成并发送预主密钥不同，两端利用互相交换的DH parameters来分别计算得出同样的预主密钥。
3. Session keys created: 同RSA
4. Server is ready: 同RSA
5. Server is ready: 同RSA
6. Secure symmetric encryption achieved: 同RSA

	（gᵃ mod p)ᵇ mod p = (gᵇ mod p)ᵃ mod p



### Forward Secrecy/Perfect Forward Secrecy

> 向前安全性，指的是长期使用的主密钥泄露不会导致过去的会话密钥泄露。  
> 典型的例子是RSA和DH两种算法，如果私钥被窃取了就可以根据握手信息(包括预主密钥)，解密以前建立的RSA会话(Wireshark抓包解密https也是这个逻辑)

> 但是DH不同，因为交换dp params时，根据DH算法特性，握手时
（gᵃ mod p)ᵇ mod p = (gᵇ mod p)ᵃ mod p
a b在两端私有，不暴露在握手过程，所以即便嗅探或者抓包了握手信息，有私钥也无法解密得到会话密钥，所以这个角度看，DH具备向前安全性。

> FS/PFS(向前安全性)这个概念很早就出现了，但在[斯诺登爆料了NSA的https破解方案后](https://thehackernews.com/2016/10/nsa-crack-encryption.html)，才使得https这个方面的问题更受重视/流行起来 DH->ECDH, DHE->ECDHE

### CipherSuites
> TLS可算作是个'混合'加密系统(同时使用对称与非对称加密算法)在CipherSuites上就能很好地体现出来，这样的'混合'方案，也普遍用于互联网上的其他协议(比如SSH,IPsec,WireGuard等)。

> 一个CipherSuites包含四部分算法内容(key exchange + authentication + encryption + message authentication code)，格式为Kx-Au-Enc-Mac

#### Kx
* 这个算法的目的看名字便一目了然，就是用于交换client和server两端的密钥(RSA中体现的是Server给的公钥和Client计算后给的预主密钥)，最后协商得出主密钥(在TLS场景体现为session key)。  
* 采用的是 非对称加密(对于少量数据能保持一个较好的性能)  
* 常见的Kx算法便是上面提到的RSA和DH，但是根据上述握手流程不难看出，DH做的事情其实用 密钥协商 描述更准确(RSA中要交换公钥和预主密钥)


#### Au
* 认证算法在CipherSuites内并非必须，因为可以选择不对对端进行认证。
* 用于Client认证Server的签名，或者Server认证Client的签名所使用的算法。
* 常见的Au算法有RSA, DSA, ECDSA三种，主流自然是RSA(2048bit以上)，而ECDSA是趋势，gmail, facebook都在往这迁移，DSA只能提供1024bit，所以不够安全。那么，使用了不同的Au算法，会导致证书内容也不一样。所以，在业务场景上来说，要支持某些CipherSuites，和使用的证书也是强相关的(所以我在想对于无SNI需使用default cert的场景，是不是也可以准备好三份不同的cert)。

#### Enc
* 用于加密传输数据。采用对称加密算法比如AES(处理大量数据时也能保持较好的性能)

#### Mac
* 此算法用于校验数据完整性，来保证数据在传输过程没被改变。
* 采用的是相关hash算法来生成信息摘要。

### Elliptical curves
>
这个一般在client hello的extension中指明支持的elliptical curve。  
>
elliptical curve cryptography(ECC)是用作公钥加密的，利用椭圆曲线背后的数学原理来保证密钥对之间的安全性。  
>
因为基于椭圆曲线的方程式容易正向计算，但是逆向很难，所以在密码学中很有价值。(高安全性+低成本)  
>
ec一般用于ECDSA和ECDH算法上。  
>
这个extension信息，其实可以和client ciphersuites结合起来一起做指纹追踪用户的，这里就不展开了。

### 会话重用
>
对于7层代理，负载均衡等场景，https加解密也是对算力消耗的一个大头.
>
为了减少对CPU的消耗，Intel提供了QAT卡把计算成本迁移到这儿来做专门的处理，提升了节点对https处理的效率，当然这是另一个话题。
>
在https算力消耗中握手中的加解密是占比非常大的，因此能做到ssl 会话重用，对效率提升是很明显而成本较低的。

#### Session ID
* 
Session ID是最早的tls session resumption方案，客户端如果支持session id，需要发送一个session id extension，服务器收到后如果也支持session cache，那么就会在server hello里响应后续进行resumption的session id，这样两端存储好这个session id(两端以这个session id为key，存储session key)


#### Session Ticket
* 
Session ID的带来效率提升的同时也引入了一个问题，Server端是集群的话就需要同步这个Session Cache，而且节点相关进程重载后，Session Cache一般在内存中维护，那么Session Cache也就丢失了。(当然也有解决方案，比如前面搞个四层负载做源IP一致性hash，Session Cache放到一个高效的DB里存储等)
同时HTTP本身是个无状态协议，Session Cache却要维护状态，有些矛盾。所以估计是参考了Cookie(个人猜测)，使用Session Ticket来做会话跟踪。
Session Ticket保存了session状态信息(比如握手用的CipherSuites和master secret)，而且Ticket本身也是加密的，这个加密的密钥只有服务端知道(指定密钥文件或者随机生成)。
这样即便同样的client被负载到不同的节点，只要节点都用同一份上面说的加密的密钥(随机生成可能就有坑了)，那么就可以无需同步做到Session Resumption了。

> 握手过程获取新的session tikcet:

	 Client                                           Server  
	
	 ClientHello  
	(empty SessionTicket extension)-------->  
	                                                 ServerHello  
	                             (empty SessionTicket extension)  
	                                                Certificate*  
	                                          ServerKeyExchange*  
	                                         CertificateRequest*  
	                              <--------      ServerHelloDone  
	 Certificate*  
	 ClientKeyExchange  
	 CertificateVerify*  
	 [ChangeCipherSpec]  
	 Finished                     -------->  
	                                            NewSessionTicket  
	                                          [ChangeCipherSpec]  
	                              <--------             Finished  
	 Application Data             <------->     Application Data  


* 使用ticket重用会话，ticket无效等握手场景可参考[RFC相关文档](https://tools.ietf.org/html/rfc5077)(当网上各种说法不一致时，看官方文档还是最靠谱)

#### ID vs Ticket
* Session ID
	* 优点
		1. Server根据ID直接索引，无需解密
	* 缺点
		1. Server端需要同步和存储会话状态
		2. 不具有向前安全性。(如果session cache泄露还是有被解密会话的风险)

* Session Ticket
	* 优点
		1. 无需同步和存储会话状态

	* 缺点	
		1. Server需根据密钥解密ticket
		2. 集群场景密钥需保持一致，这样就有密钥更新的问题
		3. 不具有向前安全性，所以要定期更新密钥
		4. 某些老版本客户端不支持，比如win7 IE

> 在业务场景来看，如果不考虑或解决了同步问题(前面做好路由)，用Session ID还是比较方便，cache可以放在内存里；如果不好解决同步问题或者大业务量下cache带来瓶颈，Session Ticket也是个不错的选择。

### Mixed Content问题
> 
某些站点可能是http与https的杂合体，比如一些关键api可能是https，但是一些资源链接是http。从浏览器去访问，一般地址栏都是https，用户感觉是个加密安全链接，但是其实不是所有资源内容全部采用了https，所以还是有潜在的安全风险。
具体细分，又分为

* 被动混合内容: 仅限于无法与页面其他部分交互的内容，如图片，视频等 
* 主动混合内容: 可以与页面进行交互的内容，如JS，API请求等
明显主动混合内容危害更大。

> 对于Mixed Content问题，浏览器一般会提供告警或阻止相关动作。

### TLS1.3

#### Why TLS1.3
> 
简单而言，TLS1.3更安全，更高效。
具体体现在

1. 抛弃了一些老的，不安全的加密算法。
2. 改善了握手流程从而升了效率(初次握手2RTT->1RTT，会话复用1RTT->0RTT)。
3. 对整个握手过程进行签名。
TLS1.3的Kx仅支持DH，
但是新的技术往往也意味着更多的未知和风险，比如最近爆出的openssl 高危漏洞就和实现TLS1.3相关(当然漏洞本身是实现问题而不是协议问题)[https://nvd.nist.gov/vuln/detail/CVE-2020-1967]。

#### 0 RTT
> 
通过Session ID和Session Ticket，可以把TLS会话RTT减到1，而TLS1.3，却能实现0 RTT。
>
当使用TLS1.3时，会生成一个PSK(pre-shared key)，Server用Session ticket的密钥加密PSK，作为Session Ticket返回给Client。当会话复用时，Client会直接用PSK加密HTTP请求，把加密内容发给Server，Server解密PSK，再用PSK解密数据得到HTTP请求，同时用PSK加密HTTP响应。从而，实现了0RTT。
>
同时，PSK也使用Session ID(那么两端就以Session ID为索引同时存储好PSK就行)
>
但是，PSK依然不具有FP，还是因为Session Ticket的秘钥问题。
>
而且，0RTT情况下给重放攻击提供了可能(不使用0RTT可以利用会话重用的一次握手防止重放攻击,因为reuse时会用新的client random和server random)。


### 证书及证书链
>
证书可以自签发(利用Openssl)或者购买CA机构授权的证书。
证书的验证用到上面CipherSuites中的Au算法，具体而言，证书验证过程中，包含这么几个过程：

1. 验证数字签名: 
	* 签名的生成一般是对消息进行hash，再用私钥对hash value进行加密生成签名；验证则从收到的消息中提取签名，用公钥解密签名得到hash value1，同时用同样的hash算法对消息正文hash，得到hash value2，对比hash value1和hash value2，相同则验证成功，从而证明证书合法。
	* 上面验证了数字签名，但是如何保证公钥是合法有效的呢？这里就需要客户端去CA(Certification Authority)来确认对应的公钥确实属于对应的域名，所以这里就会引入证书链的问题。
	因为每一张证书都是上级CA证书签发的，直到根证书，所以当客户端验证一张证书时，需要有上级证书来验证，一直到客户端内置的根证书。所以，在实际业务中，通常客户端可能会缺少中间证书，这时候比较好的解决办法是把服务端的证书内容填充上中间证书。但是如果客户端没有此证书的根证书，这种场景估计只能由客户端适配了(安装根证书)
2. 验证证书时效性: 直接验证证书内容就可以得到，但是证书内容是否可信，就依赖步骤1证书是否有效，以及步骤3证书是否被吊销。
3. 验证证书是否有效/被吊销: 一般私钥被盗窃，遗失，证书过期，就可能被吊销；这个需要去CA确认，一般有两种方法: 
	* CRL，即证书吊销列表(相当于一个黑名单)，CA定期发布，客户端需要定时同步这个信息。
	* OCSP，即在线证书状态查询协议，虽然是即时的，但是客户端要自己实现查询动作，而且有网络消耗。

### keyless
>
拓展的有些远，keyless应该是CloudFlare最先提出的概念并进行了实践。
>
背景是为了解决客户对流量代理不信任的问题，不愿意把私钥托管在不信任的节点上(特别是银行这类客户)，所以在tls握手中插入了一个中间人的过程，代理节点拿到client相关信息后，与远端(私钥托管服务器)进行交互，最终获得session key完成和client握手。

### HSTS和SNI的一些业务场景
> 
HTTP Strict Transport Security  
HSTS Preload List    
Server Name Indication

### 一些重要的SSL/TLS 漏洞
> 
RSA key transport: Doesn’t provide forward secrecy
>
CBC mode ciphers: BEAST and Lucky 13 attacks
>
RC4 stream cipher: Not secure for use in HTTPS
>
Arbitrary Diffie-Hellman groups: CVE-2016-0701
>
Export ciphers: FREAK and LogJam attacks
	
## OpenSSL

### 支持全量CipherSuites的OpenSSL
>
`enable-weak-ssl-ciphers enable-deprecated enable-rc4 enable-ssl3 enable-ssl3-method`
>
`./config --prefix=/root/openssl shared enable-weak-ssl-ciphers enable-deprecated enable-bf enable-blake2 enable-camellia enable-cast enable-cmac enable-cms enable-comp enable-des enable-dh enable-dsa enable-ec enable-ecdh enable-ecdsa enable-ec_nistp_64_gcc_128 enable-idea enable-md2 enable-md4 enable-mdc2 enable-ocsp enable-poly1305 enable-rc2 enable-rc4 enable-rc5 enable-rfc3779 enable-seed enable-siphash enable-sm2 enable-sm3 enable-sm4 enable-srp enable-ssl3 enable-ssl3-method enable-tls enable-ts`
	
### 如何使用好OpenSSL命令行
>
发起握手请求:
`openssl s_client -connect 127.0.0.1:443 -servername "test.com"`
开启tls server:
`openssl s_server -accept 443 -cert myserver.crt -key myserver.key -www`
生成自签名证书: 
`openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem`

## 主要参考
1. https://www.cloudflare.com/learning/ssl
2. https://builders.intel.com/docs/networkbuilders/understanding-the-ssl-tls-adoption-of-elliptic-curve-cryptography.pdf
3. https://www.ibm.com/developerworks/cn/linux/l-cn-sclient/index.html
4. https://www.ibm.com/support/knowledgecenter/zh/SSWHYP_4.0.0/com.ibm.apimgmt.cmc.doc/task_apionprem_gernerate_self_signed_openSSL.html
5. http://openssl.6102.n7.nabble.com/How-to-enable-RC4-in-OpenSSL-1-1-0c-td69662.html
