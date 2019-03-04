title: RPM包制作
author: Stanley Wang
date: 2017-01-15 14:27:57
categories: Linux基础
tags:
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# RPM包能制作什么
- 一个应用程序
- 库文件
- 配置文件
- 文档包

# 制作步骤
## 创建制作目录
- BUILD
> 源代码解压以后，放在这个目录，不用用户参与，只需提供这个目录，真正的制作车间。
- RPMS
> 制作完成的RPM包放在这个目录。有子目录，跟硬件架构相关，特定平台子目录，i386，ARM等等。交叉编译。
- SOURCES
> 所有的原材料。
- SPECS
> spec文件存放目录，制作RPM包的纲领性文件。软件包名.spec。
- SRPMS
> SRC格式的RPM包存放目录。没有平台相关的概念。

**注意：**一般制作RPM包，建议不要用root用户，所以，以上制作目录结构，建议使用普通用户创建，不要用系统默认的。

- 宏定义
macrofiles：~/.rpmmacros，以最后这个为准
rmpbuild --showrc|grep _topdir
所以切换普通用户
```~/.rpmmacros
%_topdir           /home/xxx/rpmbuild
```
命令：
```
# mkdir -pv rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
# rpmbuild --showrc|grep _topdir
```

## 把源文件放进适当目录

## 制作spec文件（至关重要）
### 信息说明段（introduction section）Name，Version，Release，Group必须，其他可选。
rpm -qpi 可以查看一个rpm包的相关信息
定义各种tag
- Name：软件包的名字
- Relocations:是否可换安装地址。not relocatable
- Version：版本号，至关重要，只能用X.XX.XXX不能使用-
- Release：发行版本号 1%{?dist}
- License：声明版权，例如：GPLv2
- Group：属于那个组，不要自己定义，在以下组里找，只要存在的组就可以 less /usr/share/doc/rpm-4.4.2.3/GROUPS
- URL
- Packager：制作者<邮箱>
- Vendor：提供商
- Summary：概要
- %description：描述
- Source：源文件，链接，Source0：解压缩主原材料，Source1：脚本等等
- BuildRoot：编译好的程序，临时安装根目录，配合file section，收集哪些文件，打包到RPM包，最后在clean section中删除。可以规定任意目录：/%{_tmppath}/%{name}-%{version}-%{release}-root
- BuildRequires：定义依赖关系，编译依赖和安装依赖。

### 准备段prep section
解压缩源码包到制作目录，cd进去，设定工作环境、宏，等等。
单独的宏来定义：
- %prep 
- %setup -q 静默模式

### 制作段build section
```
%build
./configure balabalabala.......................
%{__make} %{?_smp_mflags} 多CPU上，这个标识可以加快编译速度
```
### 安装段install section
```
%install
%{__rm} -rf %{buildroot}
%{__make} install DESTDIR="%{buildroot}"
%find_long %{name}
%{__install} -p -D 0755 %{SOURCE1} %{buildroot}/etc/init.d/nginx 安装自定义的资源文件 -p保留原材料时间戳
```
> 补充：Linux系统install命令：类似于cp
> install /etc/fstab /tmp
> install -d /tmp/test 创建目录
> install -D /etc/fstab /tmp/noexistsdir/fstab
> 可以直接指定安装目标处不存在的目录，但是要把安装的源文件名也指定

### 脚本段script section
```
%pre安装前 $1=0,1,2 卸载，安装，升级
$1 == 1
加个用户
%post安装后
$1 == 1
chkconfig --add 
%preun卸载前
$1 == 0
service xxx stop
chkconfig --del
%postun卸载后
```

### 清理段clean cection
```
%clean
%{__rm} -rf %{buildroot}
```

### 文件段files section
除了debug信息，都要做进RPM包
```
%files -f %{name}.lang
%defattr (-,root,root,0755) 定义文件默认权限
%doc API CHANGES COPYING CREDITS README axelrc.examlpe 文档
%config(noreplace) %{_sysconfdir}/axelrc 配置文件，noreplace不替换原来的
/usr/local/bin/axel 包含的所有文件，可以直接写目录
%attr (0755,root,root) /etc/rc.d/init.d/nginx 定义自定义资源的属性，不指定则继承%defattr
```

### 更改日志段change log section
```
%changelog
*  xxx 日期，制作人，版本号
- Initial Version release号
```

## 制作RPM包
rpmbuild命令     
     -bp 只执行到prep段
     -bi 只执行到install段
     -bc 只执行到build段
     -bb 制作二进制格式的RPM包
     -bs 制作源码格式的RPM包
     -ba 既制作二进制格式又制作源码格式
     -bl 检查files字段和buildroot文件是否一一对应，不一致则报错

## 展开rpm源码包
rpm2cpio xxxx-src.rpm | cpio -id
> 到哪去找源码包呢？
> rpmfind.net
> rpm.pbone.net

# YUM后的rpm包保留在本地的方法
```
# sed -i 's#keepcache=0#keepcache=1#g' /etc/yum.conf
```
rpm包默认存放路径
```
/var/cache/yum/base/packages 
```

# fpm制作rpm包
```
# yum install ruby
# gem source -a http://mirrors.aliyun.com/rubygems/
# gem source -r http://rubygems.org/
# gem install fpm
```

[参考文档](http://www.ibm.com/developerworks/cn/linux/l-rpm/)