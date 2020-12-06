# DNSSEC

## 什么是DNSSEC？

域名系统安全扩展（DNS Security Extensions），简称DNSSEC。它是通过数字签名来保证DNS的应答报文的真实性和完整性，主要作用于Local DNS和权威DNS之间，防止数据被中间人篡改，防止中间人在Local DNS上进行缓存投毒。当某个域名开启了DNSSEC，且需要Local DNS和权威DNS同时支持，从客户端的角度来看，查询到的解析记录一定是正确的，如果存在中间人篡改信息，则客户端会收到Local DNS的查询失败的结果，避免被重定向到非预期的地址。

相关的RFC文档可以在这里查询： [RFC DNSSEC](https://www.dnssec.net/rfc)。其中，

DNSSSEC的核心规范在如下三个文档中指出：

- [RFC 4033](https://tools.ietf.org/html/rfc4033) - DNS Security Introduction and Requirements
- [RFC 4034](https://tools.ietf.org/html/rfc4034) - Resource Records for the DNS Security Extensions
- [RFC 4035](https://tools.ietf.org/html/rfc4035) - Protocol Modifications for the DNS Security Extensions

[RFC 5155](https://tools.ietf.org/html/rfc5155) 介绍了如何进行定期更新资源记录RR，NSEC3(Next Secure)。[RFC 5910](https://tools.ietf.org/html/rfc5910) 和 [RFC 4641](https://tools.ietf.org/html/rfc4641) 分别介绍了如何进行DNSSEC与扩展协议的映射，DNSSEC在部署签名、key生成管理、签名管理等的具体实践。

## DNSSEC原理

### DNSSEC的记录类型

- RRSIG(Resource Record Signature): RRset的加密签名
- DNSKEY(DNS Public Key): 公钥，包含KSK(Key Signing Key)和ZSK(Zone Signing Key)公钥两种
- DS(Delegation Signer): KSK公钥的摘要
- NSEC/NSEC3(Next Secure): 用来证明否定应答(no name error, no data error, etc.)
- CDNSKEY和CDS: 子zone用来自动更新在父zone中的DS记录

### RRSET

一个域名,多个相同类型的资源记录的集合成为资源记录集(RRset) ,RRset是DNS传输的基本单元，即相同Owner，Type，Class的若干RR的集合。如下例所示：假设查询`163.com`，获得如下结果：

```
$ dig dig 163.com

; <<>> DiG 9.10.6 <<>> 163.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 57932
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 7, ADDITIONAL: 7

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;163.com.			IN	A

;; ANSWER SECTION:
163.com.		313	IN	A	123.58.180.7
163.com.		313	IN	A	123.58.180.8
```
可以看到Local DNS给我返回了两条A记录，那么这两个A记录就会被绑定到单个 A RRset中。

### ZSK（Zone Signing Key）

1. ZSK私钥签名RRset生成RRSIG，公钥以DNSKEY RRset的形式发布，任何Local DNS都可以获取到这个DNSKEY
2. 仅权威服务器RRset签名生成RRSIG
3. RRSIG的TTL和RRSET的TTL相同
4. DNS解析时，查询RRset时，会同时获取RRSIG，以及DNSKEY，Local DNS 利用ZSK的公钥验证签名
5. 如果权威DNS是可信的，那么验证过程到这里就可以结束了。可是如果ZSK是伪造的，则Local DNS丢弃该响应，并给客户端响应DNS查询失败。

那么如何对ZSK进行验证呢？

### KSK（Key Signing Key）

1. KSK用于验证ZSK
2. KSK私钥签名DNSKEY RRset（包含ZSK和KSK的公钥）生成DNSKEY RRSIG
3. 本zone的信任可以建立，但KSK自己签名验证自己，不可信，所以需要建立信任链打通本zone和父zone之间的信任

### DS（Delegation Signer）

1. 子zong KSK公钥的哈希，提交到父zone上，子zone提交DS记录后，则意味着子zone的DNSSEC已经准备就绪
2. Local DNS 在迭代查询中，权威DNS在返回NS记录的同时会返回DS记录
3. 递归查询到KSK公钥后进行哈希，和父zone里的DS记录进行比较，如果能匹配成功，则证明KSK没有被篡改

### 总结

DNSSEC拥有两种密钥，一种是ZSK，一种是KSK。其中ZSK是用于对域名的解析记录（例如A记录，CNAME记录等）进行签名使用的，而KSK是用于对域名的DNSKEY记录进行签名的。


## 参考

- https://www.dnssec.net/
- https://zhuanlan.zhihu.com/p/107552714

