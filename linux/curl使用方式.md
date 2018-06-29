[TOC]





## curl使用方式

注：URL代表一个请求url,可能包括query参数，列如：http://localhost:8080/workflow-api/hello/sayHello?name=yd



#### 参考博客：https://blog.csdn.net/qidi_huang/article/details/51278056



### 概述

1. curl 命令用作网络数据包收发，常应用于非交互式环境中。
2. URL 的格式依赖于命令所使用的网络协议，相关详细信息可以查看《RFC 3986》文档。
3.  如果在一条命令中访问多个文件，crul 会尝试在多个传输会话间重用一个连接，以此减少建立不必要的连接或握手，从而访问速度。多个相互独立的 curl 命令调用之间不支持连接重用。
4. curl 运行时，默认启用传输进度条用于显示总传输数据大小、传输速度、预计剩余传输时间等。
5. 进度条数据**默认**直接输出到控制台。如果不希望这些数据在你 POST 或 PUT 时扰乱了 response数据包，可以使用重定向 > 或后跟参数 -o [file] 或其它同样功能的语句将相应数据重定向到文件中。
6. 使用参数 -# 可以让她看起来更像真正的进度“条”



#### URL技巧

- 字符串匹配    http://site.{one,two,three}.com

- 字符匹配    ftp://ftp.letters.com/file[a-z].txt

- 数字匹配    ftp://ftp.numerical.com/file[1-100].txt

-  混合匹配    http://any.org/archive[1996-1999]/vol[1-4]/part{a,b,c}.html

- 步进匹配    http://www.numerical.com/file[1-100:10].txt

  ​		    http://www.letters.com/file[a-z:2].txt



#### 【option参数】

- **-h/--help**    显示 curl 命令使用方法的简要帮助信息

- ​    **-M/--manual**    显示详细帮助文档

- ​    **-a/--append**    向服务器端文件追加内容。如果服务器端文件不存在，将创建该文件。

- **-A/--user-agent <agent string>**    为 HTTP 数据包指定 User-Agent 字段内容，即浏览器信息。如果内容含有空格，请使用单引号括起来。有的网站要求访问者必须使用特定的浏览器甚至版本，这种情况下可以使用该参数绕过服务器的检测。

- **-H/--header <header string>**    为 HTTP 数据包指定 Header 字段内容。如果要删除某个 header 字段，可以使用 <字段名>: 的格式替换 <header string> 代表的内容。比如删除 Host 字段的内容，可以这样添加 -H 参数：-H "Host:"不要手动为你的 header 内容添加换行标记 \r\n，因为 curl 会帮你处理，否则只会让你的代码出现莫名奇妙的问题。

- **-I/--head**    只接收 response数据包中 header 字段的内容。

- **-b/--cookie <name=data>**    为发送的 HTTP 数据包指定 cookie 内容，通常是之前同服务器通讯时，从服务器接收到的 response数据包中 Set-Cookie 字段中的内容。如果要指定多对值，格式为“NAME1=VALUE1;NAME2=VALUE2”.如果内容中没有等号 =，则会被解析为存储 Cookie 的文件，此时还应该配合参数 -L/--location 使用。

- **-c/--cookie-jar <filename>**    存储 response数据包中的 cookie 信息到文件中。如果文件名是破折号 -，则 cookie 将在 stdout 被打印出来。Cookie信息使用 Netscape cookie文件的格式进行记录。cookie 写入失败不会导致 curl 命令中止，使用 -v 参数可以在操作失败时看到警告信息。

- **-D/--dump-header <filename>**    存储 response数据包中的 header 信息到文件中。在 FTP 传输中使用时，FTP 服务器的回复消息会被视作 headers 存储起来。

-    --connect-timeout <seconds> **   设置连接超时。这个值是对于连接建立阶段而言的，一旦连接被建立，这个限制就没用了。

- **-m/--max-time**

- **-C/--continue-at <offset>**    断点续传。<offset>是一个数值，表示从文件头开始计算的要跳过的字节数。如果<offset>被设定为破折号 -，那么 curl 命令将自行检测待传输文件的断点位置。

