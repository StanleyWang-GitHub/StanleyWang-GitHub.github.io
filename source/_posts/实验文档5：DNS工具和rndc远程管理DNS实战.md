title: 实验文档5：DNS工具和rndc远程管理DNS实战
author: Stanley Wang
categories: Web DNS技术
date: 2018-12-16 12:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# DNS管理工具
## 安装
```
# yum install bind-utils -y
```
## 工具一：nslookup
Windows操作系统也有的一个常用工具
### 交互式
```
#nslookup 
> server localhost
Default server: localhost
Address: 127.0.0.1#53
Default server: localhost
Address: 127.0.0.1#53
> www.bkjf-inc.com
Server:		localhost
Address:	127.0.0.1#53

Name:	www.bkjf-inc.com
Address: 192.144.198.128
-------以上是权威应答
> server 8.8.8.8
Default server: 8.8.8.8
Address: 8.8.8.8#53
> www.bkjf-inc.com
Server:		8.8.8.8
Address:	8.8.8.8#53

Non-authoritative answer:
Name:	www.bkjf-inc.com
Address: 192.144.198.128
-------以上是非权威应答
```
### 非交互式
```
#nslookup www.baidu.com
Server:		183.60.83.19
Address:	183.60.83.19#53

Non-authoritative answer:
www.baidu.com	canonical name = www.a.shifen.com.
Name:	www.a.shifen.com
Address: 220.181.112.244
Name:	www.a.shifen.com
Address: 220.181.111.37
```

## 工具二：host
简单粗暴的小工具
```
#host -t A www.baidu.com
www.baidu.com is an alias for www.a.shifen.com.
www.a.shifen.com has address 220.181.112.244
www.a.shifen.com has address 220.181.111.37
```

## 工具三：dig
功能强大的DNS工具，重点掌握
```
#dig -t A www.baidu.com @localhost

; <<>> DiG 9.9.4-RedHat-9.9.4-73.el7_6 <<>> -t A www.baidu.com @localhost
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46476
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 5, ADDITIONAL: 6

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.baidu.com.			IN	A

;; ANSWER SECTION:
www.baidu.com.		1200	IN	CNAME	www.a.shifen.com.
www.a.shifen.com.	300	IN	A	220.181.111.37
www.a.shifen.com.	300	IN	A	220.181.112.244

;; AUTHORITY SECTION:
a.shifen.com.		1200	IN	NS	ns3.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns2.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns4.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns5.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns1.a.shifen.com.

;; ADDITIONAL SECTION:
ns5.a.shifen.com.	1200	IN	A	180.76.76.95
ns1.a.shifen.com.	1200	IN	A	61.135.165.224
ns3.a.shifen.com.	1200	IN	A	112.80.255.253
ns4.a.shifen.com.	1200	IN	A	14.215.177.229
ns2.a.shifen.com.	1200	IN	A	220.181.57.142

;; Query time: 561 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Fri Mar 01 09:45:51 CST 2019
;; MSG SIZE  rcvd: 271
```
常用参数：
- +[no]addition
- +short

## 工具四：nsupdate
不常用，需要在zone配置文件里声明`allow-update { acl; };`
### 调整zone配置文件
```vi /etc/named.rfc1912.zones
zone "bkjf-inc.com" IN {
        type master;
        file "bkjf-inc.com.zone";
        allow-update { 10.4.7.11/32; };
};
```
重启named服务
```
# systemctl restart named
```
### 新增一条记录
```
#nsupdate 
> server 10.4.7.11
> update add update.bkjf-inc.com 60 A 10.4.7.11
> send
> quit
```
检查：
```
#nslookup update.bkjf-inc.com
Server:		10.4.7.11
Address:	10.4.7.11#53

Name:	update.bkjf-inc.com
Address: 10.4.7.11
```
### 查看区域数据库文件
```ll /var/named/
-rw-r--r-- 1 root  root   335 Feb 28 10:57 bkjf-inc.com.zone
-rw-r--r-- 1 named named  733 Mar  1 09:54 bkjf-inc.com.zone.jnl
#file bkjf-inc.com.zone.jnl 
bkjf-inc.com.zone.jnl: data
```
产生了一个jnl的数据文件，不能使用文本编辑器打开
> jnl文件（journal文件）是BIND9动态更新的时候记录更新内容所生成的日志文件。

