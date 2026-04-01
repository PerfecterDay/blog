# curl与wget命令简介
{docsify-updated}

## curl
1. 不带有任何参数时，curl 就是发出 GET 请求。
	`curl http://www.baidu.com`
2. `-A` ：指定客户端的用户代理标头，即 `User-Agent` 。curl 的默认用户代理字符串是`curl/[version]`
3. `-b` ：用来向服务器发送 Cookie。
	`curl -b 'foo=bar' https://google.com`
	会生成一个http头`Cookie: foo=bar`，向服务器发送一个名为foo、值为bar的 Cookie
4. `-c` ：将服务器设置的 Cookie 写入一个文件。
	`curl -c cookies.txt https://www.google.com`
5. `-d` ：用于发送 POST 请求的数据体。
	`curl -d'login=emma＆password=123'-X POST https://google.com/login`
	还可以读取本地文件的数据发送给服务器。
	`curl -d '@data.txt' https://google.com/login`
6. `--data-urlencode`：等同于`-d`，但是会自动进行URL编码。
7. `--data-raw`： 发送原始数据体
	```
	curl --location --request POST 'https://10.187.144.42:8000/trade/' \
	--header 'App-Name: gtjagjapp' \
	--header 'App-ID: sdsdawdsads' \
	--header 'Content-Type: application/json' \
	--data-raw '[
		{
			"code": "1231",
			"params": {
				"username":"700210",
				"token":"ABCDEFGHIJKLMNOPQRST0123456789",
				"account_type":"",
				"market_code":"",
				"currency_code":"HKD",
				"balance_type":"HKD",
				"language":"1"
			},
			"params_ext":{}
		}
	]'
	```
8. `-e` ：用来设置 HTTP 的标头Referer，表示请求的来源。
9.  `-F` ：用来向服务器上传二进制文件。
	`curl -F 'file=@photo.png' https://google.com/profile`
	会给 HTTP 请求加上标头Content-Type: multipart/form-data，然后将文件photo.png作为file字段上传。
10. `-G` ：用来构造 URL 的get请求的查询字符串。
11. `-H` ：添加 HTTP 请求的标头。  `curl -H "X-First-Name: Joe" https://example.com`
12. `-i` ：打印出服务器回应的 HTTP 标头。
13. `-I` ：向服务器发出 HEAD 请求，然会将服务器返回的 HTTP 标头打印出来。
14. `-k` ：指定跳过 SSL 检测。
15. `-L` ：让 HTTP 请求跟随服务器的重定向。curl 默认不跟随重定向。
16. `--limit-rate` ：用来限制 HTTP 请求和回应的带宽，模拟慢网速的环境。
17. `-o` ：将服务器的回应保存成文件，等同于 `wget` 命令.
18. `-O` ：将服务器回应保存成文件，并将 URL 的最后部分当作文件名。
19. `-s` ：将不输出错误和进度信息。
20. `-S` ：指定只输出错误信息，通常与-s一起使用。
21. `-u` ：用来设置服务器认证的用户名和密码。
22. `-v` ：输出通信的整个过程，用于调试。
23. `-x` ：指定 HTTP 请求的代理。
24. `-X` ：指定 HTTP 请求的方法。
25. `--cert`: 指定证书
26. `--form`: 指定 form 参数， `--from 'UserID=67buu'`
27. `--http1.1/--http2/--http3`: 指定使用的http 版本 
28. `curl --compressed -vv 'https://www.baidu.com' -w '%{size_download}'` : 显示压缩数据及数据大小
29. `-w` : 按指定格式显示本次传输的信息，用字符串表示
30. `--trace <file>`: 会把完整的http 请求和响应保存到一个文件，如果想在命令行输出这些信息，使用 `-` 作为文件名，如 \
    ```
	curl --trace - --location 'http://localhost:8080/user/bmp/queryUser' \
	--form 'a="2"' \
	--form '2="3"'
	```
31. `--trace-time` : 记录请求的详细时间
    ```
	curl --trace log.txt --trace-time -w "dnslookup: %{time_namelookup} | connect: %{time_connect} | appconnect: %{time_appconnect} | pretransfer: %{time_pretransfer} | starttransfer: %{time_starttransfer} | total: %{time_total} | size: %{size_download}\n" -k -H 'Accept:application/json' -H 'Content-type:application/json' -XPOST https://ttl-api-uat.demo.net/mobile/services/v0/accountBalanceEnquiry -d '{
		"clientID" : "621166",
		"sessionID": "nQMWR/vXi8lfiO3GbdZNOg=="
	}'
	```

### 检查Http服务是否支持gzip
```
curl https://www.baidu.com/ --silent --write-out "%{size_download}\n" --output /dev/null
curl https://www.baidu.com/ --silent -H "Accept-Encoding: gzip,deflate" --write-out "%{size_download}\n" --output /dev/null
```

### 网站测速
`-w` 参数的一些字段的意思：
+ `time_namelookup`: DNS 解析时间，可以与 --resolve 选项配合寻找最快的DNS
+ `time_connect`: 与服务端创建好 TCP 连接的时间，严格来说是客户端回复 ACK 的时间。我们可以通过 time_connect - time_namelookup 来大致推断网络延时。
+ `time_appconnect`: 完成 SSL/TLS 设置的时间，此时客户端与服务端完成密钥交换，客户端准备发起请求
+ `time_pretransfer`: 服务端收到请求的时间
+ `time_starttransfer`: 服务端准备好回应内容的时间。
+ `time_total`: 完成整个请求的所有时间
+ `time_redirect`: 若请求经过多次重定向，那么这个包含直到最后一次请求开始所耗的时间。

这些机时的单位都是秒s 。

```
curl -so /dev/null -w "dnslookup: %{time_namelookup} | connect: %{time_connect} | appconnect: %{time_appconnect} | pretransfer: %{time_pretransfer} | starttransfer: %{time_starttransfer} | total: %{time_total} | size: %{size_download}\n" 'https://appapi-uat-mo.demo.com'
```

```
dnslookup: 1.510 | connect: 1.757 | appconnect: 2.256 | pretransfer: 2.259 | starttransfer: 2.506 | total: 3.001 | size: 53107
```

<center><img src="pics/timingOfHTTPS.png" width="60%"></center>


## wget
wget 是一个非常实用的命令行工具，用于下载文件、网页等内容，支持 HTTP、HTTPS、FTP 等协议。
+ `wget http://example.com/file.txt`  file.txt 下载到当前目录
+ `wget -O newname.txt http://example.com/file.txt`: -o 指定保存的文件名
+ `wget -c http://example.com/largefile.zip`: 断点续传
+ `wget -p http://example.com/page.html`: 下载整个网页，包括资源
+ `wget --user=username --password=secret http://example.com/securefile`: 使用 http 认证
+ `wget --header="Cookie: session=abc123" http://example.com/private`: 携带cookie
+ `wget http://example.com/file1.txt http://example.com/file2.txt`: 下载多个文件
+ `wget -i urls.txt  # 文件中每行一个链接`: 下载多个文件
+ `wget --no-check-certificate https://example.com`: 忽略 HTTPS 证书错误