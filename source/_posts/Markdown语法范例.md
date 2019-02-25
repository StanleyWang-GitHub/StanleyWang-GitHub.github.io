title: Markdown语法范例
author: Stanley Wang
categories: Linux基础
mathjax: true
date: 2018-01-11 09:32:16
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6 "VIP学习QQ群")<=={% endcq %}
- - -
# 分级标题
{% tabs 分级标题%}
<!-- tab 效果 -->
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
<!-- endtab -->
<!-- tab 代码-->
{% code %}
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
{% endcode %}
<!-- endtab -->
{% endtabs %}

# 字体
## 样式
{% tabs 字体样式%}
<!-- tab 效果 -->
*斜体*
**粗体**
***加粗斜体***
~~删除线~~
<!-- endtab -->
<!-- tab 代码-->
{% code %}
\*斜体\*
\*\*粗体\*\*
\*\*\*加粗斜体\*\*\*
\~\~删除线\~\~
{% endcode %}
<!-- endtab -->
{% endtabs %}
## 颜色、字号
{% tabs 颜色字号%}
<!-- tab 效果 -->
<font color="FF7F50">颜色</font>
<font size="20">字号</font>
<!-- endtab -->
<!-- tab 代码-->
{% code %}
<font color="FF7F50">颜色</font>
<font size="20">字号</font>
{% endcode %}
<!-- endtab -->
{% endtabs %}

# 列表
## 有序列表
有序列表则使用数字接着一个英文句点。
{% tabs 有序列表%}
<!-- tab 效果 -->
1. 有序列表项 一
2. 有序列表项 二
3. 有序列表项 三
<!-- endtab -->
<!-- tab 代码-->
{% code %}
1. 有序列表项 一
2. 有序列表项 二
3. 有序列表项 三
{% endcode %}
<!-- endtab -->
{% endtabs %}
在特殊情况下，项目列表很可能会不小心产生，像是下面这样的写法：
```
1986. What a great season.
```
会显示成：
>1986. What a great season.

前面的1986成了序号，换句话说，也就是在行首出现了数字-句点-空白，要避免这样的状况，你可以在句点前面加上反斜杠：
```
1986\. What a great season.
```
则会正确的显示为：
>1986\. What a great season.
## 无序列表
使用 *，+，-表示无序列表。
{% tabs 有序列表%}
<!-- tab 效果 -->
- 无序列表项 一
- 无序列表项 二
- 无序列表项 三
<!-- endtab -->
<!-- tab 代码-->
{% code %}
- 无序列表项 一
- 无序列表项 二
- 无序列表项 三
{% endcode %}
<!-- endtab -->
{% endtabs %}
## 定义型列表
定义型列表由名词和解释组成。一行写上定义，紧跟一行写上解释。解释的写法“:”紧跟一个缩进(Tab)
{% tabs 有序列表%}
<!-- tab 效果 -->
Markdown
:	轻量级文本标记语言，可以转换成html，pdf等格式（左侧有一个可见的冒号和四个不可见的空格）
代码块 2
:	这是代码块的定义（左侧有一个可见的冒号和四个不可见的空格）
<!-- endtab -->
<!-- tab 代码-->
{% code %}
Markdown
:    轻量级文本标记语言，可以转换成html，pdf等格式（左侧有一个可见的冒号和四个不可见的空格）
代码块 2
:    这是代码块的定义（左侧有一个可见的冒号和四个不可见的空格）
{% endcode %}
<!-- endtab -->
{% endtabs %}
# 引用
## 一般引用
引用需要在被引用的文本前加上>符号。
{% tabs 一般引用%}
<!-- tab 效果 -->
> 这是一个有两段文字的引用
> 无意义的占行文字1.
> 无意义的占行文字2.
>
> 无意义的占行文字3.
> 无意义的占行文字4.
<!-- endtab -->
<!-- tab 代码-->
{% code 引用%}
> 这是一个有两段文字的引用
> 无意义的占行文字1.
> 无意义的占行文字2.
>
> 无意义的占行文字3.
> 无意义的占行文字4.
{% endcode %}
<!-- endtab -->
{% endtabs %}
## 引用嵌套
区块引用可以嵌套（例如：引用内的引用），只要根据层次加上不同数量的 > ：
{% tabs 引用嵌套%}
<!-- tab 效果 -->
> 请问 Markdwon 怎么用？ - 小白
>> 自己看教程！ - 愤青
>>> 教程在哪？ - 小白
<!-- endtab -->
<!-- tab 代码-->
{% code %}
> 请问 Markdwon 怎么用？ - 小白
>> 自己看教程！ - 愤青
>>> 教程在哪？ - 小白
{% endcode %}
<!-- endtab -->
{% endtabs %}
## 引用其它要素
引用的区块内也可以使用其他的 Markdown 语法，包括标题、列表、代码区块等：
{% tabs 引用其他%}
<!-- tab 效果 -->
> 1.   这是第一行列表项。
> 2.   这是第二行列表项。
>
> 引用代码行：
> `return shell_exec("echo $input | $markdown_script");`
> 引用代码段：
> {% code %}
for (list in $lists);do
	echo $list;
