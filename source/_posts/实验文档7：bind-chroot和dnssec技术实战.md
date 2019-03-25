title: 实验文档7：bind-chroot和dnssec技术实战
author: Stanley Wang
categories: Web DNS技术
date: 2018-12-16 8:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# 安装部署bind-chroot
## 系统环境
服务器：腾讯云主机，有公网IP
OS：CentOS Linux release 7.4.1708 (Core) 
bind-chroot：bind-chroot-9.9.4-73.el7_6.x86_64

## yum 安装
```
# yum install bind-chroot -y
=============================================================================================================================================================

 Package                                 Arch                            Version                                      Repository                        Size
=============================================================================================================================================================

Installing:
 bind-chroot                             x86_64                          32:9.9.4-73.el7_6                            updates                           88 k
Installing for dependencies:
 bind                                    x86_64                          32:9.9.4-73.el7_6                            updates                          1.8 M
Updating for dependencies:
 bind-libs                               x86_64                          32:9.9.4-73.el7_6                            updates                          1.0 M
 bind-libs-lite                          x86_64                          32:9.9.4-73.el7_6                            updates                          741 k
 bind-license                            noarch                          32:9.9.4-73.el7_6                            updates                           87 k
 bind-utils                              x86_64                          32:9.9.4-73.el7_6                            updates                          206 k

Transaction Summary
=============================================================================================================================================================

Install  1 Package  (+1 Dependent package)
Upgrade             ( 4 Dependent packages)

Installed:
  bind-chroot.x86_64 32:9.9.4-73.el7_6                                                                                                                       


Dependency Installed:
  bind.x86_64 32:9.9.4-73.el7_6                                                                                                                              


Dependency Updated:
  bind-libs.x86_64 32:9.9.4-73.el7_6  bind-libs-lite.x86_64 32:9.9.4-73.el7_6  bind-license.noarch 32:9.9.4-73.el7_6  bind-utils.x86_64 32:9.9.4-73.el7_6 

Complete!


```

## 配置bind-chroot
bind-chroot本质上是使用chroot方式给bind软件换了个“根”，这时bind软件的“根”在/var/named/chroot下，弄懂这一点，配置起来就跟BIND9没什么区别了
把yum安装的bind-chroot在/etc下的产生的配置文件硬链接到/var/named/chroot/etc下
```cd /var/named/chroot/etc/
[root@VM_0_13_centos ~]# cd /var/named/chroot/etc/
[root@VM_0_13_centos etc]# ls /etc/named
named/               named.conf           named.iscdlv.key     named.rfc1912.zones  named.root.key       
[root@VM_0_13_centos etc]# ln /etc/named.* .
```
```cd /var/named/chroot/var/named
[root@VM_0_13_centos named]# ln /var/named/named.* .
[root@VM_0_13_centos named]# mkdir data/ dynamic/ slaves/ dnssec-key/
[root@VM_0_13_centos named]# chgrp -R named *
[root@VM_0_13_centos named]# ll
drwxrwx--- 2 root  named 4096 Feb 27 18:30 data
drwxr-xr-x 3 root  named 4096 Feb 28 14:31 dnssec-key
drwxrwx--- 2 root  named 4096 Feb 28 14:33 dynamic
-rw-r----- 2 root  named 2281 May 22  2017 named.ca
-rw-r----- 2 root  named  152 Dec 15  2009 named.empty
-rw-r----- 2 root  named  152 Jun 21  2007 named.localhost
-rw-r----- 2 root  named  168 Dec 15  2009 named.loopback
drwxrwx--- 2 root  named 4096 Jan 30 01:23 slaves
```
## /etc/named.conf主配置文件
编辑主配置文件，这里把53端口开放到公网
```vi /etc/named.conf
options {
	listen-on port 53 { any; };
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";
	recursing-file  "/var/named/data/named.recursing";
	secroots-file   "/var/named/data/named.secroots";
	allow-query     { any; };

	/* 
	 - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
	 - If you are building a RECURSIVE (caching) DNS server, you need to enable 
	   recursion. 
	 - If your recursive DNS server has a public IP address, you MUST enable access 
	   control to limit queries to your legitimate users. Failing to do so will
	   cause your server to become part of large scale DNS amplification 
	   attacks. Implementing BCP38 within your network would greatly
	   reduce such attack surface 
	*/
	recursion no;

	dnssec-enable yes;
	dnssec-validation yes;
	dnssec-lookaside auto;

	/* Path to ISC DLV key */
	bindkeys-file "/etc/named.iscdlv.key";

	managed-keys-directory "/var/named/dynamic";
	
	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
	type hint;
	file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

# 使用dnssec技术维护一个业务域
在公网上使用BIND9维护的业务域，最好使用dnssec技术对该域添加数字签名
DNSSEC（DNS Security Extension）----DNS安全扩展，主要是为了解决DNS欺骗和缓存污染问题而设计的一种安全机制。

[DNSSEC技术参考文献1](https://www.cloudxns.net/Support/detail/id/1309.html)
[DNSSEC技术参考文献2](https://blog.csdn.net/zhu_tianwei/article/details/45082015)

## 打开dnssec支持选项
```vi /etc/named.conf
dnssec-enable yes;
dnssec-validation yes;
dnssec-lookaside auto;
```
## 配置一个业务域bkjf-inc.com
```vi /etc/named.rfc1912.zones
zone "bkjf-inc.com" IN {
        type master;
        file "bkjf-inc.com.zone";
        key-directory "dnssec-key/bkjf-inc.com";
        inline-signing yes;
        auto-dnssec maintain;
        allow-update { none; };
};
```
## 创建数字签名证书
```vi /var/named/chroot/var/named/dnssec-key
[root@VM_0_13_centos dnssec-key]# mkdir bkjf-inc.com
[root@VM_0_13_centos dnssec-key]# chgrp named bkjf-inc.com
[root@VM_0_13_centos dnssec-key]# cd bkjf-inc.com
[root@VM_0_13_centos bkjf-inc.com]# dnssec-keygen -a RSASHA256 -b 1024 bkjf-inc.com
Generating key pair..................................++++++ .++++++ 
Kbkjf-inc.com.+008+53901
[root@VM_0_13_centos bkjf-inc.com]# dnssec-keygen -a RSASHA256 -b 2048 -f KSK bkjf-inc.com                                                                           KSK bkjf-inc.com
Generating key pair..........................................................................................+++ ............................................
.....+++ 
Kbkjf-inc.com.+008+40759

