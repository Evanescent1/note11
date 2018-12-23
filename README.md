# note11
实验：实现ip转发（在FORWARD转发做设置，在客户机做防火墙）
192.168.66.132 客户机（关闭桥接）
172.18.133.212服务器（关闭仅主机）不在一个网段
192.168.66.135和172.18.133.210网关  打开核心转发功能echo 1 > /proc/sys/net/ipv4/ip_forward
在客户机添加路由指向网关（要到达18网段，指向网关）主机路由<---网络路由<---默认路由
route   add -net   172.18.0.0/16  gw  192.168.66.135
删除路由route  del   -net  0.0.0.0
在服务器上添加路由
route   add  -net  192.168.0.0/16   gw 172.18.133.210
检测是否能够ping同，在服务器端打开一个服务，看是否能够访问
1需要在网关上(代表客户机端的网关）控制访问服务（控制客户端能否访问服务器,源确定，目标不确定，控制源端IP，只可以访问目标80端口的httpd服务）
设置请求报文（方法2可以省略，利用方法1）
方法1（只能发出的新连接可以出去，不允许外界的新连接进来,放在第二条规则）
iptables -I  FORWARD  -s 192.168.0.0/16  -p  tcp --dport  80  -m state  --state NEW  -j  ACCEPT
方法2（对应下面的编号）：iptables  -A  FORWARD -s  192.168.0.0/16  -p  tcp  --dport  80  -j   ACCEPT
设置响应报文  目标端口80确定
方法1（连接跟踪,放在第一条规则）：iptables  -I FORWARD -m  state  --state ESTABLISHED  -j  ACCEPT
方法2：iptables  -I  FORWARD  2  -d  192.168.0.0/16  -p tcp  --sport 80  -j   ACCEPT
拒绝别的访问服务（第五条规则）
iptables  -A  FORWARD   -j   REJECT

添加规则（可以访问ftp，dns)
修改第二条规则
iptables -R FORWARD 2 -s 192.168.0.0/16  -p  tcp -m multiport  --dports 80,21,53  -m state  --state NEW  -j  ACCEPT
添加upd协议的DNS(第三天规则）
iptables -I FORWARD  3 -s 192.168.0.0/16  -p udp --dport 53  -m  state --state  NEW  -j  ACCEPT
由于客户机主动访问，服务器端的ftp端口随机（需要添加链接追踪功能）第四条规则
启动模块 modprobe  nf_conntrack_ftp
iptables -I FORWARD  4 -s 192.168.0.0/16  -p tcp  -m state  --state RELATED  -j  ACCEPT
如果网关是主机，需要在INPUT做防火墙，防止客户机访问自己

在客户机端做地址转换(隐藏客户机地址，在客户机做地址网关转换，在发出链POSTROUTING做转发）
SNAT:静态地址转换--to-source,不做端口映射）-nat指定表
          动态地址（如果有几个网卡）：--random
          --perstistent：只要第一做了地址转换，会一直使用这个地址，不在做变化
静态地址，指定IP
1设置规则第一条（如果不在一个网络，做地址转换） !  -d  192.168.0.0/16取反，目标不是这个网络做转发
iptables  -t  nat表  -A POSTROUTING  -s  192.168.0.0/16  !  -d  192.168.0.0/16  -j  SNAT  --to-source    172.18.133.210
2可以在FORWARD转发做过滤设置（设置ping速率）第二条规则
iptables -I  FORWARD  -s 192.168.0.0/16  -p  icmp  -m state --state NEW  -j  ACCEPT
设置回应报文（第一条）
iptables  -I  FORWARD  -m  state  --state  ESTABLISHED  -j  ACCEPT
插入第二条规则（设置可以访问httpd）
iptables -I  FORWARD -s 192.168.0.0/16   -p tcp  --dport 80 -m  state  --state NEW -j ACCEPT

在客户器的防火墙做地址伪装（实现动态地址转换,自动选择一个地址转换）
MASQUERADE.只能使用在POSTROUTING链上
设置规则
iptables  -t  nat表  -A POSTROUTING  -s  192.168.0.0/16  !  -d  192.168.0.0/16  -j  MASQUERADE


DNAT服务器端实现IP转换（在服务器网关做设置，改目标地址，让所有客户机访问服务器，显示都是网关的ip）
端口映射;--to-destination  要改的目标地址,可以指定端口port
                 --random
                 --persistent
修改PREROUTING链上（进入网关的目标改变成服务器的ip)
iptables -t nat -A   PREROUTING  -d公网地址  192.168.66.135  -p tcp  --dport  80  -j  DNAT  --to-destination  172.18.133.212
修改服务器端端口/etc/httpd/conf/httpd.conf
8080   42行
修改规则（端口映射）
iptables -t nat -I   PREROUTING  1  -d公网地址  192.168.66.135  -p tcp  --dport  80  -j  DNAT  --to-destination  172.18.133.212:8080

在服务器端做端口转换（在服务器主机本身做端口,不要在网关做）
REDIRECT端口映射 在PREROUTING链
iptables  -t  nat  PREROUTING  -d  172.18.133.212  -p  tcp  --dport 80  -j  REDIRECT  --to-port   8080


规则优化（服务器使用黑名单，客户器使用白名单）
1匹配严格放在前面，使用频率高的
3设置默认策略
最后一条规则设定（建议使用，不依赖本地策略）
默认策略设定

RAM编址存储单元（内存）
电子数字计算机
数据：
程序：指令+数据，指令的控制逻辑
程序：算法+数据结构
过程式编程：以算法为核心，数据服务算法
对象式编程：以数据为中心，算法服务数据
shell;命令的堆积
把数据组成特定的结构
程序执行在CPU上，CUP负责向RAM上加载指令，在cpu之上有电路规划好的计算指令，程序调用指令，cpu是运行指令的。指令放在内存当中（ram),指令是加工数据的，引用数据的，引用数据为直接数据，间接数据（变量），变量是内存地址。
内存有内存空间引用方式，每一个存储单元都有独立的标识，平面化的存储结构，一个整体空间。
一共有32根存储线路（电子通信线路）标识存储单元，线路只能标识两种状态，有电1和无电0
每一存储单元是字节byte。2的32次方个存储单元（内存）
1024byte=1k  1024k=1M  1024M=1G    1024x1024x1024x4=4G
64根总线就是4G的4G，40多亿个4G空间
由于内存空间不同，程序不能在不同主机上找到相应的存储单元，没有通用性，利用虚拟内存概念（超配），程序编好之后，都相当于自己的程序拥有4G内存空间，程序代码会调用自己的认为的虚拟内存的空间存储地址（线性地址），当需要存储数据时，由于物理地址，和程序调用的虚拟存储的空间的地址可能不一致，会使用映射的方式实现，每一存储数据的空间称为叶匡page，由内核把虚拟地址转为物理地址的某个地址。
操作系统把一份资源虚拟成多份，让多个程序同时运行，利用内核让计算机虚拟化，内核把cpu时间切片，利用时序复用的方式，每一个时间片，cpu运行的程序不同，
把内存空间也进行切片，由内核以超配的方式分给进程，利用程序使用的实际数据叶匡进行分配。把一段空间按照一定的大小进行切割分配，空间复用
内核中保存的数据的元数据，进程的真正物理空间数据，属性信息