done
{% endcode %}
<!-- endtab -->
<!-- tab 代码-->
{% code %}
> 1.   这是第一行列表项。
> 2.   这是第二行列表项。
>
> 引用代码行：
> `return shell_exec("echo $input | $markdown_script");`
> 引用代码段：
>{\% code %}
for (list in $lists);do
 	echo $list;
done
{\% endcode %}
{% endcode %}
<!-- endtab -->
{% endtabs %}
## note标签引用
{% tabs note标签引用%}
<!-- tab default -->
{% note default %}
### default with-icon
#### default 1.1
#### default 1.2
{% endnote %}
{% note default no-icon %}
### defalt no-icon
#### default 1.1
#### default 1.2
{% endnote %}
<!-- endtab -->
<!-- tab primary-->
{% note primary %}
### primary with-icon
#### primary 1.1
#### primary 1.2
{% endnote %}
{% note primary no-icon %}
### primary no-icon
#### primary 1.1
#### primary 1.2
{% endnote %}
<!-- endtab -->
<!-- tab info-->
{% note info %}
### info with-icon
#### info 1.1
#### info 1.2
{% endnote %}
{% note info no-icon %}
### info no-icon
#### info 1.1
#### info 1.2
{% endnote %}
<!-- endtab -->
<!-- tab success-->
{% note success %}
### success with-icon
#### success 1.1
#### success 1.2
{% endnote %}
{% note success no-icon %}
### success no-icon
#### success 1.1
#### success 1.2
{% endnote %}
<!-- endtab -->
<!-- tab warning-->
{% note warning %}
### warning with-icon
#### warning 1.1
#### warning 1.2
{% endnote %}
{% note warning no-icon %}
### warning no-icon
#### warning 1.1
#### warning 1.2
{% endnote %}
<!-- endtab -->
<!-- tab danger-->
{% note danger %}
### danger with-icon
#### danger 1.1
#### danger 1.2
{% endnote %}
{% note danger no-icon %}
### danger no-icon
#### danger 1.1
#### danger 1.2
{% endnote %}
<!-- endtab -->
{% endtabs %}
## 居中引用

