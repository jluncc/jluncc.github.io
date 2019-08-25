---
title: Java OOM 分析
author: 陈加菲
date: 2019-06-10 21:12:47
tags:
 - JVM
---

### 什么是 OOM

在 Java 中，OOM 是 java.lang.OutOfMemoryError 异常的缩写，简单来说是应用的内存用完了。

而这个内存，指代的是 JVM 管理的内存模型。

#### JVM 内存模型

JVM 在运行时管理的内存区域分别如下

1. 程序计数器。其作用是记录每个线程当前执行的字节码指令的位置，因为可能由多个线程并发执行不同的方法，所以程序计数器是每个线程私有的。
2. Java 虚拟机栈。Java 中方法的调用会为方法创建栈帧，然后在线程的 Java 虚拟机栈中进行出栈入栈，记录调用情况；除此，Java 虚拟机栈还会记录局部变量表，操作数栈等信息，同样为每个线程私有。
3. Java 堆内存。程序中创建的对象，便存储在 Java 堆内存中，其也是 GC 的主要地方，堆内存中的对象又按不同的 “生存” 情况分为新生代，老年代等。Java 堆内存为所有线程共有。
4. 方法区 / Metaspace。该区域主要存放类的元信息，在 JDK 1.8 后新独立出来 Metaspace。
5. 本地方法栈。为 natice 方法，调用本地操作系统类库提供内存环境。
6. 堆外内存。这一区域不属于 JVM 内存，是可直接访问的内存，如 NIO 会访问到这部分。

### 为什么会 OOM

发送 OOM，简单来说可总结为两个原因：

- 分配给 JVM 的内存确实不够用
- 分配的内存是够用的，但是代码写的不好，多余的内存没有释放，最终导致内存不够用

### OOM 类型

一般 OOM 分为三种类型

1. java.lang.OutOfMemoryError: Java heap space。Java 堆内存溢出，此种情况最常见。
2. java.lang.OutOfMemoryError: PermGen space。Java 永久代溢出，即方法区溢出了。
3. java.lang.StackOverflowError。不会抛 OOM ERROR，但也是比较常见的 Java 内存溢出。

### 分析 OOM 思路

接下来简单总结，在线上环境遇到 OOM 时，分析定位问题的思路。

1. 寻找 GC 日志和 dump 文件。
这里有个问题，就是不知道线上应用有没有开启打印 GC 日志和 dump 文件，这里寻找的思路如下

```
// 1.查找对应的应用的 pid
jps -v // 输出所有 Java 应用，找对应 pid，或使用 ps -ef | grep xxx 来寻找

// 2.查看该 pid 启动应用时的 VM 参数
jinfo -flags pid

```

获取到的 VM 参数，可关注如下几项参数的情况

```
-XX:+HeapDumpOnOutOfMemoryError  // 是否开启 OOM 时输出 dump 文件，+ 表示开启
-XX:HeapDumpPath=/data/logs      // 若配置该项，dump 文件输出到该路径下

-XX:+PrintGCDetails              // 是否打印 GC 详细信息
-XX:+PrintGCTimeStamps           // 是否打印 GC 时间戳(基准形式)
-XX:+PrintGCDateStamps           // 是否打印 GC 时间戳(日期形式)
-XX:+PrintTenuringDistribution   // 在每次新生代 GC 时，输出幸存区中对象的年龄分布
-Xloggc:/data/logs               // 若配置该项，GC 日志输出到该路径下 
```

2. 若应用没有启用相应的参数输出日志，这里简单查看应用 JVM 内存使用情况的思路如下

```
// 1.查看应用 GC 次数及平均每次 GC 时间
jstat -gc pid 5000 // 每 5 秒显示一次 pid 的进程生成 GC 情况

// 2.查看 JVM 堆内存不同代的具体使用情况
jmap -heap pid
```

3. 同时，应该在这时加上对应的参数，见上面 [获取到的 VM 参数] 示例。

那么如何加上参数呢？

有几个方法可以设置。首先是加在 Tomcat 的启动文件上（/bin/catalina.sh），如下新增配置

```
JAVA_OPTS="-XX:+HeapDumpOnOutOfMemoryError ... "
```

因为我们是一个 Tomcat 只部署一个应用，这样配置在 Tomcat 启动文件上，其粒度便只针对当前应用。

除此之外，也可以配置在 Maven 启动文件上（%MAVEN_HOME%/bin/mvn），配置语法与上面类似。因为有多个应用使用同个 Maven，因此最后没有采用该种配置方法。

配置完参数后，重新发布应用，若下次有 OOM 情况出现，便可分析对应日志来定位具体问题。

### 脑图

![OOM 分析.png](/images/oom-analysis-1/oom-analysis.png)

### 参考文章

[Linux使用jstat命令查看jvm的GC情况](https://blog.csdn.net/zlzlei/article/details/46471627)
[一次线上生产问题的全面复盘](https://mp.weixin.qq.com/s/Me1Y6Moir93EbHw_0qpA5w)
[什么是java OOM？如何分析及解决oom问题？](https://www.cnblogs.com/ThinkVenus/p/6805495.html)

**END**

