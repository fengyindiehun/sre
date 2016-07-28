# 0x00 what is cgroup?

cgroups是Linux内核的一个功能,用来限制,控制与分离一个进程组群的资源（如CPU,内存,网络,磁盘输入输出等).

## 0x01 The begin of cgroups

这个项目最早是由Google的工程师在2006年发起（主要是Paul Menage和Rohit Seth），最早的名称为进程容器(process containers).在2007年时，因为在Linux内核中，容器（container）这个名词有许多不同的意义，为避免混乱，被重命名为cgroup，并且被合并到2.6.24版的内核中去.

## 0x02 feature

cgroups的一个设计目标是为不同的应用情况提供统一的接口,从控制单一进程(像nice)到操作系统层虚拟化(像opeNVZ，Linux-VServer，LXC)。cgroups提供：

* 资源限制：组可以被设置不超过设定的内存限制；这也包括虚拟内存.
* 优先化：一些组可能会得到大量的CPU[5] 或磁盘输入输出通量.
* 报告：用来衡量系统确实把多少资源用到适合的目的上.
* 分离：为组分离命名空间，这样一个组不会看到另一个组的进程、网络连接和文件.
* 控制：冻结组或检查点和重启动.

## 0x03 the position of cgroups in kernel

作为一个内核功能可以在下图看到其所在

![architectures](https://upload.wikimedia.org/wikipedia/commons/4/44/Linux_kernel_and_daemons_with_exclusive_access.svg)

# 0x10 The struct of cgroups

## 0x11 term

* *任务(Tasks)*：就是系统的一个进程.
* *控制组(Control Group)*：一组按照某种标准划分的进程,比如官方文档中的Professor和Student,或是WWW和System之类的,其表示了某进程组.Cgroups中的资源控制都是以控制组为单位实现.一个进程可以加入到某个控制组.而资源的限制是定义在这个组上.简单点说,cgroup的呈现就是一个目录带一系列的可配置文件.
* *层级(Hierarchy)*：控制组可以组织成hierarchical的形式,既一颗控制组的树(目录结构).控制组树上的子节点继承父结点的属性.简单点说,hierarchy就是在一个或多个子系统上的cgroups目录树.
* *子系统(Subsystem)*：一个子系统就是一个资源控制器,比如CPU子系统就是控制CPU时间分配的一个控制器.子系统必须附加到一个层级上才能起作用,一个子系统附加到某个层级以后,这个层级上的所有控制族群都受到这个子系统的控制.Cgroup的子系统可以有很多,也在不断增加中.

## 0x12 resource mangement

引用这个图片,尝试解释一下cgroup的资源划分的结构.不同颜色代表不同`group`对资源的划分,不过这个设计存在一些缺陷已经遭到`Tejun Heo`的[吐槽](https://lwn.net/Articles/484254/),某个任务归类方式的多样性导致了多个`Hierarchy`的正交,导致了进程管理的复杂.多个子系统的之间的交叉使用导致了管理的复杂,不过在一些cgroup套件里面使用配置文件转移一部这个问题的复杂度.

![cgroups2](https://i1.wp.com/duffy.fedorapeople.org/blog/designs/cgroups/diagram2.png)

## 0x13 subsystem

* blkio — 这个子系统为块设备设定输入/输出限制，比如物理设备（磁盘，固态硬盘，USB 等等）。
* cpu — 这个子系统使用调度程序提供对 CPU 的 cgroup 任务访问。
* cpuacct — 这个子系统自动生成 cgroup 中任务所使用的 CPU 报告。
* cpuset — 这个子系统为 cgroup 中的任务分配独立 CPU（在多核系统）和内存节点。
* devices — 这个子系统可允许或者拒绝 cgroup 中的任务访问设备。
* freezer — 这个子系统挂起或者恢复 cgroup 中的任务。
* memory — 这个子系统设定 cgroup 中任务使用的内存限制，并自动生成​​内存资源使用报告。
* net_cls — 这个子系统使用等级识别符（classid）标记网络数据包，可允许 Linux 流量控制程序（tc）识别从具体 cgroup 中生成的数据包。
* net_prio — 这个子系统用来设计网络流量的优先级
* hugetlb — 这个子系统主要针对于HugeTLB系统进行限制，这是一个大页文件系统。

# 0x20 The Base usage in CLI

## 0x21 installation

在使用`systemd`的系统里面`hierarchy`由其启动时自动创建,在`rhel6`系列中需要`yum install libcgroup`,如果是`Ubuntu`系列的话较新的版本是自带了.
查看cgroup文件系统的挂载:

    ➜  ~ mount -t cgroup
    cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
    cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
    cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
    cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
    cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
    cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
    cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpuacct,cpu)
    cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
    cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
    cgroup on /sys/fs/cgroup/net_cls type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls)
    
查看子系统:

    ➜  ~ lssubsys
    cpuset
    cpu,cpuacct
    memory
    devices
    freezer
    net_cls
    blkio
    perf_event
    hugetlb
    
## 0x22 configuration

Linux中把`cgroups`实现成了文件系统,所以对文件系统里面的特定文件进行操作就可以完成对`cgroup`的配置.

demo:

    int main(void)
    {
        int i = 0;
        for(;;) i++;
        return 0;
    }

配置:首先在某个子系统下面建立一个目录,其目录里面会自动创建与其相关的文件(文件名表示其意义);其次置具体参数到文件名;然后把要限制的进程PID放入`task`的文件.
    
    ➜  ~ mkdir /sys/fs/cgroup/cpu/eleme
    ➜  ~ ps uax | grep deadloop
    root       4260 59.0  0.0   4164   352 pts/0    RN   23:12   3:03 ./deadloop
    ➜  ~ echo "2000" >> /sys/fs/cgroup/cpu/eleme/cpu.cfs_quota_us
    ➜  ~ echo "4260" >> /sys/fs/cgroup/cpu/eleme/tasks

## 0x23 confirmation

配置前:

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    4434 root      25   5    4164    356    280 R  92.0  0.0   0:04.18 deadloop

配置后:
    
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    4434 root      25   5    4164    356    280 R   2.0  0.0   1:14.91 deadloop
    
### reference

[1] [redhat access](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Resource_Management_Guide/chap-Introduction_to_Control_Groups.html)
[2] [wikipedia](https://en.wikipedia.org/wiki/Cgroups)
[3] [coolshell](http://coolshell.cn/articles/17049.html)
