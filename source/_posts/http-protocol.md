---
title: HTTP协议详解
categories: 开发基础
tags:
- http
- 基础
top: 100
copyright: ture
---

# 介绍
&emsp;&emsp;HTTP是Hyper Text Transfer Protocol（超文本传输协议）的缩写。它的发展是万维网协会（World Wide Web Consortium）和Internet工作小组IETF（Internet Engineering Task Force）合作的结果，最终发布了一系列的RFC，RFC 1945定义了HTTP/1.0版本。其中最著名的就是RFC 2616。RFC 2616定义了今天普遍使用的一个版本--HTTP 1.1。<!-- more -->
&emsp;&emsp;HTTP协议是用于从WWW服务器传输超文本到本地浏览器的传送协议。它可以使浏览器更加高效，使网络传输减少。它不仅保证计算机正确快速地传输超文本文档，还确定传输文档中的哪一部分，以及哪部分内容首先显示（如文本先于图形）等。
&emsp;&emsp;HTTP是一个应用层协议，由请求和响应构成，是一个标准的客户端服务器模型，而且HTTP是一个无状态的协议。

# 主要特点
- 简单快速：客户向服务器请求服务时，只需传达请求方法和路径。请求方法常用的有GET、HEAD、POST。每种方法规定了客户端与服务器联系的类型不同。由于HTTP协议简单，使得HTTP服务器的程序规模小，因而通信速度很快。
- 灵活：HTTP允许传输任意类型的数据对象。正在传输的类型由Content-Type加以标记。
- 无连接：无连接的含义是限制每次连接只处理一个请求。服务器处理完客户端的请求，并收到客户端的应答后，即断开连接。采用这种方式可以节省传输时间。
- 无状态：HTTP协议是无状态协议。无状态是指协议对于事务处理没有记忆能力。缺少状态意味着如果后续处理需要前面的信息，则它必须重传，这样可能导致每次连接传送的数据量增大；另一方面，在服务器不需要先前信息时，它的应答就很快。
- 支持[B/S及C/S模式](https://www.cnblogs.com/jingmin/p/6493216.html)

# HTTP协议请求消息结构
&emsp;&emsp;客户端发送一个HTTP请求到服务器的请求消息包括以下格式：请求行（request line）、请求头部（header）、空行和请求数据四个部分组成，下图给出了请求报文的一般格式。
![](http://pz1livcqe.bkt.clouddn.com/HTTP请求消息体结构.jpg 'HTTP请求消息体结构')

&emsp;&emsp;HTTP消息体主要包含以下实质内容（空格和换行也必不可少）：
- 请求方法
- URL：统一资源定位符
- HTTP请求头部
- HTTP请求体

## 请求方法
&emsp;&emsp;HTTP包含了多种不同的请求方式，每一种请求方式用在不同的场景。

| 序号 | 方法 | 描述 |
| :-: | :-: | :-: |
| 1 | GET | 请求指定的页面信息，并返回实体主体 |
| 2 | HEAD | 类似于GET请求，只不过返回的响应中没有具体的内容，用于获取报头 |
| 3 | POST | 向指定资源提交数据进行处理请求（例如提交表单或者上传文件）。数据被包含在请求体中。POST请求可能会导致新的资源的建立或已有资源的修改 |
| 4 | PUT | 从客户端向服务器传送的数据取代指定的文档的内容 |
| 5 | DELETE | 请求服务器删除指定的页面 |
| 6 | CONNECT | HTTP/1.1协议中预留给能够将连接改为管道方式的代理服务器。  |
| 7 | OPTIONS | 允许客户端查看服务器的性能 |
| 8 | TRACE | 回显服务器收到的请求，主要用于测试或诊断。 |

## URL——统一资源定位符
&emsp;&emsp;URL由三部分组成：资源类型、存放资源的主机域名、资源文件名。
&emsp;&emsp;URL的一般语法格式为（带方括号[]的为可选项）：
```
protocol :// hostname[:port] / path / [;parameters][?query]#fragment
```
&emsp;&emsp;举个🌰：[https://baijiahao.baidu.com/s?id=1603848351636567407&wfr=spider&for=pc](https://baijiahao.baidu.com/s?id=1603848351636567407&wfr=spider&for=pc)
- protocol：https
- hostname：baijiahao.baidu.com
- parameters：id=1603848351636567407&wfr=spider&for=pc （使用&分割参数）

&emsp;&emsp;总结如下图所示：
![](http://pz1livcqe.bkt.clouddn.com/解析图.jpg '解析图')

## 请求头
&emsp;&emsp;请求头中主要包含本次请求的附加信息，其中常用的字段如：
- Accept：指定客户端能够接收的内容类型
- Accept-Encoding：指定浏览器可以支持的web服务器返回内容压缩编码类型
- Accept-Language：浏览器可接受的语言
- Content-Length：请求的内容的长度，如：Content-Lenght：348
- Content-Type：请求的与实体对应的MIME信息，常用的类型如下：
    - text/html：HTML格式
    - text/plain：纯文本格式
    - text/xml：XML格式
    - image/gif：gif图片格式
    - image/jpeg：jpg图片格式
    - image/png：png图片格式
    - application/xhtml+xml：XHTML格式
    - application/xml：XML数据格式
    - application/json：JSON数据格式
    - application/pdf：pdf格式
    - application/msword：word文档格式
    - application/octet-stream：二进制流数据
- Date请求发送的日期和时间
- 更多的请求头字段参考：[HTTP响应头和请求头信息对照表](http://tools.jb51.net/table/http_header)

## 请求体
&emsp;&emsp;在整个报文中，请求头之后，隔一行空格，以下部分就是HTTP的请求体了。请求体是我们发送请求的时候需要传给接收端的内容。其格式需要和请求头中的`Content-Type`对应，不然会导致接受无法识别。

# HTTP响应
&emsp;&emsp;HTTP的响应同样分为：响应行、响应头和响应体。和请求报文有点类似。

## 响应行
&emsp;&emsp;响应行中包含了HTTP的版本和本次请求的状态，请求状态的对应值见下文或见[HTTP响应码大全](https://blog.csdn.net/ddhsea/article/details/79405996)

## 响应头
&emsp;&emsp;响应头用于描述服务器的基本信息、数据的描述，这些信息将告知客户端如何处理响应头中的内容。
- Allow 服务器支持哪些请求方法（如GET、POST等）。
- Content-Encoding 文档的编码（Encode）方法。只有在解码之后才能得到Content-Type头指定的内容类型。利用gzip压缩文档能够显著地减少HTML文档的下载时间。Java的GZIPOutputStream可以很方便地进行gzip压缩，但只有Unix上的Netscape和Windows上的IE 4、IE 5才支持它。因此，Servlet应该通过查看Accept-Encoding头（即`request.getHeader("Accept-Encoding")`）检查浏览器是否支持gzip，为支持gzip的浏览器返回经gzip压缩的HTML页面，为其他浏览器返回普通页面。
- Content-Lenght 表示内容长度。只有当浏览器使用持久HTTP连接时才需要这个数据。如果你想利用持久连接的优势，可以把输出文档写入ByteArrayOutputStream，完成后查看其大小，然后把该值放入Content-Length头，最后通过`byteArrayStream.writeTo(response.getOutputStream())`发送内容。
- Content-Type 表示后面的文档属于什么MIME类型。Servlet默认为text/plain，但通常需要显式地指定为text/html。由于经常要设置Content-Type，因此HttpServletResponse提供了一个专用的方法setContentType。更多的请求头字段参考：[HTTP响应头和请求头信息对照表](http://tools.jb51.net/table/http_header)

## 响应实体
&emsp;&emsp;响应实体中包含的就是客户端从服务器中获取的数据了。数据的格式和长度都会在响应体中描述。

# 状态码
&emsp;&emsp;HTTP的状态码由三位数字组成，第一个数字定义了响应的类别，共有5种类别：
- 1xx：指示信息 — 表示请求已接收，继续处理
- 2xx：成功 — 表示请求已被成功接收、理解、接受
- 3xx：重定向 — 要完成请求必须进行更进一步的操作
- 4xx：客户端错误 — 请求有语法错误或请求无法实现
- 5xx：服务器端错误 — 服务器未能实现合法的请求

## 常见的状态码：
- 200 OK：客户端请求成功
- 400 Bad Request：客户端请求有语法错误，不能被服务器所理解
- 401 Unauthorized：请求未经授权，这个状态代码必须和`WWW-Authenticate`报头域一起使用
- 403 Forbidden：服务器收到请求，但是拒绝提供服务
- 404 Not Found：请求资源不存在，不如：输入了错误的URL
- 500 Internal Server Error：服务器发生不可预期的错误
- 503 Server Unavailable：服务器当前不能处理客户端的请求，一段时间后可能恢复正常。
- 更多状态码见：[HTTP响应码大全](https://blog.csdn.net/ddhsea/article/details/79405996)

# GET 请求和POST 请求的区别
## 区别
- GET请求：请求的数据会附在URL之后（就是把数据放置在HTTP协议头中），以?分隔URL和传输数据，多个参数用&进行连接；如果数据是英文字母/数组，原样发送，如果是空格，则转换为+，如果是中文/其他字符，则直接把字符串用base64进行加密。
- POST请求：把提交的数据放置在是HTTP包的包体中。因此，GET请求的数据会在地址栏中显示出来；而POST请求，地址栏不会改变。

## 实际开发中传输大小存在的限制
- GET：特定浏览器和服务器对URL长度有限制，例如IE对URL长度的限制是2083字节（2K + 35）。对于其他浏览器，如FireFox等，理论上没有长度限制，其限制取决于操作系统的支持。因此对于GET请求时，传输数据就会受到URL长度的限制。
- POST：由于不是通过URL传值，理论上数据不受限制。但实际各个web服务器会规定对POST请求数据大小进行限制，Apache、IIS6都有各自的配置。

## 安全性
&emsp;&emsp;POST的安全性要比GET的安全性高。比如：GET提交数据，用户名和密码将以明文的形式出现在URL上，因为①登录页面有可能被浏览器缓存；②其他人查看浏览器的历史记录；除此之外，使用GET提交数据还可能会造成Cross-site request forgery攻击。

# 参考
- [简书 - 他头上长犄角 - http协议详解](https://www.jianshu.com/p/acf4efd9ebce)
- [博客园 - 不必、放弃 - http协议详解(超详细)](https://www.cnblogs.com/wangning528/p/6388464.html)
- [简书 - BennyLoo - TCP/IP协议栈 —— IP、TCP、UDP、HTTP协议详解](https://www.jianshu.com/p/dac7b8bdb682)
- [HTTP响应头和请求头信息对照表](http://tools.jb51.net/table/http_header)
- [HTTP状态码对照表](http://tools.jb51.net/table/http_status_code)