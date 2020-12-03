# EDNS

在上篇中记录了随着技术的发展，DNS应该使用UDP还是TCP，同时也提到了基于UDP传输的DNS会有512字节的限制，那么作为应对方案之一，[RFC2671](https://tools.ietf.org/html/rfc2671) 在1999年提出了EDNS机制，后来又在2013年，[RFC6891](https://tools.ietf.org/html/rfc6891)给出了补充说明，描述了EDNS向后兼容的机制。下面结合一些已有的博客文章，和自己的一些理解，对EDNS进行简单的记录。

## 什么是EDNS？

回顾下DNS报文的扩展字段，包括四个部分：分别为`Queries、Answers、Authoritative NameServer、Additional Records`，EDNS就是在遵循已有的DNS消息格式（附加信息区域 Addition Record）的基础上增加一些字段，来支持更多的DNS请求业务。RFC6891中特别指出，NameServer解析器需要考虑到兼容性问题，如果客户端发起了EDNS查询，但是递归解析器不支持EDNS，那应该按照正常的DNS解析返回，以保证交互顺畅。

借用下面两张图来说明EDNS的一个主要应用场景，在CDN中的应用：

对于CDN来说，实现区域性的调度，让用户就近下载，通常依赖于以下两种方式：

1. Anycast：通过BGP的选路原则，确保用户访问最靠近最终用户的CDN服务器，但是这种形式对CDN来说，部署成本很高，非主流CDN厂商难以大规模化部署。
2. DNS解析：依赖于客户端指定的递归LDNS，最终由LDNS返回结果，但是，如果用户指定或者自动获取了一个**不理想**的DNS，那么他可能会拿到一个错误的解析结果，如下图一，印度的用户通过新加坡的DNS，拿到了在新加坡的CDN节点；但在图二中，首先新加坡的DNS和权威DNS支持EDNS，并且客户端在发起DNS查询请求时，需要在DNS报文中增加`edns-client-subnet`(简称ECS)，即把自己的子网放在报文中，当权威DNS解析该查询报文时，会根据递归DNS中附带的subnet来分配CDN节点，而不再根据递归DNS来分配CDN节点了，因此，这里的客户端拿到了正确的在印度的CDN节点。

![image](https://blog.catchpoint.com/wp-content/uploads/2017/05/edns7.png)
图一：

![image](https://blog.catchpoint.com/wp-content/uploads/2017/05/edns8.png)
图二

## 为什么要有EDNS？

理由大致如下：

1. 标准DNS协议头中的标志位，共16bit，已经快被用尽了，需要添加新的返回类型(RCODE)和标记(FLAGS)来支持其他需求；

2. 在RFC1035中，为了将报文做到尽可能短小，进行消息压缩，只为标示domain类型的标签分配了两位，现在已经用掉了两位（00表示字符串类型，11表示压缩类型），后面如果有更多的标签类型则无法支持；

3. 在最初设计DNS时，用UDP包传输时的包大小限制为512字节，如果超过该限制会被截断，并切换到TCP进行重传，为了避免这种情况，突破这种限制，这种机制允许客户端通知DNS服务器如果响应超过了512字节，无需截断，可以返回大包，最大可支持4096字节；

## EDNS的内容是什么？

 怎样在DNS消息协议的基础上再增加一些字段呢？为了保持向后兼容性，更改已有的DNS协议格式是不可能的，所以只能在DNS协议的数据部分中做文章。

所以，EDNS中引入了一种新的伪资源记录OPT（Resource Record），之所以叫做伪资源记录是因为它不包含任何DNS数据，OPT RR不能被cache、不能被转发、不能被存储在zone文件中。OPT被放在DNS通信双方（requestor和responsor）DNS消息的Additional data区域中。

### OPT伪资源记录中的内容有哪些呢？

OPT pseudo-RR中的内容包含固定部分和可变部分。它的结构如下：

        Field Name   Field Type     Description
        ------------------------------------------------------
        NAME         domain name    empty (root domain)
        TYPE         u_int16_t      OPT
        CLASS        u_int16_t      sender's UDP payload size
        TTL          u_int32_t      extended RCODE and flags
        RDLEN        u_int16_t      describes RDATA
        RDATA        octet stream   {attribute,value} pairs

上面的结构中最下面的RDATA是可变部分，其余的部分都是固定部分：Name字段目前为空；TYPE字段是OPT RR的类型编号，IANA为其分配的是41（0x29）；TTL中是扩展的DNS消息头部，下面会有介绍；RDLEN是可变部分RDATA的长度；RDATA是KV类型的可变部分。

原来的TTL字段被用来存储扩展消息头部中的RCODE和flags，它的格式如下：

                    +0 (MSB)                            +1 (LSB)
        +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    0: |         EXTENDED-RCODE        |            VERSION            |
        +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    2: |                               Z                               |
        +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
                    extended RCODE and flags Detail 


其中高位8个bit是扩展RCODE（返回状态码），这8个bit加上DNS头部的4bit总共有12bit（8bit在高位），这样就可以表示更多的返回类型；

VERSOION字段表示EDNS的版本（EDNS根据支持不同的扩展内容会有很多版本），这篇文章提到的内容的VERSION=0

RFC2671中Z一般情况下被发送者设置为0，接收方可以忽略它。但是后续的扩展协议中会用到这16bit。


        +0 (MSB)                            +1 (LSB)
        +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    0: |                          OPTION-CODE                          |
        +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    2: |                         OPTION-LENGTH                         |
        +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    4: |                                                               |
        /                          OPTION-DATA                          /
        /                                                               /
        +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
                                    RDATA格式

其中OPTION-CODE由IANA分配；OPTION-LENGTH是OPTION-DATA的长度；OPTION-DATA是具体长度。

## 目前实际网络环境下的EDNS

这里要先说明下ECS（Edns-Client-Subnet，[RFC7871](https://tools.ietf.org/html/rfc7871) ），它的主要作用是允许递归Local DNS服务器传递用户的IP地址给权威DNS服务器，因此EDNS的实际应用，也是在RFC7871推出之后，2016年后开始有应用的。

### ECS 支持现状

RFC7871定稿之前，ECS一直作为一个RFC草案存在，并没有得到大范围的支持。

其次，如果要让域名的来访用户真正能访问到正确的区域和运营商站点，不仅仅用户使用的 Local DNS 要支持ECS，同时域名使用的权威 DNS 也要支持，而客户端在发起DNS查询请求时，也需要主动的把自己的subnet放在请求报文的Addtion Record中。

所以事实上，ECS 的普及程度并不高。目前市场上支持 ECS 的权威DNS不多，目前主要是Aws、Google、CF等大厂商支持，国内的BAT等一线互联网公司也是支持的，但是支持 ECS 的递归 DNS 却是少之又少。在中国，几乎没有哪个运营商的Local DNS 支持EDNS，而在中国作为用户，几乎都是靠着运营商的Local DNS进行上网的，所以难以普及。

### EDNS带来的问题

上面提到运营商的Local DNS几乎不会去支持EDNS，究其原因，主要是两个方面：

1. 缓存问题： 由于DNS是逐级缓存的，如果Local DNS支持EDNS，一般情况下，单条域名的解析记录的缓存需要针对不同的subnet进行缓存，即每个subnet的缓存都是独立的，通常情况下，需要缓存到24位，不过RFC7871建议Local DNS将subnet Trancate 为20位并进行缓存，但这也让Local DNS的需要缓存的记录呈现指数级的上升，同时，缓存命中率也会大大降低，需求Local DNS频繁的回源，造成额外的性能开销，用户体验上，虽然调度更加精准，但是出现DNS解析延时增大的概率也大大增加；
2. 安全问题，rfc7871 11节提到了三个可能的安全问题：

    1. 由于客户端会在Addition Record中带上自己真实的subnet，所以在整条DNS查询链路上，客户端的IP地址会被显示的暴露出来；
    2. 生日攻击：
        - 攻击者向DNS服务器发送一定数量的DNS请求包，请求的DNS选择该DNS服务器解析不了的，这样的情况下，DNS服务器会向上层DNS服务器发出相同数量的请求。
        - 攻击者快速向DNS服务器发送一定数量的伪造的DNS应答包，其中ID和端口号为随机的，当ID和端口号与正确的产生哈希碰撞时，缓存攻击成功。DNS服务器将错误的DNS解析项缓存，这样，所有DNS请求都将导向攻击者伪造的IP地址。
    3. 缓存污染：如DNS劫持、中间人攻击等手段。

因此，从目前的大环境看，EDNS的大范围普及还很难，但是从用户的角度看，DNS解析如果延时稍微增大，但是整个http的请求来说，并不会很明显的将整体时延拖慢，而且随着IPv6、物联网、5G等技术的推进，相信运营商对EDNS的支持也会逐步到来。

## TroubleShooting： DNS解析验证

之前在工作中碰到过Local DNS故障导致用户被调度到错误的CDN节点的问题，以下为TroubleShooting的过程：

### 确定local DNS的解析结果

可以通过`dig`确认。

例如，下面是一个天津联通的用户，其Local DNS为 202.99.104.68：

```
$ dig ws.163fen.com @202.99.104.68

; <<>> DiG 9.10.3-P4-Debian <<>> ws.163fen.com @202.99.104.68
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 37944
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;ws.163fen.com.			IN	A

;; ANSWER SECTION:
ws.163fen.com.		232	IN	CNAME	gslb.cdn.163fen.com.
gslb.cdn.163fen.com.	76	IN	A	125.39.38.116
gslb.cdn.163fen.com.	76	IN	A	125.39.38.117
```

### 向权威查询，需要加上subnet

#### 查出163fen.com的权威

```
$ dig NS 163fen.com

; <<>> DiG 9.10.3-P4-Debian <<>> NS 163fen.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40343
;; flags: qr rd ra; QUERY: 1, ANSWER: 7, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;163fen.com.			IN	NS

;; ANSWER SECTION:
163fen.com.		17687	IN	NS	ns6.nease.net.
163fen.com.		17687	IN	NS	ns1.nease.net.
163fen.com.		17687	IN	NS	ns8.166.com.
163fen.com.		17687	IN	NS	ns3.nease.net.
163fen.com.		17687	IN	NS	ns4.nease.net.
163fen.com.		17687	IN	NS	ns5.nease.net.
163fen.com.		17687	IN	NS	ns2.166.com.
```

### 向权威查询

```
dig ws.163fen.com +subnet=125.39.38.1 @ns2.166.com

; <<>> DiG 9.10.3-P4-Debian <<>> ws.163fen.com +subnet=125.39.38.116 @ns2.166.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 54116
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 6, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; CLIENT-SUBNET: 125.32.0.0/32/12
;; QUESTION SECTION:
;ws.163fen.com.			IN	A

;; ANSWER SECTION:
ws.163fen.com.		300	IN	CNAME	gslb.cdn.163fen.com.wscdns.com.

;; AUTHORITY SECTION:
cdn.163fen.com.		86400	IN	NS	ns3.ntes53.netease.com.
cdn.163fen.com.		86400	IN	NS	ns4.ntes53.nease.net.
cdn.163fen.com.		86400	IN	NS	ns2.ntes53.netease.com.
cdn.163fen.com.		86400	IN	NS	ns1.ntes53.netease.com.
cdn.163fen.com.		86400	IN	NS	ns6.ntes53.nease.net.
cdn.163fen.com.		86400	IN	NS	ns5.ntes53.nease.net.
```

### 结论

从上方查询看来，权威的结果是正常的，我们将天津用户CNAME到了备用域名
但local dns的解析不正常，应该是local dns的递归使用了运营商线路，而运营商DNS并没有及时的去权威DNS更新最近的解析。

## 参考链接

- https://www.jianshu.com/p/42e59ddd78af
- https://www.cnblogs.com/cobbliu/p/3188632.html
- https://tools.ietf.org/html/rfc2671
- https://ephen.me/2017/PublicDns_2/
