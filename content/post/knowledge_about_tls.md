---
date: 2022-03-26 13:22:45
title: TLS 协议工作原理
tags:
  - "OpenSSL"
  - "TLS"
  - "OCSP"
draft: false
---

这篇笔记用来记录 TLS 协议的工作原理及其相关计算机密码学的知识。

<!--more-->

```bash

                                       (@@) (  ) (@)  ( )  @@    ()    @     O     @     O      @
                                  (   )
                              (@@@@)
                           (    )

                         (@@@)
                       ====        ________                ___________
                   _D _|  |_______/        \__I_I_____===__|_________|
                    |(_)---  |   H\________/ |   |        =|___ ___|      _________________
                    /     |  |   H  |  |     |   |         ||_| |_||     _|                \_____A
                   |      |  |   H  |__--------------------| [___] |   =|                        |
                   | ________|___H__/__|_____/[][]~\_______|       |   -|                        |
                   |/ |   |-----------I_____I [][] []  D   |=======|____|________________________|_
                 __/ =| o |=-~O=====O=====O=====O\ ____Y___________|__|__________________________|_
                  |/-=|___|=    ||    ||    ||    |_____/~\___/          |_D__D__D_|  |_D__D__D_|
                   \_/      \__/  \__/  \__/  \__/      \_/               \_/   \_/    \_/   \_/

```

## 计算机密码学算法

TLS 协议的是结合了多种计算机密码学算法实现的，它用来实现传输过程中的数据安全。

在计算机密码学中，经常使用的有对称加密算法，非对称加密算法和散列算法。

对称加密算法用来加密数据，它的特点是数据加密和解密需要使用相同的密钥，所以密钥泄露时，加密数据就会被轻松破解，实际应用的对称加密算法有 DES 和 AES 。

非对称加密算法则更特殊，使用时需要预先生成密钥对，在通信时保留私钥，把公钥分发给对方，将信息使用私钥进行加密，这样对方就可以使用公钥对加密信息进行解密。这种加密方式的特点在于使用公钥加密后，只能通过对应的私钥进行解密，保证了只有通信双方能获取到加密信息。实际应用的非对称加密算法有 RSA 和 DSA 。

散列算法也就是哈希 hash 算法，主要用来验证数据的完整性。它的特点是对不定长的数据总是可以计算出定长的散列值，并且这个过程是单向不可逆的。即使是很小的数据改动，散列值总是不相同的。我们利用这部分散列信息来验证的数据的完整性，实际应用的散列算法有 MD5 和 SHA 。

## 如何实现数据安全

在前面密码学的理论支持下，我们需要做到以下三点来确保数据安全：

1. 数据不被泄露。
2. 数据是真实的，未被篡改破坏的。
3. 数据是来源可靠的。

实现第一点需要对数据进行加密，基于前面的理论，我们应该优先使用的是非对称加密算法，但实际中是结合了两种加密算法来实现。

实现第二点需要对数据进行校验，在实际应用中，我们通过散列算法计算出散列值，这个值通常称为信息摘要，再通过非对称加密算法对信息摘要进行加密，得到的加密信息称为数字签名。在验证数据真实完整性时更多是使用加密后的数字签名，而不是单纯的信息摘要。

由于我们并无法直接确认通信目标就是可信的，为了实现第三点，在这里引出了证书的概念。

虽然我们无法保证通信方是百分百可信的，但是我们可以选择性地信任某些权威机构，并委托这些机构去认证通信来源，如果这些机构认可通信来源，就会给它颁发证书。当我们与对方通信时，如果它出示的证书来自我们信任的权威机构，那么我们就可以信任对方并与之通信。

权威机构 Certificate Authority 自身也需要证书，并且我们需要主动信任 CA 证书，由这个 CA 签发的系列证书才能被正常信任，各大认证机构的 CA 证书通常已经内置于操作系统，随意信任来历不明的 CA 证书是非常危险的。

在三点要求满足的情况下，我们可以认为已经实现了数据安全。

## 生成 CA 证书和 TLS 证书

前面我们已经描述了证书在通信过程中的重要地位，其中 CA 机构自身使用的证书和签发出去的 TLS 证书是略有区别的，但它们都遵循 X.509 标准。

CA 证书包括了该机构的组织信息，持有的公钥和信息的数字签名等，我们需要 CA 证书的重要原因是验证时需要使用 CA 证书中的公钥去验证 TLS 证书的数字签名。

TLS 证书包括了该站点的组织信息，域名信息，站点所使用的公钥和信息的数字签名等， TLS 证书的公钥会用于和使用该证书的站点通信。