[root@VM_0_13_centos bkjf-inc.com]# chgrp named *
[root@VM_0_13_centos bkjf-inc.com]# chmod g+r *.private
[root@VM_0_13_centos bkjf-inc.com]# ll
total 16
-rw-r--r-- 1 root named  607 Feb 28 14:10 Kbkjf-inc.com.+008+40759.key
-rw-r----- 1 root named 1776 Feb 28 14:10 Kbkjf-inc.com.+008+40759.private
-rw-r--r-- 1 root named  433 Feb 28 14:10 Kbkjf-inc.com.+008+53901.key
-rw-r----- 1 root named 1012 Feb 28 14:10 Kbkjf-inc.com.+008+53901.private
```
这里如果生成密钥的速度很慢，需要yum安装一下haveged软件并开启
```
# systemctl start haveged.service
```

## 创建区域数据库文件
```vi /var/named/chroot/var/named/bkjf-inc.com.zone
[root@VM_0_13_centos named]# cat bkjf-inc.com.zone
$TTL 600	; 10 minutes
@               IN SOA	ns1.bkjf-inc.com. 87527941.qq.com. (
                2018121605 ; serial
                10800      ; refresh (3 hours)
                900        ; retry (15 minutes)
                604800     ; expire (1 week)
                86400      ; minimum (1 day)
                )
                NS     ns1.bkjf-inc.com.
                NS     ns2.bkjf-inc.com.
$ORIGIN bkjf-inc.com.
$TTL 60	; 1 minute
ns1             A     192.144.198.128
ns2             A     192.144.198.128
www             A     192.144.198.128
eshop           CNAME www
```

## 启动bind-chroot服务
```
# systemctl start named-chroot
```

## 自动生成了签名zone
如果启动成功且配置无误，应该自动生成了带签名的zone
```cd /var/named/chroot/var/named/
[root@VM_0_13_centos named]# ll
total 60
-rw-r--r-- 1 root  named  507 Feb 28 14:34 bkjf-inc.com.zone
-rw-r--r-- 1 named named  512 Feb 28 14:26 bkjf-inc.com.zone.jbk
-rw-r--r-- 1 named named  742 Feb 28 14:35 bkjf-inc.com.zone.jnl
-rw-r--r-- 1 named named 4102 Feb 28 14:44 bkjf-inc.com.zone.signed
-rw-r--r-- 1 named named 7481 Feb 28 14:35 bkjf-inc.com.zone.signed.jnl
```
检查签名区需要用到完全区域传送命令
```
[root@VM_0_13_centos named]# dig -t AXFR bkjf-inc.com @localhost