- **--create-dirs**    创建目录。通常与参数 -o 连用，用于为 /dir/filename 这样的参数创建目录 dir/。比如 -c /recvDir/cookie.txt --create-dirs

- **--ftp-create-dirs**    为 FTP/SFTP 传输创建远程目录。

- **--crlf **   在 FTP 上传中将 LF 转换为 CRLF。

- **-d/--data <data>**    为 POST 数据包指定要向 HTTP 服务器发送的数据并发送出去。这个过程和在浏览器中点击“submit”按钮是一样的，且数据将以 content-type application/x-www-form-urlencoded 的方式被编码。
  在同一条命令中多次使用 -d 参数，参数后的内容会被符号 & 拼接起来。比如 -d name=daniel -d skill=lousy 会被转化为 name=daniel&skill=lousy

  如果<data>的内容以符号 @ 开头，其后的字符串将被解析为文件名，curl 命令会从这个文件中读取数据发送，但文件中的内容必须是 URL 编码的格式；如果内容以符号 - 开头，curl 命令将从 stdin 读取数据发送。

- **--data-binary <data>**    为 POST 数据包指定要向 HTTP 服务器发送的数据并发送出去。数据将以二进制的形式被送出。

- **--data-urlencode <data>**    为 POST 数据包指定要向 HTTP 服务器发送的数据并发送出去。数据在发送前将进行 URL-encode 编码。

- **--digest **   为 HTTP 传输启用摘要加密算法，防止登录信息被明文传输。通常在其后使用 -u/--user 参数指定用户名和密码。

- **-e/--referer <URL>**    为 HTTP 数据包指定 Referer Page 信息，即前一个被访问页面的 URL。通常这个信息被服务器用于判断自己是否被盗链，如果发现服务器端有这样的检测机制，则可以使用该参数绕过检测。

- **-E/--cert <certificate[:password]>**    为 HTTPS/FTPS 数据包指定数字证书。数字证书必须是 PEM 格式。如果 password 没有在内容中显式给定，则会在连接建立时被服务器端询问。

- **-f/--fail **   禁止服务器在打开页面失败或脚本调用失败时向客户端发送错误说明，取而代之，curl 会返回错误码 22。

- **-F/--form <name=content>**   模拟用户在浏览器上点击“submit”按钮提交表单的操作。如果想提交文件，可以将<name=content>中 content部分的内容替换为 @filename 的格式；如果想提交文件中的内容，可以将 content部分的内容替换为 <filename 格式；如果想提交从 stdin 获取的到的数据，可以将 content部分的内容替换为 @- 或者 <-
  ​                                                   比如，要提交存储密码的文件到服务器上，命令为：curl -F password=@/etc/passwd www.mypasswords.com
  ​                                                   要将文件中的内容作为密码提交到服务器上，命令为：curl -F password=</etc/passwd www.mypasswords.com
  ​                                                   在<name=content>内容中添加“type=xxx”字段可以指定 Content-Type 类型。
  ​                                                   比如 curl -F "web=@index.html;type=text/html" url.com 
  ​                                                   或者 curl -F "name=daniel;type=text/foo" url.com
  ​                                                   还可以从文件读取待发送内容后，修改数据包中的文件名信息，像这样：curl -F "file=@localfile;filename=nameispost" url.com

- **-g/--globoff**   关闭 URL 的通配符功能。这样就可以访问名称中含有字符 {、}、[、] 的文件了。

- **-G/--get  **  将 -d 参数后指定的数据以 GET 方法打包发送，并在数据末尾添加一个问号 ?

- **--hostpubmd5 <md5>**    为数据包指定一个 32 位的十六进制数。这个数应该与远程服务器公钥的 MD5 值相同。如果 MD5 值不相同，curl 将中止连接。这个参数只在 SCP/SFTP 传输协议中使用。

- **-i/--include**   使输出信息中包含 HTTP-header 的内容，比如 server-name、HTTP-version 等。

