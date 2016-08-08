# 0x00 起因

昨天早上我还在吃早餐,老大对我讲我们的服务器挂了,kernel在临死前留下了一个dump.

# 0x10 尸检

然后,尸检的活让我来!

## 0x11 kernel version

确认一下尸体信息,以及死因.

```
      KERNEL: /usr/lib/debug/lib/modules/3.10.0-229.el7.x86_64/vmlinux
    DUMPFILE: vmcore  [PARTIAL DUMP]
        CPUS: 24
        DATE: Wed Aug  3 10:10:42 2016
      UPTIME: 95 days, 17:54:10
LOAD AVERAGE: 0.13, 0.13, 0.14
       TASKS: 544
     RELEASE: 3.10.0-229.el7.x86_64
     VERSION: #1 SMP Fri Mar 6 11:36:42 UTC 2015
     MACHINE: x86_64  (2394 Mhz)
      MEMORY: 95.6 GB
       PANIC: "divide error: 0000 [#1] SMP "
```

kernel是centos的`3.10.0-229`,除0导致了死亡.

## 0x12 log

知道了死于除0,利用log看一下凶手谁.

```
PID: 0      TASK: ffff88183d27e660  CPU: 19  COMMAND: "swapper/19"
 #0 [ffff88187fce3a90] machine_kexec at ffffffff8104c681
 #1 [ffff88187fce3ae8] crash_kexec at ffffffff810e2222
 #2 [ffff88187fce3bb8] oops_end at ffffffff8160d188
 #3 [ffff88187fce3be0] die at ffffffff810173eb
 #4 [ffff88187fce3c10] do_trap at ffffffff8160c860
 #5 [ffff88187fce3c60] do_divide_error at ffffffff81013f7e
 #6 [ffff88187fce3d10] divide_error at ffffffff816160ce
    [exception RIP: intel_pstate_timer_func+376]
    RIP: ffffffff814a9d28  RSP: ffff88187fce3dc8  RFLAGS: 00010206
    RAX: 0000000027100000  RBX: ffff880c3b059e00  RCX: 0000000000000000
    RDX: 0000000000000000  RSI: 0000000000000010  RDI: 000000002e5361f0
    RBP: ffff88187fce3e28   R8: ffff88183ca08038   R9: ffff88183ca08001
    R10: 0000000000000002  R11: 0000000000000005  R12: 000000000000513f
    R13: 0000000000271000  R14: 000000000000513f  R15: ffff880c3b059e00
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #7 [ffff88187fce3e30] call_timer_fn at ffffffff8107e046
 #8 [ffff88187fce3e68] run_timer_softirq at ffffffff8107fecf
 #9 [ffff88187fce3ee0] __do_softirq at ffffffff81077bf7
#10 [ffff88187fce3f50] call_softirq at ffffffff8161635c
#11 [ffff88187fce3f68] do_softirq at ffffffff81015de5
#12 [ffff88187fce3f80] irq_exit at ffffffff81077f95
#13 [ffff88187fce3f98] smp_apic_timer_interrupt at ffffffff81616fd5
#14 [ffff88187fce3fb0] apic_timer_interrupt at ffffffff8161569d
--- <IRQ stack> ---
#15 [ffff880c3db4bde8] apic_timer_interrupt at ffffffff8161569d
    [exception RIP: native_safe_halt+6]
    RIP: ffffffff81052dd6  RSP: ffff880c3db4be90  RFLAGS: 00000286
    RAX: 00000000ffffffed  RBX: ffffffff8109b938  RCX: 0100000000000000
    RDX: 0000000000000000  RSI: 0000000000000000  RDI: 0000000000000046
    RBP: ffff880c3db4be90   R8: 0000000000000000   R9: 0000000000000000
    R10: 0000000000000004  R11: 0000000000000005  R12: 0000000085099e00
    R13: 0000000000000013  R14: 00000002ed0ef7c8  R15: 001d63c002b695c0
    ORIG_RAX: ffffffffffffff10  CS: 0010  SS: 0018
#16 [ffff880c3db4be98] default_idle at ffffffff8101c93f
#17 [ffff880c3db4beb8] arch_cpu_idle at ffffffff8101d236
#18 [ffff880c3db4bec8] cpu_startup_entry at ffffffff810c6955
#19 [ffff880c3db4bf28] start_secondary at ffffffff810423ca
```

可以看见`intel_pstate_timer_func`函数直接导致了死亡,后面开始了`kdump`收尸.

## 0x13 backtrace

