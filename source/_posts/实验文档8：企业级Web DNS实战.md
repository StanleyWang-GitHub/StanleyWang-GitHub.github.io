title: 实验文档8：企业级Web DNS实战
author: Stanley Wang
categories: Web DNS技术
date: 2018-12-16 6:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# 系统环境
```
#cat /etc/redhat-release 
CentOS Linux release 7.5.1804 (Core) 

#uname -a
Linux node 3.10.0-862.el7.x86_64 #1 SMP Fri Apr 20 16:44:24 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```
# 安装部署namedmanager
## 准备rpm包
<https://repos.jethrocarr.com/pub/jethrocarr/linux/centos/7/jethrocarr-custom/x86_64/>

## 下载最新版
```
[root@hdss7-11 opt]# ll
total 62244
-rw-r--r-- 1 root root   102136 Feb  1 18:17 namedmanager-bind-1.9.0-2.el7.centos.noarch.rpm
-rw-r--r-- 1 root root  1084340 Feb  1 18:17 namedmanager-www-1.9.0-2.el7.centos.noarch.rpm
```

## 安装
```
[root@hdss7-11 opt]# yum localinstall namedmanager-* -y
...
Installed:
  namedmanager-bind.noarch 0:1.9.0-2.el7.centos                                 namedmanager-www.noarch 0:1.9.0-2.el7.centos                                

Dependency Installed:
  apr.x86_64 0:1.4.8-3.el7_4.1                        apr-util.x86_64 0:1.5.2-6.el7                    bind.x86_64 32:9.9.4-73.el7_6                       
  httpd.x86_64 0:2.4.6-88.el7.centos                  httpd-tools.x86_64 0:2.4.6-88.el7.centos         libzip.x86_64 0:0.10.1-8.el7                        
  mailcap.noarch 0:2.1.41-2.el7                       mariadb.x86_64 1:5.5.60-1.el7_5                  mariadb-libs.x86_64 1:5.5.60-1.el7_5                
  mariadb-server.x86_64 1:5.5.60-1.el7_5              mod_ssl.x86_64 1:2.4.6-88.el7.centos             perl-Compress-Raw-Bzip2.x86_64 0:2.061-3.el7        
  perl-Compress-Raw-Zlib.x86_64 1:2.061-4.el7         perl-DBD-MySQL.x86_64 0:4.023-6.el7              perl-DBI.x86_64 0:1.627-4.el7                       
  perl-IO-Compress.noarch 0:2.061-2.el7               perl-Net-Daemon.noarch 0:0.48-5.el7              perl-PlRPC.noarch 0:0.2020-14.el7                   
  php.x86_64 0:5.4.16-46.el7                          php-cli.x86_64 0:5.4.16-46.el7                   php-common.x86_64 0:5.4.16-46.el7                   
  php-intl.x86_64 0:5.4.16-46.el7                     php-ldap.x86_64 0:5.4.16-46.el7                  php-mysqlnd.x86_64 0:5.4.16-46.el7                  
  php-pdo.x86_64 0:5.4.16-46.el7                      php-process.x86_64 0:5.4.16-46.el7               php-soap.x86_64 0:5.4.16-46.el7                     
  php-xml.x86_64 0:5.4.16-46.el7                     

Dependency Updated:
  bind-libs.x86_64 32:9.9.4-73.el7_6  bind-libs-lite.x86_64 32:9.9.4-73.el7_6  bind-license.noarch 32:9.9.4-73.el7_6  bind-utils.x86_64 32:9.9.4-73.el7_6 

Complete!
```

## 先配mysql
### 启动mysql
```
[root@hdss7-11 mysql]# systemctl start mariadb.service
[root@hdss7-11 ~]# systemctl enable mariadb
Created symlink from /etc/systemd/system/multi-user.target.wants/mariadb.service to /usr/lib/systemd/system/mariadb.service.
[root@hdss7-11 mysql]# mysqladmin -uroot password 123456
[root@hdss7-11 mysql]# cd /usr/share/namedmanager/resources/
[root@hdss7-11 resources]# ./autoinstall.pl 
autoinstall.pl

This script setups the NamedManager database components:
 * NamedManager MySQL user
 * NamedManager database
 * NamedManager configuration files

THIS SCRIPT ONLY NEEDS TO BE RUN FOR THE VERY FIRST INSTALL OF NAMEDMANAGER.
DO NOT RUN FOR ANY OTHER REASON

Please enter MySQL root password (if any): 123456

输入123456

Searching ../sql/ for latest install schema...
../sql//version_20131222_install.sql is the latest file and will be used for the install.
Importing file ../sql//version_20131222_install.sql
Creating user...
Updating configuration file...
DB installation complete!

You can now login with the default username/password of setup/setup123 at http://localhost/namedmanager
```

