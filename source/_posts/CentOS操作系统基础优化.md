title: CentOS操作系统基础优化
author: Stanley Wang
date: 2017-01-14 15:27:57
categories: Linux基础
tags:
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# 内核优化
```
ECHOSTR='net.ipv4.tcp_fin_timeout = 2
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_keepalive_time =600
net.ipv4.ip_local_port_range = 4000    65000
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.route.gc_timeout = 100
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_synack_retries = 1
net.core.somaxconn = 16384
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_max_orphans = 16384
net.nf_conntrack_max = 25000000
net.netfilter.nf_conntrack_max = 25000000
net.netfilter.nf_conntrack_tcp_timeout_established = 180
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 120'
echo "$ECHOSTR" >> /etc/sysctl.conf &&\
modprobe ip_conntrack && modprobe bridge
echo "modprobe ip_conntrack" >> /etc/rc.local
echo "modprobe bridge" >> /etc/rc.local
/sbin/sysctl -p
```

# 文件描述符
```vi /etc/security/limits.conf
*                hard    nofile          65535
*                soft    nofile          65535
*                hard    noproc          65535
*                soft    noproc          65535
```

# 更新yum源，安装epel源
``vi /etc/yum.repo.d/CentOS-Base.repo
略
``
```
# yum install epel-release -y
```

# 系统时钟同步
```
# yum install chrony -y
# systemctl start chronyd
# systemctl enable chronyd
```

# 关闭SELinux和防火墙
```
if [ -f /etc/selinux/config ]; then
  sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config
  setenforce 0
fi
```
```
# systemctl stop firewalld
# systemctl disable firewalld
```

# 调整系统字符集
```
# echo 'export LC_ALL=C'>> /etc/profile
# echo 'export LANG=en_US.UTF-8' >> /etc/profile
# source /etc/profile
```
# 安装基础工具
```
# yum install net-tools telnet tree nmap sysstat lrzsz dos2unix -y 
```