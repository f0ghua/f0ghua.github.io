# 信息安全技术入门


谈到信息安全，我只能算是个门外汉。但是由于工作需求，经常也会接触一些，所以整理一下备查。顺便也把自己的一些心得分享出来。


## 关键字 {#关键字}

我们在看跟安全相关的文献的时候，最常看到的就是以下的一些词汇。理解它们对于理解相关技术是很重要的。

-   Confidentiality 保密性
-   Encrypt 加密
-   Integrity 完整性（你有没有被篡改过？）
-   Authentication 认证（你是谁？）


## 相关技术 {#相关技术}

所有的技术都是为了满足应用的需求而产生。安全技术也不例外。


### 对称加密算法 {#对称加密算法}

考虑一个场景。A 是一个地下工作者，每隔一段时间，他都要和自己的上线 B 联系传递信息。为了保证信息不被窃听，A 需要把自己的信息加密之后再发送给 B，而 B 要做的就是解密并查看 A 给他的信息。之后 B 同样的也可以加密自己的信息再传递给 A。

这里用到的加解密，最常用的方式就是对称式的加密算法了。也就是双方都拥有一个相同的秘钥（key），然后使用相同的算法来加密或者解密，最终获得明文。

这样的加密算法有很多，通过 `man openssl -> Encoding and Cipher Commands` 可以看到一些常用的算法列表。


### 非对称加密算法（公钥加密） {#非对称加密算法-公钥加密}

这样的方式很简便快捷，但是存在一个问题。那就是 A 的秘钥每次更新的时候都必须通知 B
以进行同步。这个通知的过程就存在着的泄漏的风险。

于是 A B 在商量之后更新了他们的加密策略，采用了公钥加密算法。

公钥加密算法和对称加密算法不同，包含了 2 个 key。一个私钥（private key）和一个公钥（public key）。公钥可以公开发布，用来加密信息；而私钥只会妥善保存好，用来解密公钥加密的信息。因为私钥不会被传播，所以就解决了对称性加密带来的秘钥泄漏的风险。注意，虽然一般情况下我们都是用公钥加密私钥解密，但是逆过来也是可行的。私钥加密的信息，使用公钥同样可以解密。

比较典型的非对称加密算法就是 RSA 了。


### PKI {#pki}

但是怎么确定 B 拿到的公钥是属于 A 的呢？如果中间有人偷偷把 A 发给 B 的公钥换掉了，那不就可以伪装成 A 了吗？

PKI 隆重登场。PKI 中文译为公钥基础设施，它是英文 Public Key Infrastructure 的缩写，基于公钥密码学，建立起一种普遍适用的基础设施，为各种网络应用提供全面的安全服务。

我们可以简单的认为 PKI 体系就是为了沟通的双方安全的拿到对方的秘钥（真实，完整）。

PKI 基本结构由证书认证机构（certificate authority, CA）、证书持有者（certificate
holder）、依赖方（relying party）三方构成

-   CA 是一个独立的可信第三方，为证书持有者签发数字证书，数字证书中声明了证书持有者的身份和公钥。CA 在签发证书前应对证书持有者的身份信息进行核实验证，并根据其核验结果为其签发证书。

-   证书持有者向 CA 申请数字证书，并向 CA 提供必要的信息以证明其身份及能力，获得由
    CA 签发的证书；证书持有者在与依赖方进行交互时，需向依赖方提供由 CA 签发的数字证书证明其有效身份。

-   依赖方是证书的验证方，依赖方与证书持有者进行交互（如建立通信连接）时，需获取证书持有者的数字证书，验证数字证书的真实性和有效性。依赖方可以指定其信任的 CA 列表，若证书持有者提供的数字证书不是受信 CA 签发的数字证书，依赖方将不认可该证书所声明的信息。

所以三者的关系，简单来说就如下图：

```bash

                         +--------------+
               +----->   |      CA      | <----+
               | +----+  +--------------+      |
       request | | identify                    |
   certficiate | | issue certificate           | trust
               | |                             |
               | v                             |
               |            provide            |
           +-------------+  certificate +-------------+
           | certificate | +--------->  | relay party |
           | holder      | <---------+  |             |
           +-------------+  verifiy     +-------------+
                            certificate

```

证书存在的意义在于回答“公钥属于谁”的问题，以帮助用户安全地获得对方的公开密钥。

在理解了 PKI 之后，我们再来看 A B 可以如何交互

-   私钥(key - private key)
-   证书请求(csr - certificate signing request)
-   签名证书(crt - certificate signed request)

<!--listend-->

```bash

                    A                      CA        B

                    +                       +        +
                    |                       |        |
   generate a pair  |                       |        |
           of keys  |                       |        |
                    |  send csr             |        |
                    | +-------------------> |        |
                    |                       |        |
                    |  1. verify identity   |        |
                    |  2. sign the csr      |        |
                    |  3. send back the cer |        |
                    | <-------------------+ |        |
                    |                       |        |
                    |                       |        |
                    |  send A's crt to B    |        |
                    | +----------------------------> | 1. verify with CA's cert
                    |                       |        | 2. store A's public key
                    |                       |        |
                    |                       |        |
                    |         send data to A|        |
   decrypt with A's | <----------------------------+ | encrypt data with
        private key |                       |        | A's public key
                    |                       |        |
                    |                       |        |
                    |                       |        |
                    |                       |        |
                    +                       +        +

```

