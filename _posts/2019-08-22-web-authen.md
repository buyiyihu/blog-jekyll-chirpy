---
title: 《Web服务的身份数据安全》读书笔记
date: 2019-08-22 00:58:18 +0800
categories: [阅读笔记,Authentication]
tags: [Authentication] 
comments: true
---

[《Web服务的身份数据安全》](https://book.douban.com/subject/30308605/)

![img]({{ site.url }}/assets/img/source/web_sec.jpeg){: .w-75 .normal}


---

# 1. SSO & LDAP

SSO -- single sign-on

一处登陆，多处使用.  最简单的SSO可以直接用cookie来实现，但各系统必须在同一个母域名下。

比较古老传统的SSO实现是使用 LDAP

LDAP  （Lightweight Directory Access Protocol） 轻型目录访问协议 ，一个开放，中立，工业标准的应用协议【RFC 4511】

LDAP在TCP/IP之上定义了一个相对简单的升级和搜索目录的协议，提供访问控制和维护分布式信息的目录信息。

LDAP目录与普通数据库的主要不同之处在于数据的组织方式，它是一种有层次的、树形结构,特点是查询速度很快，但写入的性能很差。主要用于经常查但很少改的情况

主要用途在于企业，团体内部，查询符合特定条件的内部人员联系方式

当用于实现SSO时，LDAP作为一个中心服务，用来认证登陆者信息。登录时，服务将用户的认证信息带到LDAP做比对，如果通过，则登陆成功。

对现行sso的一项不满来源于缺少分层分权限功能，所以对一些重要的需要限制权限的资源，往往需要额外另加一层认证方式



# 2. OAuth


OAuth开始于2006年11月 ,在Twitter和Ma.gnolia（一个书签服务） 讨论在Ma.gnolia API上使用Twitter OpenID进行委托授权,他们讨论后得出结论——业界缺少API访问委托的开放标准。于是便开始了OAuth的研究开发。

OAuth分为OAuth以及OAuth 2.0。OAuth Core 1.0 版本发布于2007年12月4日，在2010年4月，OAuth成为了RFC标准： 【RFC 5849】

OAuth 2.0的草案是在2010年5月初在IETF发布的。OAuth 2.0是OAuth协议的下一版本，但不向后兼容OAuth 1.0。 OAuth 2.0的客户端开发比较简易，方便为Web应用，桌面应用和手机等提供认证流程。


## 2.1 OAuth 2.0定义了四种授权方式。

授权码模式（authorization code）
简化模式（implicit）
密码模式（resource owner password credentials）
客户端模式（client credentials）


### 2.1.1 授权码模式


授权码模式（authorization code）是功能最完整、流程最严密的授权模式。

![img]({{ site.url }}/assets/img/source/websec_1.jpg)


（1）用户访问客户端，后者将前者导向认证服务器。

请求参数:

```
- response_type：表示授权类型，必选项，此处的值固定为"code"
- client_id：表示客户端的ID，必选项
- redirect_uri：表示重定向URI，可选项
- scope：表示申请的权限范围，可选项
- state：表示客户端的当前状态，可以指定任意值，认证服务器会原封不动地返回这个值。 防止XSS，CSRF攻击
```

（2）用户选择是否给予客户端授权。

（3）假设用户给予授权，认证服务器将用户导向客户端事先指定的"重定向URI"（redirection URI），同时附上一个授权码。

返回参数:

```
- code：表示授权码，必选项。该码的有效期应该很短，通常设为10分钟，客户端只能使用该码一次，否则会被授权服务器拒绝。该码与客户端ID和重定向URI，是一一对应关系。
- state：如果客户端的请求中包含这个参数，认证服务器的回应也必须一模一样包含这个参数。
```

（4）客户端收到授权码，附上早先的"重定向URI"，向认证服务器申请令牌。这一步是在客户端的后台的服务器上完成的，对用户不可见。
```
- grant_type：表示使用的授权模式，必选项，此处的值固定为"authorization_code"。
- code：表示上一步获得的授权码，必选项。
- redirect_uri：表示重定向URI，必选项，且必须与A步骤中的该参数值保持一致。
- client_id：表示客户端ID，必选项。
```

（5）认证服务器核对了授权码和重定向URI，确认无误后，向客户端发送访问令牌（access token）和更新令牌（refresh token）。
```
- access_token：表示访问令牌，必选项。
- token_type：表示令牌类型，该值大小写不敏感，必选项，可以是bearer类型或mac类型。
- expires_in：表示过期时间，单位为秒。如果省略该参数，必须其他方式设置过期时间。
- refresh_token：表示更新令牌，用来获取下一次的访问令牌，可选项。
- scope：表示权限范围，如果与客户端申请的范围一致，此项可省略。
```

参数使用JSON格式发送（```Content-Type: application/json```）。此外，HTTP头信息中明确指定不得缓存（`Cache-Control: no-store`）

该模式又叫 3-Legged OAuth 三桅OAuth ,三腿OAuth 



### 2.1.2 简化模式



简化模式（implicit grant type）不通过第三方应用程序的服务器，直接在浏览器中向认证服务器申请令牌，跳过了"授权码"这个步骤。所有步骤在浏览器中完成，令牌对访问者是可见的，且客户端不需要认证。

![img]({{ site.url }}/assets/img/source/websec_2.jpg)


（1）客户端将用户导向认证服务器。
```
response_type：表示授权类型，此处的值固定为"token"，必选项。
client_id：表示客户端的ID，必选项。
redirect_uri：表示重定向的URI，可选项。
scope：表示权限范围，可选项。
state：表示客户端的当前状态，可以指定任意值，认证服务器会原封不动地返回这个值。
```

（2）用户决定是否给于客户端授权。

（3）假设用户给予授权，认证服务器将用户导向客户端指定的"重定向URI"，并在URI的Hash部分包含了访问令牌。
```
access_token：表示访问令牌，必选项。
token_type：表示令牌类型，该值大小写不敏感，必选项。
expires_in：表示过期时间，单位为秒。如果省略该参数，必须其他方式设置过期时间。
scope：表示权限范围，如果与客户端申请的范围一致，此项可省略。
state：如果客户端的请求中包含这个参数，认证服务器的回应也必须一模一样包含这个参数。
```


（4）浏览器向资源服务器发出请求，其中不包括上一步收到的Hash值。

（5）资源服务器返回一个网页，其中包含的代码可以获取Hash值中的令牌。

（6）浏览器执行上一步获得的脚本，提取出令牌。

（7）浏览器将令牌发给客户端。

该模式又叫2-Legged OAuth 二桅OAuth ，二腿OAuth 


### 2.1.3 密码模式

密码模式（Resource Owner Password Credentials Grant）中，用户向客户端提供自己的用户名和密码。客户端使用这些信息，向"服务商提供商"索要授权。

在这种模式中，用户必须把自己的密码给客户端，但是客户端不得储存密码。这通常用在用户对客户端高度信任的情况下，比如客户端是操作系统的一部分，或者由一个著名公司出品。而认证服务器只有在其他授权模式无法执行的情况下，才能考虑使用这种模式。


（A）用户向客户端提供用户名和密码。

（B）客户端将用户名和密码发给认证服务器，向后者请求令牌。

```
grant_type：表示授权类型，此处的值固定为"password"，必选项。
username：表示用户名，必选项。
password：表示用户的密码，必选项。
scope：表示权限范围，可选项。
```

（C）认证服务器确认无误后，向客户端提供访问令牌。

整个过程中，客户端不得保存用户的密码。

### 2.1.4客户端模式

客户端模式（Client Credentials Grant）指客户端以自己的名义，而不是以用户的名义，向"服务提供商"进行认证。严格地说，客户端模式并不属于OAuth框架所要解决的问题。在这种模式中，用户直接向客户端注册，客户端以自己的名义要求"服务提供商"提供服务，其实不存在授权问题。

（A）客户端向认证服务器进行身份认证，并要求一个访问令牌。

```
granttype：表示授权类型，此处的值固定为"clientcredentials"，必选项。
scope：表示权限范围，可选项
```

（B）认证服务器确认无误后，向客户端提供访问令牌。



更新令牌：

如果用户访问的时候，客户端的"访问令牌"已经过期，则需要使用"更新令牌"申请一个新的访问令牌。
客户端发出更新令牌的HTTP请求，包含以下参数：
```
granttype：表示使用的授权模式，此处的值固定为"refreshtoken"，必选项。
refresh_token：表示早前收到的更新令牌，必选项。
scope：表示申请的授权范围，不可以超出上一次申请的范围，如果省略该参数，则表示与上一次一致。
```

## 2.2 下面是某第三方微博客户端的验证流程示例：


![img]({{ site.url }}/assets/img/source/websec_3.jpg)
![img]({{ site.url }}/assets/img/source/websec_4.jpg)
![img]({{ site.url }}/assets/img/source/websec_5.jpg)




# 3. OpenID

OpenID 一个去中心化的身份认证协议和开放标准

OAuth解决了授权（ authorization）的问题，提供了Access Token来解决授权第三方客户端访问受保护资源的问题，
但没有解决 认证（authentication）的问题，即用户授权客户服务访问资源，但客户端并不知道该用户是谁.

OpenID 就是用来 认证（ authentication） 的。

概念：
```
终端用户（End User）—— 想要向某个网站表明身份的人。
标识（Identifier）—— 最终用户用以标识其身份的URL或XRI。
身份提供者（Identity Provider, IdP）—— 提供OpenID URL或XRI注册和验证服务的服务提供者。
依赖方（Relying Party, RP）—— 想要对最终用户的标识进行验证的网站。
```

1. EU登录时，在依赖方的页面上填入自己在身份提供者 下注册的账号对应的OpenID，通常是一个URI
2. RP拿到URI做处理后，跳转到这个URI 中，使EU在IDP 登陆
3. 登录完成并同意后，再跳转回依赖方RP，并带回 身份标识



## 3.1 OAuth2.0  与 OpenID 的异同

相同点——都是跳转到认证网站，再带着信息跳回来

但OpenID只带了这个人的身份ID，而OAuth带着access token可以用来向原网站请求信息资源

OpenID有discovery功能，使用OpenID客户服务服务提供方不需要硬编码，任何认证服务都可以

OAuth需要客户服务方硬编码，只能访问客户服务端允许范围内授权方，并且没有identity的概念——不认人，甚至这个token后面不是个人

OAuth 取代OpenID 造成的滥用


![img]({{ site.url }}/assets/img/source/websec_6.jpg)




## 3.2 隐蔽重定向漏洞


2014年5月，新加坡南洋理工大学研究人员王晶发现，OAuth2.0和OpenID授权接口的网站存隐蔽重定向漏洞（Covert Redirect）

黑客先制造一个正常的钓鱼网站，引诱用户登录，在登录后传给验证服务器另外一个url，导致access token被重定向到恶意服务

漏洞不是出现在OAuth这个协议本身，这个协议本身是没有问题的，之所以存在问题是因为各个厂商没有严格参照官方文档，只是实现了简版。

OAuth的提供方提供OAuth授权过程中没有对回调的URL进行校验，从而导致可以被赋值为非原定的回调URL，甚至在对回调URL进行了校验的情况可以被绕过。利用这种URL跳转或XSS漏洞，可以获取到相关授权token


有的OAuth提供方只对回调URL的根域等进行了校验，当回调的URL根域确实是原正常回调URL的根域，但实际是该域下的一个存在URL跳转漏洞的URL，就可以构造跳转到钓鱼页面，就可以绕过回调URL的校验了。
```
1) redirect_uri=http%3A%2F%2Fwww.a.com?www.b.com
2) redirect_uri=http%3A%2F%2Fwww.a.com\www.b.com
3) redirect_uri=http%3A%2F%2Fwww.a.com:\@www.b.com
```
Facebook 、Google、Yahoo、Linkedin、Github、eBay、PayPal、
腾讯QQ、新浪微博、淘宝、支付宝、搜狐网、网易、人人网、开心网、亚马逊、微软Live、WordPress 都受到了影响




# 4. OpenID Connect


OIDC是OpenID Connect的简称，OIDC=(Identity, Authentication) + OAuth 2.0。它在OAuth2上构建了一个身份层，是一个基于OAuth2协议的身份认证标准协议。2014年初发布

OAuth解决了授权（ authorization）的问题，提供了Access Token来解决授权第三方客户端访问受保护资源的问题，
但没有解决 认证（ authentication）的问题，即用户授权客户服务访问资源，但客户端并不知道该用户是谁
access token并没有告诉Client任何东西。在OAuth 中, token对Client不透明

OIDC在这个基础上提供了ID Token来解决第三方客户端标识用户身份认证的问题。OIDC的核心在于在OAuth2的授权流程中，一并提供用户的身份认证信息（ID Token）给第三方客户端，ID Token使用JWT格式来包装。


OIDC和OAuth认证流程类似

```
EU：End User，用户。
RP：Relying Party ，用来代指OAuth2中的受信任的客户端，身份认证和授权信息的消费方；
OP：OpenID Provider，有能力提供EU身份认证的服务方（比如OAuth2中的授权服务），用来为RP提供EU的身份认证信息；
ID-Token：JWT格式的数据，包含EU身份认证的信息。
UserInfo Endpoint：用户信息接口（受OAuth2保护），当RP使用ID-Token访问时，返回授权用户的信息，此接口必须使用HTTPS。
```

RP发送一个认证请求给OP，其中附带client_id；
OP对EU进行身份认证；
OP返回响应，发送授权码给RP；
RP使用授权码向OP索要ID-Token和Access-Token，RP验证无误后返回给RP；
RP使用Access-Token发送一个请求到UserInfo EndPoint； UserInfo EndPoint返回EU的Claims。

在RP得到Access Token后可以请求UserInfo EndPoint，然后获得一组EU相关的Claims，这些信息可以说是ID-Token的扩展，（避免ID Token过于庞大和暴露用户敏感信息）
此资源必须部署在TLS之上。



# 5. token

## 5.1 最常用token——JWT


## 5.2 OAuth2.0 的token


看三腿OAuth2.0 的第五步

```
（5）认证服务器核对了授权码和重定向URI，确认无误后，向客户端发送访问令牌（access token）和更新令牌（refresh token）。
access_token：表示访问令牌，必选项。
token_type：表示令牌类型，该值大小写不敏感，必选项，可以是bearer类型或mac类型。
expires_in：表示过期时间，单位为秒。如果省略该参数，必须其他方式设置过期时间。
refresh_token：表示更新令牌，用来获取下一次的访问令牌，可选项。
scope：表示权限范围，如果与客户端申请的范围一致，此项可省略。
```

OAuth2.0协议在规定下发accessToken时，包含access_token，token_type，expires_in、refresh_token，以及scope字段，其中部分字段可选，
```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache
{
    "access_token":"2YotnFZFEjr1zCsicMWpAA",
    "token_type":"example",
    "expires_in":3600,
    "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
    "example_parameter":"example_value"
}
```



OAuth2.0使用的两种token

BEARER  和 MAC 

BEARER 是在RFC6750中定义的一种token类型。BEARER类型token定义了三种token传递策略，客户端在传递token时必须使用其中的一种，且最多一种。
该类型token需要TLS（Transport Layer Security）提供安全支持

(1) 放在Authorization请求首部

在传输时，Authorization首部的authentication-scheme需要设置为Bearer，请求示例：
```
GET /resource HTTP/1.1
Host: server.example.com
Authorization: Bearer mF_9.B5f-4.1JqM
```

(2) 放在请求实体中

Token需放置在access_token参数后面，且Content-Type需要设置为application/x-www-form-urlencoded，请求示例如下：
```
POST /resource HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded
access_token=mF_9.B5f-4.1JqM
```

(3) 放在URI请求参数中

该方式通过在请求URl后面添加access_token参数来传递token，请求示例如下：
```
GET /resource?access_token=mF_9.B5f-4.1JqM HTTP/1.1
Host: server.example.com
```
客户端在请求时需要设置Cache-Control: no-store，服务端在成功响应时也需要设置Cache-Control: private。
由于很多服务都会以日志方式去记录用户的请求，此类方式存在较大的安全隐患，所以一般不推荐使用，除非前两种方案均不可用。


## 5.3 MAC(Message Authentication Code)

MAC类型的token设计的主要目的就是为了应对不可靠的网络环境。

MAC类型相对于BEARER类型对于用户资源请求的区别在于，BEARER类型只需要携带授权服务器下发的token即可，
而对于MAC类型来说，除了携带授权服务器下发的token，客户端还要携带时间戳，nonce，以及在客户端计算得到的mac值等信息，并通过这些额外的信息来保证传输的可靠性。

一些开放API接口可能会强制要求以MAC类型令牌来请求


该类型token则增加了mac_key和mac_algorithm两个字段，

mac_key是一个客户端和服务端共享的对称密钥，
mac_algorithm则指明了加密算法（比如hmac-sha-1，hmac-sha-256），示例响应如下：
```
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache
{
    "access_token":"SlAV32hkKG",
    "token_type":"mac",
    "expires_in":3600,
    "refresh_token":"8xLOxBtZp8",
    "mac_key":"adijq39jdlaska9asud",
    "mac_algorithm":"hmac-sha-256"
}
```
nonce是客户端生成的一个字符串形式的签名，是对ts和id两个维度的唯一、可重复性标识。

在请求资源时，需要带上一个 mac 字段
这是对本次请求参数的一个签名。通过对请求数据进行本地加密计算得到，用于防止请求过程中参数被更改。
服务器端收到请求之后，会以相同的算法和密钥重新计算一遍mac值，并与客户端传递过来的作比较，如果不一致则拒绝该请求。
因为密钥仅保存在客户端和服务端本地，所以无需担心mac值被更改或伪造，从而确保在没有TLS保证的环境下可靠传输，实际上这里可以看做是MAC类型请求自己实现了一遍TLS。


# 6. 初始身份验证和密码安全

## 6.1 安全观念：

(1)  把安全验证的责任交给用户，
(2) 为了安全，不惜可用性 
(3) 侥幸认为奇葩的事情不会发生在我身上



## 6.2 信息熵
![img]({{ site.url }}/assets/img/source/websec_7.jpg)


若S為一個三個面的骰子,
P(面一)=1/5,
P(面二)=2/5,
P(面三)=2/5

则信息熵：
![img]({{ site.url }}/assets/img/source/websec_8.jpg)



一般认为 36.86比特的密码才算是好密码，等于从5000个不重复的词中随机选择3个的信息熵

| 字符集                      | 字符集数量          | 信息熵（bit） |
|:-----------------------------|:-----------------|--------:|
| 10个阿拉伯数字 | 10       | 3.322    |
| 十六进制数字      | 16    | 4.000      |
| 不区分大小写的拉丁子母 | 26 | 4.700   |
| 不区分大小写的字母数字 | 36 | 5.170   |
| 区分大小写的拉丁子母 | 52 | 5.700   |
| 区分大小写的字母数字 | 62 | 5.954  |
| ASCII 中所有可打印的字符 | 95 | 6.570  |
| 一个字节数据 | 256 | 8.000  |




## 6.3  账号密码系统存在的问题

密码疲劳 导致大部分人要么使用 极其简单的密码，要么很多平台用同一个密码
为此现在有很多服务来解决这一个问题 包括 1password， lastpass 还有chrome，safari，ios系统自带的密码

安全问题 ——不好记，信息泄漏（同学案列）
社会工程——客服说漏嘴，个人信息泄漏


## 6.4  密码加密与破解

加密算法
- PBKDF2    bcrypt   scrypt
- SHA（Secure Hash Algorithm）家族的演算法，由美国国家安全局（NSA）设计，是美国的政府标准

    + SHA-0：1993年发布，当时称做安全散列标准（Secure Hash Standard），发布之后很快就被NSA撤回，是SHA-1的前身。

    + SHA-1：1995年发布，SHA-1在许多安全协定中广为使用，包括TLS和SSL、PGP、SSH、S/MIME和IPsec，曾被视为是MD5的后继。但SHA-1的安全性在2000年以后已经不被大多数的加密场景所接受。

    + SHA-2：2001年发布，包括SHA-224、SHA-256、SHA-384、SHA-512、SHA-512/224、SHA-512/256。虽然至今尚未出现对SHA-2有效的攻击，它的算法跟SHA-1基本上仍然相似，有人开始设计其他算法来替代。

    + SHA-3：2015年正式发布，SHA-3並不是要取代SHA-2，因为SHA-2目前並沒有出現明顯的弱點。由於對MD5出現成功的破解，以及對SHA-0和SHA-1出現理論上破解的方法，NIST感覺需要一個與之前演算法不同的，可替換的加密雜湊演算法，也就是現在的SHA-3。

不再安全的算法
- MD5
- SHA-0
- SHA-1 (2005 年) 少于 2^69 次计算就可以找到一组碰撞，2017年荷兰密码学研究小组CWI和Google正式宣布攻破了SHA-1

## 6.5 密码攻击方法

- 钓鱼——低级简单却十分好用
- 社会工程
- 暴力攻击【主要用于离线破解】
- 字典攻击 【从常见的密码库中比对】，
- 反向查询表  不划算
- 彩虹表
- 恶意软件（键盘记录器，屏幕截取器）

应对方法

- 验证码
- CAPTCHA (completely antomated public Turing test to tell computers and humans apart)——reCAPTCHA
- 2FA (two factor authentication )  ——中间人攻击
- 加盐。在大部份情况，盐是可以明文存储的，但盐值是需要一直变的
- 撒胡椒。胡椒值相当于一个私钥而非公钥，所以需要保密。有的时候胡椒值是在代码层根据一定的逻辑生成的
- 密钥延伸。循环重复应用密码散列函数或块加密方法，或使用需要大量内存的密码散列函数。 


## 6.6 密码以外的身份鉴定方法

- 信任区构建
- 通过额外信息生成指纹
- 环境信息——连接的设备（蓝牙等），GPS，WIFI，生物特征，摄像头和其他传感器
- 浏览器指纹识别
    2010年EFF一项研究报告指出，浏览器有84%具有独特特征，只有1%的浏览器出现了重复指纹。接受测试的浏览器 熵 是18.1bit，1/286777

阻碍浏览器指纹识别的配置
1 禁用js,2 安装了tor浏览器 3 移动设备（配置项不多，比较普适），高度克隆的企业桌面设备无法深度配置

| 不同浏览器特性的熵 |
|:-----------------------------|:---------:|
| 用户代理   |  10bit |
| 插件  |  15.4   |  
| 字体  |  13.9  |  
| 视频  |  4.83   |  
| 超级cookie  |  2.12  |  
| http accept首部  |  6.06  |  
| 时区  |  3.04   |  
| 启用 cookie  |  0.353  |  

可识别的浏览器信息
用户代理/插件/字体，accept首部，屏幕分辨率，时区
GPS，设备指纹（操作系统版本，设备名称，型号，蓝牙配对设备，mac地址）


## 6.7 其他身份验证方法

- 一次性密码 
- OTP/ 2FA/MFA 多因素身份验证 
- 短信/Google Authenticator / Authy
    短信下发：中间人攻击

- TOTP(基于时间的一次性密码算法) /HOTP (基于散列消息验证码的一次性密码算法)
  
- 生物特征
    指纹。Chaos computer club 2013年使用一张高分辨率的图片代替指纹，骗过了iphone 5s的TouchID传感器，成功解锁手机

- 面部识别
- 视网膜和虹膜——都是通过摄像头识别，差别在于识别过程
    视网膜扫描血管对光的吸收——要求很近。虹膜先获取眼球的图像在做分析——10～30cm。视网膜可能遭受短期和永久改变——糖尿病、高血压， 虹膜出生到死都不会

- 静脉识别
    过静脉识别仪取得个人静脉分布图，依据专用比对算法从静脉分布图提取特征值，另一种方式通过红外线 CCD摄像头获取手指、手掌、手背静脉的图像，将静脉的数字图像存贮在计算机系统中，实现特征值存储

- 新方案
   FIDO（Fast IDentity Online） Alliance












