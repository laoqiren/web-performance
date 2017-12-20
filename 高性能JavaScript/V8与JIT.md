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

为了解释器的问题即可能对相同代码重复翻译多次以及优化问题，js引擎可以在适当的时候混入编译方式。

不同的浏览器的JIT实现方式略有区别，但基本思想一致。即在js引擎中加入一个叫做`monitor`的部分来监控代码执行，并记录下每部分代码执行的次数以及使用的类型。如果`monitor`发现同一段代码执行多次，就会将此段代码标记为`warm`，如果执行次数非常多，就将其标记为`hot`。`warm`标记的代码会被放到`baseline compiler`，`hot`标记的代码会被放到`optimizing compiler`：

![https://cloud.netlifyusercontent.com/assets/344dbf88-fdf9-42bb-adb4-46f01eedd629/099b0385-1354-4c2f-adfd-46cbba596682/07-jit09-opt.png](https://cloud.netlifyusercontent.com/assets/344dbf88-fdf9-42bb-adb4-46f01eedd629/099b0385-1354-4c2f-adfd-46cbba596682/07-jit09-opt.png)

### baseline cimpiler
当某段代码被标记为`warm`，将不直接通过解释执行，而是通过`baseline cimpiler`，`baseline compiler`会存储这段代码的编译结果：

![https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/02/02-05-jit06.png](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/02/02-05-jit06.png)

js引擎会为图示中函数的每一行代码生成一个"桩"，"桩"中包含代码行号和变量类型信息，当`monitor`发现同一段代码被执行，且其中变量类型相同，则会输出它之前已经编译好版本。

编译器还能做一系列的优化工作，优化工作会花费一定的时间从而阻塞代码执行，但是如果某段代码执行的频率确实很高，那么牺牲一定时间来做这些优化工作是值得的。

### Optimizing compiler
如果某段代码被标记为`hot`，这段代码会被放到`Optimizing compiler`，`Optimizing compiler`会对代码进行更多优化，产生一个执行更快的编译版本并进行存储：

![https://hacks.mozilla.org/files/2017/02/02-06-jit09-768x560.png](https://hacks.mozilla.org/files/2017/02/02-06-jit09-768x560.png)

但是`Optimizing compiler`是基于一定假设的，比如，它假设通过某一构造器实例化出来的对象具有完全一样的结构，即拥有的属性相同，且属性顺序相同。

`Optimizing compiler`使用`monitor`收集到的代码信息来进行判断代码是否符合上述假设，在一个循环语句中，如果某个对象在前面的迭代中满足上面的条件，`Optimizing compiler`就会认为，在后续的迭代中，此对象依然会满足条件。

但是由于JS的动态性，我们不能完全保证此假设成立，尽管是在同一循环期间，对象的结构也会发生变化。所以经过`Optimizing compiler`编译的编译版本在执行之前需要进行检查，如果假设错误，将会丢弃此编译版本，然后将代码交回给解释器或者`baseline cimpiler`，此种行为被称为（弃优化）:

![https://hacks.mozilla.org/files/2017/02/02-07-jit11-768x555.png](https://hacks.mozilla.org/files/2017/02/02-07-jit11-768x555.png)

可见，`Optimizing compiler`机制可能会造成一些性能问题，如果js引擎针对某段代码在优化和弃优化直接不断循环，将会导致代码执行速度比没有使用优化机制更慢。

所以，浏览器在某段代码在优化和弃优化直接徘徊到一定次数后，将会放弃优化。

// todo