## 配置namedmanager
### 改config.php
```vi /etc/namedmanager/config.php
$_SERVER['HTTPS'] = "TRUE";
```
### 改config-bind.php
```vi /etc/namedmanager/config-bind.php
$config["api_url"]              = "http://dns-manager.od.com/namedmanager";     // Application Install Location
$config["api_server_name"]      = "dns-manager.od.com";                         // Name of the DNS server (important: part of the authentication process)
$config["api_auth_key"]         = "verycloud";                                  // API authentication key
```

### 绑host
```vi /etc/hosts
10.4.7.11   dns-manager.od.com
```

### 配apache
```vi /etc/httpd/conf/httpd.conf
Listen 10.4.7.11:8080
	ServerName dns-manager.od.com
	<Directory />
		AllowOverride none
		allow from all
		#Require all denied
	</Directory>
```

### 配nginx
```vi /etc/nginx/conf.d/dns-manager.od.com.conf
server {
		server_name dns-manager.od.com;
   
		location =/ {
			rewrite ^/(.*) http://dns-manager.od.com/namedmanager  permanent;
		}
		location / {
			proxy_pass http://10.4.7.11:8080;
			proxy_set_header Host       $http_host;
			proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
		}
	}
```

## 启动apache和nginx
- 启动apache
```
[root@hdss7-11 ~]# systemctl start httpd
[root@hdss7-11 ~]# systemctl enable httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
```
- 启动nginx
```
[root@hdss7-11 ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@hdss7-11 ~]# nginx 
```
访问<http://dns-manager.od.com>，看看页面是否正常

## 继续改namedmanager的配置
### 改namedmanager_bind_configwriter.php
```vi /usr/share/namedmanager/bind/namedmanager_bind_configwriter.php
if (flock($fh_lock, LOCK_EX ))
{
	log_write("debug", "script", "Obtained filelock");
}
```
### 启动namedmanager_logpush.rcsysinit
#### 启动该脚本
```vi /usr/share/namedmanager/resources/namedmanager_logpush.rcsysinit
[root@hdss7-11 resources]# sh namedmanager_logpush.rcsysinit start
Starting namedmanager_logpush service:
[root@hdss7-11 resources]# nohup: redirecting stderr to stdout
```
#### 检查是否启动
```
[root@hdss7-11 resources]# ps -ef|grep php|egrep -v grep
root      10738      1  0 10:49 pts/1    00:00:00 php -q /usr/share/namedmanager/bind/namedmanager_logpush.php
```
#### 检查日志
```vi /var/log/namedmanager_logpush
[root@hdss7-11 resources]# tail -fn 200 /var/log/namedmanager_logpush
Error: Unable to authenticate with NamedManager API - check that auth API key and server name are valid
```
有报错，所以需要继续配置
### 改inc_soap_api.php
```vi /usr/share/namedmanager/bind/include/application/inc_soap_api.php
preg_match("/^http:\/\/(\S*?)[:0-9]*\//", $GLOBALS["config"]["api_url"], $matches);
```
### 重启namedmanager_logpush.rcsysinit
```vi /usr/share/namedmanager/resources/namedmanager_logpush.rcsysinit
[root@hdss7-11 resources]# sh namedmanager_logpush.rcsysinit restart
Stopping namedmanager_logpush services:
Starting namedmanager_logpush service:
nohup: redirecting stderr to stdout
```