效果：
{% cq %}Something{% endcq %}
代码：
```
{% cq %}Something{% endcq %}
```
# 表格
第一行为表头，第二行分隔表头和主体部分，第三行开始每一行为一个表格行。
列于列之间用管道符|隔开。原生方式的表格每一行的两边也要有管道符。
第二行还可以为不同的列指定对齐方向。默认为左对齐，在“-”右边加上“:”就右对齐，在“-”两边都加上“:”就居中对齐。
{% tabs 表格%}
<!-- tab 效果：左对齐 -->
学号|姓名|分数
-|-|-
小明|男|75
小红|女|79
小陆|男|92
<!-- endtab -->
<!-- tab 代码：左对齐-->
{% code 成绩表%}
学号|姓名|分数
-|-|-
小明|男|75
小红|女|79
小陆|男|92
{% endcode %}
<!-- endtab -->
<!-- tab 效果：右对齐 -->
学号|姓名|分数
-:|-:|-:
小明|男|75
小红|女|79
小陆|男|92
<!-- endtab -->
<!-- tab 代码：右对齐-->
{% code 成绩表%}
学号|姓名|分数
-:|-:|-:
小明|男|75
小红|女|79
小陆|男|92
{% endcode %}
<!-- endtab -->
<!-- tab 效果：居中 -->
学号|姓名|分数
:-:|:-:|:-:
小明|男|75
小红|女|79
小陆|男|92
<!-- endtab -->
<!-- tab 代码：居中-->
{% code 成绩表%}
学号|姓名|分数
:-:|:-:|:-:
小明|男|75
小红|女|79
小陆|男|92
{% endcode %}
<!-- endtab -->
{% endtabs %}
# 分割线
你可以在一行中用三个以上的星号、减号、底线来建立一个分隔线，行内不能有其他东西。你也可以在星号或是减号中间插入空格。下面每种写法都可以建立分隔线：
{% tabs 分割线%}
<!-- tab 效果 -->
* * *
***
*****
- - -
---------------------------------------
<!-- endtab -->
<!-- tab 代码-->
{% code %}
\* \* \*
\***
\*****
- - -
\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-
{% endcode %}
<!-- endtab -->
{% endtabs %}
# 代码
## 行内代码
{% tabs 行内代码%}
<!-- tab 效果 -->
C语言里的函数 `scanf()` 怎么使用？
C语言里的函数 {% label default@scanf() %} 怎么使用？
C语言里的函数 {% label primary@scanf() %} 怎么使用？
C语言里的函数 {% label info@scanf() %} 怎么使用？
C语言里的函数 {% label success@scanf() %} 怎么使用？
C语言里的函数 {% label warning@scanf() %} 怎么使用？
C语言里的函数 {% label danger@scanf() %} 怎么使用？
<!-- endtab -->
<!-- tab 代码-->
{% code %}
C语言里的函数 \`scanf()\` 怎么使用？
C语言里的函数 {\% label default@scanf() %} 怎么使用？
C语言里的函数 {\% label primary@scanf() %} 怎么使用？
C语言里的函数 {\% label info@scanf() %} 怎么使用？
C语言里的函数 {\% label success@scanf() %} 怎么使用？
C语言里的函数 {\% label warning@scanf() %} 怎么使用？
C语言里的函数 {\% label danger@scanf() %} 怎么使用？
{% endcode %}
<!-- endtab -->
{% endtabs %}
## 多行代码
{% tabs 多行代码%}
<!-- tab 效果 -->
{% code %}
for (list in $lists);do
	echo $list
done
{% endcode %}
<!-- endtab -->
<!-- tab 代码-->
{% code %}
\`\`\`
for (list in $lists);do
	echo $list
done
\`\`\`
{% endcode %}
<!-- endtab -->
{% endtabs %}
# 超链接
## 行内式超链接
{% tabs 行内式%}
<!-- tab 效果 -->
不带title：
欢迎来到[我的博客](https://blog.stanley.wang)
带title：
欢迎来到[我的博客](https://blog.stanley.wang "Stanley's Blog")
<!-- endtab -->
<!-- tab 代码-->
{% code %}
不带title：
欢迎来到\[我的博客\]\(https://blog.stanley.wang\)
带title：
欢迎来到\[我的博客\]\(https://blog.stanley.wang "Stanley's Blog"\)
{% endcode %}
<!-- endtab -->
{% endtabs %}
## 自动超链接
{% tabs 自动%}
<!-- tab 效果 -->
<https://blog.stanley.wang>
<stanley.wang.m@qq.com>
<!-- endtab -->
<!-- tab 代码-->
{% code %}
\<https://blog.stanley.wang\>
\<stanley.wang.m@qq.com\>
{% endcode %}
<!-- endtab -->
{% endtabs %}
## 锚点超链接
暂不支持锚点超链接
# 公式
## 示例
{% tabs 公式%}
<!-- tab 效果 -->
When $a \ne 0$, there are two solutions to $ax^2$ + bx + c = 0 and they are： $$x= {-b \pm \sqrt{b^2-4ac} \over 2a}$$
<!-- endtab -->
<!-- tab 代码-->
{% code %}
When $a \ne 0$, there are two solutions to $ax^2$ + bx + c = 0 and they are： $$x= {-b \pm \sqrt{b^2-4ac} \over 2a}$$
{% endcode %}
<!-- endtab -->
{% endtabs %}
## 语法规范
### 呈现位置
- 行内公式:	使用`$…$`定义，此时公式在一行内显示
{% tabs 行内 %}
<!-- tab 效果 -->
$\displaystyle\sum_{i=0}^N\int_{a}^{b}g(t,i)\text{d}t$
<!-- endtab -->
<!-- tab 代码-->
{% code %}
$\displaystyle\sum_{i=0}^N\int_{a}^{b}g(t,i)\text{d}t$
{% endcode %}
<!-- endtab -->
{% endtabs %}
- 文内公式:	使用`$$…$$`定义，此时公式居中放大显示
{% tabs 文内 %}
<!-- tab 效果 -->
$$\sum_{i=0}^N\int_{a}^{b}g(t,i)\text{d}t$$
<!-- endtab -->
<!-- tab 代码-->
{% code %}
$$\sum_{i=0}^N\int_{a}^{b}g(t,i)\text{d}t$$
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 字母、运算符与杂项
#### 希腊字母

显示|命令|显示|命令
:-:|:-:|:-:|:-:
$\alpha$|\alpha|$\beta$|\beta
$\gamma$|\gamma|$\delta$|\delta
$\epsilon$|\epsilon|$\zeta$|\zeta
$\eta$|\eta|$\theta$|\theta
$\iota$|\iota|$\kappa$|\kappa
$\lambda$|\lambda|$\mu$|\mu
$\nu$|\nu|$\xi$|\xi
$\pi$|\pi|$\rho$|\rho
$\sigma$|\sigma|$\tau$|\tau
$\upsilon$|\upsilon|$\phi$|\phi
$\chi$|\chi|$\psi$|\psi
$\omega$|\omega|—|—
- 如果要大写希腊字母，则首字母大写即可，如`$\Gamma$`显示为$\Gamma$
- 如果要使希腊字母显示为斜体，则前面添加var即可，如`$\varGamma$`显示为$\varGamma$

#### 字母修饰
##### 上下标
- 上标：`^`
- 下标：`_`
> 举例：`$C_n^2$`显示为：$C_n^2$

##### 矢量
- 单字母向量:
>`$\vec a$`显示为$\vec a$
>`$\overrightarrow a$`显示为$\overrightarrow a$
- 多字母向量:
>`$\vec {abcde}$`显示为$\vec {abcde}$
>`$\overrightarrow {abcde}$`显示为$\overrightarrow {abcde}$
- 特殊修饰:
>上尖号:`$\hat {abcde}$`显示为$\hat {abcde}$
>宽上尖号: `$\widehat {abcde}$`显示为$\widehat {abcde}$
>上划线:`$\overline {abc}de$`显示为$\overline {abc}de$
>下划线:`$\underline ab{cde}$`显示为$\underline ab{cde}$

##### 字体
- TypeWriter:`$\mathtt {A}$`显示为:$\mathtt {ABCDEFGHIJKLMNOPQRSTUVWXYZ}$
- Blackboard blod:`$\mathbb {A}$`显示为:$\mathbb {ABCDEFGHIJKLMNOPQRSTUVWXYZ}$
- Sans Serif:`$\mathsf {A}$`显示为:$\mathsf {ABCDEFGHIJKLMNOPQRSTUVWXYZ}$

##### 空格
- 语法本身忽略空格，`$ab$`和`$a b$`都显示为$ab$ $a b$
- 小空格:`$a\ b$`显示为$a\ b$
- 4格空格:`$a\quad b$`显示为$a\quad b$

#### 分组
- 使用`{}`将同一级的括在一起，成组处理
> `$x_i^2$`显示为$x_i^2$
> `$x_{i^2}$`显示为$x_{i^2}$

#### 括号
- 小括号:`$(...)$`显示为$(...)$
- 中括号:`$[...]$`显示为$[...]$
- 大括号:`$\\{...\\}$`显示为$\\{...\\}$
- 尖括号：`$\langle ... \rangle$`显示为$\langle ... \rangle$
- 绝对值：`$\vert ... \vert$`显示为$\vert ... \vert$
- 双竖线：`$\Vert ... \Vert$`显示为$\Vert ... \Vert$
- 使用`$\left$`和`$\right$`使符号大小与邻近的公式相适应,该语句适用于所有括号类型
> `$\\{\frac{(x+y)}{[\alpha+\beta]}\\}$`显示为$\\{\frac{(x+y)}{[\alpha+\beta]}\\}$
> `$\left\\{\frac{(x+y)}{[\alpha+\beta]}\right\\}$`显示为$\left\\{\frac{(x+y)}{[\alpha+\beta]}\right\\}$

#### 常用数学运算符
##### 基础符号
运算符|说明|应用举例|命令
:-:|:-:|:-:|:-:
+|加|$x+y$|`$x+y$`
-|减|$x−y$|`$x-y$`
\times|叉乘|$x\times y$|`$x\timesy$`
\cdot|点乘|$x\cdot y$|`$x\cdot y$`
\ast(*)|星乘|$x\ast(y)$|`$x\ast(y)$`
\div|除|$x\div y$|`$x\div y$`
\pm|加减|$x\pm y$|`$x\pm y$`
\mp|减加|$x\mp y$|`$x\mp y$`
\approx|约等于|$x\approx y$|`$x\approx y$`	 
\equiv|恒等于|$x\equiv y$|`$x\equiv y$`
\cong|全等于|$\triangle ABC\cong \triangle BCD$|`$\triangle ABC\cong \triangle BCD$`
\sim|相似于|$x\sim y$|`$x\sim$ y`
\bigodot|定义运算符|$x\bigodot y$|`$x\bigodot y$`
\bigotimes|定义运算符|$x\bigotimes y$|`$x\bigotimes y$`

##### 比较运算符
运算符|说明|应用举例|命令
:-:|:-:|:-:|:-:
=|等于|$x=y$|`$x=y$`
\lt|小于|$x\lt y$|`$x\lt y$`
\gt|大于|$x\gt y$|`$x\gt y$`
\le|小于等于|$x\le y$|`$x\le y$`
\ge|大于等于|$x\ge y$|`$x\ge y$`
\ne|不等于|$x\ne y$|`$x\ne y$`

##### 逻辑运算符
运算符|说明|应用举例|命令
:-:|:-:|:-:|:-:
\land|与|$x\land y$|`$x\land y$`
\lor|或|$x\lor y$|`$x\lor y$`
\lnot|非|$\lnot x$|`$\lnot x$`
\oplus|异或|$x\oplus y=(\lnot x\land y)\lor(x\land \lnot y)$|`$x\oplus y=(\lnot x\land y)\lor(x\land \lnot y)$`
\forall|针对所有|$\forall x \in N$|`$\forall x \in N$`
\exists|存在|$\exists \xi$|`$\exists \xi$`


##### 集合符号
运算符|说明|应用举例|命令
:-:|:-:|:-:|:-:
\in|属于|$x\in y$|`$x\in y$`
\subseteq|子集|$x\subseteq y$|`$x\subseteq y$`
\subset|真子集|$x\subset y$|`$x\subset y$`
\supset|超集|$x\supset y$|`$x\supset y$`
\supseteq|超集|$x\supseteq y$|`$x\supseteq y$`
\varnothing|空集|$\varnothing$|`$\varnothing$`
\cup|并|$x\cup y$|`$x\cup y$`
\cap|交|$x\cap y$|`$x\cap y$`

##### 特殊符号
符号|命令|符号|命令
:-:|:-:|:-:|:-:
$\infty$|`$\infty$`|$\partial$|`$\partial$`
$\nabla$|`$\nabla$`|$\triangle$|`$\triangle$`
$\top$|`$\top$`|$\bot$|`$\bot$`
$\vdash$|`$\vdash$`|$\vDash$|`$\vDash$`
$\star$|`$\star$`|$\ast$|`$\ast$`
$\circ$|`\circ`|$\bullet$|`$\bullet$`


**注:**想要表达非的概念只需前加`\not`，会添加删除斜线，如:`$x\not=y$`显示为$x\not=y$，`$x\not\in y$`显示为$x\not\in y$

#### 其他
运算符|说明|应用举例|命令
:-:|:-:|:-:|:-:
\overbrace|上大括号|$\overbrace{a+\underbrace{b+c}_{1.0}+d}^{2.0}$|`$\overbrace{a+\underbrace{b+c}_{1.0}+d}^{2.0}$`
\underbrace|下大括号|$\underbrace{a+d}_3$|`$\underbrace{a+d}_3$`
\partial|偏导数|$\frac{\partial z}{\partial x}$|`$\frac{\partial z}{\partial x}$`
\ldots|底端对齐的省略号|$1,2,\ldots,n$|`$1,2,\ldots,n$`
\cdots|中线对齐的省略号|$1,2,\cdots,n$|`$1,2,\cdots,n$`
\uparrow|上箭头|$\uparrow$|`$\uparrow$`
\Uparrow|双上箭头|$\Uparrow$|`$\Uparrow$`
\downarrow|下箭头|$\downarrow$|`$\downarrow$`
\Downarrow|双下箭头|$\Downarrow$|`$\Downarrow$`
\leftarrow|左箭头|$\leftarrow$|`$\leftarrow$`
\Leftarrow|双左箭头|$\Leftarrow$|`$\Leftarrow$`
\rightarrow|右箭头|$\rightarrow$|`$\rightarrow$`
\Rightarrow|双右箭头|$\Rightarrow$|`$\Rightarrow$`

### 求和、极限与积分
#### 求和、求积
- 求和符号`$\sum$`显示为$\sum$，**注意**要用`\displaystyle`显示成文内公式的模样,或者使用`$$\sum$$`（这时样式为居中）
>举例:不加`\displaystyle`,`$\sum_{i=0}^n$`显示为$\sum_{i=0}^n$
>举例:加`\displaystyle`,`$\displaystyle\sum_{i=0}^n$`显示为$\displaystyle\sum_{i=0}^n$
>举例:`$$\sum_{i=0}^n$$`显示为$$\sum_{i=0}^n$$
- 求积符号`$\prod`显示为$\prod$
>举例:`$\displaystyle\prod_{i=0}^n$`显示为$\displaystyle\prod_{i=0}^n$

#### 集合
- 大交集`$\bigcap$`显示为$\bigcap$
> 举例:`$\displaystyle\bigcap_{i=0}^n$`显示为$\displaystyle\bigcap_{i=0}^n$
- 大并集`$\bigcup$`显示为$\bigcup$
> 举例:`$\displaystyle\bigcup_{i=0}^n$`显示为$\displaystyle\bigcup_{i=0}^n$

#### 极限
- 极限符号`$\lim$`显示为$\lim$
>举例: `$\displaystyle\lim_{x\to\infty}$`显示为$\displaystyle\lim_{x\to\infty}$

#### 积分
- 积分符号
{% tabs 积分%}
<!-- tab 效果 -->
$\int$
$\iint$
$\iiint$
$\oint$
<!-- endtab -->
<!-- tab 代码-->
{% code %}
$\int$
$\iint$
$\iiint$
$\oint$
{% endcode %}
<!-- endtab -->
{% endtabs %}
>举例:`$\int_0^\infty{fxdx}$`显示为$\int_0^\infty{fxdx}$

### 分式与根式
#### 分式
- `$\frac{公式1}{公式2}$`显示为$\frac{公式1}{公式2}$
>举例:`$\frac{b_i^2}{a_i^2}$`显示为$\frac{b_i^2}{a_i^2}$
- 连分式用`$\cfrac{公式1}{公式2}$`，样式与`$\frac{公式1}{公式2}$`略有不同
>举例: 连分式`$$x=a_0 + \cfrac {1^2}{a_1 + \cfrac {2^2}{a_2 + \cfrac {3^2}{a_3 + \cfrac {4^2}{a_4 + ...}}}}$$`显示为$$x=a_0 + \cfrac {1^2}{a_1 + \cfrac {2^2}{a_2 + \cfrac {3^2}{a_3 + \cfrac {4^2}{a_4 + ...}}}}$$
>举例: 连分式`$$x=a_0 + \frac {1^2}{a_1 + \frac {2^2}{a_2 + \frac {3^2}{a_3 + \frac {4^2}{a_4 + ...}}}}$$`显示为$$x=a_0 + \frac {1^2}{a_1 + \frac {2^2}{a_2 + \frac {3^2}{a_3 + \frac {4^2}{a_4 + ...}}}}$$

#### 根式
- `$\sqrt[x]{y}$`显示为$\sqrt[x]{y}$

### 特殊函数
- 语法:`$\函数名$`
>举例: `$\sin x$`,`$\ln x$`,`$\log_n^2$`,`$\max(A,B,C)$`显示为$\sin x$,$\ln x$,$\log_n^2$,$\max(A,B,C)$

### 矩阵
#### 基本语法
- 起始标记:`$\begin{matrix}`,结束标记:`\end{matrix}$`
- 每一行末尾标记`\\\`，行间元素之间以`&`分隔
{% tabs 矩阵%}
<!-- tab 效果 -->
$\begin{matrix}
1&0&0\\\
0&1&0\\\
0&0&1\\\
\end{matrix}$
<!-- endtab -->
<!-- tab 代码-->
{% code %}
$\begin{matrix}
1&0&0\\\
0&1&0\\\
0&0&1\\\
\end{matrix}$
{% endcode %}
<!-- endtab -->
{% endtabs %}

#### 矩阵边框
- 在起始、结束标记处用下列词替换**matrix**
{% tabs 矩阵边框%}
<!-- tab pmatrix -->
$\begin{pmatrix}
1&0&0\\\
0&1&0\\\
0&0&1\\\
\end{pmatrix}$
<!-- endtab -->
<!-- tab 小括号-->
{% code %}
$\begin{pmatrix}
1&0&0\\\
0&1&0\\\
0&0&1\\\
\end{pmatrix}$
{% endcode %}
<!-- endtab -->
<!-- tab bmatrix -->
$\begin{bmatrix}
1&0&0\\\
0&1&0\\\
0&0&1\\\
\end{bmatrix}$
<!-- endtab -->
<!-- tab 中括号-->
{% code %}
$\begin{bmatrix}
1&0&0\\\
0&1&0\\\
0&0&1\\\
\end{bmatrix}$
{% endcode %}
<!-- endtab -->
<!-- tab Bmatrix -->
$\begin{Bmatrix}
1&0&0\\\
0&1&0\\\
0&0&1\\\
\end{Bmatrix}$
<!-- endtab -->
<!-- tab 大括号-->
{% code %}
$\begin{Bmatrix}
1&0&0\\\
0&1&0\\\
0&0&1\\\
\end{Bmatrix}$
{% endcode %}
<!-- endtab -->
<!-- tab vmatrix -->
$\begin{vmatrix}
1&0&0\\\
0&1&0\\\
0&0&1\\\
\end{vmatrix}$
<!-- endtab -->
<!-- tab 单竖线-->
{% code %}
$\begin{vmatrix}
1&0&0\\\
0&1&0\\\
0&0&1\\\
\end{vmatrix}$
{% endcode %}
<!-- endtab -->
<!-- tab Vmatrix -->
$\begin{Vmatrix}
1&0&0\\\
0&1&0\\\
0&0&1\\\
\end{Vmatrix}$
<!-- endtab -->
<!-- tab 双竖线-->
{% code %}
$\begin{Vmatrix}
1&0&0\\\
0&1&0\\\
0&0&1\\\
\end{Vmatrix}$
{% endcode %}
<!-- endtab -->
{% endtabs %}

#### 省略元素
- 横省略号：`$\cdots$`
- 竖省略号：`$\vdots$`
- 斜省略号：`$\ddots$`
{% tabs 省略元素 %}
<!-- tab 效果 -->
$\begin{bmatrix}
{a_{11}}&{a_{12}}&{\cdots}&{a_{1n}}\\\
{a_{21}}&{a_{22}}&{\cdots}&{a_{2n}}\\\
{\vdots}&{\vdots}&{\ddots}&{\vdots}\\\
{a_{m1}}&{a_{m2}}&{\cdots}&{a_{mn}}\\\
\end{bmatrix}$
<!-- endtab -->
<!-- tab 代码-->
{% code %}
$\begin{bmatrix}
{a_{11}}&{a_{12}}&{\cdots}&{a_{1n}}\\\
{a_{21}}&{a_{22}}&{\cdots}&{a_{2n}}\\\
{\vdots}&{\vdots}&{\ddots}&{\vdots}\\\
{a_{m1}}&{a_{m2}}&{\cdots}&{a_{mn}}\\\
\end{bmatrix}$
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 阵列
- 需要array环境:起始、结束处以`$\begin{array}`、`\end{array}$`声明
- 对齐方式:在`{array}`后以`{}`逐行统一声明,左对齐:`l`；居中：`c`；右对齐：`r`
- 竖直线:在声明对齐方式时，插入`|`建立竖直线
- 插入水平线:`\hline`
- 换行: `\\\`，行间元素之间以`&`分隔
{% tabs 阵列 %}
<!-- tab 效果 -->
$\begin{array}{c|lll}
{\downarrow}&{a}&{b}&{c}\\\
\hline
{R_1}&{c}&{b}&{a}\\\
{R_2}&{b}&{c}&{c}\\\
\end{array}$
<!-- endtab -->
<!-- tab 代码-->
{% code %}
$\begin{array}{c|lll}
{\downarrow}&{a}&{b}&{c}\\\
\hline
{R_1}&{c}&{b}&{a}\\\
{R_2}&{b}&{c}&{c}\\\
\end{array}$
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 方程组
- 需要cases环境:起始、结束处以`$\begin{cases}`、`\end{cases}$`声明
- 换行：`\\\`,行间元素之间以`&`分隔
{% tabs 方程组 %}
<!-- tab 效果 -->
$\begin{cases}
a_1x+b_1y+c_1z=d_1\\\
a_2x+b_2y+c_2z=d_2\\\
a_3x+b_3y+c_3z=d_3\\\
\end{cases}
$
<!-- endtab -->
<!-- tab 代码-->
{% code %}
$\begin{cases}
a_1x+b_1y+c_1z=d_1\\\
a_2x+b_2y+c_2z=d_2\\\
a_3x+b_3y+c_3z=d_3\\\
\end{cases}
$
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 分段函数
- 与方程组定义方法类似
{% tabs 分段函数 %}
<!-- tab 效果 -->
$f(n)=
\begin{cases}
\cfrac n2, &if\ n\ is\ even\\\
3n + 1, &if\ n\ is\ odd\\\
\end{cases}$
<!-- endtab -->
<!-- tab 代码-->
{% code %}
$f(n)=
\begin{cases}
\cfrac n2, &if\ n\ is\ even\\\
3n + 1, &if\ n\ is\ odd\\\
\end{cases}$
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 多行表达式
{% tabs 多行表达式 %}
<!-- tab 效果 -->
$\begin{equation}\begin{split} 
a&=b+c-d \\\
&\quad +e-f\\\ 
&=g+h\\\
&=i\\\
\end{split}\end{equation}$
<!-- endtab -->
<!-- tab 代码-->
{% code %}
$\begin{equation}\begin{split} 
a&=b+c-d \\\ 
&\quad +e-f\\\ 
&=g+h\\\ 
&=i\\\
\end{split}\end{equation}$
{% endcode %}
<!-- endtab -->
{% endtabs %}


# 插入图片
语法说明：`![图片Alt](图片地址 "图片Title")`
{% tabs 图片%}
<!-- tab 效果 -->
![哆啦A梦](/images/duola.jpg "哆啦A梦")
<!-- endtab -->
<!-- tab 代码-->
{% code %}
!\[哆啦A梦\]\(/images/duola.jpg "哆啦A梦"\)
{% endcode %}
<!-- endtab -->
{% endtabs %}