在操作系统中分层级结构，多级路由实现
内存有40多亿个存储单元，内存做多级路由，实现映射关系快速实现，cpu引进了MMU（内存管理单元），mmU地址转换是cpu的一个芯片，把地址查找表装入mmu，由于它是硬件设施，可以快速查找关系映射，程序运行是由局部性的（热区），时间局部性和空间局部性，可以使用缓存进行加速数据访问，mmu的缓存区TLB转换后援缓冲器。
磁盘虚拟化：一个硬盘由多个程序进行访问，文件系统把磁盘分成更细更小的存储单元，利用文件，
网卡虚拟化：千兆网卡，一秒钟可以发送一千个数据位。把1秒切分成多个多个单元，每一个时间的偏转都能发送一个数据出去，划分的越精细对外在同样的时间内传送的数据越多，
利用电压发送信号，在每一个时间偏转，如果线路有电表示1，没电表示0，利用的电压持续的时间长短，发出信号不同。当进程发数据，把数据放到网卡的发送队列，网卡频率越高，工作效率越高

cpu的指令分两类
普通指令集：算数  环3
特权指令集：关机   环0
内核就是为使用环0而设计的，程序只能利用环3指令，如果需要使用特权指令，通过内核封装

在宿主机上安装虚拟机，在虚拟机上安装操作系统，
虚拟机用户空间--->虚拟内核---> 虚拟cpu-->宿主机内核--->宿主机cpu
BT二进制转换：添加了环复1

半虚拟化：内核直接调用虚拟软件  para-virt
完全虚拟化：full-virt  硬件辅助实现   纯软件模拟
cmulation模拟器：模拟环0和环3

三级虚拟化：底层内核，用户空间，在用户空间上跑个软件，在软件上运行了虚拟机，虚拟机用自己的内核和用户空间，在虚拟上的进程映射了虚拟机的物理地址，而虚拟机的地址，是被宿主机已经虚拟过的线性地址，

嵌套的页表NPT，硬件支持的mmu
扩展的页表EPT,硬件支持的mmu
带标签的TLB：需要添加虚拟机的先线性地址到物理地址的转换

虚拟机把宿主机上的一个文件当硬盘用，需要驱动文件，用软件的方式模拟出驱动器

让虚拟机使用真正的硬盘，IO透传，
 云计算，虚拟工作在操作系统上，而提供物理地址的主机随机
