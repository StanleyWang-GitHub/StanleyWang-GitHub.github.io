title: 实验文档6：智能DNS实战
author: Stanley Wang
categories: Web DNS技术
date: 2018-12-16 10:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# BIND9的acl访问控制列表
## 4个内置acl
- any：任何主机
- none：没有主机
- localhost：本机
- localnet：本地子网所有IP

## 自定义acl
### 简单acl
```
acl "someips" {                               //定义一个名为someips的ACL  
  10.0.0.1; 192.168.23.1; 192.168.23.15;      //包含3个单个IP  
 };
```
### 复杂acl
```
acl "complex" {             //定义一个名为complex的ACL  
  "someips";                //可以嵌套包含其他ACL  
  10.0.15.0/24;             //包含10.0.15.0子网中的所有IP  
  !10.0.16.1/24;            //非10.0.16.1子网的IP  
  {10.0.17.1;10.0.18.2;};   //包含了一个IP组  
  localhost；               //本地网络接口IP（含实际接口IP和127.0.0.1）  
 };  
```
## 使用acl
```
allow-update { "someips"; };
allow-transfer { "complex"; };
...
```

# BIND9的view视图功能
view语句定义了视图功能。视图是BIND9提供的强大的新功能，允许DNS服务器根据客户端的不同，有区别地回答DNS查询，每个视图定义了一个被特定客户端子集见到的DNS名称空间。这个功能在一台主机上运行多个形式上独立的DNS服务器时特别有用。

## view的语法范例
```
view view_name [class] {
    match-clients { address_match_list } ;
    match-destinations { address_match_list } ;
    match-recursive-only { yes_or_no } ;
    [ view_option; ...]
    [ zone-statistics yes_or_no ; ]
    [ zone_statement; ...]
};
```

## view配置范例1：按照不同业务环境解析
**注：**以下是内网DNS的view使用范例
```
acl "env-test" {
    10.4.7.11;
};
acl "env-prd" {
    10.4.7.12;
};

view "env-test" {
    match-clients { "env-test"; };
    recursion yes;
    zone "od.com" {
        type master;
        file "env-test.od.com.zone";
    };
};
view "env-prd" {
    match-clients { "env-prd"; };
    recursion yes;
    zone "od.com" {
        type master;
        file "env-prd.od.com.zone";
    };
};
view "default" {
    match-clients { any; };
    recursion yes;
    zone "." IN {
	type hint;
	file "named.ca";
    };
    include "/etc/named.rfc1912.zones";
};
```

## view配置范例2：智能DNS
**注：**以下特指公网智能DNS配置范例
```
//电信IP访问控制列表
acl "telecomip"{ telecom_IP; ... };
//联通IP访问控制列表
acl "netcomip"{ netcom_IP; ... };
view "telecom" {
    match-clients { "telecomip"; };
    zone "ZONE_NAME" IN {
        type master;
        file "ZONE_NAME.telecom.zone";
    };
};
view "netcom" {
    match-clients { "netcomip"; };
    zone "ZONE_NAME" IN {
        type master;
        file "ZONE_NAME.netcom.zone";
    };
};
view "default" {
    match-clients { any; };
    zone "ZONE_NAME" IN {
        type master;
        file "ZONE_NAME.zone";
    };
};

```