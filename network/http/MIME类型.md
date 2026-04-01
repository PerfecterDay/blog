# MIME 类型
{docsify-updated}

媒体类型（也通常称为多用途互联网邮件扩展或 MIME 类型）是一种标准，用来表示文档、文件或一组数据的性质和格式。它在 IETF 的 RFC 6838 中进行了定义和标准化。
互联网号码分配局（IANA）负责跟踪所有官方 MIME 类型，你可以在媒体类型页面中找到最新的完整列表。

## MIME 类型的结构
MIME 类型通常仅包含两个部分：类型（type）和子类型（subtype），中间由斜杠 / 分割，中间没有空白字符，还可以有一个可选的参数，能够提供额外的信息：
```
type/subtype[;parameter=value]
```
类型代表数据类型所属的大致分类，例如 video 或 text。
子类型标识了 MIME 类型所代表的指定类型的确切数据类型。以 text 类型为例，它的子类型包括：plain（纯文本）、html（HTML 源代码）、calender（iCalendar/.ics 文件）。
每种类型都有自己的一组可能的子类型。一个 MIME 类型总是包含类型与子类型这两部分，且二者必需成对出现。

MIME 类型对大小写不敏感，但是传统写法都是小写。参数值可以是大小写敏感的。

## MIME的种类
类型可分为两类：
+ 独立的（discrete）
+ 多部分的（multipart）。

### 独立类型
独立类型代表单一文件或媒介，比如一段文字、一个音乐文件、一个视频文件等。
而多部份类型，可以代表由多个部件组合成的文档，其中每个部分都可能有各自的 MIME 类型；此外，也可以代表多个文件被封装在单次事务中一同发送。多部分 MIME 类型的一个例子是，在电子邮件中附加多个文件。

多部分类型指的是一类可分成不同部分的文件，其各部分通常是不同的 MIME 类型；也可用于——尤其在电子邮件中——表示属于同一事务的多个独立文件。它们代表一个复合文档。
HTTP 不会特殊处理多部分文档：信息会被传输到浏览器（如果浏览器不知道如何显示文档，很可能会显示一个“另存为”窗口）。除了几个例外，在 HTML 表单的 POST 方法中使用的 `multipart/form-data`，以及用来发送部分文档，与 206 Partial Content 一同使用的 `multipart/byteranges`。

### 多部分的类型
有两种多部分类型：

1. message
封装其他信息的信息。例如，这可以用来表示将转发信息作为其数据一部分的电子邮件，或将超大信息分块发送，就像发送多条信息一样。例如，message/rfc822（用于转发或回复信息的引用）和 message/partial（允许将大段信息自动拆分成小段，由收件人重新组装）是两个常见的例子。（查看 IANA 上 message 类型的注册表）

2. multipart
由多个组件组成的数据，这些组件可能各自具有不同的 MIME 类型。例如，multipart/form-data（用于使用 FormData API 生成的数据）和 multipart/byteranges（定义于 RFC 7233, section 5.4.1，当获取到的数据仅为部分内容时——如使用 Range 标头传输的内容——与返回的 HTTP 响应 206 “Partial Content”组合使用）。（查看 IANA 上 multipart 类型的注册表）



`multipart/form-data`

```
<form
  action="http://localhost:8000/"
  method="post"
  enctype="multipart/form-data">
  <label>名字：<input name="myTextField" value="Test" /></label>
  <label><input type="checkbox" name="myCheckBox" /> 勾选</label>
  <label>
    上传文件：<input type="file" name="myFile" value="test.txt" />
  </label>
  <button>发送文件</button>
</form>
```

发送的HTTP报文：
```
POST / HTTP/1.1
Host: localhost:9999
Connection: keep-alive
Content-Length: 386
Cache-Control: max-age=0
sec-ch-ua: "Not/A)Brand";v="8", "Chromium";v="126", "Google Chrome";v="126"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "macOS"
Upgrade-Insecure-Requests: 1
Origin: null
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryBzBehasJKoeCVU5z
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7

------WebKitFormBoundaryBzBehasJKoeCVU5z
Content-Disposition: form-data; name="myTextField"

Test
------WebKitFormBoundaryBzBehasJKoeCVU5z
Content-Disposition: form-data; name="myCheckBox"

on
------WebKitFormBoundaryBzBehasJKoeCVU5z
Content-Disposition: form-data; name="myFile"; filename="a.txt"
Content-Type: text/plain

123

------WebKitFormBoundaryBzBehasJKoeCVU5z--
```
