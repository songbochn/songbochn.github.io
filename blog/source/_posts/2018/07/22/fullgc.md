---
title: fullGC分析记录
date: 2018-07-22 12:08:17
tag: 
	- java
	- jvm
	- gc
category:
	- 技术总结
---
在开发过程中发现平均几个小时就会发生一次fullGC,在fullgc的这段时间接口响应十分之慢，甚至会出现超时现象，因此要解决掉这种问题。以下是排查过程。
#### jvm参数设置
首先看下jvm参数设置：

- -XX:+DisableExplicitGC 关闭System.gc()
- -XX:+PrintGCDetails 打印GC详细信息
- -XX:+PrintHeapAtGC 打印GC前后堆使用情况
- -XX:+PrintTenuringDistribution  查看每次minor GC后新的存活周期的阈值 就是年龄
- -XX:ParallelGCThreads=20  并行收集器的线程数
- -XX:+UseConcMarkSweepGC   使用CMS内存收集
- -XX:+PrintGCTimeStamps 打印GC时间
- -XX:+PrintGCDateStamps   以日期格式
<!--more-->
- -XX:CMSFullGCsBeforeCompaction=0  上面配置开启的情况下,这里设置多少次Full GC后,对年老代进行压缩
- -XX:+UseCMSCompactAtFullCollection 使用并发收集器时,开启对年老代的压缩
- -XX:CMSInitiatingOccupancyFraction=80 老年代使用80% 就开始fullGC
- -Xmn1024m 年轻代大小-XX:SurvivorRatio=8 Eden区与Survivor区的大小比值
- -XX:MetaspaceSize=256m 永久代，初始值一定是2180710byte 扩容到 256m就fullgc,此后扩容一次gc一次
- -XX:MaxMetaspaceSize=256m -XX:+HeapDumpOnOutOfMemoryError 
- -XX:ReservedCodeCacheSize=128m 这个参数主要设置codecache的大小，比如我们jit编译的代码都是放在codecache里的，所以codecache如果满了的话，那带来的问题就是无法再jit编译了，而且还会去优化。因此大家可能碰到这样的问题：cpu一直高，然后发现是编译线程一直高（系统运行到一定时期）-XX:InitialCodeCacheSize=128m 初始化-Xmx3g 最大堆大小-Xms3g 初始堆大小

#### GC日志文件