图示只标注了 B 往 A 发送数据的过程，如果是 A 往 B 传递信息，那么同样的方法，只是换成使用
B 的 key 而已。

在这里面还有几点需要说明的

-   所谓的签名，其实可以简单的理解成把原始证书先做 hash，再使用 CA 的 private key
    对这个 hash 值进行加密，最后添加到证书里面的一个过程
-   证书可以由 CA 签名，也可以自签名
-   证书一般使用 X509 的格式，其中中包含了持有者的公钥和发布者的签名，以及其他的一些表明证书身份相关的信息


#### CA {#ca}

CA 分为 Signed CA 和 Self Signed CA。两者的差别在于其证书（certificate）被认证的方式不同。

Self Signed CA 是指由自己给自己的 certificate 做签名认证的 CA ，由于一般来讲此类
CA 总是处于 CA 链的最顶端，所以又叫 Root CA。

Signed CA 是指 certificate 由其他 CA 进行签名认证的 CA。而认证它的 CA，我们称为
Signed CA 的上层 CA，也叫 Parent CA；而 Signed CA 则称为 Sub CA 或 Child CA。


## 实例说明 {#实例说明}


### PKI {#pki}

下面，就用 openssl 举例生成 RSA 的 CA 和用户证书。


#### 生成 CA 的私钥 {#生成-ca-的私钥}

```bash
[fog@fog-ibm ca] openssl genrsa -des3 -out ca.key 1024

Generating RSA private key, 1024 bit long modulus
...........................+++++
............................................+++++
e is 65537 (0x10001)
Enter PEM pass phrase:
Verifying password - Enter PEM pass phrase:

Please backup this ca.key file and remember the pass-phrase you had to
enter at a secure location.
```


#### 生成 CA 的自我签名证书 {#生成-ca-的自我签名证书}

```bash
fog@tmp$ openssl req -new -x509 -days 365 -key ca.key -out ca.crt
Enter pass phrase for ca.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:Fog
Email Address []:
```

查看一下生成的证书

```bash
fog@tmp$ openssl x509 -noout -text -in ca.crt
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            6c:5b:7a:5c:37:e3:1b:2d:99:2d:99:20:79:54:c4:d6:52:9e:20:17
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = AU, ST = Some-State, O = Internet Widgits Pty Ltd, CN = Fog
        Validity
            Not Before: Jul  5 08:43:44 2022 GMT
            Not After : Jul  5 08:43:44 2023 GMT
        Subject: C = AU, ST = Some-State, O = Internet Widgits Pty Ltd, CN = Fog
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (1024 bit)
                Modulus:
                    00:c2:ad:39:a9:64:a5:85:5b:39:ab:eb:a8:da:82:
                    66:90:60:f9:35:d9:ea:67:4e:6f:d6:99:14:1c:32:
                    6a:23:e4:03:eb:11:44:1a:94:4d:42:a1:74:9e:26:
                    62:10:0b:4b:a8:3c:10:28:32:c9:d9:1b:34:aa:03:
                    5d:d6:63:70:32:68:c0:d2:9d:17:e7:b4:47:6e:6e:
                    86:66:a5:bc:f1:52:4e:2a:47:2a:0e:7b:00:d7:f9:
                    d0:70:c2:12:58:ab:df:b3:82:51:23:cf:a8:59:40:
                    c6:a7:7e:f6:8c:91:1e:00:42:87:d1:42:1c:98:15:
                    4e:ff:1c:38:55:8a:d1:9d:fb
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                3F:46:C9:DA:5C:4E:60:65:CB:48:8E:DF:FE:92:67:2C:03:A4:B0:A1
            X509v3 Authority Key Identifier:
                keyid:3F:46:C9:DA:5C:4E:60:65:CB:48:8E:DF:FE:92:67:2C:03:A4:B0:A1

            X509v3 Basic Constraints: critical
                CA:TRUE
    Signature Algorithm: sha256WithRSAEncryption
         72:98:2c:4b:ac:5d:3e:aa:21:ca:5f:46:1a:b9:14:2c:2f:b8:
         e9:3d:8a:bd:19:ea:da:32:13:fc:02:9c:f7:f2:2b:99:56:20:
         11:86:5b:7c:1b:ea:5a:0e:f4:bd:da:e2:44:ec:b9:71:7b:3a:
         01:a3:d1:dc:bc:fc:52:49:6a:c8:f4:2c:27:45:ab:e6:65:bf:
         75:75:1e:66:56:cc:82:71:95:7f:0b:da:8a:bf:2d:b6:7b:fd:
         72:a2:68:f5:13:59:6f:ac:72:20:bf:59:0c:20:60:3d:46:6e:
         d7:4c:83:90:42:ab:04:b9:e1:e4:9f:1c:7e:1e:2f:96:32:b0:
         89:6e
```

