## HTTP

### HTTP协议概念

HTTP 协议是一种基于文本的传输协议，它位于 OSI 网络模型中的应用层。

### 通讯报文

请求

```
POST http://www.baidu.com HTTP/1.1
Host: www.baidu.com
Connection: keep-alive
Content-Length: 7
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36

wd=HTTP
```

响应

```
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Type: text/html;charset=utf-8
Date: Thu, 14 Feb 2019 07:23:49 GMT
Transfer-Encoding: chunked

<html>...</html>
```

### HTTP的缺点：不安全

#### **HTTP存在中间人攻击**

因为HTTP协议中的报文都是以**明文**的形式进行传输，不做任何加密，这样容易在发送数据包之后被中间人挟持，之后修改数据包再发送。

在 HTTP 传输过程中，中间人能看到并且修改 HTTP 通讯中所有的请求和响应内容，所以使用 HTTP 是非常的不安全的。

#### **防止中间人攻击**

**对称加密**

既然内容是明文那我使用对称加密的方式将报文加密这样中间人不就看不到明文了吗，

于是双方约定加密方式，使用AES加密报文

这样看起来中间人就获取不到报文信息，但是在通讯过程中还是会以明文的方式暴露加密方式和密匙。

如果第一次通信被拦截到了，那么秘钥就会泄露给中间人，中间人仍然可以解密后续的通信



**非对称加密**

既然对称加密会有风险，那么能不能将密匙进行加密呢，采用非对称加密，可以通过RSA算法来实现

```
在约定加密方式的时候由服务器生成一对公私钥，服务器将公钥返回给客户端，客户端本地生成一串秘钥(AES_KEY)用于对称加密，并通过服务器发送的公钥进行加密得到(AES_KEY_SECRET)，之后返回给服务端，服务端通过私钥将客户端发送的AES_KEY_SECRET进行解密得到AEK_KEY,最后客户端和服务器通过AEK_KEY进行报文的加密通讯
```

可以看到这种情况下中间人是窃取不到用于AES加密的秘钥，所以对于后续的通讯是肯定无法进行解密了，那么这样做就是绝对安全了吗？



不一定，中间人可以将自己伪装成客户端和服务端的结合体：

在用户->中间人的过程中，模拟服务器的行为，这样可以拿到用户请求的明文；

在中间人->服务器的过程中间，模拟客户端行为，这样可以拿到服务器响应的明文，以此来进行中间人攻击



这一次通信再次被中间人截获，中间人自己也伪造了一对公私钥，并将公钥发送给用户以此来窃取客户端生成的AES_KEY，在拿到AES_KEY之后就能轻松的进行解密了



因此，产生了HTTPS制裁中间人

## HTTPS 协议

### HTTPS 简介

HTTPS 其实是SSL+HTTP的简称,当然现在SSL基本已经被TLS取代了，不过接下来我们还是统一以SSL作为简称，SSL协议其实不止是应用在HTTP协议上，还在应用在各种应用层协议上，例如：FTP、WebSocket。



其实SSL协议大致就和上一节非对称加密的性质一样，握手的过程中主要也是为了交换秘钥，然后再通讯过程中使用对称加密进行通讯，服务器是通过 SSL 证书来传递公钥，客户端会对 SSL 证书进行验证，其中证书认证体系就是确保SSL安全的关键，接下来我们就来讲解下CA 认证体系，看看它是如何防止中间人攻击的。



### CA 认证体系

客户端需要对服务器返回的 SSL 证书进行校验，那么客户端是如何校验服务器 SSL 证书的安全性呢？

**权威认证机构**

在 CA 认证体系中，所有的证书都是由权威机构来颁发，而权威机构的 CA 证书都是已经在操作系统中内置的，我们把这些证书称之为CA根证书



**签发证书**

我们的应用服务器如果想要使用 SSL 的话，需要通过权威认证机构来签发CA证书，我们将服务器生成的公钥和站点相关信息发送给CA签发机构，再由CA签发机构通过服务器发送的相关信息用CA签发机构进行加签，由此得到我们应用服务器的证书，证书会对应的生成证书内容的签名，并将该签名使用CA签发机构的私钥进行加密得到证书指纹，并且与上级证书生成关系链。



这里我们把百度的证书下载下来看

可以看到

```
-GlobalSign R1
	-GlobalSign G2
		-baidu.com
```

可以看到百度是受信于GlobalSign G2，同样的GlobalSign G2是受信于GlobalSign R1，当客户端(浏览器)做证书校验时，会一级一级的向上做检查，直到最后的根证书，如果没有问题说明服务器证书是可以被信任的。



**如何验证服务器证书**

那么客户端(浏览器)又是如何对服务器证书做校验的呢，首先会通过层级关系找到上级证书，通过上级证书里的公钥来对服务器的证书指纹进行解密得到签名(sign1)，再通过签名算法算出服务器证书的签名(sign2)，通过对比sign1和sign2，如果相等就说明证书是没有被篡改也不是伪造的。



这里有趣的是，证书校验用的 RSA 是通过私钥加密证书签名，公钥解密来巧妙的验证证书有效性。

这样通过证书的认证体系，我们就可以避免了中间人窃取AES_KEY从而发起拦截和修改 HTTP 通讯的报文。