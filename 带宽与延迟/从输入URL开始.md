# 从输入URL开始
这是一个古老的问题，即我们输入URL后按下回车到网页测呈现都发生了什么？

## 预处理URL
URL格式：
![http://img.voidcn.com/vcimg/000/005/096/442_7e4_fe7.png](http://img.voidcn.com/vcimg/000/005/096/442_7e4_fe7.png)

以http为例：
```
http://www.example.com:80/path/to/myfile.html?key1=value1&key2=value2#SomewhereInTheDocument
```

有时候我们并没有输入完整的url，浏览器会智能地进行自动补全，如浏览器会对不同协议自动添加端口，如http的80端口，https的443端口等等。除此之外浏览器还会根据我们的历史记录，书签等信息进行智能提示。如果不合法，将会转为搜索关键字。

## DNS解析
浏览器在得到合法的URL后，就要进行域名解析：
* 从浏览器自身DNS缓存中查找结果，上一章中我们提到了DNS预解析优化策略就是提前进行DNS解析放入缓存(要查看 Chrome 当中的缓存， 打开 `chrome://net-internals/#dns`)
* 在操作系统DNS缓存中查询，如linux上的`NSCD`缓存服务
* 读取hosts文件，如*nix上的`/etc/hosts`
* 查询配置的DNS服务器查询
* **递归查询过程:** 本地DNS服务器查询自己的缓存记录，如果有则返回，如果没有则去查询根DNS服务器。
* **迭代查询过程：** 根DNS服务器告诉本地DNS服务器顶级域的DNS地址，本地DNS再查询通过顶级域DNS找到下一级域的DNS地址，以此迭代查询直到找到域名和IP的对应信息
* 找到关系后，本地DNS返回结果给客户机，并进行相应的缓存

![](/assets/dns_lookup.jpg)

图片来源：51cto