```bash
{Heap before GC invocations=2693 (full 10):
par new generation   total 943744K, used 914622K [0x0000000700000000, 0x0000000740000000, 0x0000000740000000)
eden space 838912K, 100% used [0x0000000700000000, 0x0000000733340000, 0x0000000733340000)
from space 104832K,  72% used [0x00000007399a0000, 0x000000073e38f9f8, 0x0000000740000000)
to   space 104832K,   0% used [0x0000000733340000, 0x0000000733340000, 0x00000007399a0000)
concurrent mark-sweep generation total 2097152K, used 1674190K [0x0000000740000000, 0x00000007c0000000, 0x00000007c0000000)
Metaspace       used 82793K, capacity 84248K, committed 84480K, reserved 1124352K
class space    used 8521K, capacity 8836K, committed 8960K, reserved 1048576K
2018-06-27T10:30:03.956+0800: 132844.082: [GC (Allocation Failure) 132844.082: [ParNew
Desired survivor size 53673984 bytes, new threshold 9 (max 15)
- age   1:    9419072 bytes,    9419072 total
- age   2:    6701816 bytes,   16120888 total
- age   3:    4611720 bytes,   20732608 total
- age   4:    4955944 bytes,   25688552 total
- age   5:    7069448 bytes,   32758000 total
- age   6:    7478208 bytes,   40236208 total
- age   7:    6091736 bytes,   46327944 total
- age   8:    5505400 bytes,   51833344 total
- age   9:    5498208 bytes,   57331552 total
: 914622K->83580K(943744K), 0.0676204 secs] 2588812K->1765017K(3040896K), 0.0678538 secs] [Times: user=0.45 sys=0.00, real=0.07 secs] 
Heap after GC invocations=2694 (full 10):
par new generation   total 943744K, used 83580K [0x0000000700000000, 0x0000000740000000, 0x0000000740000000)
eden space 838912K,   0% used [0x0000000700000000, 0x0000000700000000, 0x0000000733340000)
from space 104832K,  79% used [0x0000000733340000, 0x00000007384df160, 0x00000007399a0000)
to   space 104832K,   0% used [0x00000007399a0000, 0x00000007399a0000, 0x0000000740000000)
concurrent mark-sweep generation total 2097152K, used 1681437K [0x0000000740000000, 0x00000007c0000000, 0x00000007c0000000)
Metaspace       used 82793K, capacity 84248K, committed 84480K, reserved 1124352K
class space    used 8521K, capacity 8836K, committed 8960K, reserved 1048576K
}
2018-06-27T10:30:04.025+0800: 132844.151: [GC (CMS Initial Mark) [1 CMS-initial-mark: 1681437K(2097152K)] 1766485K(3040896K), 0.0106641 secs] [Times: user=0.05 sys=0.00, real=0.01 secs] 
2018-06-27T10:30:04.036+0800: 132844.162: [CMS-concurrent-mark-start]
2018-06-27T10:30:04.069+0800: 132844.195: [CMS-concurrent-mark: 0.030/0.033 secs] [Times: user=0.16 sys=0.00, real=0.04 secs] 
2018-06-27T10:30:04.069+0800: 132844.195: [CMS-concurrent-preclean-start]
2018-06-27T10:30:04.076+0800: 132844.201: [CMS-concurrent-preclean: 0.006/0.006 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
2018-06-27T10:30:04.076+0800: 132844.201: [CMS-concurrent-abortable-preclean-start]
 CMS: abort preclean due to time 2018-06-27T10:30:09.275+0800: 132849.401: [CMS-concurrent-abortable-preclean: 5.197/5.200 secs] [Times: user=5.99 sys=0.10, real=5.20 secs] 
2018-06-27T10:30:09.276+0800: 132849.402: [GC (CMS Final Remark) [YG occupancy: 240189 K (943744 K)]132849.402: [Rescan (parallel) , 0.0363990 secs]132849.439: [weak refs processing, 0.0347831 secs]132849.473: [class unloading, 0.0227507 secs]132849.496: [scrub symbol table, 0.0075422 secs]132849.504: [scrub string table, 0.0013443 secs][1 CMS-remark: 1681437K(2097152K)] 1921626K(3040896K), 0.1050885 secs] [Times: user=0.34 sys=0.00, real=0.11 secs] 
2018-06-27T10:30:09.382+0800: 132849.508: [CMS-concurrent-sweep-start]
2018-06-27T10:30:10.562+0800: 132850.688: [CMS-concurrent-sweep: 1.181/1.181 secs] [Times: user=1.40 sys=0.02, real=1.18 secs] 
2018-06-27T10:30:10.562+0800: 132850.688: [CMS-concurrent-reset-start]
2018-06-27T10:30:10.567+0800: 132850.693: [CMS-concurrent-reset: 0.005/0.005 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
{Heap before GC invocations=2694 (full 11):
 par new generation   total 943744K, used 922492K [0x0000000700000000, 0x0000000740000000, 0x0000000740000000)
  eden space 838912K, 100% used [0x0000000700000000, 0x0000000733340000, 0x0000000733340000)
  from space 104832K,  79% used [0x0000000733340000, 0x00000007384df160, 0x00000007399a0000)
  to   space 104832K,   0% used [0x00000007399a0000, 0x00000007399a0000, 0x0000000740000000)
 concurrent mark-sweep generation total 2097152K, used 85252K [0x0000000740000000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 82793K, capacity 84248K, committed 84480K, reserved 1124352K
  class space    used 8521K, capacity 8836K, committed 8960K, reserved 1048576K
2018-06-27T10:30:34.966+0800: 132875.092: [GC (Allocation Failure) 132875.092: [ParNew
Desired survivor size 53673984 bytes, new threshold 9 (max 15)
- age   1:    8523248 bytes,    8523248 total
- age   2:    6541000 bytes,   15064248 total
- age   3:    6627176 bytes,   21691424 total
- age   4:    4592336 bytes,   26283760 total
- age   5:    4939016 bytes,   31222776 total
- age   6:    7055128 bytes,   38277904 total
- age   7:    7459240 bytes,   45737144 total
- age   8:    6061064 bytes,   51798208 total
- age   9:    5429624 bytes,   57227832 total
: 922492K->90909K(943744K), 0.0349351 secs] 1007745K->181536K(3040896K), 0.0351277 secs] [Times: user=0.20 sys=0.00, real=0.03 secs] 
Heap after GC invocations=2695 (full 11):
 par new generation   total 943744K, used 90909K [0x0000000700000000, 0x0000000740000000, 0x0000000740000000)
  eden space 838912K,   0% used [0x0000000700000000, 0x0000000700000000, 0x0000000733340000)
  from space 104832K,  86% used [0x00000007399a0000, 0x000000073f267448, 0x0000000740000000)
  to   space 104832K,   0% used [0x0000000733340000, 0x0000000733340000, 0x00000007399a0000)
 concurrent mark-sweep generation total 2097152K, used 90627K [0x0000000740000000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 82793K, capacity 84248K, committed 84480K, reserved 1124352K
  class space    used 8521K, capacity 8836K, committed 8960K, reserved 1048576K
}
{Heap before GC invocations=2695 (full 11):
 par new generation   total 943744K, used 929821K [0x0000000700000000, 0x0000000740000000, 0x0000000740000000)
  eden space 838912K, 100% used [0x0000000700000000, 0x0000000733340000, 0x0000000733340000)
  from space 104832K,  86% used [0x00000007399a0000, 0x000000073f267448, 0x0000000740000000)
  to   space 104832K,   0% used [0x0000000733340000, 0x0000000733340000, 0x00000007399a0000)
 concurrent mark-sweep generation total 2097152K, used 90627K [0x0000000740000000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 82793K, capacity 84248K, committed 84480K, reserved 1124352K
  class space    used 8521K, capacity 8836K, committed 8960K, reserved 1048576K
2018-06-27T10:31:07.676+0800: 132907.802: [GC (Allocation Failure) 132907.802: [ParNew
Desired survivor size 53673984 bytes, new threshold 9 (max 15)
- age   1:    7681112 bytes,    7681112 total
- age   2:    5856440 bytes,   13537552 total
- age   3:    6507528 bytes,   20045080 total
- age   4:    6624920 bytes,   26670000 total
- age   5:    4586624 bytes,   31256624 total
- age   6:    4938792 bytes,   36195416 total
- age   7:    7049544 bytes,   43244960 total
- age   8:    7452280 bytes,   50697240 total
- age   9:    6060952 bytes,   56758192 total
: 929821K->74939K(943744K), 0.0370969 secs] 1020448K->170947K(3040896K), 0.0372984 secs] [Times: user=0.24 sys=0.00, real=0.04 secs] 
Heap after GC invocations=2696 (full 11):
 par new generation   total 943744K, used 74939K [0x0000000700000000, 0x0000000740000000, 0x0000000740000000)
  eden space 838912K,   0% used [0x0000000700000000, 0x0000000700000000, 0x0000000733340000)
  from space 104832K,  71% used [0x0000000733340000, 0x0000000737c6ed58, 0x00000007399a0000)
  to   space 104832K,   0% used [0x00000007399a0000, 0x00000007399a0000, 0x0000000740000000)
 concurrent mark-sweep generation total 2097152K, used 96007K [0x0000000740000000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 82793K, capacity 84248K, committed 84480K, reserved 1124352K
  class space    used 8521K, capacity 8836K, committed 8960K, reserved 1048576K
}
```
#### 分析