虚拟机迁移：真正的文件放在一个单独的共享服务器，在不同的物理机操作
宿主机的内核有一个模块，桥接模块，
二层设备，纯软件实现的
同一个宿主机上跑多个虚拟机，每个虚拟机在不同的网段，需要在宿主机打开核心转发功能，充当路由，
内核资源的虚拟化，可以协议栈虚拟多个，

内核拆分模拟空间：
网络net    mount根文件系统，pid  uts主机名   ipc进程间的通信   user用户

查看网络名称空间：ip netns  ls  模拟路由器使用
在名称空间执行命令：ip netns exec router1 ifconfig
添加地址激活：ip netns exec router1 ifconfig lo 127.0.0.1/8 up

brctl 管理软交换机
查看当前的交换机brctl show
添加交换机brctl addbr virbr

添加虚拟以太网网卡
ip link add veth1.1 type veth peer name veth1.2

把网卡的一头添加到网络名称空间
p link set veth1.1 netns router1
把另一头连接交换机（如果用网卡连接，交换机不能有地址的）
brctl addif virbr  veth1.2
取消绑定
brctl delif virbr veth1.2

给网络名称空间的网卡改名
ip netns exec router1 ip link set veth1.1 name eth0
添加ip并激活
ip netns exec router1 ifconfig eth0 10.0.0.1/24 up

给另一个网卡添加地址并激活
ifconfig veth1.2 10.0.0.2/24 up

添加一个路由（路由是三层设备，可以有地址）
ip link set veth1.2 router2
把网卡绑定
p link set veth1.2 netns router2

要与宿主机外部网络连接
1vnet1  虚拟的空间的交换机有个DNS，只为虚拟机提供地址
2vnet8
仅主机：仅能与宿主机通信
nat：在宿主机打开核心转发，所有来自私网的，要访问以外的网络，都通过SNAT转发出去，
桥接：把物理机的网卡当交换机用，虚拟的交换当网卡用

通过nat机制，两个物理的间的虚拟机连接，地址是不可见的
解决方案
1把物理网卡与虚拟机的网卡绑定，所有到物理网卡的报文都转发给它，通过nat隐藏机制，两个虚拟主机都会认为访问是对端的物理网卡的地址
2把一个大网划分多个子网，每个子网给每个宿主机上的虚拟机使用，在每个宿主机上打开隧道接口，把虚拟机要发出的报文，封装一层另一个宿主机的ip地址，当另个宿主机收到报文，拆封装，看到里面的报文，进而交给宿主机上的虚拟机，用一个网络承载另一个网络，称为叠加网络（隧道机制）

主机级虚拟化：OpcnVSwitch  
实现软件：xen安装之后，重启会代替原来的宿主机 ，宿主机会变成虚拟机（特权虚拟机），
                KVM：模块，一旦安装，变成虚拟机
                vmware 类型2
                  vmwareFSX类型1
                 hyperV
类型2：宿主上的内核，内核之上跑虚拟机
类型1：硬件之上直接跑虚拟化软件，软件上跑多个虚拟机

容器级：在内核之上创建多个用户空间，内核内部把各种资源切成N份，为各个用户空间服务，互相之间隔离，每一个用户空间称为容器

程序级虚拟化：php运行在解释器上

云操作系统，统一管理虚拟机的
CLoudOS：laaS主机级
        OpenStack
         CloudStack
CloudOS(PaaS）容器级管理系统
          kubernctes---> OpenShift(k8s的二次发行版）
          docker compose  swarm  machinc
         Mesos,Marathon

KVM
1只支持x86_64
2HVM硬件级虚拟化（网络，硬盘）cpu必须支持硬件虚拟化
内核中的模块，虚拟化软件
qemu模拟各种cpu，

在虚拟机中创建虚拟机（虚拟机嵌套）
添加选项

查看是否支持虚拟化vmx|svm必须有一个
grep -i -E '(vmx|svm|lm)' /proc/cpuinfo
要支持KVM
启动模块nodprobe kvm
查看模块是否装入lsmod  | grep kvm
安装yum   install qemu-kvm
使用virt-manager管理KVM
安装yum  install qemu-kvm  libvirt-daemon-kvm守护进程   libvirt  virt-manager图形化
启动守护进程systemctl  start  libvirtd.service
创建桥接网卡：cp  ens33   br0
vim   ens33(当交换机）
删除网关，掩码，路由，dns
DEVICE=ens33
NAME=ens33
ONBOOT=yes
NETBOOT=yes
IPV6INIT=no
BOOTPROTO=none
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
IPV4_FAILURE_FATAL=no
BRIDGE=br0 

vim  br0(当网桥）
DEVICE=br0
NAME=br0
ONBOOT=yes
NETBOOT=yes
IPV6INIT=no
BOOTPROTO=none
TYPE=Bridge
PROXY_METHOD=none
BROWSER_ONLY=no
IPV4_FAILURE_FATAL=no
重启服务
查看getenforce必须是Disabled
运行virt-manager
出现图形化
网络引导---》类型Linux  版本centos7.0--->





安装yum  groupinstall "GNOME  Desktop"
