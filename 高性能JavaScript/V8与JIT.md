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
![http://7xsi10.com1.z0.glb.clouddn.com/01-02-perf_graph10-768x633.png](http://7xsi10.com1.z0.glb.clouddn.com/01-02-perf_graph10-768x633.png)

### 解释与编译
通常有两种方法将高级语言代码转换为机器代码，`解释`与`编译`。

解释器一行一行的翻译代码，因此我们不需要等待所有代码都编译完成便可开始执行，能够尽快看到执行效果，这非常适合Web的场景；但是解释执行的问题是，解释器可能针对同一行代码重复翻译多次，如循环语句。然后解释器在翻译过程中不能花太多时间在各种优化机制上面。

编译的方式则相反，其在执行前将完整的程序翻译成机器码，执行的时候就不用考虑翻译的事情，所以就不会出现同一段代码重复翻译的问题。然后编译方式可以在翻译的时候花更多的事情在优化上面。

### 结合两种方式

为了解释器的问题即可能对相同代码重复翻译多次以及优化问题，js引擎可以在适当的时候混入编译方式。

不同的浏览器的JIT实现方式略有区别，但基本思想一致。即在js引擎中加入一个叫做`monitor`的部分来监控代码执行，并记录下每部分代码执行的次数以及使用的类型。如果`monitor`发现同一段代码执行多次，就会将此段代码标记为`warm`，如果执行次数非常多，就将其标记为`hot`。`warm`标记的代码会被放到`baseline compiler`，`hot`标记的代码会被放到`optimizing compiler`：

![http://7xsi10.com1.z0.glb.clouddn.com/02-06-jit09-768x560.png](http://7xsi10.com1.z0.glb.clouddn.com/02-06-jit09-768x560.png)

### baseline cimpiler
当某段代码被标记为`warm`，将不直接通过解释执行，而是通过`baseline cimpiler`，`baseline compiler`会存储这段代码的编译结果：

![http://7xsi10.com1.z0.glb.clouddn.com/02-05-jit06-768x565.png](http://7xsi10.com1.z0.glb.clouddn.com/02-05-jit06-768x565.png)

js引擎会为图示中函数的每一行代码生成一个"桩"，"桩"中包含代码行号和变量类型信息，当`monitor`发现同一段代码被执行，且其中变量类型相同，则会输出它之前已经编译好版本。

编译器还能做一系列的优化工作，优化工作会花费一定的时间从而阻塞代码执行，但是如果某段代码执行的频率确实很高，那么牺牲一定时间来做这些优化工作是值得的。

### Optimizing compiler
如果某段代码被标记为`hot`，这段代码会被放到`Optimizing compiler`，`Optimizing compiler`会对代码进行更多优化，产生一个执行更快的编译版本并进行存储：

![http://7xsi10.com1.z0.glb.clouddn.com/02-06-jit09-768x560.png](http://7xsi10.com1.z0.glb.clouddn.com/02-06-jit09-768x560.png)

但是`Optimizing compiler`是基于一定假设的，比如，它假设通过某一构造器实例化出来的对象具有完全一样的结构，即拥有的属性相同，且属性顺序相同。

`Optimizing compiler`使用`monitor`收集到的代码信息来进行判断代码是否符合上述假设，在一个循环语句中，如果某个对象在前面的迭代中满足上面的条件，`Optimizing compiler`就会认为，在后续的迭代中，此对象依然会满足条件。

但是由于JS的动态性，我们不能完全保证此假设成立，尽管是在同一循环期间，对象的结构也会发生变化。所以经过`Optimizing compiler`编译的编译版本在执行之前需要进行检查，如果假设错误，将会丢弃此编译版本，然后将代码交回给解释器或者`baseline cimpiler`，此种行为被称为（弃优化）:

![http://7xsi10.com1.z0.glb.clouddn.com/02-07-jit11-768x555.png](http://7xsi10.com1.z0.glb.clouddn.com/02-07-jit11-768x555.png)

可见，`Optimizing compiler`机制可能会造成一些性能问题，如果js引擎针对某段代码在优化和弃优化直接不断循环，将会导致代码执行速度比没有使用优化机制更慢。

所以，浏览器在某段代码在优化和弃优化直接徘徊到一定次数后，将会放弃优化。

### 优化实例：类型特化

`Optimizing compiler`针对代码优化有许多手段，其中比较典型的一个例子就是`类型特化`(`type specialization`)，比如下面一段代码：
```js
function arraySum(arr) {
  var sum = 0;
  for (var i = 0; i < arr.length; i++) {
    sum += arr[i];
  }
}
```
上面的代码看似简单，但是由于js是动态类型语言，在执行上面代码时，需要做许多额外的工作，当上面的代码被标记为`warm`时，将被交给`baseline compiler`，`baseline compiler`会为上面的每一行代码创建"桩"，比如上面的`sum += arr[i]`，来表示整形相加合赋值操作。

但是，我们不能保证`sum`何`arr[i]`一定是整形，因为有可能`arr`的某一项并不是整形，如果其中一项是字符串类型，那么上面的加法操作执行策略就会不一样，需要被编译成新的版本。`JIT`的策略是对每一段代码产生多个"桩"，如果该段代码在执行期间变量类型不变，只会产生一个"桩"，否则会有多个"桩"。这样，该段代码每次执行前都需要进行一个"决策"过程:

![http://7xsi10.com1.z0.glb.clouddn.com/02-08-decision_tree01-768x394.png](http://7xsi10.com1.z0.glb.clouddn.com/02-08-decision_tree01-768x394.png)

这样引擎将会花大量时间在询问相同问题（决策）过程中：

![http://7xsi10.com1.z0.glb.clouddn.com/02-09-jit_loop02-768x496.png](http://7xsi10.com1.z0.glb.clouddn.com/02-09-jit_loop02-768x496.png)

而`Optimizing compiler`的一个优化策略就是简化这样一个决策过程，对于其中一些变量的类型检查提前到循环之前:

![http://7xsi10.com1.z0.glb.clouddn.com/02-10-jit_loop02-768x488.png](http://7xsi10.com1.z0.glb.clouddn.com/02-10-jit_loop02-768x488.png)

有些`JIT`实现会在这方面做更进一步优化，比如firefox定义了一种只含有整数的特别数组，如果`arr`满足条件，上面图中每次执行前关于`arr[i]`的类型检查也可省去。

### 存在的问题
上面的优化策略也会带来一些新的问题，其中之一就是会带来额外的开销：

* 优化和弃优化的开销
* `monitor`用于记录以及弃优化发生时恢复先前信息的开销
* 存储`baseline`和优化版本的代码需要耗费一定内存

针对上面的问题，还可以通过进一步优化来解决，最近一段时间炒得火热的`WebAssembly`便是一种手段。我们会在下一章节进一步探讨。

此篇文章主要内容翻译自文章:
[https://hacks.mozilla.org/2017/02/a-crash-course-in-just-in-time-jit-compilers/](https://hacks.mozilla.org/2017/02/a-crash-course-in-just-in-time-jit-compilers/)