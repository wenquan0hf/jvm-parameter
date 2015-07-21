# GC 日志

原文地址：[https://blog.codecentric.de/en/2014/01/useful-jvm-flags-part-8-gc-logging/](https://blog.codecentric.de/en/2014/01/useful-jvm-flags-part-8-gc-logging/)

作者：[PATRICK PESCHLOW](https://blog.codecentric.de/en/author/patrick-peschlow/)，译者：Greenster 校对：梁海舰

本系列的最后一部分是有关垃圾收集（GC）日志的 JVM 参数。GC 日志是一个很重要的工具，它准确记录了每一次的 GC 的执行时间和执行结果，通过分析 GC 日志可以优化堆设置和 GC 设置，或者改进应用程序的对象分配模式。

## -XX:+PrintGC

参数 - XX:+PrintGC（或者 - verbose:gc）开启了简单 GC 日志模式，为每一次新生代（young generation）的 GC 和每一次的 Full GC 打印一行信息。下面举例说明：

>[GC 246656K->243120K(376320K), 0.0929090 secs]  
[Full GC 243120K->241951K(629760K), 1.5589690 secs]

每行开始首先是 GC 的类型（可以是 “GC” 或者 “Full GC”），然后是在 GC 之前和 GC 之后已使用的堆空间，再然后是当前的堆容量，最后是 GC 持续的时间（以秒计）。

第一行的意思就是 GC 将已使用的堆空间从 246656K 减少到 243120K，当前的堆容量（译者注：GC 发生时）是 376320K，GC 持续的时间是 0.0929090 秒。

简单模式的 GC 日志格式是与 GC 算法无关的，日志也没有提供太多的信息。在上面的例子中，我们甚至无法从日志中判断是否 GC 将一些对象从 young generation 移到了 old generation。所以详细模式的 GC 日志更有用一些。

## -XX:PrintGCDetails

如果不是使用 - XX:+PrintGC，而是 - XX:PrintGCDetails，就开启了详细 GC 日志模式。在这种模式下，日志格式和所使用的 GC 算法有关。我们首先看一下使用 Throughput 垃圾收集器在 young generation 中生成的日志。为了便于阅读这里将一行日志分为多行并使用缩进。

>[GC  
[PSYoungGen: 142816K->10752K(142848K)] 246648K->243136K(375296K), 0.0935090 secs]   
[Times: user=0.55 sys=0.10, real=0.09 secs]

我们可以很容易发现：这是一次在 young generation 中的 GC，它将已使用的堆空间从 246648K 减少到了 243136K，用时 0.0935090 秒。此外我们还可以得到更多的信息：所使用的垃圾收集器（即 PSYoungGen）、young generation 的大小和使用情况（在这个例子中 “PSYoungGen” 垃圾收集器将 young generation 所使用的堆空间从 142816K 减少到 10752K）。

既然我们已经知道了 young generation 的大小，所以很容易判定发生了 GC，因为 young generation 无法分配更多的对象空间：已经使用了 142848K 中的 142816K。我们可以进一步得出结论，多数从 young generation 移除的对象仍然在堆空间中，只是被移到了 old generation：通过对比绿色的和蓝色的部分可以发现即使 young generation 几乎被完全清空（从 142816K 减少到 10752K），但是所占用的堆空间仍然基本相同（从 246648K 到 243136K）。

详细日志的 “Times” 部分包含了 GC 所使用的 CPU 时间信息，分别为操作系统的用户空间和系统空间所使用的时间。同时，它显示了 GC 运行的 “真实” 时间（0.09 秒是 0.0929090 秒的近似值）。如果 CPU 时间（译者注：0.55 秒 + 0.10 秒）明显多于” 真实 “时间（译者注：0.09 秒），我们可以得出结论：GC 使用了多线程运行。这样的话 CPU 时间就是所有 GC 线程所花费的 CPU 时间的总和。实际上我们的例子中的垃圾收集器使用了 8 个线程。

接下来看一下 Full GC 的输出日志

>[Full GC
	[PSYoungGen: 10752K->9707K(142848K)]
    [ParOldGen: 232384K->232244K(485888K)]  243136K->241951K(628736K)  
[PSPermGen: 3162K->3161K(21504K)], 1.5265450 secs
]

除了关于 young generation 的详细信息，日志也提供了 old generation 和 permanent generation 的详细信息。对于这三个 generations，一样也可以看到所使用的垃圾收集器、堆空间的大小、GC 前后的堆使用情况。需要注意的是显示堆空间的大小等于 young generation 和 old generation 各自堆空间的和。以上面为例，堆空间总共占用了 241951K，其中 9707K 在 young generation，232244K 在 old generation。Full GC 持续了大约 1.53 秒，用户空间的 CPU 执行时间为 10.96 秒，说明 GC 使用了多线程（和之前一样 8 个线程）。

对不同 generation 详细的日志可以让我们分析 GC 的原因，如果某个 generation 的日志显示在 GC 之前，堆空间几乎被占满，那么很有可能就是这个 generation 触发了 GC。但是在上面的例子中，三个 generation 中的任何一个都不是这样的，在这种情况下是什么原因触发了 GC 呢。对于 Throughput 垃圾收集器，在某一个 generation 被过度使用之前，GC ergonomics（参考本系列第 6 节）决定要启动 GC。

Full GC 也可以通过显式的请求而触发，可以是通过应用程序，或者是一个外部的 JVM 接口。这样触发的 GC 可以很容易在日志里分辨出来，因为输出的日志是以 “Full GC(System)” 开头的，而不是 “Full GC”。

对于 Serial 垃圾收集器，详细的 GC 日志和 Throughput 垃圾收集器是非常相似的。唯一的区别是不同的 generation 日志可能使用了不同的 GC 算法（例如：old generation 的日志可能以 Tenured 开头，而不是 ParOldGen）。使用垃圾收集器作为一行日志的开头可以方便我们从日志就判断出 JVM 的 GC 设置。

对于 CMS 垃圾收集器，young generation 的详细日志也和 Throughput 垃圾收集器非常相似，但是 old generation 的日志却不是这样。对于 CMS 垃圾收集器，在 old generation 中的 GC 是在不同的时间片内与应用程序同时运行的。GC 日志自然也和 Full GC 的日志不同。而且在不同时间片的日志夹杂着在此期间 young generation 的 GC 日志。但是了解了上面介绍的 GC 日志的基本元素，也不难理解在不同时间片内的日志。只是在解释 GC 运行时间时要特别注意，由于大多数时间片内的 GC 都是和应用程序同时运行的，所以和那种独占式的 GC 相比，GC 的持续时间更长一些并不说明一定有问题。

正如我们在第 7 节中所了解的，即使 CMS 垃圾收集器没有完成一个 CMS 周期，Full GC 也可能会发生。如果发生了 GC，在日志中会包含触发 Full GC 的原因，例如众所周知的”concurrent mode failure“。

为了避免过于冗长，我这里就不详细说明 CMS 垃圾收集器的日志了。另外，CMS 垃圾收集器的作者做了详细的说明（在这里），强烈建议阅读。

## -XX:+PrintGCTimeStamps 和 - XX:+PrintGCDateStamps

使用 - XX:+PrintGCTimeStamps 可以将时间和日期也加到 GC 日志中。表示自 JVM 启动至今的时间戳会被添加到每一行中。例子如下：

>0.185: [GC 66048K->53077K(251392K), 0.0977580 secs]  
0.323: [GC 119125K->114661K(317440K), 0.1448850 secs]  
0.603: [GC 246757K->243133K(375296K), 0.2860800 secs]

如果指定了 - XX:+PrintGCDateStamps，每一行就添加上了绝对的日期和时间。

>2014-01-03T12:08:38.102-0100: [GC 66048K->53077K(251392K), 0.0959470 secs]  
2014-01-03T12:08:38.239-0100: [GC 119125K->114661K(317440K), 0.1421720 secs]  
2014-01-03T12:08:38.513-0100: [GC 246757K->243133K(375296K), 0.2761000 secs]  

如果需要也可以同时使用两个参数。推荐同时使用这两个参数，因为这样在关联不同来源的 GC 日志时很有帮助。

## -Xloggc

缺省的 GC 日志时输出到终端的，使用 - Xloggc: 也可以输出到指定的文件。需要注意这个参数隐式的设置了参数 - XX:+PrintGC 和 - XX:+PrintGCTimeStamps，但为了以防在新版本的 JVM 中有任何变化，我仍建议显示的设置这些参数。

## 可管理的 JVM 参数

一个常常被讨论的问题是在生产环境中 GC 日志是否应该开启。因为它所产生的开销通常都非常有限，因此我的答案是需要开启。但并不一定在启动 JVM 时就必须指定 GC 日志参数。

HotSpot JVM 有一类特别的参数叫做可管理的参数。对于这些参数，可以在运行时修改他们的值。我们这里所讨论的所有参数以及以 “PrintGC” 开头的参数都是可管理的参数。这样在任何时候我们都可以开启或是关闭 GC 日志。比如我们可以使用 JDK 自带的 jinfo 工具来设置这些参数，或者是通过 JMX 客户端调用 HotSpotDiagnostic MXBean 的 setVMOption 方法来设置这些参数。