- **--interface <name>**   使用指定网卡访问/传输数据。命令格式：curl --interface eth0:1 http://www.netscape.com/

- **--keepalive-time <seconeds>  **  为连接设定保活时间。这个值同时也是相邻 2 次发送保活消息的时间间隔。

- ​

  --K/--config <config file>   为 curl 命令指定一个配置文件，curl 会从该文件中读取内容并作为自己的运行参数。默认的配置文件是 ~/.curlrc。当 <config file> 的内容被写成符号 - 时，curl 会从 stdin 读取配置。
                                                  配置文件每一行只能写一个参数；支持使用符号 \ 对字符进行转义；如果内容中含有空格，需用引号括起来；以符号 # 开始的内容会被视为注释；文件中的 URL 信息需要写成 url = URL 这样的格式。
                                                  配置文件示例如下：

  --- Example file ---

  this is a comment

                                                  url = "curl.haxx.se"
                                                  output = "curlhere.html"
                                                  user-agent = "superagent/1.0"

  and fetch another URL too

                                                  url = "curl.haxx.se/docs/manpage.html"
                                                  -O
                                                  referer = "http://nowhereatall.com/"

  --- End of example file ---
  ​


- **-q **   如果这个参数是 curl 命令的第一个参数，那么 curl 命令将不会去读取默认的 curlrc 配置文件。
- **--libcurl <file> **   将本条命令实现的功能转换成调用 libcurl 的 C 代码，并保存在 <file> 文件中。但是目前还不能很好地转化带 -F 参数的命令。
- **--limit-rate <speed>**    为数据包指定最大传输速率，这样就不会占用整个带宽。通常在数据处理管道不够用时使用。默认的速率单位是 bytes/s，如果在数字后加上 k、m、g 等后缀，如 200k，则表示 200Kb/s。
- **--local-port <num>[-num]**    为连接指定一个本地端口。
- **--L/--location**   当服务器报告被请求的页面已被移动到另一个位置时（通常返回 3XX 错误代码），允许 curl 使用新的地址重新访问。如果跳转链接指向了一个不同的主机，curl 将不向其发送用户名和密码。
- **--location-trusted **   该参数和 -L 参数类似，也可让 curl 继续访问跳转链接，区别在于该参数允许向跳转链接发送明文用户名和密码。
- **--max-redirs <num>**   指定最大跳转次数。默认的次数限制为 50。如果将 <num> 设置为 -1 表示不进行限制。
- **-m/--max-time <seconds>**    为数据传输过程指定超时时间。如果传输过程在这个时间内没有完成，连接将被中止。防止脚本因为网络不好或链接失效而挂死。
- **-o/--output <file>**    将获取到的数据存入文件中，而不是打印出来。如果在 URL 中使用了 {} 或 [] 进行通配，我们就可以在 <file> 中使用 #1、#2 ... 的形式来顺序表示这些通配符中的内容。
  ​                                     比如下面这样的写法：
  ​                                     curl http://{one,two}.site.com -o "file_#1.txt"
  ​                                     curl http://{site,host}.host[1-5].com -o "#1_#2"
- **-O/--remote-name  **  在本地保存获取的数据时，使用她们在远程服务器上的文件名进行保存。
- **--random-file <file>   ** 为 SSL 连接指定一个存有随机数的种子的文件。
- **--retry <num>  **  在数据包的传输失败时，为其指定重试次数。默认次数为 0，即不重传。连接超时、500错误代码都会引起重传
- **-s/--silent  ** 将 curl 设定在静默模式下工作。进度条和错误消息都不会被显示。
- **--stderr <file>**    重定向错误输出到文件。
- **-T/--upload-file <file>  ** 这个参数用于向远程服务器传输指定文件。上传文件时，URL必须以符号 / 结尾才会被识别为目录。当文件名是符号 - 时，curl 将从 stdin 读取输入数据进行上传。如果远端是 HTTP 服务器，那么应该使用 PUT 方法。
  ​                                          命令格式如下：
  ​                                          curl -T "{file1,file2}" http://www.uploadtothissite.com
  ​                                          curl -T "img[1-1000].png" ftp://ftp.picturemania.com/upload/
