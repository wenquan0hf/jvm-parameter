# 打印所有 XX 参数及值

原文地址：[https://blog.codecentric.de/en/2012/07/useful-jvm-flags-part-3-printing-all-xx-flags-and-their-values/](https://blog.codecentric.de/en/2012/07/useful-jvm-flags-part-3-printing-all-xx-flags-and-their-values/)

**译者**：李洪柱     **校对**：方腾飞

本篇文章基于 Java 6（update 21oder 21 之后）版本， HotSpot JVM 提供给了两个新的参数，在 JVM 启动后，在命令行中可以输出所有 XX 参数和值。

<pre>-XX:+PrintFlagsFinal and -XX:+PrintFlagsInitial</pre>

让我们现在就了解一下新参数的输出。以 -client 作为参数的 -XX:+PrintFlagsFinal   的结果是一个按字母排序的 590 个参数表格（注意，每个 release 版本参数的数量会不一样）

<pre>$ java <span style="color: #660033;">-client</span> -XX:+PrintFlagsFinal Benchmark
<span style="font-weight: bold; color: #7a0874;">[</span>Global flags<span style="font-weight: bold; color: #7a0874;">]</span>
uintx AdaptivePermSizeWeight               = 20               <span style="font-weight: bold; color: #7a0874;">{</span>product<span style="font-weight: bold; color: #7a0874;">}</span>
uintx AdaptiveSizeDecrementScaleFactor     = 4                <span style="font-weight: bold; color: #7a0874;">{</span>product<span style="font-weight: bold; color: #7a0874;">}</span>
uintx AdaptiveSizeMajorGCDecayTimeScale    = 10               <span style="font-weight: bold; color: #7a0874;">{</span>product<span style="font-weight: bold; color: #7a0874;">}</span>
uintx AdaptiveSizePausePolicy              = 0                <span style="font-weight: bold; color: #7a0874;">{</span>product<span style="font-weight: bold; color: #7a0874;">}</span><span style="font-weight: bold; color: #7a0874;">[</span>...<span style="font-weight: bold; color: #7a0874;">]</span>
uintx YoungGenerationSizeSupplementDecay   = 8                <span style="font-weight: bold; color: #7a0874;">{</span>product<span style="font-weight: bold; color: #7a0874;">}</span>
uintx YoungPLABSize                        = 4096             <span style="font-weight: bold; color: #7a0874;">{</span>product<span style="font-weight: bold; color: #7a0874;">}</span>
 bool ZeroTLAB                             = <span style="font-weight: bold; color: #c20cb9;">false</span>            <span style="font-weight: bold; color: #7a0874;">{</span>product<span style="font-weight: bold; color: #7a0874;">}</span>
 intx hashCode                             = 0                <span style="font-weight: bold; color: #7a0874;">{</span>product<span style="font-weight: bold; color: #7a0874;">}</span></pre>

(校对注：你可以尝试在命令行输入上面的命令，亲自实现下)

表格的每一行包括五列，来表示一个 XX 参数。第一列表示参数的数据类型，第二列是名称，第四列为值，第五列是参数的类别。第三列”=” 表示第四列是参数的默认值，而“:=” 表明了参数被用户或者 JVM 赋值了。

注意对于这个例子我只是用了 Benchmark 类，因为这个系列前面的章节也是用的这个类。甚至没有一个主类的情况下你能得到相同的输出，通过运行 java 带另外的参数 -version. 现在让我们检查下 server VM 提供了多少个参数。我们也能指定参数 -XX:+UnlockExperimentalVMOptions 和 -XX:+UnlockDiagnosticVMOptions；来解锁任何额外的隐藏参数。

<pre><span style="color: #666666;">$ </span>java <span style="color: #660033;">-server</span> -XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions -XX:+PrintFlagsFinal Benchmark
</pre>

724 个参数，让我们看一眼那些已经被赋值的参数。

<pre>$ java <span style="color: #660033;">-server</span> -XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions -XX:+PrintFlagsFinal Benchmark <span style="font-weight: bold;">|</span> <span style="font-weight: bold; color: #c20cb9;">grep</span> <span style="color: #ff0000;">":"</span>
uintx InitialHeapSize                     := 57505088         <span style="font-weight: bold; color: #7a0874;">{</span>product<span style="font-weight: bold; color: #7a0874;">}</span>
uintx MaxHeapSize                         := 920649728        <span style="font-weight: bold; color: #7a0874;">{</span>product<span style="font-weight: bold; color: #7a0874;">}</span>
uintx ParallelGCThreads                   := 4                <span style="font-weight: bold; color: #7a0874;">{</span>product<span style="font-weight: bold; color: #7a0874;">}</span>
 bool PrintFlagsFinal                     := <span style="font-weight: bold; color: #c20cb9;">true</span>             <span style="font-weight: bold; color: #7a0874;">{</span>product<span style="font-weight: bold; color: #7a0874;">}</span>
 bool UseParallelGC                       := <span style="font-weight: bold; color: #c20cb9;">true</span>             <span style="font-weight: bold; color: #7a0874;">{</span>product<span style="font-weight: bold; color: #7a0874;">}</span></pre>

（校对注：这个命令非常有用）我们仅设置一个自己的参数 -XX:+PrintFlagsFinal。其他参数通过 server VM 基于系统设置的，以便以合适的堆大小和 GC 设置运行。

如果我们只想看下所有 XX 参数的默认值，能够用一个相关的参数，-XX:+PrintFlagsInitial  。 用 -XX:+PrintFlagsInitial, 只是展示了第三列为 “=” 的数据（也包括那些被设置其他值的参数）。

然而，注意当与 - XX:+PrintFlagsFinal 对比的时候，一些参数会丢失，大概因为这些参数是动态创建的。

研究表格的内容是很有意思的，通过比较 client 和 server VM 的行为，很明显了解哪些参数会影响其他的参数。有兴趣的读者，可以看一下这篇不错文章 [Inspecting HotSpot JVM Options](http://q-redux.blogspot.com/2011/01/inspecting-hotspot-jvm-options.html)。这个文章主要解释了第五列的参数类别。

**-XX:+PrintCommandLineFlags**
让我们看下另外一个参数，事实上这个参数非常有用: -XX:+PrintCommandLineFlags。这个参数让 JVM 打印出那些已经被用户或者 JVM 设置过的详细的 XX 参数的名称和值。

换句话说，它列举出 -XX:+PrintFlagsFinal的结果中第三列有“:=”的参数。以这种方式，我们可以用 - XX:+PrintCommandLineFlags 作为快捷方式来查看修改过的参数。看下面的例子。

<pre>$ java <span style="color: #660033;">-server</span> -XX:+PrintCommandLineFlags Benchmark 
</pre>

<pre> -XX:<span style="color: #007800;">InitialHeapSize</span>=57505088 -XX:<span style="color: #007800;">MaxHeapSize</span>=920081408 -XX:<span style="color: #007800;">ParallelGCThreads</span>=4 -XX:+PrintCommandLineFlags -XX:+UseParallelGC
</pre>

现在如果我们每次启动 java 程序的时候设置 -XX:+PrintCommandLineFlags 并且输出到日志文件上，这样会记录下我们设置的 JVM 参数对应用程序性能的影响。类似于 -showversion(见 Part1)，我建议 –XX:+PrintCommandLineFlags 这个参数应该总是设置在 JVM 启动的配置项里。因为你从不知道你什么时候会需要这些信息。

奇怪的是在这个例子中，通过 -XX:+PrintCommandLineFlags 列出堆的最大值会比通过 - XX:+PrintFlagsFinal 列举出的相应值小一点。如果谁知道两者之间不同的原因，请告诉我。