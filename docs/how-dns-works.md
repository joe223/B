# DNS查询流程

![示例](https://joe223.github.io/note/how-dns-works/assets/how-dns-works.png)
用一个示例描述整个过程（这并非唯一https://tools.ietf.org/html/rfc1034#section-4.3.1）:

1. 在Chromium端键入www.example.com地址，交由host_resolver处理
2. host_resolver首先判断这个url是否为localhost，结果为否，那么它先从自己的DNS缓存查找
3. 缓存坦言：不知道
4. 没有从缓存中查到结果，host_resolver转头向hosts文件查询：这里有登记过www.example.com吗？
5. hosts文件也说没有
6. host_resolver无能为力，只得请求本地DNS服务器帮忙（通常情况下本地DNS服务器由当地ISP提供）
7. 为了尽可能缩短DNS查询时间，本地DNS服务器会将以往的记录暂时保存。当收到hosts_resolver的DNS请求后，服务器首先向自己的缓存查询，如果能查到，那么它会直接返回www.example.com的IP地址
8. 可惜的是，服务器的缓存也说不知道
9. 此时本地DNS服务器开始了一项艰巨的任务，它先向根域名服务器询问：老大，www.example.com的IP是多少啊？
10. 做老大的顾不上这样细枝末节的事，根域名服务器告诉本地域名服务器：你去问.com顶级域名服务器吧，它负责这个
11. 本地DNS服务器按照根域名服务器的指引找到.com的顶级域名服务器：二哥，www.example.com的IP是多少啊？
12. 当然，二哥肩上的担子子也很重，便告诉本地DNS服务器（一条SOA (Start of Authority) Record）：你去问example.com的权威域名服务器吧
13. 本地DNS服务器转向example.com的权威域名服务器，它负责example.com下所有域名的解析
14. example.com权威域名服务器查询后将IP X.X.X.X 告知本地DNS服务器
15. 本地DNS服务器谨慎地将此次记录留到缓存中，并将IP返回给host_resolver
客户端终于收到IP地址并建立TCP连接，发起HTTP请求获取网站资源。

# 为什么只有13台根域名服务器?

RFC1035中定义：进行DNS查询时，UDP payload 为 512 bytes。

DNS报文格式如下：

    +---------------------+
    |        Header       |
    +---------------------+
    |       Question      | the question for the name server
    +---------------------+
    |        Answer       | RRs answering the question
    +---------------------+
    |      Authority      | RRs pointing toward an authority
    +---------------------+
    |      Additional     | RRs holding additional information
    +---------------------+

其中：

- [576 bytes - MTU](https://tools.ietf.org/html/rfc791#section-3.1)
- [512 bytes - UDP payload](https://tools.ietf.org/html/rfc1035#section-4.2.1)
- [12 bytes - DNS message Header](https://tools.ietf.org/html/rfc1035#section-4.1.1)
- [5 bytes - DNS message Question](https://tools.ietf.org/html/rfc1035#section-4.1.2)
- [31 bytes - DNS message Answer](https://tools.ietf.org/html/rfc1035#section-4.1.3)
- [15 bytes - DNS message Answer with compression](https://tools.ietf.org/html/rfc1035#section-4.1.4)
- [16 bytes - DNS message Question with compression](https://tools.ietf.org/html/rfc1035#section-4.1.4)

那么得到如下公式：

    ① 512 - 12 - 5 = 31 + 15n + 16m
    ② m = n + 1

    m: Additional 字段数
    n: Answer 字段数

最终得到 `n ≈ 14.45`

> Bill Manning: “做人留一线，以后好相见”

所以选择 **13**

# DNS协议存在什么缺陷？

1. 单点故障

DNS采层次化的树形结构，由树叶走向树根就可以形成一个完全合格域名(Fully Qualified Domain Name，FQDN),DNS服务器作为该FQDN唯一对外的域名数据库和对内部提供递归域名查询的系统，其安全和稳定就存在单点故障的风险。

2. 无认证机制

DNS没有提认证机制，查询者在收到应答时无法确认应答信息的真假，黑客可以将一个虚假的IP地址作为应答信息返回给请求者，从而引发DNS欺骗。

3. DNS本身漏洞

DNS是域名软件，它在提供高效服务的同时也看在许多安全性漏洞。现已证明在DNS版本4和版本8上存在缺陷，攻击者利用这些缺陷能成功地进行DNS击。DNS又面临着来自网络的各种威胁，黑客利用DNS协议或者软件设计的漏洞，通过网络向DNS发起攻击以达到非法目的，攻击主要包括如下几种。
- 内部攻击
攻击者在非法或合法地控制一台DNS服务器后，可以直接作域名数据库，修改
指向域名所对应的IP，当客户发出对指定域名的查询请求后，将得到伪造的IP地址。
- 序列号攻击
DNS协议格式中定义了用来匹配请求数据包和响应数据报序列的ID，欺骗者利用序列号伪装成DNS服务器向客户端发送DNS响应数据包，在DNS服务器发送的真实DNS响应数据报之前到达客户端，从而将客户端带到攻者所希望的网站，进行DNS欺骗。
- 信息插入攻击
攻击者可以在DNS应答报文中随意添加某些信息，指示权威域名服务器的域名及IP。如果在被影响的域名服务器上查询该域的请求，则请求都会被转向攻击者所指定的域名服务器上去，从而威胁到网络数据的完整性。

4. 缓存中毒

DNS使用超高速缓存，当一个名称服务器收到有关域名和IP的映射信息时，它会将该信息存放在高速缓存中。这种映射表是动态更新的，但刷新有一个周期，假冒者如果在下次更新之前成功修改了这个映射表，就可以进行DNS欺骗。

5. 信息泄露
DNS的默认设置允许任何人进行域传送（域传送一般用于主服务器和辅服务器之间的数据同步)，而域传送可能会造成信息泄露。


# DNS查询中用到什么协议？

- DNS 查询主要使用UDP协议。TCP作为备选项，通常而言，域名服务器不会启用TCP查询方式。因为资源消耗太高。
- DNS 进行Zone Transfers时会使用TCP协议。
