# MIME 类型
{docsify-updated}
> https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/MIME_types

媒体类型（也通常称为多用途互联网邮件扩展或 MIME 类型）是一种标准，**用来表示文档、文件或一组数据的性质和格式**。它在 IETF 的 RFC 6838 中进行了定义和标准化。
互联网号码分配局（IANA）负责跟踪所有官方 MIME 类型，你可以在媒体类型页面中找到最新的完整列表。

## MIME 类型的结构
MIME 类型通常仅包含两个部分：**类型（type）和子类型（subtype）**，中间由斜杠 `/` 分割，中间没有空白字符，还可以有一个可选的参数，能够提供额外的信息：
```
type/subtype[;parameter=value]
```
类型代表数据类型所属的大致分类，例如 `video` 或 `text` 。
子类型标识了 MIME 类型所代表的指定类型的确切数据类型。以 `text` 类型为例，它的子类型包括：`plain`（纯文本）、 `html`（HTML 源代码）、`calender`（ `iCalendar/.ics` 文件）。
每种类型都有自己的一组可能的子类型。一个 MIME 类型总是包含类型与子类型这两部分，且二者必需成对出现。

MIME 类型对大小写**不敏感**，但是传统写法都是小写。参数值可以是大小写敏感的。

## MIME的种类
类型可分为两类：
+ 独立的（discrete）
+ 多部分的（multipart）。

### 独立类型
独立类型代表单一文件或媒介，比如一段文字、一个音乐文件、一个视频文件等。

IANA 目前注册的独立类型如下：
1. `application`: 不明确属于其他类型之一的任何二进制数据；要么是将以某种方式执行或解释的数据，要么是需要借助某个或某类特定应用程序来使用的二进制数据。
2. `audio`: 音频或音乐数据。常见的示例如 `audio/mpeg` 、 `audio/vorbis` 
3. `example`: 在演示如何使用 MIME 类型的示例中用作占位符的保留类型。这一类型永远不应在示例代码或文档外使用
4. `font`: 字体/字型数据。常见的示例如 `font/woff` 、 `font/ttf` 和 `font/otf`
5. `image`: 图像或图形数据，包括位图和矢量静态图像，以及静态图像格式的动画版本,如 GIF 动画或 APNG。常见的例子有 `image/jpeg` 、 `image/png` 和 `image/svg+xml`
6. `model`: 三维物体或场景的模型数据。示例包含 `model/3mf` 和 `model/vrml`
7. `text`: 纯文本数据，包括任何人类可读内容、源代码或文本数据——如逗号分隔值（comma-separated value，即 CSV）格式的数据。示例包含：`text/plain` 、 `text/csv`  和 `text/html`
8. `video`: 视频数据或文件，例如 MP4 电影（ `video/mp4` ）

### 多部分的类型
而多部份类型，可以代表由多个部件组合成的文档，其中每个部分都可能有各自的 MIME 类型；此外，也可以代表多个文件被封装在单次事务中一同发送。多部分 MIME 类型的一个例子是，在电子邮件中附加多个文件。

多部分类型指的是一类可分成不同部分的文件，其各部分通常是不同的 MIME 类型；也可用于——尤其在电子邮件中——表示属于同一事务的多个独立文件。它们代表一个复合文档。
HTTP 不会特殊处理多部分文档：信息会被传输到浏览器（如果浏览器不知道如何显示文档，很可能会显示一个“另存为”窗口）。除了几个例外，在 HTML 表单的 POST 方法中使用的 `multipart/form-data`，以及用来发送部分文档，与 206 Partial Content 一同使用的 `multipart/byteranges`。

有两种多部分类型：

1. `message`  
封装其他信息的信息。例如，这可以用来表示将转发信息作为其数据一部分的电子邮件，或将超大信息分块发送，就像发送多条信息一样。例如，`message/rfc822`（用于转发或回复信息的引用）和 `message/partial`（允许将大段信息自动拆分成小段，由收件人重新组装）是两个常见的例子。（

1. `multipart`  
由多个组件组成的数据，这些组件可能各自具有不同的 `MIME` 类型。例如， `multipart/form-data`（用于使用 FormData API 生成的数据）和 `multipart/byteranges`（定义于 RFC 7233, section 5.4.1，当获取到的数据仅为部分内容时——如使用 Range 标头传输的内容——与返回的 HTTP 响应 206 “Partial Content”组合使用）。


## 常见的 MIME 类型
`multipart/form-data` ：可用于 HTML 表单从浏览器发送信息给服务器。

作为多部分文档格式，它由边界线（一个由双横滑线 `--` 开始的字符串）划分出的不同部分组成。每一部分有自己的实体，以及自己的 HTTP 请求头，`Content-Disposition` 和 `Content-Type` 用于文件上传字段。
```
Content-Type: multipart/form-data; boundary=aBoundaryString
(other headers associated with the multipart document as a whole)

--aBoundaryString
Content-Disposition: form-data; name="myFile"; filename="img.jpg"
Content-Type: image/jpeg

(data)
--aBoundaryString
Content-Disposition: form-data; name="myField"

(data)
--aBoundaryString
(more subparts)
--aBoundaryString--
```

示例：
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