# 配置BIND9
## 先配rndc
### rndc.key
```
[root@hdss7-11 ~]# cat /etc/rndc.key 
key "rndc-key" {
	algorithm hmac-sha256;
	secret "CD/4vqb9l0WiMy5TXjfeu1cMhyRerQ9kL2jwdBFWwa4=";
};
```
如果没有，使用如下命令生成rndc.key
```
[root@hdss7-11 ~]# rndc-keygen -r /dev/urandom
```
### 配rndc.conf
```vi /etc/rndc.conf
key "rndc-key" {
	algorithm hmac-sha256;
	secret "CD/4vqb9l0WiMy5TXjfeu1cMhyRerQ9kL2jwdBFWwa4=";
};

options {
	default-key "rndc-key";   
	default-server 10.4.7.11;
	default-port 953;   
}; 
```
### 删除rndc.key
```
[root@hdss7-11 ~]# rm -f /etc/rndc.key
```
## BIND9主配置文件
```vi /etc/named.conf
options {
    listen-on port 53 { 10.4.7.11; };
    directory 	"/var/named";
    dump-file 	"/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    allow-query     { any; };
    allow-transfer { 10.4.7.12; };
    also-notify { 10.4.7.12; };

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
    recursion yes;  

    dnssec-enable no;
    dnssec-validation no;


    /* Path to ISC DLV key */
    bindkeys-file "/etc/named.iscdlv.key";

    managed-keys-directory "/var/named/dynamic";

    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
};

key "rndc-key" {
    algorithm hmac-sha256;
    secret "CD/4vqb9l0WiMy5TXjfeu1cMhyRerQ9kL2jwdBFWwa4=";
};

controls {
    inet 10.4.7.11 port 953
        allow { 10.4.7.11;} keys { "rndc-key"; };
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
include "/etc/named.namedmanager.conf";
```
## 改named.namedmanager.conf文件属性
```vi /etc/named.namedmanager.conf
[root@hdss7-11 named]# chown apache.named /etc/named.namedmanager.conf
[root@hdss7-11 named]# ls -l /etc/named.namedmanager.conf 
-rw-r--r-- 1 apache named 112 Dec  16 11:19 /etc/named.namedmanager.conf
```

## 检查配置并启动BIND9
### 检查配置
```
[root@hdss7-11 ~]# named-checkconf
```
### 启动BIND9
```
[root@hdss7-11 ~]# systemctl start named
```
### 开机自启动
```
[root@hdss7-11 ~]# systemctl enable named
```
### 检查启动情况
```
[root@hdss7-11 ~]# netstat -luntp|grep 53
tcp        0      0 10.4.7.11:53            0.0.0.0:*               LISTEN      10922/named         
tcp        0      0 10.4.7.11:953           0.0.0.0:*               LISTEN      10922/named         
udp        0      0 10.4.7.11:53            0.0.0.0:*                           10922/named 
```

# 配置Web DNS页面
浏览器打开<http://dns-manager.od.com>（提前绑好host），用户名/密码：setup/setup123
## 配置Configuration选项卡
### Zone Configuration Defaults
- DEFAULT_HOSTMASTER
> 87527941@qq.com
- DEFAULT_TTL_SOA
> 86400
- DEFAULT_TTL_NS
> 120
- DEFAULT_TTL_MX
> 60
- DEFAULT_TTL_OTHER
> 60

### API Configuration
- ADMIN_API_KEY
> verycloud

### Date and Time Configuration
- DATEFORMAT
> yyyy-mm-dd
- TIMEZONE_DEFAULT
> Asia/Shanghai

### Save Changes

## 配置New Servers选项卡
### Add NewServer
#### Server Details
- Name Server FQDN *
> dns-manager.od.com
- Description
> dns server for od.com
#### Server Type
- Server Type
> API (supports Bind)
- API Authentication Key *
> verycloud
#### Server Domain Settings
勾选
- Nameserver Group *
> default -- Default Nameserver Group
- Primary Nameserver *
> Make this server the primary one used for DNS SOA records.
不勾选
- Use as NS Record *
> Adds this name server to all domains as a public NS record.
### Save Changes

## 配置Domain/Zones选项卡
两种方式
- 手动添加域
- 自动导入域
### Add Domain（手动）
#### Domain Details
- Domain Type * 
> Standard Domain
> Reverse Domain (IPv4)
> Reverse Domain (IPv6)
根据实际情况选择，这里选择Standard Domain（正解域）
- Domain Name *
> od.com
- Description
> od.com domain
#### Domain Server Groups
**注意**：一定要勾选域服务器组
> default -- Default Nameserver Group
#### Start of Authority Record
- Email Administrator Address *
> Email Administrator Address *
- Domain Serial *
> 2018121601
- Refresh Timer *
> 21600
- Refresh Retry Timeout *
> 3600
- Expiry Timer *
> 604800
- Default Record TTL *
> 60

