# V8与JIT
在众多的JavaScript引擎中，V8脱引而出，其优良的性能得到广大开发者的喜爱。

Google将webkit中集成的js引擎替换为V8：
![http://my.csdn.net/uploads/201207/18/1342625901_1361.jpg](http://my.csdn.net/uploads/201207/18/1342625901_1361.jpg)

我们熟悉的Node.js就使用了V8引擎来解析执行js，[https://benchmarksgame.alioth.debian.org/u64q/compare.php?lang=node](https://benchmarksgame.alioth.debian.org/u64q/compare.php?lang=node)可以通过此链接查看Node与其他语言的基准测试比较。可以看到在V8中执行的JS可以是吊打其他脚本语言，与Java相比也相差不大。

V8采取了一系列优化机制提升JS效率，我们将从`JIT`、`垃圾回收`等方面去了解V8内核。

## JIT
即时编译（英语：Just-in-time compilation），动态编译的一种形式，是一种提高程序运行效率的方法。通常，程序有两种运行方式：`静态编译`与`动态解释`。静态编译的程序在执行前全部被翻译为机器码，而解释执行的则是一句一句边运行边翻译。

即时编译器则混合了这二者，一句一句编译源代码，但是会将翻译过的代码缓存起来以降低性能损耗。相对于静态编译代码，即时编译的代码可以处理延迟绑定并增强安全性。

JIT运用的地方很多，如`.NET`、`JVM`、`PyPy`等。在多个JS引擎均使用JIT技术提升js执行速度：
![https://cloud.netlifyusercontent.com/assets/344dbf88-fdf9-42bb-adb4-46f01eedd629/e4c17eec-5fcf-4b37-aa90-4180d4ac4dca/03-perf-graph10-opt.png](https://cloud.netlifyusercontent.com/assets/344dbf88-fdf9-42bb-adb4-46f01eedd629/e4c17eec-5fcf-4b37-aa90-4180d4ac4dca/03-perf-graph10-opt.png)

### 解释与编译
通常有两种方法将高级语言代码转换为机器代码，`解释`与`编译`。

解释器一行一行的翻译代码，因此我们不需要等待所有代码都编译完成便可开始执行，能够尽快看到执行效果，这非常适合Web的场景；但是解释执行的问题是，解释器可能针对同一行代码重复翻译多次，如循环语句。然后解释器在翻译过程中不能花太多时间在各种优化机制上面。

编译的方式则相反，其在执行前将完整的程序翻译成机器码，执行的时候就不用考虑翻译的事情，所以就不会出现同一段代码重复翻译的问题。然后编译方式可以在翻译的时候花更多的事情在优化上面。

### 结合两种方式

// todo