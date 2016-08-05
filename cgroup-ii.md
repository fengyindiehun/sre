# 0x00 net_cls and backgroud knowledge

这个子系统使用等级识别符（classid）标记网络数据包.

* 可用流量控制器(tc)从不同的`cgroup`中关联不同的优先级到数据包,这个场景多数是用于`QOS`相关.
* 也可以用`iptables`来对这些具有这些特征(-m cgroup)流量做一些具体的操作,这个操作幅度比`QOS`的要大,根据`iptables`的不同`table`提供的`-J`有很多动作可以做.


# 0x10 the queuing of traffic controller

IP服务模型是尽力而为,这样的模型不能体现某些流量的重要性,所以诞生了`QOS`技术,linux很早就提供了流量控制接口,命令行工具是`tc`.

## 0x11 Basic Queuing technology

在描述Linux QOS之前可以想象一下最基本的一个处理方式就是普通队列(FIFO),而后对其有了点改进诞生了由3个队列一起工作(pfifo),由多个队列一起工作的(Stochastic Fair Queuing)和拿着令牌才能走包的(token bucket filter).
需要补充说明的是虽然linux也支持基于字节的队列技术`bfifo`,但是`bfifo`在功能上要远逊色于`pfifo`.

>![pfifo](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/images/pfifo_fast-qdisc.png)
><center>接口默认队列技术的pfifo_fast</center>
>pfifo(基于packet的fifo)默认是使用三个队列,能提供基本的优先级功能.