- **--trace <file> **   启用对所有数据包传输的追踪，并记录到指定文件中。当文件名是符号 - 时，追踪消息将输出到标准输出。
- **--trace-ascii <file>**    类似于 --trace ，但是是以 ascii 的形式输出的而非十六进制，便于阅读。
- **-u/--user <user:password>  **  为数据传输在服务器端的身份验证提供用户名/密码信息。
- **-U/--proxy-user <username:password>**    为数据传输在代理服务器上的身份验证提供用户名/密码信息。
- **-v/--verbose **   显示数据传输过程中的详细信息。
- **-V/--version**    显示 curl 及其所使用的 libcurl 的版本信息。
- **-w/--write-out <format>**    在数据传输完成后，输出和本次传输相关的参数信息，比如 header 的大小、下载速度等。<format>部分即格式化输出内容，变量使用格式为 %{varible}，使用 %% 输出百分号 %，使用 \r、\n、\t 输出 回车、换行、制表符。
  ​                                               所有支持的变量如下表所示：
  ​                                               url_effective    上次访问的URL。
  ​                                               http_code    上一次 HTTP 或 FTP 数据传输过程中的 response 数值代码。
  ​                                               http_connect    上一次 CONNECT 请求中的数值代码
  ​                                               time_total    数据传输消耗的总时间，以秒为单位，精度为毫秒。
  ​                                               time_namelookup    从数据传输开始到域名解析完成所花费的时间。
  ​                                               time_connect    TCP连接建立成功所花费的时间。
  ​                                               time_appconnect    应用层协议，如 SSL/SSH、三次握手等过程完成所花费的时间。
  ​                                               time_redirect    从跳转链接被激活到真正开始从跳转链接下载数据所经过的时间。
  ​                                               time_starttransfer    从请求连接开始，到第一个字节被传送前所经过的时间。
  ​                                               size_download    数据传输过程中下载的总数据大小。
  ​                                               size_upload    数据传输过程中上传的总数据大小。
  ​                                               size_header    下载的数据包中，header 字段的总数据大小。
  ​                                               size_request    被发送的 HTTP request 的总数据大小。
  ​                                               speed_download    整个数据传输过程中的平均数据下载速度。
  ​                                               speed_upload    整个数据传输过程中的平均数据上传速度。
  ​                                               content_type    被请求访问的文件的 Content_Type 类型。
  ​                                               num_redirects    访问请求中包含的跳转链接数量。
  ​                                               redirect_url    跳转链接指向的URL
  ​                                               ssl_verify_result    SSL验证的结果。值为 0 时表示验证成功。
- **-x/--proxy <proxyhost[:port]>**    为数据传输指定代理服务器。如果没有显示指定端口号，则使用默认端口号1080
- **-X/--request <command>**    为 HTTP 数据包指定一个方法，比如 PUT、DELETE。默认的方法是 GET。
- **-z/--time-cond <date expression>**    请求一个时间点比 <date expression> 中描述的时间更早或更晚的文件。<date expression>以符号 - 开始时表示访问比该时间点更早的文件，以数字开始时表示访问比该时间点更晚的文件。
- **-0/--http1.0  **  强制在 HTTP 传输时使用 HTTP 1.0 协议。默认使用 HTTP 1.1
- **-1/--tlsv1   ** 强制在 SSL 传输时使用 TLS v1 协议
- **-2/--sslv2 **   强制在 SSL 传输时使用 SSL v2 协议
- **-3/--sslv3 **   强制在 SSL 传输时使用 SSL v3 协议
- **-4/--ipv4  **  强制解析域名为 IPv4 地址
- **-6/--ipv6**    强制解析域名为 IPv6 地址





####【退出代码/返回值】

