---
title: nginx的location匹配规则
tags:
 - nginx
 - location
abstract: 本文整理总结nginx的location语法及匹配规则，location中的proxy_pass转发规则，以及location中的root与alias的区别。location是nginx配置中常用的指令，其用法灵活，本文可能挂一漏万，希望大家批评指正。
posturl: nginx-location
date: 2018-09-23 21:26:22
---
### location的语法规则
[官网](http://nginx.org/en/docs/http/ngx_http_core_module.html#location)所示location语法规则如下：
```bash
Syntax:	 location [ = | ~ | ~* | ^~ ] uri { ... }
         location @name { ... }
Default: —
Context: server, location
```

第一种语法的location为无名location（ps:我自己起的名字，便于区分，以下location如没有特别说明，都是这种无名location），uri参数为必须的，` = ， ~ ， ~* ， ^~`这几个符号可选。

第二种语法的location称为命名location，用于server内部重定向，不能直接用来处理正常的请求,所以也没有uri的参数。本文不对其展开。

语法规则可见，location没有默认值，上下文为server和location，所以location也可以嵌套在location内部。但是不能随便嵌套，有两个规则：
1. 内部uri不可能匹配到时，不能嵌套。比如`/ab` 不能嵌套在 `/b`内, 因为当 `/b`匹配后，`/ab`显然不可能再匹配。
2. 命名location内不能嵌套其他location，并且也不能嵌套在其他location中，命名location只能在server内。

### location匹配符号的含义
` = ， ^~, ~ ， ~*`四个匹配符号的基本优先级按下面列表逐渐降低，=优先级最高，基本含义如下：
```
= 表示绝对匹配。只有请求的url路径与后面的字符串完全相等时，才会命中。
^~ 表示如果该符号后面的字符是最佳匹配，采用该规则，不再进行后续的查找。
~ 表示该规则是使用正则定义的，区分大小写。
~* 表示该规则是使用正则定义的，不区分大小写。
四个符号都不用，也就是省略匹配符的，只要包含uri参数中的字符即可匹配成功。所以`location / {}`可以匹配到任何其他location不匹配的url请求。不使用匹配符的location优先级最低，
```

### 常见配置
这里列出常见的共存配置规则，如下。为了方便测试，所有location在nginx中直接返回文本内容，可以直接拷贝以下配置去测试。
```
	location = / {
	   default_type text/html;
	   return 200 'configuration A';
	}
	location = /login {
	   default_type text/html;
	   return 200 'configuration B';
	}
	location ^~ /static/ {
	   default_type text/html;
	   return 200 'configuration C';
	}
	location ~* \.jpg$ {
	   default_type text/html;
	   return 200 'configuration E of jpg';
	}
	location ~ \.(gif|jpg|png|js|css)$ {
	   default_type text/html;
	   return 200 'configuration D';
	}
	location ~* \.png$ {
	   default_type text/html;
	   return 200 'configuration E';
	}
	location ^~ /s1 {
	   default_type text/html;
	   return 200 'configuration F';
	}
	location ^~ /s1long {
	   default_type text/html;
	   return 200 'configuration G';
	}
	location / {
	   default_type text/html;
	   return 200 'configuration H';
	}
```
基于以上配置，实验结果如下
**提示：本文实验使用nginx版本为1.12.2，运行在centos系统下**

| 测试url                   | 返回值（即匹配规则） | 说明                                                                                                |
| --------------------------- | ---------------------- | ----------------------------------------------------------------------------------------------------- |
| http://domain/              | configuration A        |                                                                                                       |
| http://domain/test          | configuration H        |                                                                                                       |
| http://domain/login         | configuration B        | B优先于H                                                                                           |
| http://domain/static/login/ | configuration C        | B为精确匹配，此url不符合B                                                                   |
| http://domain/login/        | configuration H        | 同上                                                                                                |
| http://domain/static/p.png  | configuration C        | ^~最佳匹配优先于正则匹配，故C优先于D                                                  |
| http://domain/test/p.png    | configuration D        | D和E均匹配，并且都是正则匹配，写在上方的优先，D生效                                                      |
| http://domain/test/p.jpg    | configuration E of jpg | 同上 |
| http://domain/test/p.pNG    | configuration E        | D区分大小写，不匹配pNG，所以E生效                                                       |
| http://domain/s1long        | configuration G        | F、G同是最佳匹配，G可以匹配到的内容较长，G优先                                   |                                              |

### uri后的斜杆
先祭出官网上的一段话
> If a location is defined by a prefix string that ends with the slash character, and requests are processed by one of proxy_pass, fastcgi_pass, uwsgi_pass, scgi_pass, memcached_pass, or grpc_pass, then the special processing is performed. In response to a request with URI equal to this string, but without the trailing slash, a permanent redirect with the code 301 will be returned to the requested URI with the slash appended. If this is not desired, an exact match of the URI and location could be defined like this:
> location /user/ {
	proxy_pass http://user.example.com;
}
location = /user {
	proxy_pass http://login.example.com;
}

就是说如果有一个uri是以斜杆开始以斜杆结束的，并且内部有转发请求的，如果访问的url除了末尾的斜杆不匹配以外，其他都匹配，nginx会返回永久重定向。如果这不是想要的结果，可以加上一个精确匹配的location。

### proxy_pass的转发规则
proxy_pass用在location中转发请求给后端服务，proxy_pass的转发url有两种，nginx对它们的处理方法不同。
1. 一种是不带路径部分的。`proxy_pass http://127.0.0.1;`，就是只有后端服务地址，没有路径的，即没有出现`/`。这种情况下，nginx会把客户端请求的url中的所有路径拼接在这个地址后面，然后发请求。
2. 另一种是带有路径部分。`proxy_pass http://127.0.0.1/remote/;`。这种情况，nginx先把location中匹配的部分路径去掉，剩下的拼接，然后发起请求。特别注意，`proxy_pass http://127.0.0.1/;`这个也属于这一类。

### alias与root在location中的区别
在配置location的静态文件目录时，用到alias与root，两者处理规则略有不同。这里用两个简单例子说明。
#### alias
```
location ^~ /test/ {
 	alias /www/root/html/my_test/;
}
```

这种情况，如果一个请求的URI是/test/one.html时，web服务器将会返回服务器上的/www/root/html/my_test/one.html的文件。因为alias是把location后面配置的路径（也就是匹配部分）丢弃掉。

#### root
```
location ^~ /test/ {
	root /www/root/html/my_test/;
}
```
这里就不一样了。如果一个请求的URI是/test/one.html时，web服务器将会返回服务器上的/www/root/html/my_test/test/one.html的文件。root是在指定的目录下，寻找请求路径文件，不会去除匹配到的部分。

### 总结

location的匹配优先级如下：
1. 精确匹配 > 最佳匹配(^~) > 正则匹配
2. 多字符匹配 > 少字符匹配
3. 配置文件中写在上方的规则优先于写在下方的规则

location的配置乃至nginx的配置都很灵活，很强大。本文只说了一些基本的，遇到具体问题，不在本文所说的规则中时，可以用搜索引擎搜索，也可以自己动手配置实验验证。