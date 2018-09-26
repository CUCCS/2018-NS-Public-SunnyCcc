### 基于VirtualBox的网络攻防基础环境搭建实例讲解
#### 一. 实验要求
- 节点：靶机、网关、攻击者主机
- 连通性
    - [ ] 靶机可以直接访问攻击者主机
    - [ ] 攻击者主机无法直接访问靶机
    - [ ] 网关可以直接访问攻击者主机和靶机
    - [ ] 靶机的所有对外上下行流量必须经过网关
    - [ ] 所有节点均可以访问互联网
- 其他要求
    - [ ] 所有节点制作成基础镜像（多重加载的虚拟硬盘）
#### 二. 实验环境
- 节点：
    - 靶机名称：**windowscookie**(cn_windows_10_enterprise_x64_dvd_6846957.iso)
    - 网关名称：**ubuntucookie**（ubuntu-18.04-live-server-amd64.iso）
    - 攻击者主机名称：**kalicookie**(kali-linux-2018.3-amd64.iso)

- 三台虚拟机网络配置：
    - 靶机：
        ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCEea5cafb7bab8ede30bb3fa992823c0c8/21264)
        ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE8cbff10db825e3bb1c3512ea5f42e368/21260)
    - 攻击者：
        ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE19f33b9bea4e6921e63bf660b70b9ade/21267)
        ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE5734715993566ee575fba68027827976/21262)
    - 网关：
        ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCEcdc6b833a3a5d35685df54424a2c4a14/21265)
        ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE1cbc6d32d7426ce108a518f008fc8443/21228)
- 最终网络拓扑图：
    ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE9fe3ea2b18e121120b5b5b98a26c0b75/21474)
- 三台主机增强功能：均配置（实验问题一）
#### 三. 实验过程
1. 靶机ping攻击者：
    - 实验原理：靶机与网关处于同一内网，网关与攻击者处于同一外网；靶机发送给攻击者的数据包通过网关转发给攻击者。
    - 实验过程：
        - 根据实验原理可以实现靶机ping攻击者，但是实际实验过程中发现靶机ping攻击者不通问题（实验问题二）。此时监听一下经过网关的流量发现网关没有将经过内网网卡（enp0s8)的流量通过外网网卡（enp0s3)传递出去。
            ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE1668853cd5938236df360fbdb7b6471c/21089)
            ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE397d94c8a266bb4db02a10f13b47b246/21099)
            ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/F92D20AE69B746F88026A1CD7CE9AC9A/21344)
        - 解决实验问题二: 开启网关IPV4转发功能，保存，将修改写入防火墙
            - 出于安全考虑，Linux系统默认是禁止数据包转发的。所谓转发即当主机拥有多于一块的网卡时，其中一块收到数据包，根据数据包的目的ip地址将数据包发往本机另一块网卡，该网卡根据路由表继续发送数据包。这通常是路由器所要实现的功能。

            -  检查：检查/proc下的文件发现ipv4转发没有开启 (值为 0)：
            
                ```
                    cat /proc/sys/net/ipv4/ip_forward 
                ```
                ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE54f8a47ae628f345d8ddfde3f500e4d0/21070)
            - 开启：通过sysctl可以开启ipv4的转发功能：
                ```
                    sysctl -w net.ipv4.ip_forward=1
                ```

            - 再次检查：：ipv4转发开启 (值为 1)：
                ```
                    cat /proc/sys/net/ipv4/ip_forward 
                ```
                ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE7b0a0fbedcb30cc39b576ddc230fed8e/21066)
            - 保存：修改/etc/sysctl.conf：/etc/sysctl.conf:
                ```
                    net.ipv4.ip_forward = 1 
                ```
                ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE44b97820769a95ba071472feaad6a27e/21086)
            
        - 开启网关IPV4转发后靶机依然ping不通攻击者，此时监听一下经过网关的流量，发现实验问题二解决后，网关可以将经过内网的流量通过外网传递出去，但是进一步监听经过攻击者的流量，发现攻击者虽然回复了request，但是回复的目的地址直接是靶机的IP地址，而攻击者和靶机处于不同网段，无法直接实现包转发（实验问题三）
        - 解决实验问题三：配置防火墙并保存配置

            ```
                iptables -t nat -A POSTROUTING -s 192.168.68.0/24 -o enp0s3 -j MASQUERADE
            ```

            ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE2695c9def6734864e294aa8b3aef5d72/21206)
            ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE6168eb6a834b2a4259591c205471144d/21318)
            ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE28e7945d789ec38fde4e27728e313f09/21220)
        - 再次使靶机ping攻击者，发现可以ping通； 打开attacker,gateway监听
            - gateway监听：
                ```
                tcpdump -n -i enp0s8 icmp
                tcpdump -n -i enp0s3 icmp
                ```
            - attacker监听：
    
                ```
                tcpdump -n -i eth0 icmp
                ```
            ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCEeafecb4dc50ffb32ffc5069cc834ed48/21208)
            ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE645360f0f86aa5ff80e9327cbd3689c1/21209)
            ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE06cdaa6b6e399412d7aead1487115b43/21212)