- ​    1    curl 不支持该协议
- ​    2    curl 初始化失败
- ​    3    URL 格式错误
- ​    5    解析代理服务器失败
- ​    6    解析主机失败
- ​    7    建立与主机的连接失败
- ​    8    无法解析 FTP 服务器返回的消息
- ​    9    FTP 服务器拒接访问。可能是拒绝登录或拒绝访问特定目录，但很多情况下是访问了一个不存在的位置导致的
- ​    11    无法解析 FTP 服务器的 PASS 回复消息
- ​    13    无法解析 FTP 服务器的 PASV 回复消息
- ​    14    无法解析 FTP 服务器的 227-line 回复消息
- ​    15    无法解析 FTP 主机
- ​    17    无法与 FTP 服务器建立二进制传输模式
- ​    18    文件传输不完整。只有文件的一部分被传送了。
- ​    19    FTP 下载/访问指定的文件失败
- ​    21    FTP 引用错误
- ​    22    HTTP 页面获取失败。将返回 400 及其以上的错误码。
- ​    23    写入数据到本地文件系统发生错误
- ​    25    FTP 服务器无法存储被上传的文件
- ​    26    读取数据出错
- ​    27    内存分配失败
- ​    28    操作超时
- ​    30    FTP 服务器运行 PORT 命令失败
- ​    31    FTP 服务器运行 REST 命令失败
- ​    33    HTTP 服务器执行 range 命令失败
- ​    34    HTTP 服务器 post 方法错误
- ​    35    SSL 连接出错。通常是 SSL 握手失败。
- ​    36    FTP 断点续传出错，无法继续前一次下载任务。
- ​    37    FILE 协议无法打开文件。可能是没有权限导致的
- ​    38    LDAP 协议无法执行 bind 操作
- ​    39    LDAP 协议执行搜索功能失败
- ​    41    没有找到匹配的 LDAP 功能
- ​    42    curl 被其它应用程序中止运行
- ​    43    发生内部错误。通常是因为调用函数时传递了错误的参数
- ​    45    接口调用错误
- ​    47    跳转链接数量达到上限
- ​    48    为 telnet 协议指定了一个未知的参数
- ​    49    telnet 协议的参数格式有误
- ​    51    SSL 认证或 SSH 的 MD5 指纹不正确
- ​    52    服务器无应答
- ​    53    未找到 SSL 加密引擎
- ​    54    无法将 SSL 加密引擎设为默认引擎
- ​    55    发送网络数据失败
- ​    56    接收网络数据失败
- ​    58    本地证书有问题
- ​    59    无法使用指定的 SSL 密码
- ​    60    已知的 CA证书无法用于验证
- ​    61    无法识别传输编码
- ​    62    无效的 LDAP URL
- ​    63    文件大小超过限制
- ​    64    请求的 FTP SSL 级别获取失败
- ​    65    发送需要进行 rewind 的数据失败
- ​    66    解析 SSL 引擎失败
- ​    67    用户名/密码验证失败，curl 无法登录
- ​    68    TFTP 服务器上没有找到指定的文件
- ​    69    对 TFTP 服务器的访问出现权限问题
- ​    70    TFTP 服务器已耗尽磁盘空间
- ​    71    非法的 TFTP 操作
- ​    72    未知的 TFTP 传输 ID
- ​    73    TFTP 企图传输一个已经存在的文件
- ​    74    用户不存在导致 TFTP 操作失败
- ​    75    字符转换失败
- ​    76    需要使用字符转换函数
- ​    77    读取 SSL CA 证书出错。可能是证书路径不正确，也可能是没有权限
- ​    78    URL 中指定的资源不存在
- ​    79    SSH 会话中发生了未知错误
- ​    80    中断 SSH 连接失败
- ​    82    无法加载 CRL 文件。可能是没有指定正确的格式
- ​    83    发布者身份验证失败



### 语法：curl 【options】【URL】

curl -d "name=yd" -G -i --keepalive-time 30 --limit-rate 200 http://192.168.1.58:8091/workflow-api/hello/sayHello -o "a.txt" --stderr "b.txt" --trace "c.txt" -w %{http_code}







