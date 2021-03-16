- [堆（Heap）](#堆heap)
  - [堆的概述](#堆的概述)
  - [堆的内存结构](#堆的内存结构)
  - [堆空间大小的设置](#堆空间大小的设置)
    - [年轻代&老年代](#年轻代老年代)
  - [对象分配的一般过程](#对象分配的一般过程)
  - [Minor GC & Major GC & Full GC](#minor-gc--major-gc--full-gc)
    - [Minor GC触发机制](#minor-gc触发机制)
    - [老年代GC（Major GC/Full GC）触发机制](#老年代gcmajor-gcfull-gc触发机制)
  - [内存分配策略](#内存分配策略)
  - [TLAB（Thread Local Allocation Buffer）](#tlabthread-local-allocation-buffer)
    - [为什么有TLAB？](#为什么有tlab)
    - [什么是TLAB？](#什么是tlab)
  - [堆空间的参数设置](#堆空间的参数设置)

## 堆（Heap）

### 堆的概述

* 一个JVM实例只存在一个堆内存，堆也是Java内存管理的核心区域
* Java堆区在JVM启动时就被创建，其空间大小就确定了。是JVM管理的最大一块内存空间
* 堆可以处于物理上不连续的内存空间，但在逻辑上应该被视为连续的
* 所有的线程共享Java堆，在这里还可以划分线程私有的缓冲区（Thread Local Allocation Buffer，TLAB）
* 所有的对象实例和数组都应在运行时分配在堆上
* 数组和对象可能永远不会存储在栈上，因为栈帧中保存引用，这个引用指向对象或者数组中堆的位置

![](https://tva1.sinaimg.cn/large/e6c9d24egy1gokp2bmdi7j20cy06cgmu.jpg)

* 在方法结束后，堆中的对象不会马上被删除，仅仅在垃圾收集时才会被移除
* 堆，是GC执行垃圾回收的重点区域



### 堆的内存结构

* JDK7，堆内存逻辑上分为：新生区+养老区+永久区
* JDK8，堆内存逻辑上分为：新生区+养老区+元空间

> 约定：
>
> 新生区=新生代=年轻代
>
> 养老区=老年区=老年代
>
> 永久区=永久代

* 物理上，堆内存包括：年轻代（伊甸园区+幸存者0区+幸存者1区）+老年代

![](https://tva1.sinaimg.cn/large/e6c9d24egy1gokt079accj208m04mt9k.jpg)



### 堆空间大小的设置

-Xms：用于表示堆区的起始内存

-Xmx：用于表示堆区的最大内存

* 一旦堆区中的内存大小超过-Xmx所指定的最大内存时，会抛出OutOfMemoryError异常
* **通常会将-Xms和-Xmx两个参数配置相同的值**，其目的是为了能够在java垃圾回收机制清理完堆区后不需要重新分隔计算堆区的大小，从而提高性能 

>  查看设置的参数：
>
> 方式一：jps/jstat -gc 进程id
>
> 方式二：-XX:+PrintGCDetails

#### 年轻代&老年代

默认 -XX:NewRatio=2，表示年轻代：老年代=1:2，年轻代占整个堆的1/3

默认 -XX:SurvivorRatio=8，表示年轻代中Eden区与Survivor区的比例是8:1:1

* 几乎所有的Java对象都是在Eden区被new出来的
* 绝大部分的Java对象的销毁都在新生代进行了

-Xmn：用来设置年轻代的空间的大小（一般不设置）



### 对象分配的一般过程

![](https://tva1.sinaimg.cn/large/e6c9d24egy1goktyswd4sj20i10ehwh2.jpg)

阈值：-XX:MaxTenuringThreshold=<N>进行设置，默认是15次



### Minor GC & Major GC & Full GC

JVM在进行GC时，并非每次都对三个内存区域（新生代、老年代、方法区）一起回收的，大部分回收的都是指新生代

对于Hotspot VM，GC按照回收区域可以分为：

* 部分收集
  * 新生代收集（Minor GC/Young GC）：只是新生代（Eden、S0、S1）的垃圾收集
  * 老年代收集（Major GC/Old GC）：只是老年代的垃圾收集
  * 混合收集（Mixed GC）：收集整个新生代以及部分老年代的垃圾收集
* 整堆收集（Full GC）：收集整个Java堆和方法区的垃圾收集

#### Minor GC触发机制

* 当年轻代空间不足时，就会触发Minor GC，这里的年轻代满指的是Eden满，Survivor满不会触发GC（每次Minor GC会清理年轻代的内存）

* 因为Java对象大多具备**朝生夕灭**的特性，所以Minor GC非常频繁，一般回收速度也比较快

* Minor GC会引发STW，暂停其他的用户线程，等垃圾回收结束，用户线程才恢复运行

#### 老年代GC（Major GC/Full GC）触发机制

* 出现了Major GC，经常会伴随至少一次的Minor GC（但并非绝对的，在Parallel Scavenge收集器的收集策略里就有直接进行Major GC的策略选择过程）
  * 老年代空间不足时，会先尝试触发Minor GC，如果之后空间还不足，则触发Major GC
* Major GC的速度一般会比Minor GC慢10倍以上，STW的时间更长
* 如果Major GC以后，内存还不足，就报OOM了

>  Full GC是开发或调优中尽量要避免的，这样暂时时间会短一些



### 内存分配策略

* 优先分配到Eden
* 大对象直接分配到老年代，尽量避免出现过多的大对象
* 长期存活的对象分配到老年代
* 动态对象年龄判断



### TLAB（Thread Local Allocation Buffer）

#### 为什么有TLAB？

* 堆区是线程共享区域，任何线程都可以访问到堆区中的共享数据
* 由于对象实例的创建在JVM中非常频繁，因此在并发环境下从堆区中划分内存空间是线程不安全的
* 为避免多个线程操作同一地址，需要使用加锁等机制，从而影响分配速度

#### 什么是TLAB？

* 对Eden区域继续进行划分，JVM为每个线程分配了一个私有缓存区域
* 多线程同时分配内存时，使用TLAB可以避免一系列的非线程安全问题，同时还能够提升内存分配的吞吐量
* 默认情况下，TLAB的内存非常小，仅占有整个Eden区的1%



### 堆空间的参数设置

* -XX:+PrintFlagsInitial:查看所有参数的默认初始值
* -XX:+PrintFlagsFinal:查看所有的参数的最终值
* -Xms:初始堆内存空间
* -Xmx:最大堆空间内存
* -Xmn:设置新生代的大小
* -XX:NewRatio:配置新生代与老年代在堆结构中的占比
* -XX:SurvivorRatio:设置新生代中Eden和S0/S1空间的比例
* -XX:MaxTenuringThreshold:设置新生代垃圾的最大年龄
* -XX:+PrintGCDetails:输出详细的GC处理日志
* -XX:HandlePromotionFailure:是否设置空间分配担保