签发证书的通常过程是这样的：

1. 域名拥有者将域名信息，申请者所属组织等必要信息生成签发证书的请求，给到具体的 CA 机构。
2. CA 对签发请求的信息生成信息摘要，并使用 CA 的私钥生成数字签名，再附上 CA 自身的信息，生成证书文件。

在实际应用中，我们可以直接自行签发 TLS 证书，也可以通过自建 CA 来签发需要的 TLS 证书。

```bash
# 自签名 TLS 证书

# 生成私钥
$ openssl genrsa -out key.pri 2048
# 以 www.ibm.com 为例，生成证书签发请求
$ openssl req -new -sha256 -key key.pri \
 -subj "/C=US/ST=IL/L=Chicago/O=IBM Corporation/OU=IBM Software Group/CN=www.ibm.com" -out request.csr
# 生成自签证书，由于它不会被自动信任， signkey 可以重新生成或者直接沿用自身的私钥
$ openssl x509 -req -sha256 -days 365 -in request.csr -signkey key.pri -out ibm.crt

# 上述例子生成的证书就是 RSA 证书，下面的例子是生成 ECC 证书
# 查看当前 openssl 版本支持的椭圆曲线
$ openssl ecparam -list_curves
# 一般使用 prime256v1 来生成私钥
$ openssl ecparam -genkey -name prime256v1 -out key.pri
$ openssl req -new -sha256 -key key.pri \
 -subj "/C=US/ST=IL/L=Chicago/O=IBM Corporation/OU=IBM Software Group/CN=www.ibm.com" -out request.csr
$ openssl x509 -req -sha256 -days 365 -in request.csr -signkey key.pri -out ibm.crt
```

自签名意味着自身就是签发的 CA 机构，在访问使用自签名 TLS 证书的站点时，通常会显示证书不受信任的提示，手动把这个自签名证书加入到操作系统的信任链中后这个提示就不会再出现了。但是将单一站点的 TLS 证书加入到信任链中是非常少见的操作，一般只会直接信任 CA 证书。

所以如果需要多个自签 TLS 证书，则自建 CA 会更方便，它与自签名 TLS 的区别在于不随意使用私钥去签名，而是固定一对密钥，把它持续用于后续 TLS 证书的签发，并且将固定公钥生成 CA 证书，那么只需要信任 CA 证书就可以自动信任它所签发的 TLS 证书。

```bash
# 自建 CA

# 生成私钥
$ openssl genrsa -out key.pri 2048
# 生成证书签发请求
$ openssl req -new -sha256 -key key.pri \
 -subj "/C=US/ST=IL/L=Chicago/O=IBM Corporation/OU=IBM Software Group/CN=IBM Root Certificate" -out request.csr
# 生成 CA 证书
$ openssl x509 -req -sha256 -days 365 -in request.csr -signkey key.pri -out CA.crt

# 使用自建 CA 签发 TLS 证书

# 站点生成证书签发请求
$ openssl genrsa -out server_key.pri 2048
# 这里可以通过配置多个 CN ，也可以将单个 CN 指定类似 *.software.ibm.com 的泛解析来满足多域名的情况
$ openssl req -new -sha256 -key server_key.pri \
 -subj "/C=US/ST=IL/L=Chicago/O=IBM Corporation/OU=IBM Software Group/CN=software.ibm.com" -out request.csr
# CA 处理签发请求，生成证书
$ openssl x509 -req -sha256 -days 365 -in request.csr -CA CA.crt -CAkey key.pri -CAcreateserial -out software.crt
```

> 在签发 CA 证书时最好通过 `-extfile <(printf "basicConstraints=CA:TRUE")` 表明自身是用作 CA 证书用途。

在 Go 1.15 之后的版本，进行 tls 握手时发生报错 `"x509: certificate relies on legacy Common Name field, use SANs or temporarily enable Common Name matching with GODEBUG=x509ignoreCN=0"` 是因为当前使用的证书依赖于 CN 作为域名绑定，需要使用 x509 拓展字段 Subject Alternative Name 才能进行正常验证，也就是报错信息所述的 SAN 。