这里我们就可以看到一个 x509 格式的证书包含了哪些字段。这个对于理解证书很重要，所以详细分析一下

```bash
Issuer: C = AU, ST = Some-State, O = Internet Widgits Pty Ltd, CN = Fog
Subject: C = AU, ST = Some-State, O = Internet Widgits Pty Ltd, CN = Fog
```

因为这是一个自签名的证书，所以 Issuer 和 Subject 的内容是一样（Issuer 表示是谁颁发的证书，而 Subject 表示这个证书属于谁）。

```bash
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (1024 bit)
```

Public Key Info 告诉我们这个 Public Key 对应的加密算法是 RSA，秘钥为 1024 bit，然后明文附上了对应的秘钥（注意这是秘钥对里面的 public key）。

```bash
        X509v3 extensions:
            X509v3 Basic Constraints: critical
                CA:TRUE
```

这里可以看到 CA 的值是 TRUE，表示这个证书可以作为 CA 证书使用。

```bash
Signature Algorithm: sha256WithRSAEncryption
```

签名的算法为先用 sha256 做 hash 算法，然后再用 RSA 算法使用 Issuer 的 private key
进行加密。


#### 生成用户的私有密钥 {#生成用户的私有密钥}

```bash
fog@tmp$ openssl genrsa -des3 -out my.key 1024
```

生成用户的签名请求证书

```bash
fog@tmp$ openssl req -new -key my.key -out my.csr
```

由 CA 为请求证书签名

```bash
fog@tmp$ openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -in my.csr -out my.crt -days 365
```

现在再来看看这个生成的证书是什么样的

```bash
fog@tmp$ openssl x509 -noout -text -in my.crt
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number:
            14:a2:7c:94:fa:b8:0c:28:0d:9b:c3:86:44:3a:cf:40:0e:ea:71:c3
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = AU, ST = Some-State, O = Internet Widgits Pty Ltd, CN = Fog
        Validity
            Not Before: Jul  5 09:10:51 2022 GMT
            Not After : Jul  5 09:10:51 2023 GMT
        Subject: C = AU, ST = Some-State, O = Internet Widgits Pty Ltd, CN = User
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (1024 bit)
                Modulus:
                    00:bd:8c:96:bb:a7:3f:39:36:61:fa:41:d1:70:42:
                    0f:b8:36:d4:3a:d2:0a:e1:85:f6:fe:25:d7:92:63:
                    e3:a6:72:70:1d:93:e1:69:70:e7:38:7b:32:49:ae:
                    bc:05:8e:9f:00:dd:9a:9c:c9:9b:62:97:a1:4f:c3:
                    09:d8:5c:af:83:fa:fb:1f:fd:a9:32:74:4f:6b:a4:
                    e7:52:d3:8d:bd:2e:1f:11:55:9e:b1:bf:fe:c6:6f:
                    25:cd:d4:23:10:34:39:26:e2:37:eb:2f:83:84:91:
                    d3:39:b4:d0:e1:4a:ab:c6:fd:96:98:4c:31:9f:b7:
                    17:39:f4:08:09:77:11:51:53
                Exponent: 65537 (0x10001)
    Signature Algorithm: sha256WithRSAEncryption
         47:dd:5e:81:6a:f1:e8:8d:11:86:3d:4b:6e:30:51:64:af:30:
         d4:e8:0d:c9:5e:07:f0:59:1d:f0:64:b9:e9:76:e1:66:61:59:
         f9:b9:bc:21:92:93:db:13:6e:ec:08:3c:7f:6c:a4:96:a3:9e:
         cf:76:fb:f6:8b:52:34:f4:dd:5f:56:97:df:98:c5:53:6c:97:
         18:ef:8c:ce:09:77:43:27:04:f9:14:b2:4d:a1:7f:38:bb:fe:
         4a:aa:45:1a:c3:f5:c5:46:f7:b8:fb:9a:cd:34:4e:1a:a5:3c:
         28:b0:f7:57:28:7d:85:52:da:0f:48:b2:f5:90:7d:80:fa:76:
         07:20
```

通过 Issuer 和 Subject 可以看出，这个证书是由 Fog（CA）颁发给 User 的。

对于 private key 大多数时候我们不需要密码保护，可以通过以下命令来去除它

```bash
fog@tmp$ openssl rsa -in my.key -out myunsecure.key
Enter pass phrase for my.key:
writing RSA key
```


## Reference {#reference}

-   [Difference between self-signed CA and self-signed certificate](https://stackoverflow.com/questions/4024393/difference-between-self-signed-ca-and-self-signed-certificate)
-   [TLS encryption and mutual authentication using syslog-ng Open Source Edition](https://www.linux.com/training-tutorials/tls-encryption-and-mutual-authentication-using-syslog-ng-open-source-edition/)
-   [Making sense of SSL, RSA, X509 and CSR](https://blog.gisspan.com/2016/04/making-sense-of-ssl-rsa-x509-and-csr.html)
-   [A journey into verifying signatures on x.509 certificates](https://medium.com/@ebuschini/a-journey-into-verifying-signatures-on-x-509-certificates-168a5bafaa14)