; <<>> DiG 9.9.4-RedHat-9.9.4-73.el7_6 <<>> -t AXFR bkjf-inc.com @localhost
;; global options: +cmd
bkjf-inc.com.		600	IN	SOA	ns1.bkjf-inc.com. 87527941.qq.com. 2018121608 10800 900 604800 86400
bkjf-inc.com.		86400	IN	RRSIG	NSEC 8 2 86400 20190330063503 20190228053503 53901 bkjf-inc.com. 0fyLJXxaDOI+RWnYjK2tGpd6WgbWmgeIADtjpPQFQLrv1X9fuDLi2MFR q0+csg5P22eVUdasKi3q5tMmFW8GZtLEBBVtOtSba3/FvtoitvyBGcG6 KJ155dPbhEFe/eR0/JhWtFsIsyj/UHtgELB4eGYJYCeEI+WzUopT7voz 4UE=
bkjf-inc.com.		86400	IN	NSEC	eshop.bkjf-inc.com. NS SOA RRSIG NSEC DNSKEY TYPE65534
bkjf-inc.com.		600	IN	RRSIG	NS 8 2 600 20190330063017 20190228053309 53901 bkjf-inc.com. Y/T0m4p0yNrJwJiHc0mjDgit/9E4h7MXPb5F2WgBd+huXYgL0pS0vOb3 c2aRvHHW/zngPjShOfy3sYY5203SzPS15tN6E/RAs36/I33sZE7jZBFo 9q0KjEdKHNsoC9XISSdbLPCX879/B1rKZcmhpPNmhpAK6P351nWWgd9L jtU=
bkjf-inc.com.		600	IN	RRSIG	SOA 8 2 600 20190330063503 20190228053503 53901 bkjf-inc.com. eE3nKlCmAZrjJ3DwdzPStYmrC38X6VCqCxIc6otLJDX65Uk2uSqGSPre WIu16zEsbuuxq7/38ABrupQNwkPAgaSaiLIRC/000PXzKsUPhll0xO4x u9tLg2LBRATQ+4dHpKtLsoBTX0nXVHlz09YeAAA82r5wyQye2/ebesxH +A4=
bkjf-inc.com.		0	IN	RRSIG	TYPE65534 8 2 0 20190330054441 20190228053309 53901 bkjf-inc.com. sEX7jpdTbUZ3hlIR2CRWHbgceAQFVOVKnVl6CXvyQhavIFjUyBMMhXTw hKYwXd2Hc0LGg9koWJqlt0oYS8YbXacKbeBUrLovmcbYP46Uhm05zaVo jswG7oYYsYDE3ekbl5ImnAEyjksSNOgk8if/WoUvXfF5QH6Rdl+6Q3qG cEI=
bkjf-inc.com.		600	IN	RRSIG	DNSKEY 8 2 600 20190330063309 20190228053309 53901 bkjf-inc.com. rUGjMTxmbthB6UbmemoorQOfuen8u0xeOosl7lPRNLV2Hk7KsAZzUD2/ tRAJaY9NRZ1JhZHkmX/N5hncuVpPxZnrp8UB7qOoairqgjA73IFGoT0F 00KIU0FZaqsQAbBSzpzfbwr9KVbn1hTAq6/5Q/wrWZvQOASMYrF5Xhr9 lW4=
bkjf-inc.com.		600	IN	RRSIG	DNSKEY 8 2 600 20190330063309 20190228053309 40759 bkjf-inc.com. lBXWXbTshdeH/oOkBGdwIspet0ABbhUZfzAXUjOP3ivCMW5sse3ZayEA qPe6mZncURqomWNA/xQKemoJJjtlAwc5F4CjmtrUierdy3EVVKS0NFnz 9L3PxiJcOxl1VVtSBX+XAOPa0xkS3cpEbFVOym4NaKsoLgcqKKBjjBu4 dhWoXoxXk7PE5fogo9/BM0heGI4XpnixUSTbucMw4bcnNYPY0qKUBs2o alt1CvrGz78oOO10//pXpw/ml89UwWo28/FDvxeuXS7soeImDRklTLlE xV/Q3//v7o73ZosAdSR+9xFdcZtVs43Jjo3Cy8WL1Zjz6BdRd59Fyu6h WghEKg==
bkjf-inc.com.		0	IN	TYPE65534 \# 5 08D28D0001
bkjf-inc.com.		0	IN	TYPE65534 \# 5 089F370001
bkjf-inc.com.		600	IN	DNSKEY	256 3 8 AwEAAflXAWLXAVJUEj29iidwVvZALuQr03hLn1bEl81XDtD63H7wwHS9 i9fNDYL0q0FkRDkuzXEQpb3UUleu/RYtSd9w6Ads0RWNUyB6X1E4Djmv sPwFwvo570svZSVky2rjEHnySgVI2ywqhcRYLMKjxE6pXuzXrqecQcF2 qrMq2xmJ
bkjf-inc.com.		600	IN	DNSKEY	257 3 8 AwEAAbxFYlbq+R8y/hGg/xL8xDBasZGYtgPOqVd3bP68p98YHsFwHyG8 u3svatzRoq8STNjKKZEluDC2bcUIn9/mRHyorTYPtwyePxPEgVE4yhBy 9xqD4ES+ty7kuHOUz/WEHdNdYRhYyHe+SGf4dHnmU49pHIBCE8xFX6fs t270webjuXs4Pt6qRlyoFC3XmpRDiMNVwtM+doUxo/MRK4mw5zTeHyyf dFLVOvE3mW/ZKgBfnrsj0zE71bnD5nTxJIjDv1bUppbiRy5RK40jPhHu zaa3quxg1yS/BceYcjJpZJUc3LS55HGzatfuK799KvukuDKf7u71ylW+ 5ynT7Sxhbt0=
bkjf-inc.com.		600	IN	NS	ns1.bkjf-inc.com.
bkjf-inc.com.		600	IN	NS	ns2.bkjf-inc.com.
eshop.bkjf-inc.com.	86400	IN	RRSIG	NSEC 8 3 86400 20190330063503 20190228053503 53901 bkjf-inc.com. dHM2PhYs7BVuhD//iGhcwPZGZmHDkBCfWKju6ZZlvSx3I+QmWWvVdKCj 8YCw2AkWhgARxFfRMzhxRwDjgEgHhxUr4UGPH9+kJpvGi+UpFBVoBvPw iL43qCn/4J2f6URuAY8Dcq0DFpR0QLVJgIXBZpyhUYu5hZNWI2tzfyhO GlM=
eshop.bkjf-inc.com.	86400	IN	NSEC	ns1.bkjf-inc.com. CNAME RRSIG NSEC
eshop.bkjf-inc.com.	60	IN	RRSIG	CNAME 8 3 60 20190330063503 20190228053503 53901 bkjf-inc.com. 9ONt81AjpHFrM8YwDm7pQAg62oDBgaNzdtDIqtBHt5h/BPl83fOP/dOp P0Xi+y/OsFjDzHBSBDU4sy3fJwHBqm8uuMc6m33pIZfTq15fxFXF+2hU ift1bc0b0dk/L7ANZ5haEsDcl+hSVjwru2o2ISJtvp5zySZ61pdMvA6y ktg=
eshop.bkjf-inc.com.	60	IN	CNAME	www.bkjf-inc.com.
ns1.bkjf-inc.com.	60	IN	RRSIG	A 8 3 60 20190330063017 20190228053309 53901 bkjf-inc.com. 9MUZhsTxlmn5B6QXg/iCQoFyilRh8H4OJcTgpu1KgSyMTiBoEwJGdhIx k2XimlJZr9/MrSeRbuLwMZOnwFJ7w9fcIunrYHiE1T71y0BcLnQOKaJf SkJI5VKUam80+J6unkscCj0i/Y1kXTjXWLODKsZzw4+zLz5cGJk6hvsn XP4=
ns1.bkjf-inc.com.	86400	IN	RRSIG	NSEC 8 3 86400 20190330063017 20190228053309 53901 bkjf-inc.com. EFeX2LsEd/flN2/5lCgKlSTtC93WH0LDw9GW1RAlLIfxFAptPsXkmy7y B0Blt7tOuaxA/cTNbnFZBnyo8G3YW90LnYagqeuNzl+90gjUxsbbhE4f pTkQkRXRsvcagYDKQjs9nkN1SAF13SagnupR8D2crHADICjy8RHjHtgA byM=
ns1.bkjf-inc.com.	86400	IN	NSEC	ns2.bkjf-inc.com. A RRSIG NSEC
ns1.bkjf-inc.com.	60	IN	A	192.144.198.128
ns2.bkjf-inc.com.	60	IN	RRSIG	A 8 3 60 20190330063017 20190228053309 53901 bkjf-inc.com. N2ssp0Eh6SyHBYHskedxUpfIp29DETt2g74sCuhrXwMuwLjOdVwuB02i /LqzDLyDbVZnMZncqoQ367AV2b/ttU/FJZcHiAlI2tLRTxVuNyj/E2YN BIDAtIqueNdJzsyE7n1yz9sPcsTrOidrIqqbM3qom5tMQvdo+2jrnhR3 UoY=
ns2.bkjf-inc.com.	86400	IN	RRSIG	NSEC 8 3 86400 20190330063017 20190228053309 53901 bkjf-inc.com. sTTRnUQxPBbeAG0WrQpn4iK/U62D2s8umLwx8w8bx+bwxQdhR8Yyz8Ke tSelkffgctCtyUi5i7ibSTnvUJTcvOcvWWteMOQfQqXJmAngADx87cba /M+OJqRwp8tu3PEniPpTYN3msGSEFILyxLCO/2cyBzK+8jhFFKYyMOn/ ViQ=
ns2.bkjf-inc.com.	86400	IN	NSEC	www.bkjf-inc.com. A RRSIG NSEC
ns2.bkjf-inc.com.	60	IN	A	192.144.198.128
www.bkjf-inc.com.	60	IN	RRSIG	A 8 3 60 20190330063017 20190228053309 53901 bkjf-inc.com. aKI5N4y6eqN/xunC7+4vYa3cSHyXcW533iGA6/q34/ahvq0sTgYN36aF oBO0t8fRvwS3chZaPxwuqbk6hGSW+tRhJ8x/Nnwtbcn004W0ZxI1k046 JW/ePLhq1Cw2GPHXJTsfCjYmAOcwssX2yUv6q9/vocXx/mipuTMljrId yhE=
www.bkjf-inc.com.	86400	IN	RRSIG	NSEC 8 3 86400 20190330063017 20190228053309 53901 bkjf-inc.com. 0q3C+xMKE1p586q+p8U4AHGiNjzzI899TcmL2P4x8x1B7rkc22rsakX9 AnNFAzkPOTVLr81GQtBraI1K6El2QDKcPkE9+0e+34tirpuUzVlzjYB2 f4WHGxTscdOMpCestqnmspQpmXm37+EBWS0alBBq3Db8T+F/3CSEGRS7 Ao0=
www.bkjf-inc.com.	86400	IN	NSEC	bkjf-inc.com. A RRSIG NSEC
www.bkjf-inc.com.	60	IN	A	192.144.198.128
bkjf-inc.com.		600	IN	SOA	ns1.bkjf-inc.com. 87527941.qq.com. 2018121608 10800 900 604800 86400
;; Query time: 1 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Thu Feb 28 15:22:46 CST 2019
;; XFR size: 31 records (messages 1, bytes 3433)
```
这里看到了每个记录都附带了一个`RRSIG`记录，说明已经进行了数字签名

## 检查本地解析
```
[root@VM_0_13_centos named]# dig -t A www.bkjf-inc.com @localhost +dnssec +short
192.144.198.128
A 8 3 60 20190330063017 20190228053309 53901 bkjf-inc.com. aKI5N4y6eqN/xunC7+4vYa3cSHyXcW533iGA6/q34/ahvq0sTgYN36aF oBO0t8fRvwS3chZaPxwuqbk6hGSW+tRhJ8x/Nnwtbcn004W0ZxI1k046 JW/ePLhq1Cw2GPHXJTsfCjYmAOcwssX2yUv6q9/vocXx/mipuTMljrId yhE=
```

## DS记录
在生成证书的目录对`ZSK`执行`dnssec-dsfromkey`命令，得到`bkjf-inc.com`的DS记录，这里我们使用比较长的那个
```cd /var/named/chroot/var/named/dnssec-key/bkjf-inc.com
[root@VM_0_13_centos bkjf-inc.com]#  dnssec-dsfromkey `grep -l zone-signing *key`
bkjf-inc.com. IN DS 53901 8 1 5E13F6C0ECEE84248C2543693CE7D8617920983B
bkjf-inc.com. IN DS 53901 8 2 3006068B784AFBBC67133F123A0C389514959FCB6CAB0032DB200F08E6E5C384
```
其中：
- 53901：关键标签，用于标识域名的DNSSEC记录，一个小于65535的整数值
- 8：生成签名的加密算法，8对应RSA/SHA-256
- 2：构建摘要的加密算法，2对应SHA-256
- 最后一段：摘要值，就是DS记录值

参考万网（阿里云）上关于dnssec配置的文档：[参考文档](https://help.aliyun.com/document_detail/101717.html?spm=a2c4g.11186623.6.626.37c01e9bzbyBGq)

DS记录需要通过运营商提交到上级DNS的信任锚中，这里是通过万网的配置页面，提交到`.com`域

**注意：**要在阿里云上将该域名的dns服务器指向自定义DNS服务器：[参考文档](https://help.aliyun.com/knowledge_detail/39846.html)

## 后续维护
dnssec需要定期轮转，所以需要经常变更签名，其中
- ZSK轮转
> 建议每年轮转
- KSK轮转
> 建议更新ssl证书后尽快轮转？

轮转方法：
- ZSK(zone-signing key)
```cd /var/named/chroot/var/named/dnssec-key/bkjf-inc.com
$ cd /var/named/chroot/var/named/dnssec-key/bkjf-inc.com
$ dnssec-settime -I yyyy0101 -D yyyy0201 Kbkjf-inc.com.+008+53901
$ dnssec-keygen -S Kbkjf-inc.com.+008+53901
$ chgrp bind *
$ chmod g+r *.private
```

- KSK轮转(key-signing key)
```cd /var/named/chroot/var/named/dnssec-key/bkjf-inc.com
$ cd /var/named/chroot/var/named/dnssec-key/bkjf-inc.com
$ dnssec-settime -I yyyy0101 -D yyyy0201 Kbkjf-inc.com.+008+40759
$ dnssec-keygen -S Kbkjf-inc.com.+008+40759
$ chgrp bind *
$ chmod g+r *.private
```
**注意：**KSK轮转需要同步在万网上更新DS记录

## 在任意客户端验证解析
```
#dig -t A www.bkjf-inc.com @8.8.8.8 +dnssec +short
192.144.198.128
A 8 3 60 20190330063017 20190228053309 53901 bkjf-inc.com. aKI5N4y6eqN/xunC7+4vYa3cSHyXcW533iGA6/q34/ahvq0sTgYN36aF oBO0t8fRvwS3chZaPxwuqbk6hGSW+tRhJ8x/Nnwtbcn004W0ZxI1k046 JW/ePLhq1Cw2GPHXJTsfCjYmAOcwssX2yUv6q9/vocXx/mipuTMljrId yhE=

#dig CNAME eshop.bkjf-inc.com @8.8.8.8 +dnssec +short
www.bkjf-inc.com.
CNAME 8 3 60 20190330063503 20190228053503 53901 bkjf-inc.com. 9ONt81AjpHFrM8YwDm7pQAg62oDBgaNzdtDIqtBHt5h/BPl83fOP/dOp P0Xi+y/OsFjDzHBSBDU4sy3fJwHBqm8uuMc6m33pIZfTq15fxFXF+2hU ift1bc0b0dk/L7ANZ5haEsDcl+hSVjwru2o2ISJtvp5zySZ61pdMvA6y ktg=
```

## 在第三方网站验证
<https://en.internet.nl/site/www.bkjf-inc.com/473349/>

## 浏览器插件
<https://www.dnssec-validator.cz/>

[参考文献](https://ftp.isc.org/isc/kb-files/DNSSEC-QR-B4.pdf)