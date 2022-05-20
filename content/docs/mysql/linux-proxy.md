---
title: "Linux端口转发方式"
date: 2022-05-18 23:52:08
tags: ["Linux"]
draft: false
---

# Linux端口转发方式

## ssh端口转发模式

> 这种方式，有以下特点：
> 
> 1. 可以在本地执行，也可以在服务器执行，部署方案灵活；
> 2. 通信会经过ssh加密，安全性好；
> 3. 同样是因为经过ssh加密，所以性能比较低

```shell
ssh -L ${local_port}:${server_ip}:${server_port} -fN root@${proxy_ip}
```

## *xinetd* 端口转发

> 1. 没有加密，性能高；
> 2. 在应用层转发，不需要root权限

- 安装xinetd：`yum install -y xinetd`

- 增加service配置：
  
  ```
  [root@1111 /etc/xinetd.d]# cat proxy-mysql
  service proxy-mysql
  {
     disable = no
     type = UNLISTED
     socket_type = stream
     protocol = tcp
     wait = no
     redirect = ${server_ip} ${server_port}
     bind = 0.0.0.0
     port = 3306
     user = nobody
  }
  ```

- 启动转发：`xinetd -f /etc/xinetd.d/proxy-mysql`

## Iptables方式

> 1. 在内核层来进行转发，性能高；
> 2. 需要root权限。

```shell
iptables -P FORWARD ACCEPT
iptables -t nat -A PREROUTING -p tcp -m tcp --dport ${local_port} -j DNAT --to-destination ${server_ip}:${server_port}
iptables -t nat -A POSTROUTING -p tcp -m tcp --dst ${server_ip} --dport ${server_port} -j MASQUERADE
```
