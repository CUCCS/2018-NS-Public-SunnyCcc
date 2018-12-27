# chap0x12 实战Bro网络入侵取证

## 一、实验目的

利用bro,通过分析流量包中的extract file和log文件得到攻击主机的IP.

## 二、实验环境

- 安装bro

  ```
  apt-get install bro bro-aux
  ```

- 实验环境

  ```
  lsb_release -a   # 查看系统版本信息
  uname -a    # 查看系统名
  bro -v   # 查看bro版本信息
  ```

  ![](/ns-0x12/1.jpg)

- 配置bro

  1) 编辑`/etc/bro/site/local.bro`，在该文件末尾添加加两行代码

  ```
  @load frameworks/files/extract-all-files 
  # 提取所有文件
  @load mytuning.bro
  ```

  ![](/ns-0x12/2.jpg)

  2) 在`/etc/bro/site/`目录下创建名为`mytuning.bro`的新文件，写入`redef ignore_checksums = T;`忽略校验和认证。

  >原因：通常，Bro的事件引擎将丢弃没有有效校验和的数据包。如果想要在系统上分析本地生成/捕获流量，所有发送/捕获的数据包将具有不良的校验和，因为它们尚未由NIC计算，因此这些数据包将不会在Bro策略脚本中进行分析，所以要设置成忽略校验和验证。

  ![](/ns-0x12/3.jpg)

## 三、 实验过程

## 3.1 bro 分析

- 下载pcap包：执行命`wget https://sec.cuc.edu.cn/huangwei/textbook/ns/chap0x12/attack-trace.pcap`

![](/ns-0x12/4.jpg)

- 使用bro自动化分析下载的attack-trace.pcap包

```
bro -r attack-trace.pcap /etc/bro/site/local.bro
```

![](/ns-0x12/5.jpg)

​	出现一个新的文件夹`extract-files`和`conn.log`、`files.log`等日志文件。

![](/ns-0x12/6.jpg)

- 分析相关文件

进入`extract_files`文件夹，把`extract_files`文件夹里的文件上传至`VirusTotal`网站进行分析，发现匹配了一个历史扫描报告 ，该报告表明这是一个已知的后门程序。

![](/ns-0x12/7.jpg)

![](/ns-0x12/8.jpg)



- 逆向倒推，寻找入侵线索

阅读`usr/share/bro/base/files/extract/main.bro`源代码，可知文件名的最后一部分```FHUsSu3rWdP07eRE4lid```,就是`files.log`文件的文件的唯一标识id

![](/ns-0x12/9.jpg)

- 查看分析`files.lo`可知，该文件提取自FTP会话，文件名的最后一个-右侧的字符串FHUsSu3rWdP07eRE4l是files.log中文件的唯一标识

![](/ns-0x12/10.jpg)

- 查看分析`conn.log`，通过conn.log的会话标识匹配,我们发现该PE文件来自于IPv4地址为:98.114.205.102的主机

![](/ns-0x12/11.jpg)

## 3.2 Wireshark分析

- 使用tshark查看主机信息

```
# -r 读取本地文件
# -z 设置统计参数
# -q 只在结束捕获时输出数据，针对于统计类的命令非常有用
# -n 禁止网络对象名称解析
tshark -r /root/下载/attack-trace.pcap -z ip_hosts,tree -qn
```

![](/ns-0x12/12.jpg)

- 查看攻击者

使用tshark查看攻击者，发现攻击者为`98.114.205.102`

```
# 设置读取过滤表达式（read filter expression）,只查找SYN包
tshark -r attack-trace.pcap -R "tcp.flags==0x02" -n
```

![](/ns-0x12/13.jpg)

- 查询攻击者IP地址

![](/ns-0x12/14.jpg)

- 查看攻击持续时间

使用tshark查看攻击时间，发现攻击时间为16.22秒左右

```
capinfos attack-trace.pcap
```

![](/ns-0x12/15.jpg)



## 参考链接

[基于bro的计算机入侵取证实战分析](https://www.freebuf.com/articles/system/135843.html)

[2018-NS-Public-MrCuihi](https://github.com/CUCCS/2018-NS-Public-MrCuihi/blob/2fc582766696909b3f1eb8eeb04376b4b5c0871e/网络安全/chap0x12/chap0x12%20实战Bro网络入侵取证.md)

[2018-NS-Public-Lyc-heng](https://github.com/CUCCS/2018-NS-Public-Lyc-heng/blob/83fee5c1f48cb21a0cd26ffff7c215b2f320313f/ns_chap0x12/实验报告.md)