以上是GC日志，

按时间先后发生了一次minor gc --> major gc --> minor gc

下边逐字逐句分析这段日志：

```
1.{Heap before GC invocations=2693 (full 10): 

  在minorGC之前，已经执行了2693次GC，10次FullGC

2. par new generation   total 943744K, used 914622K [0x0000000700000000, 0x0000000740000000, 0x0000000740000000)  
  
  使用ParNew垃圾收集器，年轻代共943744K,已使用914622K  //todo  

3. eden space 838912K, 100% used [0x0000000700000000, 0x0000000733340000, 0x0000000733340000)  
  
   eden区共有内存 838912k 使用率100%，//todo

4. from space 104832K,  72% used [0x00000007399a0000, 0x000000073e38f9f8, 0x0000000740000000)

    from survivor区总大小 104832k，使用率 72% 
    
5.  to   space 104832K,   0% used [0x0000000733340000, 0x0000000733340000, 0x00000007399a0000)     
    to survivor区总大小 104832k，使用率 0% 
    
6. concurrent mark-sweep generation total 2097152K, used 1674190K [0x0000000740000000, 0x00000007c0000000, 0x00000007c0000000)

    cms负责老年代垃圾回收，老年代共有2097152K的内存，已使用1674190K的内存 
    
7. Metaspace       used 82793K, capacity 84248K, committed 84480K, reserved 1124352K

    java8去掉了perm内存，用metaspace取代，关于这些参数，
        
8. class space    used 8521K, capacity 8836K, committed 8960K, reserved 1048576K9. 2018-06-27T10:30:03.956+0800: 132844.082: [GC (Allocation Failure) 132844.082: [ParNew

    时间点：GC原因：Allocation Failure  132844.082表示GC开始时间，相对于JVM启动时间的偏移量
    
9. Desired survivor size 53673984 bytes, new threshold 9 (max 15)

   这一句话非常关键，这里的意思是期望survivor的大小,53673984 bytes。新的晋升年龄为9，最大值为15。因为是复制-清理算法，也就是说，把eden和from的存活对象，复制到to的区域，其实to的接受生效范围是可以制定的，不指定，默认为一个survivor的50%，通过参数【-XX:TargetSurvivorRatio 】调整53673984 bytes就是上述一个survivor大小(104832K)的一半。那么这个有什么用？作用在于计算动态年龄，我期望放下这么多对象，那么多于大小的存活对象，我希望移到老年代，据此算出一个年龄，下次大于这个年龄的晋升到老年代。
   
10. - age   1:    9419072 bytes,    9419072 total

		年龄为1（经历过一次youngGC）的存活对象大小为9419072 bytes, 截止到age 1 总共对象大小9419072 bytes;

	- age   2:    6701816 bytes,   6701816 total

		年龄为2（经历过二次youngGC）的存活对象大小为6701816 bytes, 截止到age 2 总共对象大小6701816 bytes;

	- age   3:    4611720 bytes,   20732608 total

		同上

	- age   4:    4955944 bytes,   25688552 total
		
		同上

	- age   5:    7069448 bytes,   32758000 total

		同上

	- age   6:    7478208 bytes,   40236208 total

		同上

	- age   7:    6091736 bytes,   46327944 total

		同上

	- age   8:    5505400 bytes,   51833344 total

		同上
		
	- age   9:    5498208 bytes,   57331552 total

		同上

11.: 914622K->83580K(943744K), 0.0676204 secs] 2588812K->1765017K(3040896K), 0.0678538 secs] [Times: user=0.45 sys=0.00, real=0.07 secs] 

年轻代共有从914622K回收到83580K，年轻代大小：943744K，花费时间0.0676204secs,  堆大小2588812K到1765017K，总共大小3040896K

2018-06-27T10:30:04.025+0800: 132844.151: [GC (CMS Initial Mark) [1 CMS-initial-mark: 1681437K(2097152K)] 1766485K(3040896K), 0.0106641 secs] [Times: user=0.05 
sys=0.00, real=0.01 secs] 
2018-06-27T10:30:04.036+0800: 132844.162: [CMS-concurrent-mark-start]
2018-06-27T10:30:04.069+0800: 132844.195: [CMS-concurrent-mark: 0.030/0.033 secs] [Times: user=0.16 sys=0.00, real=0.04 secs] 
2018-06-27T10:30:04.069+0800: 132844.195: [CMS-concurrent-preclean-start]
2018-06-27T10:30:04.076+0800: 132844.201: [CMS-concurrent-preclean: 0.006/0.006 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
2018-06-27T10:30:04.076+0800: 132844.201: [CMS-concurrent-abortable-preclean-start] 
CMS: abort preclean due to time 2018-06-27T10:30:09.275+0800: 132849.401: [CMS-concurrent-abortable-preclean: 5.197/5.200 secs] [Times: user=5.99 sys=0.10, real=5.20 secs] 
2018-06-27T10:30:09.276+0800: 132849.402: [GC (CMS Final Remark) [YG occupancy: 240189 K (943744 K)]132849.402: [Rescan (parallel) , 0.0363990 secs]132849.439: [weak 
refs processing, 0.0347831 secs]132849.473: [class unloading, 0.0227507 secs]132849.496: [scrub symbol table, 0.0075422 secs]132849.504: [scrub string table, 0.0013443 
secs][1 CMS-remark: 1681437K(2097152K)] 1921626K(3040896K), 0.1050885 secs] [Times: user=0.34 sys=0.00, real=0.11 secs] 
2018-06-27T10:30:09.382+0800: 132849.508: [CMS-concurrent-sweep-start]
2018-06-27T10:30:10.562+0800: 132850.688: [CMS-concurrent-sweep: 1.181/1.181 secs] [Times: user=1.40 sys=0.02, real=1.18 secs] 
2018-06-27T10:30:10.562+0800: 132850.688: [CMS-concurrent-reset-start]
2018-06-27T10:30:10.567+0800: 132850.693: [CMS-concurrent-reset: 0.005/0.005 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
关于CMS日志分析，参考下面链接
```
#### GC日志结论