### Save Changes

### View Domains 
#### domian records
- Nameserver Configuration
- Mailserver Configuration
- Host Records Configuration
Type|TTL|Name|Content|ReversePTR|
-|-|-|-|-|-
A|60|dns-manager|10.4.7.11|no|-

### Save Changes

### Import Domain（自动导入）
#### Upload
- Import Source
> Bind 8/9 Compatible Zonefile
- Zone File
> 选择文件host.txt
附：host.txt
```
$ORIGIN .
$TTL 600	; 10 minutes
host.com			IN SOA	dns-manager.host.com. 87527941.qq.com. (
				2019013106 ; serial
				10800      ; refresh (3 hours)
				900        ; retry (15 minutes)
				604800     ; expire (1 week)
				86400      ; minimum (1 day)
				)
			   NS   ns1.host.com.
$ORIGIN host.com.
$TTL 60	; 1 minute
ns1                        A    10.4.7.11
dns-manager                A	10.4.7.11
```
#### Save Changes
#### Domain Details
#### Start of Authority Record
#### Domain Records
#### Save Changes
#### Domain Details
#### Domain Server Groups
**注意**：这里一定要勾选服务器组
#### Start of Authority Record
#### Save Changes

### 再导入一个反解域
略

## 返回Name Servers选项卡
### 查看页面状态
- Logging Status
> status_synced
- Zonefile Status
> status_synced

## 查看服务器上配置文件（都是由namedmanager服务自动生成的）
### named.namedmanager.conf
```cat /etc/named.namedmanager.conf
//
// NamedManager Configuration
//
// This file is automatically generated any manual changes will be lost.
//
zone "od.com" IN {
	type master;
	file "od.com.zone";
	allow-update { none; };
};
```
### od.com.zone
```vi /var/named/od.com.zone
$ORIGIN od.com.
$TTL 86400
@		IN SOA dns-manager.od.com. 87527941.qq.com. (
			2019020202 ; serial
			10800 ; refresh
			900 ; retry
			604800 ; expiry
			86400 ; minimum ttl
		)

; Nameservers

@	120 IN NS ns1.od.com.
od.com.	86400 IN NS dns-manager.od.com.

; Mailservers


; Reverse DNS Records (PTR)


; CNAME


; HOST RECORDS

dns-manager	60 IN A 10.4.7.11
ns1	60 IN A 10.4.7.11
```
## 检查资源记录是否生效
```
[root@hdss7-11 named]# dig -t A dns-manager.od.com @10.4.7.11 +short
10.4.7.11
```
## 验证页面增、删、改是否均生效
**注意**：这里在页面上操作资源记录，会先写mysql，再由php脚本定期刷到磁盘文件上，所以大概需要1分钟的时间生效

## 最后配置主辅同步
略

# 用户系统及操作审计功能
## 用户系统
可以创建不同的管理员用户
### User Management选项卡
### Create a new User Account
#### User Details
- Username *
> wangdao
- Real Name *
> StanleyWang
- Contact Email *
> stanley.wang.m@qq.com
#### User Password
- password *
> 123456
- password_confirm *
> 123456
#### Save Changes
### User's Permissions选项卡
#### User Permissions
- disabled
> 勾上，用户不生效
> 不勾，用户生效
> 这里不勾
- admin（超级管理员）
> 勾上，可以创建用户管理用户权限
> 不勾，不可以创建用户管理用户权限
> 这里不勾
- namedadmins（管理员）
> 勾上，dns管理员，可以管理zone和资源记录
> 不勾，不可以管理zone和资源记录
> 这里勾选
#### Save Changes
### 使用wangdao用户登录
可以进行DNS服务管理，但无法管理用户

## 审计
### 使用wangdao用户在页面增加一条资源记录
操作过程略
### Changelog选项卡
可以看到wangdao用户的操作记录，实现审计功能，做到操作可溯

### Tips
- 生产上强烈建议新生成一个超级管理员用户并将setup用户删除！
- 超级管理员用户不要轻易外泄，可以创建多个管理员账户。（一般根据业务而定，每个管理员负责一个子域）
- 管理员账户创建好后，可自行登录修改密码。
- 超级管理员用户密码的复杂度要足够高，定期更换超级管理员用户密码。