>![SFQ](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/images/sfq-qdisc.png)
><center>看上去公平的sfq</center>
>sfq的公平是由hash与轮序算法保证的.更多信息[这里](http://wiki.mikrotik.com/wiki/Manual:Queue#SFQ)

>![tbf](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/images/tbf-qdisc.png)
><center>拿着令牌才放行的tbf</center>

## 0x12 hierarchy Queuing technology

**但是这样多数是不够的**.
比如"A,B,C,D四个服务,其中A是延迟敏感的视频会议,B是吞吐量大的bt下载,C,D普通的web流量",上面提供的这些功能或多或少只能满足一部分服务.这个时候一个层次化的划分队列被开发出来了,虽然linux也实现了其它的有类队列规则,但是他们远不如其中的HTB(Hierarchical Token Bucket)使用更加广泛.

![HTB](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/images/htb-borrow.png)
htb允许创建层次的队列结构与决定队列之间的关系(父-子,子-子).

使用一般步骤如下:

* 1: 匹配和标记流量: 将流量分类待使用,利用包含一个或者多个匹配参数来选择数据包构成一个class
* 2: 创建策略到标记的流量上: 把具体的class放到特定的队列中并定义每个class携带的动作.
* 3: 附加策略到到特定接口: 附加策略到全局接口(全局进,全局出,全局进出),特定接口或者父队列.

### htb demo:

    # This line sets a HTB qdisc on the root of eth0, and it specifies that the class 1:30 is used by default. It sets the name of the root as 1:, for future references.
    tc qdisc add dev eth0 root handle 1: htb default 30

    # This creates a class called 1:1, which is direct descendant of root (the parent is 1:), this class gets assigned also an HTB qdisc, and then it sets a max rate of 6mbits, with a burst of 15k
    tc class add dev eth0 parent 1: classid 1:1 htb rate 6mbit burst 15k

    # The previous class has this branches:

    # Class 1:10, which has a rate of 5mbit
    tc class add dev eth0 parent 1:1 classid 1:10 htb rate 5mbit burst 15k

    # Class 1:20, which has a rate of 3mbit
    tc class add dev eth0 parent 1:1 classid 1:20 htb rate 3mbit ceil 6mbit burst 15k

    # Class 1:30, which has a rate of 1kbit. This one is the default class.
    tc class add dev eth0 parent 1:1 classid 1:30 htb rate 1kbit ceil 6mbit burst 15k

    # Martin Devera, author of HTB, then recommends SFQ for beneath these classes:
    tc qdisc add dev eth0 parent 1:10 handle 10: sfq perturb 10
    tc qdisc add dev eth0 parent 1:20 handle 20: sfq perturb 10
    tc qdisc add dev eth0 parent 1:30 handle 30: sfq perturb 10
    
HTB使用信息[这里](http://luxik.cdi.cz/~devik/qos/htb/manual/userg.htm),更多理论信息[这里](http://luxik.cdi.cz/~devik/qos/htb/manual/theory.htm),实现信息[这里](http://lxr.free-electrons.com/source/net/sched/sch_htb.c).

# 0x20 netfilter & iptables

>Netfilter，在Linux内核中的一个软件框架，用于管理网络数据包。不仅具有网络地址转换（NAT）的功能，也具备数据包内容修改、以及数据包过滤等防火墙功能。

以上内容引自Wikipedia,netfilter做为一个内核框架,它可以在不同阶段将函数`hook`进网络栈,框架本身并不处理数据包.

>iptables，一个运行在用户空间的应用软件，通过控制Linux内核netfilter模块，来管理网络数据包的流动与转送。在大部分的Linux系统上面，iptables是使用/usr/sbin/iptables来操作，文件则放置在手册页（Man page[2]）底下，可以通过 man iptables 指令获取。

`iptables`做为一个用户态工具,提供一些术语(table,chain,match,target)准确描述了一些网络管理,这些术语离开`iptables`上下文可能意义不一样.

![netflow](https://upload.wikimedia.org/wikipedia/commons/thumb/3/37/Netfilter-packet-flow.svg/1450px-Netfilter-packet-flow.svg.png)
<center>浏览器放大看</center>

`iptables`默认提供四个`table`,不同的`table`内置了不同的`chain`,不同`chain`提供了不近相同的`target`.

* filter: 用于应用过滤规则.
* nat: 用于应用地址转化.
* mangle: 用于修改分组数据.
* raw: 想独立与netfilter链接跟踪子系统作用的规则应用于这.(可以在图里由`raw`表所在位置确认)

具体的的各个`chain`对数据包的处理流程可以参考上图,数据包先进入`raw`表的`preroute`链,而后进入`mangle`的`preroute`链,如此推理.

### iptables demo:

    iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080
    
把默认HTTP端口的数据包由80转向8080端口,在路由决策前被处理,而后进入`mangle`的`input`后,又进入`filter`的`input`交与socket.
`-t`参数后面认识是`table`,`-A`表示对后面的`chain`进行增加条目,在往后一些事`match`规则,`-j`后面就是`target`.
    

# 0x30 net_cls demo

对前面的`QOS`与`iptables`有了解,加上在[2014](https://git.netfilter.org/libnftnl/commit/?id=1d4a4808bb967532a30230f1957236586ab6f2b6)年`netfilter`支持了`cgroup`特性,用户态需要安装新的iptables,而后可以`match`出`cgroup`相关的流量,这个时候`net_cls`才算能和`netfilter`一起工作.

    #!/bin/sh

    INTERFACE=eno16777736

    # configuring tc:
    tc qdisc add dev $INTERFACE root handle 10: htb

    # creating traffic class 10:1 and configure the rate with 40mbit
    tc class add dev $INTERFACE parent 10: classid 10:1 htb rate 40mbit

    # filter special traffic
    tc filter add dev $INTERFACE parent 10: protocol ip prio 10 handle 1: cgroup

    # create new Hierarchy
    if [ ! -d /sys/fs/cgroup/net_cls/0 ]; then
        mkdir /sys/fs/cgroup/net_cls/0
    fi

    # send 0010:0001 to net_cls.classid
    echo 0x00100001 >  /sys/fs/cgroup/net_cls/0/net_cls.classid

    # use iptables to match cgroup traffic and drop it.
    iptables -A OUTPUT -m cgroup --cgroup 0x00100001 -j DROP

### reference

[1] [cgroup net_cls doc](https://www.kernel.org/doc/Documentation/cgroup-v1/net_cls.txt)
[2] [Linux taffic control](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/)
[3] [linux firewalls](https://www.nostarch.com/firewalls.htm)
[4] [MikroTik RouterOS packet flow](http://wiki.mikrotik.com/wiki/Manual:Packet_Flow)
[5] [classful qdiscs](https://wiki.archlinux.org/index.php/Advanced_traffic_control#Classful_Qdiscs)
[6] [HTB](http://wiki.mikrotik.com/wiki/Manual:HTB#Theory)