```bash
# 生成 SAN 证书

# 前序步骤和使用 CN 的证书相似
# 生成私钥
$ openssl genrsa -out server_key.pri 2048
# 生成证书签发请求
$ openssl req -new -sha256 -key server_key.pri \
 -subj "/C=US/ST=IL/L=Chicago/O=IBM Corporation/OU=IBM Software Group/CN=software.ibm.com" -out request.csr
# 使用 CA 处理签发请求，生成 SAN 证书
$ openssl x509 -req -sha256 -days 365 -in request.csr -CA CA.crt -CAkey key.pri -CAcreateserial \
 -extfile <(printf "subjectAltName=DNS:www.software.ibm.com,DNS:software.ibm.com,IP:1.1.1.1") -out software.crt

# 如果是高版本的 openssl 可以尝试这样做
$ openssl x509 -req -sha256 -days 365 -in request.csr -CA CA.crt -CAkey key.pri -CAcreateserial \
 -addext "subjectAltName=subjectAltName=DNS:www.software.ibm.com,DNS:software.ibm.com,IP:1.1.1.1" -out software.crt

# 可以看到实际的签发方式和使用 CN 的证书类似，但是使用了 x509 ext 字段，这样可以扩展证书的泛用性，匹配多个域名和 IP 地址
# 签发之后就可以在证书的 X509v3 extensions 看到这部分信息
```

通过信任上述例子中的 `CA.crt` ，后续如 `software.crt` 等由 `CA.crt` 签发的证书都是自动授信的。

```bash
# 在 CentOS 中添加信任 CA
$ mv TrustCA.crt /etc/pki/ca-trust/source/anchors/
$ update-ca-trust

# 执行 update-ca-trust 后应该可以在 ca-bundle.crt 的文件开头看到新增的信任 CA
$ cat /etc/pki/tls/certs/ca-bundle.crt
```

以下是 `openssl` 的一些常用命令以供参考：

```bash
# 查看证书生成请求
$ openssl req -in request.csr -text -noout

# 查看证书内容信息
$ openssl x509 -in x509.crt -text -noout

# 生成 4096 bit 长度的 rsa 私钥，可选的长度还有 2048 ， 1024 ， 512
$ openssl genrsa -out key.pri 4096
# 检查生成的私钥
$ openssl rsa -in key.pri -check

# 提取对应的 rsa 公钥
$ openssl rsa -in key.pri -pubout -out key.pub

# 对文件进行数字签名
$ openssl dgst -sha256 -out sign.txt -sign key.pri file

# 使用公钥和私钥分别验证数字签名
$ openssl dgst -sha256 -prverify key.pri -signature sign.txt file
$ openssl dgst -sha256 -verify key.pub -signature sign.txt file

# 查看证书过期时间
$ openssl x509 -in x509.crt -noout -enddate
# 配合 s_client 可以通过发起 ssl 连接来测试证书是否过期
$ echo | openssl s_client -servername www.github.com -connect "www.github.com:443" 2>/dev/null | \
  openssl x509 -noout -enddate 2>/dev/null | awk -F '=' '{print $2}'

# 证书格式转换
$ openssl x509 -in x509.der -inform der -outform pem -out x509.pem

# 追踪 tls 握手过程
# -status 选项会在 clientHello 中带上 status_request extensions ，用于要求服务端附带 ocsp 响应信息
# -tlsextdebug 可以看到 tls 扩展信息
$ echo | openssl s_client -status -tlsextdebug -servername www.github.com -connect "www.github.com:443" 2>/dev/null

# 指定加密算法的 tls 握手请求，主要用来测试 ECC 证书，对应前面的不同种类证书的生成命令
# RSA 证书测试，理论上可以在输出中追踪到 rsa 的相关证书信息
$ echo | openssl s_client -cipher ECDHE-RSA-AES128-SHA256 -status -tlsextdebug \
  -servername www.ibm.com -connect "127.0.0.1:443"
# ECC 证书测试，理论上可以在输出中追踪到 ecdsa 的相关证书信息
$ echo | openssl s_client -cipher ECDHE-ECDSA-AES128-SHA256 -status -tlsextdebug \
  -servername www.ibm.com -connect "127.0.0.1:443"

# 查看 openssl 对各个版本 tls 协议的算法支持
$ openssl ciphers -v
# cipher 遵从对应的 RFC 命名标准，例如在 ECDHE-RSA-AES128-SHA256 中：
# ECDHE 指的是 Key Exchange 算法， ECDHE 是 DH 算法基于椭圆曲线的变种
# RSA 指的是认证算法， RSA 是非对称加密算法
# AES128 指的是用于 session 过程中的加密算法， AES128 是 128 bit 的对称加密算法
# SHA256 指的是摘要算法， SHA256 是 256 bit 的摘要算法
```

## 计算公钥指纹

在安卓开发过程中的 SSL pinning 会需要用到证书公钥指纹，可以通过私钥或者证书计算得出。