既然能大体确认凶手了,下面尝试看一下犯罪现场,利用`bt`的`rip`值`ffffffff814a9d28`找一下代码.

```
/usr/src/debug/kernel-3.10.0-229.el7/linux-3.10.0-229.el7.x86_64/drivers/cpufreq/intel_pstate.c: 47
0xffffffff814a9d15 <intel_pstate_timer_func+357>:	movslq %r12d,%r14
/usr/src/debug/kernel-3.10.0-229.el7/linux-3.10.0-229.el7.x86_64/drivers/cpufreq/intel_pstate.c: 52
0xffffffff814a9d18 <intel_pstate_timer_func+360>:	movslq %r13d,%rax
/usr/src/debug/kernel-3.10.0-229.el7/linux-3.10.0-229.el7.x86_64/drivers/cpufreq/intel_pstate.c: 605
0xffffffff814a9d1b <intel_pstate_timer_func+363>:	shl    $0x8,%rdx
/usr/src/debug/kernel-3.10.0-229.el7/linux-3.10.0-229.el7.x86_64/drivers/cpufreq/intel_pstate.c: 52
0xffffffff814a9d1f <intel_pstate_timer_func+367>:	shl    $0x8,%rax
/usr/src/debug/kernel-3.10.0-229.el7/linux-3.10.0-229.el7.x86_64/include/linux/math64.h: 29
0xffffffff814a9d23 <intel_pstate_timer_func+371>:	movslq %edx,%rcx
/usr/src/debug/kernel-3.10.0-229.el7/linux-3.10.0-229.el7.x86_64/include/linux/math64.h: 30
0xffffffff814a9d26 <intel_pstate_timer_func+374>:	cqto   
0xffffffff814a9d28 <intel_pstate_timer_func+376>:	idiv   %rcx
```
结合backtrace发现`rcx`是0且`0xffffffff814a9d28`处指令`idiv   %rcx`,kernel发生了除0异常.
且还可以初步断定问题是:`drivers/cpufreq/intel_pstate.c`中的函数调用了`include/linux/math64.h`30行的指令.

## 0x14 cause

到底死亡背后的原因是什么?
`3.10.0-229.el7`对应着源码包是`3.10.0-229.el7.src.rpm`,并不能对应着`upstream`代码直接看,因为并不清楚3.10的哪个小版本.
其中对应着`intel_pstate_timer_func()`的实现,且还利用的log的堆栈找出来inline函数的调用次序.

```
static void intel_pstate_timer_func(unsigned long __data)
{
	struct cpudata *cpu = (struct cpudata *) __data;
	struct sample *sample;
	intel_pstate_sample(cpu);
	sample = &cpu->sample;
	intel_pstate_adjust_busy_pstate(cpu);  // 这里! 看函数名猜测是判断busy状态
    ...
}

```

```
static inline void intel_pstate_adjust_busy_pstate(struct cpudata *cpu)
{
	int32_t busy_scaled;
	struct _pid *pid;
	signed int ctl;
	pid = &cpu->pid;
	busy_scaled = intel_pstate_get_scaled_busy(cpu); // 获取相关状态
    ...
}

```

```
static inline int32_t intel_pstate_get_scaled_busy(struct cpudata *cpu)
{
	int32_t core_busy, max_pstate, current_pstate, sample_ratio;
	u32 duration_us;
	u32 sample_time;
    ...
	duration_us = (u32) ktime_us_delta(cpu->sample.time,
					   cpu->last_sample_time);
	if (duration_us > sample_time * 3) {
		sample_ratio = div_fp(int_tofp(sample_time), // 死前一刀,int_tofp始duration_us是0.
				      int_tofp(duration_us));
		core_busy = mul_fp(core_busy, sample_ratio);
	}

	return core_busy;
}
```

这个时候贴出上下文(省略掉不相关的):

```
crash> struct cpudata ffff88187fce3e28
struct cpudata {
    ...
  last_sample_time = {
    tv64 = -131888820469800
  },
  prev_aperf = 18446744071588294792,
  prev_mperf = 321,
  sample = {
    core_pct_busy = 2144223048,
    aperf = 18446744071579335671,
    mperf = 18446612184889081816,
    freq = 270540864,
    time = {
      tv64 = 12567117954
    }
  }
}
```

结合`duration_us = (u32) ktime_us_delta(cpu->sample.time, cpu->last_sample_time)`与上下文,计算采样间隔是131901387587754ns,进行(u32)类型转换也溢出.
后面`int_tofp`实现为`#define int_tofp(X) ((int64_t)(X) << 8)`,所以`int_tofp(duration_us))`调用导致其变成`duration_us`变成0.又因为`div_fp`的调用,所以crash出现了,但是`last_sample_time`已经溢出,可能还有一些bug没有暴露出来.

