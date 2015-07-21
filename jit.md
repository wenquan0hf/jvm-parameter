# 参数分类和即时（JIT）编译器诊断

作者： [PATRICK PESCHLOW](https://blog.codecentric.de/en/2012/07/useful-jvm-flags-part-2-flag-categories-and-jit-compiler-diagnostics/%20)       [原文地址](https://blog.codecentric.de/en/author/patrick-peschlow/)    译者：赵峰 校对：许巧辉

在这个系列的第二部分，我来介绍一下 HotSpot JVM 提供的不同类别的参数。我同样会讨论一些关于 JIT 编译器诊断的有趣参数。

JVM 参数分类

HotSpot JVM 提供了三类参数。第一类包括了标准参数。顾名思义，标准参数中包括功能和输出的参数都是很稳定的，很可能在将来的 JVM 版本中不会改变。你可以用 java 命令（或者是用 java -help）检索出所有标准参数。我们在第一部分中已经见到过一些标准参数，例如：-server。

第二类是 X 参数，非标准化的参数在将来的版本中可能会改变。所有的这类参数都以 - X 开始，并且可以用 java -X 来检索。注意，不能保证所有参数都可以被检索出来，其中就没有 - Xcomp。

第三类是包含 XX 参数（到目前为止最多的），它们同样不是标准的，甚至很长一段时间内不被列出来（最近，这种情况有改变 ，我们将在本系列的第三部分中讨论它们）。然而，在实际情况中 X 参数和 XX 参数并没有什么不同。X 参数的功能是十分稳定的，然而很多 XX 参数仍在实验当中（主要是 JVM 的开发者用于 debugging 和调优 JVM 自身的实现）。值的一读的介绍非标准参数的文档 [HotSpot JVM documentation](http://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html)，其中明确的指出 XX 参数不应该在不了解的情况下使用。这是真的，并且我认为这个建议同样适用于 X 参数（同样一些标准参数也是）。不管类别是什么，在使用参数之前应该先了解它可能产生的影响。

用一句话来说明 XX 参数的语法。所有的 XX 参数都以”-XX:” 开始，但是随后的语法不同，取决于参数的类型。

对于布尔类型的参数，我们有”+” 或”-“，然后才设置 JVM 选项的实际名称。例如，-XX:+<name> 用于激活 <name> 选项，而 - XX:-<name> 用于注销选项。
对于需要非布尔值的参数，如 string 或者 integer，我们先写参数的名称，后面加上”=”，最后赋值。例如，  -XX:<name>=<value> 给 <name> 赋值 <value>。
现在让我们来看看 JIT 编译方面的一些 XX 参数。

-XX:+PrintCompilation and -XX:+CITime

当一个 Java 应用运行时，非常容易查看 JIT 编译工作。通过设置 - XX:+PrintCompilation，我们可以简单的输出一些关于从字节码转化成本地代码的编译过程。我们来看一个服务端 VM 运行的例子：

<table border="1" cellspacing="0" cellpadding="2">
<tbody>
<tr>
<td>
<pre>$ java -server -XX:+PrintCompilation Benchmark
  1       java.lang.String::hashCode (64 bytes)
  2       java.lang.AbstractStringBuilder::stringSizeOfInt (21 bytes)
  3       java.lang.Integer::getChars (131 bytes)
  4       java.lang.Object::&lt;init&gt; (1 bytes)
---   n   java.lang.System::arraycopy (static)
  5       java.util.HashMap::indexFor (6 bytes)
  6       java.lang.Math::min (11 bytes)
  7       java.lang.String::getChars (66 bytes)
  8       java.lang.AbstractStringBuilder::append (60 bytes)
  9       java.lang.String::&lt;init&gt; (72 bytes)
 10       java.util.Arrays::copyOfRange (63 bytes)
 11       java.lang.StringBuilder::append (8 bytes)
 12       java.lang.AbstractStringBuilder::&lt;init&gt; (12 bytes)
 13       java.lang.StringBuilder::toString (17 bytes)
 14       java.lang.StringBuilder::&lt;init&gt; (18 bytes)
 15       java.lang.StringBuilder::append (8 bytes)
[...]
 29       java.util.regex.Matcher::reset (83 bytes)</pre>
</td>
</tr>
</tbody>
</table>

每当一个方法被编译，就输出一行 - XX:+PrintCompilation。每行都包含顺序号（唯一的编译任务 ID）和已编译方法的名称和大小。因此，顺序号 1，代表编译 String 类中的 hashCode 方法到原生代码的信息。根据方法的类型和编译任务打印额外的信息。例如，本地的包装方法前方会有”n” 参数，像上面的 System::arraycopy 一样。注意这样的方法不会包含顺序号和方法占用的大小，因为它不需要编译为本地代码。同样可以看到被重复编译的方法，例如 StringBuilder::append 顺序号为 11 和 15。输出在顺序号 29 时停止 ，这表明在这个 Java 应用运行时总共需要编译 29 个方法。

没有官方的文档关于 - XX:+PrintCompilation，但是[这个描述](https://gist.github.com/rednaxelafx/1165804#file_notes.md)是对于此参数比较好的。我推荐更深入学习一下。

JIT 编译器输出帮助我们理解客户端 VM 与服务端 VM 的一些区别。用服务端 VM，我们的应用例子输出了 29 行，同样用客户端 VM，我们会得到 55 行。这看起来可能很怪，因为服务端 VM 应该比客户端 VM 做了 “更多” 的编译。然而，由于它们各自的默认设置，服务端 VM 在判断方法是不是热点和需不需要编译时比客户端 VM 观察方法的时间更长。因此，在使用服务端 VM 时，一些潜在的方法会稍后编译就不奇怪了。

通过另外设置 - XX:+CITime，我们可以在 JVM 关闭时得到各种编译的统计信息。让我们看一下一个特定部分的统计：

<table border="1" cellspacing="0" cellpadding="2">
<tbody>
<tr>
<td>
<pre>$ java -server -XX:+CITime Benchmark
[...]
Accumulated compiler times (for compiled methods only)
------------------------------------------------
  Total compilation time   :  0.178 s
    Standard compilation   :  0.129 s, Average : 0.004
    On stack replacement   :  0.049 s, Average : 0.024
[...]</pre>
</td>
</tr>
</tbody>
</table>

总共用了 0.178 s（在 29 个编译任务上）。这些，”on stack replacement” 占用了 0.049 s，即编译的方法目前在堆栈上用去的时间。这种技术并不是简单的实现性能显示，实际上它是非常重要的。没有”on stack replacement”，方法如果要执行很长时间（比如，它们包含了一个长时间运行的循环），它们运行时将不会被它们编译过的副本替换。

再一次，客户端 VM 与服务端 VM 的比较是非常有趣的。客户端 VM 相应的数据表明，即使有 55 个方法被编译了，但这些编译总共用了只有 0.021 s。服务端 VM 做的编译少但是用的时间却比客户端 VM 多。这个原因是，使用服务端 VM 在生成本地代码时执行了更多的优化。

在本系列的第一部分，我们已经学了 - Xint 和 - Xcomp 参数。结合使用 - XX:+PrintCompilation 和 - XX:+CITime，在这两个情况下（校对者注，客户端 VM 与服务端 VM），我们能对 JIT 编译器的行为有更好的了解。使用 - Xint，-XX:+PrintCompilation 在这两种情况下会产生 0 行输出。同样的，使用 - XX:+CITime 时，证实在编译上没有花费时间。现在换用 - Xcomp，输出就完全不同了。在使用客户端 VM 时会产生 726 行输出，然后没有更多的，这是因为每个相关的方法都被编译了。使用服务端 VM，我们甚至能得到 993 行输出，这告诉我们更积极的优化被执行了。同样，JVM 拆机 (JVM teardown) 时打印出的统计显示了两个 VM 的巨大不同。考虑服务端 VM 的运行：

<table border="1" cellspacing="0" cellpadding="2">
<tbody>
<tr>
<td>
<pre>$ java -server -Xcomp -XX:+CITime Benchmark
[...]
Accumulated compiler times (for compiled methods only)
------------------------------------------------
  Total compilation time   :  1.567 s
    Standard compilation   :  1.567 s, Average : 0.002
    On stack replacement   :  0.000 s, Average : -1.#IO
[...]</pre>
</td>
</tr>
</tbody>
</table>

使用 - Xcomp 编译用了 1.567 s，这是使用默认设置（即，混合模式）的 10 倍。同样，应用程序的运行速度要比用混合模式的慢。相比较之下，客户端 VM 使用 - Xcomp 编译 726 个方法只用了 0.208 s，甚至低于使用 - Xcomp 的服务端 VM。

补充一点，这里没有”on stack replacement” 发生，因为每一个方法在第一次调用时被编译了。损坏的输出 “Average: -1.#IO”（正确的是: 0）再一次表明了，非标准化的输出参数不是非常可靠。

-XX:+UnlockExperimentalVMOptions

有些时候当设置一个特定的 JVM 参数时，JVM 会在输出 “Unrecognized VM option” 后终止。如果发生了这种情况，你应该首先检查你是否输错了参数。然而，如果参数输入是正确的，并且 JVM 并不识别，你或许需要设置 - XX:+UnlockExperimentalVMOptions 来解锁参数。我不是非常清楚这个安全机制的作用，但我猜想这个参数如果不正确使用可能会对 JVM 的稳定性有影响（例如，他们可能会过多的写入 debug 输出的一些日志文件）。

有一些参数只是在 JVM 开发时用，并不实际用于 Java 应用。如果一个参数不能被 -XX:+UnlockExperimentalVMOptions 开启，但是你真的需要使用它，此时你可以尝试使用 debug 版本的 JVM。对于 Java 6 HotSpot JVM 你可以从[这里找到](https://java.net/projects/jdk6/download.html)。

-XX:+LogCompilation and -XX:+PrintOptoAssembly

如果你在一个场景中发现使用 -XX:+PrintCompilation，不能够给你足够详细的信息，你可以使用 -XX:+LogCompilation 把扩展的编译输出写到 “hotspot.log” 文件中。除了编译方法的很多细节之外，你也可以看到编译器线程启动的任务。注意 - XX:+LogCompilation 需要使用 - XX:+UnlockExperimentalVMOptions 来解锁。

JVM 甚至允许我们看到从字节码编译生成到本地代码。使用 - XX:+PrintOptoAssembly，由编译器线程生成的本地代码被输出并写到 “hotspot.log” 文件中。使用这个参数要求运行的服务端 VM 是 debug 版本。我们可以研究 - XX:+PrintOptoAssembly 的输出，以至于了解 JVM 实际执行什么样的优化，例如，关于死代码的消除。一个非常有趣的文章提供了一个[例子](https://weblogs.java.net/blog/2008/03/30/deep-dive-assembly-code-java)。

关于 XX 参数的更多信息

如果这篇文章勾起了你的兴趣，你可以自己看一下 HotSpot JVM 的 XX 参数。这里是一个很好的[起点](http://stas-blogspot.blogspot.de/2011/07/most-complete-list-of-xx-options-for.html)。