设置的老年代清理阈值为80%,可以看到在进行CMS回收的时候，老年代使用大小为 1681437K，在回收完之后老年代容量90627K，回收了75.8%,这明显是年轻代对象过早提升到老年代造成的老年代充斥大量垃圾的情况。

那么造成过早提升的原因分析大概是以下这些，目标是让让短命对象活在年轻代：

1️⃣：年轻代内存大小设置过小，

2️⃣：TargetSurvivorRatio参数太小，每次youngGC允许呆在年轻代的数量过小，

3️⃣：短命对象持续生成过多，使得动态年龄降低，使得大量对象晋升老年代

分析出原因后，接下来便是调整参数，通过-Xmn2g把年轻代调大，并调整TargetSurvivorRatio为60，有所缓解但还是明显有增长。于是准备Dump一份堆快照看下大对象都是什么，这里注意不要在业务高峰期dump文件，有可能造成请求超时。

下下来dump文件这里采用mat分析，[MAT工具使用](http://www.songbohome.com/2018/07/22/2018-07-22-mat/)

这里使用mat分析时，要把keepUnreachableObjects勾上，要不然会把无GCROOT引用不到的删除。而我们主要是看垃圾都是什么。
![mat总的看](https://res.cloudinary.com/dataset/image/upload/v1532242641/image/matmain.png)
可以看到jdbc4Connection有17763个对象，占了585M多，byte[]数组有444158个，占了318M多，合起来占用了47.14%的内存，我们是用了dbcp2的连接池，但为何依然有如此多连接需要分析。待会儿也会看一下

在GC完之后jdbc4Connection有对象890个 占用内存70M。相当于jdbc4Connection的94%都是废弃链接。

byte[]的内容。
![mat细节](https://res.cloudinary.com/dataset/image/upload/v1532242641/image/matdetail.png)
这是两个Connection前一个是无引用的垃圾，后一个是真正的链接。可以看到该链接正常是被ConcurrentHashMap所持有，而垃圾没有被持有，

看下dbcp2的连接池代码：

这是销毁对象的：

```
public void invalidateObject(final T obj) throws Exception {
        final PooledObject<T> p = allObjects.get(new IdentityWrapper<>(obj));
        if (p == null) {
            if (isAbandonedConfig()) {
                return;
            }
            throw new IllegalStateException(
                    "Invalidated object not currently part of this pool");
        }
        synchronized (p) {
            if (p.getState() != PooledObjectState.INVALID) {
                destroy(p);
            }
        }
        ensureIdle(1, false);
    }

    private void destroy(final PooledObject<T> toDestroy) throws Exception {
        toDestroy.invalidate();
        idleObjects.remove(toDestroy);//idleObjects为空闲链接集合。
        allObjects.remove(new IdentityWrapper<>(toDestroy.getObject()));//所有链接集合
        try {
            factory.destroyObject(toDestroy);
        } finally {
            destroyedCount.incrementAndGet();
            createCount.decrementAndGet();
        }
    }
```

根据我们整理的结果，现象是大量链接被关闭

链接的生命周期，因为使用而被创建入池，使用完空闲，有使用再拿出来用。空闲一段时间被关闭

这里有几个参数需要我们权衡 minEvictableIdleTimeMillis表示空闲多久被关闭

timeBetweenEvictionRunsMillis：清除空闲链接线程多久运行一次。将空闲链接关闭 //默认30s

numTestsPerEvictionRun：检查空间链接线程每次检查几个链接

config 内容:

```
public EvictionConfig(final long poolIdleEvictTime, final long poolIdleSoftEvictTime,
            final int minIdle) {
        if (poolIdleEvictTime > 0) {
            idleEvictTime = poolIdleEvictTime;
        } else {
            idleEvictTime = Long.MAX_VALUE;
        }
        if (poolIdleSoftEvictTime > 0) {
            idleSoftEvictTime = poolIdleSoftEvictTime;
        } else {
            idleSoftEvictTime  = Long.MAX_VALUE;
        }
        this.minIdle = minIdle; //默认0
    }

这是线程逻辑

    public boolean evict(final EvictionConfig config, final PooledObject<T> underTest,
            final int idleCount) {

        if ((config.getIdleSoftEvictTime() < underTest.getIdleTimeMillis() &&
                config.getMinIdle() < idleCount) ||
                config.getIdleEvictTime() < underTest.getIdleTimeMillis()) {
            return true;
        }
        return false;
    }
```

//为true表示要关闭

判断来说 numTestsPerEvictionRun可以改小，timeBetweenEvictionRunsMillis改大，minEvictableIdleTimeMillis改大。

使得空闲链接停留时间加长，避免被关闭。

那么还有一个问题：为什么链接在年轻代没有被回收掉，而堆积在老年代等到阈值触发才被回收：

这引出来一个问题，这些链接在刚刚晋升到老年代时，是否还被引用，这里分成两种情况来看。

如果没被引用，那就只能等到阈值触发被回收。但是在晋升之前已经被年轻代回收掉了，假设不成立。

如果被引用，那么可以得出以下结论。

#### 结论

首先扫描空闲连接的时间是恒定不变的。30s

youngGC频率：

撷取2018-06-20中午12点和下午15点的数据来看：

12点数据
![12](https://res.cloudinary.com/dataset/image/upload/v1532242641/image/gcfirst.png) 
                                                                               15点数据
![15](https://res.cloudinary.com/dataset/image/upload/v1532242641/image/gcsecond.png)
可见60s内youngGC频率非常之高。

从一批链接【记做A】的诞生来分析，A和一批对象诞生在eden区域，伴随着大量临时对象的生成以及youngGC频率可以看出未等到清理空闲链接，A中的部分链接已经晋升到了老年代，扫描空闲链接的线程，将空闲链接剔除掉后，这些垃圾链接就安安逸逸的生活在老年代。在需要链接的时候又去eden生成一批链接，周而复始。直到老年代达到阈值。

#### 如何解决？

目前上线的做法增大minEvictableIdleTimeMillis参数，也就是说让链接一直不被回收。这样也就不会产生大量的垃圾链接。而这批链接也安安静静的生活在老年代。

还有两个问题需要思考

这些链接生活在老年代是应该的吗？

为什么youngGC如此频繁，younGC的频率正常应该是多久？

#### 参考文档：

[线上FullGC频繁的排查](https://blog.csdn.net/wilsonpeng3/article/details/70064336)

[一次频繁Full GC的排查过程](https://blog.csdn.net/u012422829/article/details/78154495)

[又一次线上OOM排查经过](http://www.importnew.com/24393.html)

[JVM系列三:JVM参数设置、分析](http://www.cnblogs.com/redcreen/archive/2011/05/04/2037057.html)

[GC概念理解](http://darktea.github.io/notes/2013/09/08/java-gc.html)

[JVM 调优 —— GC 长时间停顿问题及解决方法](http://www.importnew.com/22886.html)

[记一次JVM老生代增长过快问题排查](https://neway6655.github.io/gc/2017/10/28/%E8%AE%B0%E4%B8%80%E6%AC%A1JVM%E8%80%81%E7%94%9F%E4%BB%A3%E5%A2%9E%E9%95%BF%E8%BF%87%E5%BF%AB%E9%97%AE%E9%A2%98%E6%8E%92%E6%9F%A5.html)

[MetaSpace的误解](https://blog.csdn.net/u011381576/article/details/79635867)

[metaSpace参数理解](https://www.cnblogs.com/benwu/articles/8312699.html)

[快速解读GC日志](https://blog.csdn.net/renfufei/article/details/49230943)

[从实际案例聊聊Java应用的GC优化](https://tech.meituan.com/jvm_optimize.html)

[一次GC Tuning小记](https://neway6655.github.io/java,%20gc%20tuning/2016/09/24/gc-tuning.html)

[GC之详解CMS收集过程和日志分析](https://www.cnblogs.com/zhangxiaoguang/p/5792468.html)