```
static inline int32_t div_fp(int32_t x, int32_t y)
{
	return div_s64((int64_t)x << FRAC_BITS, y);
}

```
出于完整性考虑,这里贴出了inline函数的调用次序:

```
/**
 * div_s64 - signed 64bit divide with 32bit divisor
 */
#ifndef div_s64
static inline s64 div_s64(s64 dividend, s32 divisor)
{
	s32 remainder;
	return div_s64_rem(dividend, divisor, &remainder); // 捅出了那一刀
}
#endif
```

到这里基本就和log显示调用栈一致了,代码在math.h的30行.

```
/**
 * div_s64_rem - signed 64bit divide with 32bit divisor with remainder
 */
static inline s64 div_s64_rem(s64 dividend, s32 divisor, s32 *remainder)
{
	*remainder = dividend % divisor;
	return dividend / divisor; // 30行,死亡!
}

```

google一下,看见了曾经有人遇到了这个问题:

>The kernel may delay interrupts for a long time which can result in timers
being delayed.  If this occurs the intel_pstate driver will crash with
a divide by zero error

# 0x20 背景与解决方案

## 0x21 背景

这个驱动是intel为"SandyBridge+"微架构CPU提供的调整频率的控制接口,[更多](https://www.kernel.org/doc/Documentation/cpu-freq/intel-pstate.txt)信息.

## 0x22 解决方案

通过Google找到这个bug已经存在[patch](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/drivers/cpufreq/intel_pstate.c?id=7180dddf7c32c49975c7e7babf2b60ed450cb760)的,所以最好方案是升级内核,centos升级到3.10.0-229.20.1.el7.

有个临时的解决方案是**禁用pstate**.

>
* pstate 的新的功率驱动程序将会在以下的驱动程序之前自动为现代的 Intel CPU 启用。该驱动会优先于其他的驱动程序，因为它是内置驱动，而不是作为一个模块来加载。该驱动自动作用于 Sandy Bridge 和 Ivy Bridge 这两个类型的 CPU。如果您在使用这个驱动的时候遇到问题，建议您在 Grub 的内核参数中对其禁用（即修改 `/etc/default/grub` 文件，在 `GRUB_CMDLINE_LINUX_DEFAULT=` 后添加 `intel_pstate=disable`）。

## 0x23 后语

关闭了`P-State`后，内核层面失去了控制硬件CPU频率的接口，不再对CPU的频率进行控制。根据下图我们可以看到在我们禁用这个模块前，CPU的频率受`P-State`的控制，同时turbo被打开，一直处于一个2.6GHz的水平。禁用后CPU频率不再受`P-State`控制，之前使用`P-State`打开的turbo也停止工作，最终频率稳定的工作在CPU的默认配置2.4GHz。

![CPU-Clock](https://raw.githubusercontent.com/eleme/sre/master/images/cpu-clock.png)

通过基准测试套件`unixbench`对比`intel_pstate=disable`前后性能变化非常小.

前:

```
Benchmark Run: Mon Aug 08 2016 14:28:49 - 14:56:54
24 CPUs in system; running 1 parallel copy of tests

Dhrystone 2 using register variables       30692375.7 lps   (10.0 s, 7 samples)
Double-Precision Whetstone                     3840.3 MWIPS (9.9 s, 7 samples)
Execl Throughput                               4143.8 lps   (30.0 s, 2 samples)
File Copy 1024 bufsize 2000 maxblocks        913341.5 KBps  (30.0 s, 2 samples)
File Copy 256 bufsize 500 maxblocks          239488.5 KBps  (30.0 s, 2 samples)
File Copy 4096 bufsize 8000 maxblocks       2143348.6 KBps  (30.0 s, 2 samples)
Pipe Throughput                             1628899.6 lps   (10.0 s, 7 samples)
Pipe-based Context Switching                 160517.9 lps   (10.0 s, 7 samples)
Process Creation                              11188.7 lps   (30.0 s, 2 samples)
Shell Scripts (1 concurrent)                   8369.8 lpm   (60.0 s, 2 samples)
Shell Scripts (8 concurrent)                   4390.3 lpm   (60.0 s, 2 samples)
System Call Overhead                        2486206.1 lps   (10.0 s, 7 samples)

System Benchmarks Index Values               BASELINE       RESULT    INDEX
Dhrystone 2 using register variables         116700.0   30692375.7   2630.0
Double-Precision Whetstone                       55.0       3840.3    698.2
Execl Throughput                                 43.0       4143.8    963.7
File Copy 1024 bufsize 2000 maxblocks          3960.0     913341.5   2306.4
File Copy 256 bufsize 500 maxblocks            1655.0     239488.5   1447.1
File Copy 4096 bufsize 8000 maxblocks          5800.0    2143348.6   3695.4
Pipe Throughput                               12440.0    1628899.6   1309.4
Pipe-based Context Switching                   4000.0     160517.9    401.3
Process Creation                                126.0      11188.7    888.0
Shell Scripts (1 concurrent)                     42.4       8369.8   1974.0
Shell Scripts (8 concurrent)                      6.0       4390.3   7317.2
System Call Overhead                          15000.0    2486206.1   1657.5
                                                                   ========
System Benchmarks Index Score                                        1581.0
```

后:

```
Benchmark Run: Mon Aug 08 2016 15:15:36 - 15:43:41
24 CPUs in system; running 1 parallel copy of tests

Dhrystone 2 using register variables       30671853.5 lps   (10.0 s, 7 samples)
Double-Precision Whetstone                     3837.8 MWIPS (9.9 s, 7 samples)
Execl Throughput                               4075.8 lps   (30.0 s, 2 samples)
File Copy 1024 bufsize 2000 maxblocks        923975.8 KBps  (30.0 s, 2 samples)
File Copy 256 bufsize 500 maxblocks          241757.9 KBps  (30.0 s, 2 samples)
File Copy 4096 bufsize 8000 maxblocks       2219859.9 KBps  (30.0 s, 2 samples)
Pipe Throughput                             1623321.7 lps   (10.0 s, 7 samples)
Pipe-based Context Switching                 160197.8 lps   (10.0 s, 7 samples)
Process Creation                              11004.0 lps   (30.0 s, 2 samples)
Shell Scripts (1 concurrent)                   8376.0 lpm   (60.0 s, 2 samples)
Shell Scripts (8 concurrent)                   4395.4 lpm   (60.0 s, 2 samples)
System Call Overhead                        2477794.6 lps   (10.0 s, 7 samples)

System Benchmarks Index Values               BASELINE       RESULT    INDEX
Dhrystone 2 using register variables         116700.0   30671853.5   2628.3
Double-Precision Whetstone                       55.0       3837.8    697.8
Execl Throughput                                 43.0       4075.8    947.8
File Copy 1024 bufsize 2000 maxblocks          3960.0     923975.8   2333.3
File Copy 256 bufsize 500 maxblocks            1655.0     241757.9   1460.8
File Copy 4096 bufsize 8000 maxblocks          5800.0    2219859.9   3827.3
Pipe Throughput                               12440.0    1623321.7   1304.9
Pipe-based Context Switching                   4000.0     160197.8    400.5
Process Creation                                126.0      11004.0    873.3
Shell Scripts (1 concurrent)                     42.4       8376.0   1975.5
Shell Scripts (8 concurrent)                      6.0       4395.4   7325.7
System Call Overhead                          15000.0    2477794.6   1651.9
                                                                   ========
System Benchmarks Index Score                                        1582.9
```

## 0x24 疑问

时间有限，不能投入太多的时间用于继续追杀这个问题。虽然问题找到了解决方案，但是整个debug过程中依然有些疑点没有深入去了解：

1. 0x12的log中看到是还有一个`apic_timer_interrupt`的异常。
2. 官方提供的patch中虽然修复了除0的bug，但是实际上我们的dump中`last_sample_time`早就已经发生了溢出，进而才导致了最终出现除0。但是`last_sample_time`的问题官方也没有去解决，所以kernel中可能还隐藏有另一个bug没修。
3. 在 `P-State` 控制下的CPU频率有的稳定，有的不稳定，会处于抖动状态。


### reference

[1] [[PATCH] cpufreq, Fix overflow in busy_scaled due to long delay](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/drivers/cpufreq/intel_pstate.c?id=7180dddf7c32c49975c7e7babf2b60ed450cb760)
[2] [Setting CPU governor to on demand or conservative](http://unix.stackexchange.com/questions/121410/setting-cpu-governor-to-on-demand-or-conservative)
[3] [archlinux wiki](https://wiki.archlinux.org/index.php/CPU_frequency_scaling_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#.E8.B0.83.E6.95.B4.E8.B0.83.E9.80.9F.E5.99.A8)
