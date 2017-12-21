# nginx实现负载均衡

Nginx 的负载均衡功能，其实实际上和 nginx 的代理是同一个功能，只是把代理一台机器改为多台机器而已。

## 配置文件

```
upstream backend {
	server 192.168.0.18;
	server 192.168.0.28;
	server 192.168.0.38;
}

server {
	listen 80;
	server_name 192.168.0.08;
	location / {
		proxy_pass http://backend;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}
}
```

通过上述配置，Nginx会作为HTTP反向代理，把访问本机的HTTP请求，均分到后端集群的3台服务器Server1、Server2、Server3。

![](/assets/nginx-request.png)

nginx的upstream目前支持4种方式的分配

* 轮询（默认）每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。

* weight 指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。

* iphash 每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题。

* fair 按后端服务器的响应时间来分配请求，响应时间短的优先分配。