---
layout: title
title: Apache Rewrite with Htaccess理解与技巧
date: 2019-05-15 17:50:12
categories: Apache
tags: Apache
---
台湾人写的繁体字，我给改成了简体字方便阅读。

来源：[Apache Rewrite with Htaccess理解与技巧](https://medium.com/@awonwon/htaccess-with-rewrite-3dba066aff11 "Apache Rewrite with Htaccess理解与技巧")

<!--more-->

在搜寻Apache针对网址改写、避免访问敏感档案时，都会看到RewriteRule、RewriteCond等的Directive，但因为有时候不理解其中特性或Rule跟Cond的差异等，加上若不熟悉正则表达式，完全只想逃避这些Rule，因此本文将介绍：

一、开启Rewrite功能
二、RewriteRule
三、RewriteCond
四、如何排错
五、一些小特性
六、使用Htaccess的缺点


# 一、开启Rewrite功能

进行以下操作开启Rewrite模块（CentOS7编译安装）

找到apache的配置文件httpd.conf并寻找下面这行：
```Apache
LoadModule rewrite_module modules/mod_rewrite.so
```

将前面“#”去掉，如果不存在则添加此句。

确认是否已经开启模块了

```Bash
$ cd /usr/local/apache/bin
$ sudo apachectl -M | grep rewrite
rewrite_module (shared)
```

接着就可以开始在Site Config里面撰写所需的RewriteRule了。

在理解Rewrite的过程中，常常会出现与之搭配的.htaccess文件。.htaccess称作「Hypertext Access」，以一个文件夹为单位改变Apache设定的配置（Override Config），简单来说就是可以根据每个文件夹Override原本Site Config，可以针对一个文件夹改写网址，所以RewriteRule并非就只能写在.htaccess当中哦！

但本篇文章还是会以使用.htaccess作为范例，若要打开Override的功能，只要修改Site Config，加入AllowOverride All就可以了（例如/usr/local/apache/conf/vhost/example.conf）
```Apache
<Directory "/var/www/html">
    AllowOverride All
</Directory>
```
All、FlieInfo都可以让.htaccess使用Rewrite功能，详情可以参考官方设定值的意义。

# 二、Rewrite Rule

** 基本用法 **

就是改写网址条件的规则，它的写法结构如下：
```Apache
RewriteRule [match_uri] [rewrite_uri] [flags]
```
match_uri：符合Pattern的URI
rewrite_uri：将被改写的URI
这两个也都可以使用正则表达式撰写，一个Rule范例长这样：
```Apache
Rewrite ^match\.html$ rewrite.html [NC,L]
```
意义等同于以下Pseudo Code：
```JavaScript
假设输入网址：http://domain.com/a/b/c.html
uri = a/b/c.html
if (uri.match("match.html") 
{
    url = "rewrite.html"
}
```
** RewriteRule的Flag **

最后面的flags代表设定Rule的行为，可用逗号代表多个Flag，中间不能有空格，介绍以下常用的：
```
[L]：Last，代表成功执行这个Rule后就会停止，不继续往下执行。
[NC]：Non Case-sensitive，代表match_uri不比对大小写差异。
[QSA]：Query String Append，代表保留网址尾端带的GET参数，没使用flag的默认是会把参数去掉的。
[QSD]：Query String Discard（丢弃），与QSA相反的作用，apache v2.4才有。
[R]: Redirect，代表用转址的方式转到新的网址，默认是302 Status Code，如：[R=301]，也可以回传400、200、404等的Status Code，通常会跟[L]一起代表结束，也是排错常用的Flag
[DPI]: 不要再接续的Rule中结尾中加上 PathInfo，会在「五、一些小特性」的段落说明。
[F]: Forbidden 就是不给看啦！
※ more flags: http://httpd.apache.org/docs/2.4/rewrite/flags.html
```

** 范例 **

接着来举例多个Rules加上Flag的功用，假设网站文件夹结构如下：
```
root/
 ├ match.html
 ├ rewrite.html
 ├ .htaccess
 └ secret/
    └ database_password.json
```
而范例的.htaccess的內容为：
```Apache
# 前面有井字号是注释
### 开启Rewrite
RewriteEngine On

### 设定Rewrite前面会加上的path，默认会是DocumentRoot(如：/var/www/html）
RewriteBase /

# Rules 将会由上往下依序执行
# 直到最后一行或遇到有符合且有 L flag 的 Rule 就会停止
### Rule 1. 输入 domain.com/match.html 将会显示 rewrite.html 的內容RewriteRule ^match\.html$ rewrite.html [NC,L]

### Rule 2. 输入 domain.com/redirect.html 将会被导至 domain.com/rewrite.html
RewriteRule ^redirect\.html$ rewrite.html [NC,R=302,L]

### Rule 3. 如果输入 domain.com/secret/… 这样格式的网址，则去掉 secret/ 后，转回 root 并加上 .html

### $1 是正则表达式的 group capture，就是 $1=(.*) 取得括号內的值
RewriteRule ^secret/(.*)$ $1.html [NC,L]
```
在浏览器上输入 match.html或 redirect.html均会显示 rewrite.html的內容，明显是 Rule 2 将网址改变了，而 Rule 1 没有，因为就是 302 Status Code 的关系。

在此建议可以使用 POSTMAN或是 cURL命令列或 Request 擷取等工具观察整个 Response的差異，将会非常容易除错，详细原因将在「四、如何除错」段落中解释，以下使用 cURL示范：

1.Request：http://localhost/match.html
```
$ curl 'http://localhost/match.html'
I’m rewrite.html
```

2.Request： http://localhost/redirect.html
```Bash
$ curl 'http://localhost/redirect.html'

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>302 Found</title>
</head><body>
<h1>Found</h1>
<p>The document has moved <a href="http://localhost/rewrite.html">here</a>.</p>
<hr>
<address>Apache/2.4.33 (Win32) OpenSSL/1.1.0h PHP/7.2.7 Server at localhost Port 80</address>
</body></html>
```
与浏览器不同的是，这里的Response会指出网址被转移至另外一个新网址http://localhost/rewrite.html，浏览器自动帮忙处理转址的工作，根据上面回传的网址，再发出另一个Request取得网页內容，让使用者仅感觉到网址与页面改变而已。

这时候读者可试着把RewriteBase这行注解，将会发现回传结果出错了，Rewrite结果与Domain之间被加上DocumentRoot预设的路径。
```Bash
$ curl 'http://localhost/redirect.html'

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>302 Found</title>
</head><body>
<h1>Found</h1>
<p>The document has moved <a href="http://localhost/var/www/html/rewrite.html">here</a>.</p>
<hr>
<address>Apache/2.4.33 (Win32) OpenSSL/1.1.0h PHP/7.2.7 Server at localhost Port 80</address>
</body></html>
```
前面两个Rule仅是一个基本的示范，平常应用当然会难上许多，来继续看第三个Rule吧

```
### Rule 3. 如果输入domain.com/secret/…这样格式的网址，则去掉 secret/后，转回root并加上.html

### $1是正则表达式的group capture，就是取得那个满足括号内的值
RewriteRule ^secret/(.*)$ $1.html [NC,L]
```
第三个Rule是禁止使用者访问敏感的secret文件夹，来试着访问database_password.json看能不能得到结果：

```Bash
$ curl 'http://localhost/secret/database_password.json'

HTTP/1.1 404 Not Found
Date: Sun, 07 Oct 2018 13:39:11 GMT
Server: Apache/2.4.33 (Win32) OpenSSL/1.1.0h PHP/7.2.7
Vary: accept-language,accept-charset
Accept-Ranges: bytes
Content-Type: text/html; charset=utf-8
Content-Language: en
```
不管网址改成任何http://localhost/secret/...，都会回传404找不到页面，是理想中的状况，非常棒！但是访问 http://localhost/secret/反而变成403禁止访问了耶，咦？
```Bash
$ curl 'http://localhost/secret/'

HTTP/1.1 403 Forbidden
Date: Sun, 07 Oct 2018 13:43:08 GMT
Server: Apache/2.4.33 (Win32) OpenSSL/1.1.0h PHP/7.2.7
Vary: accept-language,accept-charset
Accept-Ranges: bytes
Content-Type: text/html; charset=utf-8
Content-Language: en
```
这时候不知出了什么错的话，可以将 Rule 的 Flag 加上 R=302，检查最后改写的结果出了什么状况：
```Bash
$ curl 'http://localhost/secret/'

<!DOCTYPE HTML PUBLIC “-//IETF//DTD HTML 2.0//EN”>
<html><head>
<title>302 Found</title>
</head><body>
<h1>Found</h1>
<p>The document has moved <a href=”http://localhost/.html">here</a>.</p>
<hr>
<address>Apache/2.4.33 (Win32) OpenSSL/1.1.0h PHP/7.2.7 Server at localhost Port 80</address>
</body></html>
```
发现 $1没有抓到任何字串，所以没有任何檔名加上 .html，就变成 403 的结果，所以只要在 Rule 3 之前加上以下新的 Rule 就可以！
```Apache
RewriteRule ^secret/$ / [R=302,NC,L]
```
以上就是RewriteRule与简单的Debug方法，只要再加上看得懂正则表达式就能懂一般常见的Rules！继续看更难一点的RewriteCond吧～

# 三、RewriteCond

RewriteRule仅仅只能判断Request URI是否匹配而改写URI，但有很多需求是希望根据一些Request Header（Host、User-agent）与Apache的环境变量做改写，先满足某些条件后，再次Rewrite URI，因此有了RewriteCond的出现，它的写法结构如下：
```Apache
RewriteCond [test_string] [match_string] [flags]
RewriteRule …
```
test_string：要比对的条件
match_string：符合的条件

以上这两个都可以使用正则表达式撰写，而且RewriteCond结束一定会接着一个RewriteRule，真正的范例会长这样：
```Apache
RewriteCond %{HTTP_USER_AGENT} (facebookexternalhit)
RewriteRule ^blog/(.*)$ fb-bot.html?path=$1&type=%1 [L]
```
以上意义等同于以下Pseudo Code：
```
if ($HTTP_USER_AGENT == 'facebookexternalhit') 
{
    if (url.match('^blog/(.*)$')) 
    {
        url = 'fb-bot.html?path=$1&type=facebookexternalhit';
    }
}
```
※ 前面说到$1是Group Capture的用法，而RewriteCond则是用%1表示

** RewriteCond可使用的变量 **

${HTTP_USER_AGENT}是RewriteCond可使用的变量，有以下常见的变量：
```
%{REQUEST_URI}：Domain后面完整的URI Path，Rule其实会拿到不完整的URI，详情可以参考「五、一些小特性」段落
%{QUERY_STRING}：后面GET带的参数
%{HTTP_HOST}：Domain 例：163.com
%{HTTP_COOKIE}：Cookie
%{HTTPS}：判断是否用https或http，如果是https就等于「on」，否则为“off”
%{HTTP_USER_AGENT}：User Agent
%{REQUEST_FILENAME}：访问的文件名称

※ more variables：https://httpd.apache.org/docs/current/mod/mod_rewrite.html#rewritecond

※ 其实变量也可以放在RewriteRule的rewrite url当中
```
以及判断文件时，很常见搭配这两个match用法：
```
-d：directory. 代表如果有这个文件夹
-f：regular file. 代表如果有这个文件
```
搭配起来写法就像以下范例，代表着如果没有这个文件就转到首页：
```
RewriteCond %{REQUEST_FILENAME} !-f
RewriteRule ^(.*)$ index.html
```
其实还有不同的match写法，可以参考Apache官网的 RewriteCond 。

** RewriteCond 的 Flag 用法 **

```Apache
NC：Non case-sensitive
OR：就是OR条件，下面会说明
```
RewriteCond也有OR跟AND的条件，先前提到RewriteCond后面一定会接一个RewriteRule，有个特性是它只吃接续的第一个rule ，来看下面的范例：
```Apache
### 范例 A
RewriteCond 1
RewriteRule 1
RewriteRule 2
// 等同于
if (Cond1) 
{
    Rule1
}
Rule 2

### 范例 B
RewriteCond 1
RewriteCond 2
RewriteRule 1
// 等同于
if (Cond1 && Cond2) 
{
    Rule1
}

### 范例 C
RewriteCond 1 [OR]
RewriteCond 2
RewriteRule 1
// 等同于
if (Cond1 || Cond2) 
{
    Rule1
}
```
了解RewriteCond跟RewriteRule的用法与每一行执行下来的逻辑，就可以更轻易的改写网络上別人写好的规则。

# 四、如何排错

先前在RewriteRule的段落有稍微简单展示Debug的流程，这里仅使用文字讲解一些小秘诀。

** 1.该使用什么Debug工具 **

使用POSTMAN、cURL等的工具，浏览器除了有Cache外，也会帮忙转址，此时就没办法观察第一次转址的网址內容，譬如：最常见就是浏览器直接显示转址太多次的错误，但使用工具的话，就可以看到Response回传的转址结果。

** 2.如何知道撰写的Regular Expression是否正确？ **

把RewriteRule的Flag加上[R=302]，302 Status Code代表Moved Temporarily，浏览器并不会Cache302的转址结果，但301会，可以确定Rule无误后再拿掉或改为原本301就好，像这样观察转址的结果：
```Apache
RewriteRule ^(.*)$ =$1 [L,R=302]
```

** 3.非得要修改正在运行中的网站怎么办？ **

可以用一些识別的Header，加上RewriteCond来测试撰写的RewriteRule，譬如自定义一个User-agent，每次Reuqest都用这个User-agent即可。

** 4.浏览器有Cache **

前面提到浏览器cache的问题，若认为写的没问题，但访问网站仍是旧有结果的话，就开启私密浏览访问看看，最后仍没办法只好重开Apache看看。

# 五、关于一些小特性

1.RewriteRule在嵌套.htaccess当中不会取得完整URI Path

直接看范例，假设文件夹结構如下：

root/
 ├ a/
   ├ b.html
   └ .htaccess
 ├ c.html
 └ .htaccess


```Apache
### 两个.htaccess都只有这一行內容
RewriteRule ^(.*)$ $1 [L]
```
1.Request：c.html

使用 root/.htaccess
Rule 结果：c.html

2.Request： a/b.html

使用root/a/.htaccess
Rule结果： b.html
没错，发生了不会拿到 a/ 的路径的情况。
※ Apache 会自动选择最接近的 .htaccess 文件（详情会在下一段落说明）

3.如果拿掉 root/a/.htaccess文件，重新 Request： a/b.html

使用 root/.htaccess
Rule 结果：a/b.html
这样的结果又正常了，如果真的想确保拿到完整的 URI Path，可以使用 %{REQUEST_URL}变量来取得 URI ！
```Apache
RewriteCond %{REQUEST_URI} ^(.*)$
RewriteRule ^ %1 [L]
```

2.RewriteRule自动附加在结尾的 PATH_INFO
有时候会希望遇到 Rule1 改写之后，再传递至下一个 Rule2 判断与改写，可是会遇到后面莫名多了先前的 URI，来看一个简单的示范例子：
```Apache
RewriteRule ^(.*)$ web/$1 [NC]
RewriteRule ^(.*)$ sec======$1====== [NC,R=301]
Request： a/b/c.html
```
经过第一个Rule变成：web/a/b/c.html
最后到第二个Rule变成：sec======web/a/b/c.html/b/c.html======
发现它在Rule1的结果末端多了一个不需要的/b/c.html ，这是因为PATH_INFO的缘故，若不需要后面Path 的话，可以在Rule 1加上DPI Flag移除它，由于一些php的CMS或是Framework会使用到PATH_INFO的功能，所以是否关掉PATH_INFO的作用还是要注意一下！

3.重新寻找.htaccess文件
有时候要的Rule很纯，就只是将所有Request都转到web/文件夹下：
```Apache
RewriteBase /
RewriteRule ^(.*)$ web/$1 [QSA,L]
```
但在Apache 2.4后就会遇到出现Redirect Loop的错误，原因是第一次Request URI被改写后成web/xxxx，它会根据新URI找对应最近的.htaccess并再重跑一次RewriteRule

此时有两种解法，第一种是在 web/ 文件夹下放一个空的 .htaccess ，第二种是可以参考下一点停止 Redirect Loop 的写法，放在 RewriteRule 前面。

4.停止 Redirect Loop 的情况

有时候会遇到无穷Loop 的问题：

Request exceeded the limit of 10 internal redirects due to probable configuration error. Use 'LimitInternalRecursion' to increase the limit if necessary. Use 'LogLevel debug' to get a backtrace.
可以在所有 RewriteRule 之前加上判断，若 Redirect Status 是 200 的话，就停止 Loop：

RewriteCond %{ENV:REDIRECT_STATUS} 200
RewriteRule ^ - [L]
不过建议还是检查 RewriteRule 哪边有写错的，毕竟这解不太算是萬靈藥。

Ref：https://stackoverflow.com/a/20930010

# 六、使用Htaccess的缺点

使用Override很方便，只要将.htaccess放到文件夹下面就有效果，但官方其实不推荐开启Override的功能，它会降低效能，主要是有以下缺点：

1.每次Request的嵌套搜寻
每一次Request都会使得Apache透过嵌套递归的方式搜寻.htaccess文件，导致Apache缓慢，譬如送出这个 Request：

Request： /example/sub/index.html

Apache 则会根据路径寻找以下 .htaccess文件

/var/www/.htaccess
/var/www/example/.htaccess
/var/www/example/sub/.htaccess

最后 Apache 选择离文件最近的 /var/www/example/sub/.htaccess

2.重复Compile RewirteRule

由于每次 Request 都会嵌套搜寻 .htaccess，所以再遇到 RewriteRule 都会重新 Compile 一次，不像 Site Config 只会 Compile 一次后做 cache，所以 RewriteRule 非常多的话，也会导致 apache 缓慢。

笔者使用Apache Benchmark做了一个小型的压力测试，连续访问10000次，同时 1000个连线，取其三次执行ab指令的平均值，分别比较是否有打开Override功能，和Rewrite数量多寡是否有影响，Rewrite的状况都是以跑到倒数第三行结束为主，且每行RewriteRule跟RewriteCond不重复。

{% asset_img 1.png %}
<center>图1. 针对 orverride 功能开关和 Rule 数量的压力测试比较图</center>

如图上的Total平均值，有开启Override功能还是比没开启花的时间多了一些，而行数的多寡也会影响到花費的时间，但筆者还是有点疑惑，差0.1秒好像也没什么关系吶。

官方认为.htaccess主要还是给无法编辑Site Config的情况下使用，像是一台主机上有多站共用，但真的想用.htaccess效果又不想开启Override的话怎么办呢？也是有一个取得中间值的做法。

在Site Config中预先指定.htaccess文件路径，
把Override功能关掉，并使用Include指定引入的.htaccess，如下写法：
```Apache
DocumentRoot /var/www/
<Directory /var/www/>
 AllowOverride None
 Include /var/www/.htaccess
 …
</Directory>
```

但这方法也是有个小缺点：每次更新.htaccess都必须重新启动Apache重新读取设定。

如果主机流量不大的话，效能问题并没这么严重，最后还是得依据主机上的网站与情况，衡量哪种做法比较好哦！

Reference

Apache.org [When (not) to use .htaccess files](https://httpd.apache.org/docs/current/howto/htaccess.html#when)