### 删除一条记录
```
#nsupdate 
> server 10.4.7.11
> update delete update.bkjf-inc.com
> send
> quit
```
检查
```
#nslookup update.bkjf-inc.com
Server:		10.4.7.11
Address:	10.4.7.11#53

** server can't find update.bkjf-inc.com: NXDOMAIN
```

### 更新一条记录
不支持直接更新，需要先执行删除，再新增

### nsupdate使用小结：
- 优点
  - 命令简单，便于记忆
  - 不用手动变更SOA的serial序列号，自动滚动
  - 不需要重启/重载BIND9服务/配置，生效快
  - 可以通过配置acl实现远程管理
- 缺点
  - jnl文件无法使用文本文件的方式打开
  - 只能依赖完全区域传送查看所有区域的记录
  - 更新操作复杂，先删再增
  - 远程管理有安全隐患，需要加强审计
  - 动态域在rndc管理上多一步

# rndc远程管理DNS
## 生成rndc-key
```
#rndc-confgen -r /dev/urandom
# Start of rndc.conf
key "rndc-key" {
	algorithm hmac-md5;
	secret "MFM4AocpN0lcoL4fN2lA6Q==";
};

options {
	default-key "rndc-key";
	default-server 127.0.0.1;
	default-port 953;
};
# End of rndc.conf

# Use with the following in named.conf, adjusting the allow list as needed:
# key "rndc-key" {
# 	algorithm hmac-md5;
# 	secret "MFM4AocpN0lcoL4fN2lA6Q==";
# };
# 
# controls {
# 	inet 127.0.0.1 port 953
# 		allow { 127.0.0.1; } keys { "rndc-key"; };
# };
# End of named.conf
```

## 把rndc-key和controls配置到bind的主配置文件的options段里
```vi /etc/named.conf
key "rndc-key" {
	algorithm hmac-md5;
	secret "MFM4AocpN0lcoL4fN2lA6Q==";
};

controls {
      inet 10.4.7.11 port 953
               allow { 10.4.7.11;10.4.7.12; } keys { "rndc-key"; };
};
```
**注意：**这里要配置一下controls段的acl，限定好哪些主机可以使用rndc管理DNS服务

## 重启bind9服务
```
# systemctl restart named
```
rndc的服务端监听在953端口，检查一下端口是否起来
```
# netstat -luntp|grep 953
tcp        0      0 10.4.7.11:953           0.0.0.0:*               LISTEN      11136/named 
```

## 在远程管理主机上安装bind
rndc命令在bind包里，所以远程管理主机需要安装bind（不需要启动named）

## 在远程管理主机上做rndc.conf
使用rndc进行远程管理的主机上，都需要配置rndc.conf，且rndc-key要和DNS服务器上的key一致
```vi /etc/rndc.conf
key "rndc-key" {
	algorithm hmac-md5;
	secret "MFM4AocpN0lcoL4fN2lA6Q==";
};

options {
	default-key "rndc-key";   
	default-server 10.4.7.11;
	default-port 953;   
}; 
```

## 使用rndc命令远程管理DNS
### 查询DNS服务状态（可以取值做监控）
```
#rndc status 
version: 9.9.4-RedHat-9.9.4-73.el7_6 <id:8f9657aa>
CPUs found: 2
worker threads: 2
UDP listeners per interface: 2
number of zones: 105
debug level: 0
xfers running: 0
xfers deferred: 0
soa queries in progress: 0
query logging is OFF
recursive clients: 0/0/1000
tcp clients: 0/100
server is up and running
```
### 管理静态域(allow-update { none; };)
```vi 静态域zone文件
zone "od.com" IN {
	type master;
	file "od.com.zone";
	allow-update { none; };
};
```
增、删、改一条记录后
```
# rndc reload od.com
zone reload up-to-date
```

### 管理动态域（allow-update { 10.4.7.11; };）
```vi 动态域zone文件
zone "host.com" IN {
	type master;
	file "host.com.zone";
	allow-update { 10.4.7.11; };
};
```
增、删、改一条记录后
```
#rndc reload host.com
rndc: 'reload' failed: dynamic zone
```
直接reload会报错，需要先`freeze`再`thaw`才行
```
#rndc freeze host.com
#rndc thaw host.com
The zone reload and thaw was successful.
```