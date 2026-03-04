---
title: 通过ingress-nginx转发后的服务获取访问客户端的真实IP地址
date:  2021-09-26 12:40:00
tags: 
  - kubernetes
  - nginx
categories: 
  - 后端
  - 容器
---

nginx获取客户端真实IP的配置一般用

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

<!-- more -->

这个配置项（如果要获取每一级的完整链路，每一级代理都得加这个配置项）。详细参考：https://www.cnblogs.com/diaosir/p/6890825.html

本地调试服务可以正确拿到客户端真实IP，但是通过k8s的ingress转发之后，拿到的却只有ingress网关的IP。

![image-20210126084610905](/images/image-20210126084610905.png)

可以先进容器里面查看ingress-nginx的配置：

![image-20210126083439015](/images/image-20210126083439015.png)

一般x-forwarded-for头都是$remote_addr，这只能获取到ingress代理的IP。

服务里面拿到的IP为：

[10.244.0.0, 10.244.2.2]

![img](/images/image-20210126084610906.png)

因为服务里面取的是x-forwarded-for这个头，所以要将ingress-nginx的x-forwarded-for修改为$proxy_add_x_forwarded_for

编辑ingress的configmap：

sudo kubectl -n ingress-nginx edit cm nginx-configuration

data根元素下加入：

```yaml
data:
  compute-full-forwarded-for: "true"
  enable-underscores-in-headers: "true"
  use-forwarded-headers: "true"
  use-proxy-protocol: "true"
```

保存会立即生效。再次查看ingress容器中的nginx配置文件，发现头已变成：

![image-20210126084058660](/images/image-20210126084058660.png)

服务打印出的日志已变为：

> 2021/01/26 08:30:27 interceptor.go:63: x-forwarded-for value is [101.95.178.162, 127.0.0.1, 10.244.0.0, 10.244.2.2]

最前面一个就是客户端的ip，如果没有的话，说明客户端到第一级代理的地方没有配置X-Forwarded-For这个头。



grpc取x-forwarded-for的方法:

```
md, ok := metadata.FromIncomingContext(ctx)
if !ok {
	return
}
ips, ok := md["x-forwarded-for"]
```