#### 四. 实验结果
- [x] 靶机可以直接访问攻击者主机
    - 由前文实验过程可知
- [x] 攻击者主机无法直接访问靶机

    ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCEc3339ce9bffc513436a8f49f01a597bd/21214)
- [x] 网关可以直接访问攻击者主机和靶机
    ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCEe077ae7f7f459660a4a8b739bf40966e/21384)
    ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE66b8080f8787e66c4707106a59c7ce67/21385)
- [x] 靶机的所有对外上下行流量必须经过网关
    - 由前文实验过程可知
- [x] 所有节点均可以访问互联网
    ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE3de0f83a22d6cd59d6147f71601b23a2/21394)
    ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE25bdf0647d3c7ba3cfdc29acfab5861a/21395)
    ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE034c72e2111ed098565032f42459f95b/21432)
- [x] 所有节点制作成基础镜像（多重加载的虚拟硬盘）    
    ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE337ed98248d4e7ec8a083885133c2bf6/21484)
    ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCEe783b34ec70cd790deeb7200b06194ad/21483)
    ![image](https://note.youdao.com/yws/public/resource/d3d5cd790cd89d5053dea9f779cc7154/xmlnote/WEBRESOURCE2f8ab32c85f4ce9b907e16757a16eb78/21482)
    
### 五. 实验总结
- Virtualbox的一个高级功能——多重加载，他不仅可以有多个加载点，而且还可以将虚拟镜像文件和加载点文件复制到其他电脑里直接创建一个虚拟机（前提是那台机器也安装了virtualbox），不用再一次安装系统。
- Tcpdump 的用法：
    - 以数字显示主机及端口：tcpdump -n（协议的关键字，主要包括fddi,ip,arp,rarp,tcp,udp等类型。Fddi指明是在FDDI(分布式光纤数据接口网络)上的特定 的网络协议，实际上它是"ether"的别名，fddi和ether具有类似的源地址和目的地址，所以可以将fddi协议包当作ether的包进行处理和 分析。其他的几个关键字就是指明了监听的包的协议内容。如果没有指定任何协议，则tcpdump将会监听所有协议的信息包。）

- 在Gateway中使用iptables配置转发规则并保存
    [参考链接1](https://www.cnblogs.com/frankb/p/7427944.html)    [参考链接2](https://blog.csdn.net/xftony/article/details/80584251)
    ```
    iptables -t nat -A POSTROUTING -s 192.168.68.0/24 -o ens0p3 -j MASQUEEADE
    ```

    >- nat表：修改数据包中的源，目标IP地址或端口（端口转发: 当内网主机对外提供服务时，由于使用的是内部私有IP地址，外网无法直接访问。因此，需要在网关上进行端口转发，将特定服务的数据包转发给内网主机。）
    >- -A, --append chain rule-specification：追加新规则于指定链的尾部； 
    >- POSTROUTING链:在进行路由选择后处理数据包(
        1、目的地址是本地，则发送到INPUT，让INPUT决定是否接收下来送到用户空间，
     即：PREROUTIGN--->INPUT;  
        2、**若满足PREROUTING的nat表上的转发规则，则发送给FORWARD，然后再经过POSTROUTING发送出去，
     即：PREROUTING--->FRORWARD--->POSTROUTING**  )
    >- -s:制定数据包的源地址， IP hostname
    >- SNAT：源 NAT，解决私网用户用同一个公网 IP 上网的问题。
MASQUERADE：是 SNAT 的一种特殊形式，适用于动态的、临时会变的 IP 上。
- 实验问题一**kali上安装增强功能**：已解决；之前执行`apt update`没有反应，推测是源的问题，这一次更换了源之后再一次执行，成功安装上增强功能。
    
## 六. 参考实验报告
- 2018-NS-Public-[TheMasterOfMagic](https://github.com/CUCCS/2018-NS-Public-TheMasterOfMagic/blob/62f9a992ca432c05c7e11436695f2bd4403ed85d/ns/chap0x01/基于VirtualBox的网络攻防基础环境搭建.md)
    