```bash
# 从私钥提取对应的公钥
$ openssl rsa -in key.pri -pubout -out key.pub

# 从证书中提取对应的公钥
$ openssl x509 -in x509.crt -noout -pubkey -out key.pub

# 计算公钥指纹
$ openssl asn1parse -noout -in key.pub -inform pem -out key_der.pub
$ openssl dgst -sha256 -binary key_der.pub | openssl enc -base64
```

## 运行 OCSP Responder

`openssl` 可以运行一个用于响应 OCSP 请求的服务端，能够满足一些基本的测试要求，实际中 OCSP 具体如何在 Nginx 中进行应用则可以参考[这里](https://yuweizzz.github.io/post/practical_tips_in_nginx/#%E5%BC%80%E5%90%AF-ocsp-stapling)。

```bash
# 可以参照前面的例子生成证书签发请求，在具体签发 CA 证书时，需要主动声明证书用途
$ openssl x509 -req -sha256 -days 3650 -in ca.csr -signkey ca.key \
  -extfile <(printf "basicConstraints=CA:TRUE") -out ca.crt

# 具体证书则应该带有 OCSP Responder 信息，这里通过 set_serial 来设置序列号，方便测试
$ openssl x509 -req -sha256 -days 3650 -in x509.csr -CA ca.crt -CAkey ca.key -set_serial 1 \
  -extfile <(printf "authorityInfoAccess=OCSP;URI:http://localhost:8080/") -out x509.crt

# 还需要一个额外证书来签发 OCSP 响应，可以不对序列号进行要求
$ openssl x509 -req -sha256 -days 3650 -in signer.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -extfile <(printf "extendedKeyUsage=OCSPSigning") -out signer.crt

# 生成 CA 数据库，参考以下的 CA 数据库的格式：
# V/R 到期时间 吊销时间 序列号 unknown CN
# 字段之间使用 \t 隔开，即使某些字段不存在，比如吊销时间，也需要保留制表符
# 经过实践测试，最后的 CN 字段并不一定要与对应证书中的 CN 一致，但是在 index 中每行的 CN 都必须不同
$ cat index.txt
V 340101075959Z  1 unknown /C=CN/OU=IBM Software Group/CN=software.ibm.com1
R 340101075959Z 240124135959Z 3 unknown /C=CN/OU=IBM Software Group/CN=software.ibm.com2
# 需要注意有效证书记录中序列号字段前的制表符数量
$ cat -T index.txt
V^I340101075959Z^I^I1^Iunknown^I/C=CN/OU=IBM Software Group/CN=software.ibm.com1
R^I340101075959Z^I240124135959Z^I3^Iunknown^I/C=CN/OU=IBM Software Group/CN=software.ibm.com2

# 另外标准的吊销过程应该通过相关的 CA 命令来完成，使用命令吊销会自动更新数据库，而这里只是手动更改了 CA 数据库来达到这个效果
# 经过测试，如果你的证书没有记录在 CA 数据库中会返回 unknown 状态，否则一般是 good 或者 revoked 两种状态

# 运行 OCSP Responder
$ openssl ocsp -index index.txt -port 8080 -rsigner signer.crt -rkey signer.key -CA ca.crt
# 可以指定响应具体的请求数量后自动退出
$ openssl ocsp -index index.txt -port 8080 -rsigner signer.crt -rkey signer.key -CA ca.crt -nrequest 1
```

## TLS 协议握手过程

在 HTTP 通信之前，先在通信双方之间通过 TLS 协议建立一条安全通道，后续的 HTTP 通信内容就不再是明文的，只有通信双方才能解读这些信息，这也就是常说的 HTTPS 协议。

现有的主流 TLS 版本是 TLS v1.2 和 TLS v1.3 ，低于这两个版本的协议安全性比较低，不建议使用。

```text
TLS v1.2 的握手过程:

      Client                                               Server

      ClientHello                  -------->
                                                      ServerHello
                                                     Certificate*
                                               ServerKeyExchange*
                                              CertificateRequest*
                                   <--------      ServerHelloDone
      Certificate*
      ClientKeyExchange
      CertificateVerify*
      [ChangeCipherSpec]
      Finished                     -------->
                                               [ChangeCipherSpec]
                                   <--------             Finished
      Application Data             <------->     Application Data

   * Indicates optional or situation-dependent messages that are not
   always sent.

TLS v1.3 的握手过程:

       Client                                           Server

Key  ^ ClientHello
Exch | + key_share*
     | + signature_algorithms*
     | + psk_key_exchange_modes*
     v + pre_shared_key*       -------->
                                                  ServerHello  ^ Key
                                                 + key_share*  | Exch
                                            + pre_shared_key*  v
                                        {EncryptedExtensions}  ^  Server
                                        {CertificateRequest*}  v  Params
                                               {Certificate*}  ^
                                         {CertificateVerify*}  | Auth
                                                   {Finished}  v
                               <--------  [Application Data*]
     ^ {Certificate*}
Auth | {CertificateVerify*}
     v {Finished}              -------->
       [Application Data]      <------->  [Application Data]

         +  Indicates noteworthy extensions sent in the
            previously noted message.

         *  Indicates optional or situation-dependent
            messages/extensions that are not always sent.

         {} Indicates messages protected using keys
            derived from a [sender]_handshake_traffic_secret.

         [] Indicates messages protected using keys
            derived from [sender]_application_traffic_secret_N.

```

虽然两个主流 TLS 版本的握手过程是略有差异的，但它们的核心过程是一致的，我们可以通过 TLS v1.2 的握手过程来学习：

1. 客户端发起 ClientHello ， ClientHello 主要内容是时间戳，随机数，算法协议簇等握手需要的信息。
2. 服务端接收 ClientHello 后，会返回 ServerHello ，主要内容是时间戳，随机数，确定使用的算法协议等信息。
3. 通常地，服务端证书还会和 ServerHello 一起返回给客户端。
4. 除了证书，服务端还会发送 ServerKeyExchange ，这个部分主要内容是后续对称密钥协商所需要的参数，具体内容会由使用的协商算法所决定，如果是 RSA 协商，这个步骤是可以省略的。
5. 客户端验证证书的有效性，如果证书成功通过验证，假设使用 RSA 作为协商算法，客户端将生成新的随机数 Pre-Master Key ，然后通过证书中的公钥加密，发送 ClientKeyExchange 给到服务端。
6. 客户端完成 ClientKeyExchange ，会将 Pre-Master Key ， ClientHello 中随机数， ServerHello 中随机数三者协商出对称加密密钥，完成后发送 ChangeCipherSpec 。
7. 服务端拿到加密的 Pre-Master Key ，可以通过私钥进行解密，然后结合三个随机数，通过前面定好的协商算法计算出对称加密密钥，完成后它也应该向客户端发送 ChangeCipherSpec 。
8. 双方均切换至对称密钥加密，完成握手协议，切换至具体的应用协议。

可以看到，整个过程是非对称加密和对称加密的组合使用，因为非对称加密有着密钥安全性的保证，但解密速度比较差，而对称加密的密钥安全性比较难保障，但加密解密速度快，握手协议交换密钥的过程完美地整合了两者的优点。

在 TLS v1.3 中，它基于 v1.2 做出了一些协议优化，最直观的优化点在于省去 ChangeCipherSpec 过程，第二个优化点是 ClientKeyExchange 过程，主要改进内容在于对称密钥的协商算法。

在原有的握手协议中有 ClientHello 和 ServerHello 两个明文信息，在未改进协商算法前，如果在 ClientKeyExchange 过程，服务端的私钥已经泄露， Pre-Master Key 就可以被破获，相应的加密密钥也可以被推算出来。所以 v1.3 禁用了不安全的 RSA 协商算法，只允许使用安全的 ECDHE 协商算法，使用新算法之后，可以弃用 Pre-Master Key ，新的协商参数放在了 ClientHello 和 ServerHello 各自的扩展 key_share 中，所以可以将 KeyExchange 和 ChangeCipherSpec 过程也一并省去，直接在这两个 Hello 步骤就完成了密钥协商。

可以看到， TLS 握手最终是为了生成数据加密使用的对称密钥，在 v1.2 的版本中，使用 RSA 协商算法可以认为只有客户端参与了密钥的生成，其他参数都是不安全的明文信息。而 v1.3 则是改进了这一点， ECDHE 协商算法使得双方都参与到了密钥的生成中，虽然这一过程还远不止这么简单，整个握手过程还有签名验证和派生的 HKDF 密钥参与握手过程的信息加密，但整体的核心目标还是不变的，并且在此基础上提升了性能和安全性。

## 总结

实际中的 TLS 握手过程会根据实际情况变动，并且服务端和客户端各自的握手扩展并不总是相同的，但纵观不同版本的 TLS 协议，它们的总体思路并没有特别大的变动，证书依然承载着认证的功能，而各种算法协议的改动最终也是为了生成安全的对称密钥。

所以只需要做到对握手过程有总览性的认知就足够了，具体算法细节可以不必过于深入了解。
