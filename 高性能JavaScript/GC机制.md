# 垃圾回收机制

## V8堆内存的划分

### 堆内分配

* 新生代：大部分新对象都在这里
* 老生代：
    * 对象的布局结构信息在 `Map Space` 分配
    * 编译出来的代码在 `Code Space` 分配
    * 太大不能直接放进来的对象在 `Large Object Space` 分配
    * 创建的对象常常被晋升到 `Old Space` 的函数，在这些对象达到一定的生存率（`survival rate`）之后它再创建的对象会被自动在`Old Space` 分配(`pretenuring`)
    * 由新生代晋升(`Promotion`)而来

### 堆外分配
由C++直接分配内存，如`Buffer`。参照后续有关`Buffer和Stream`的小节。

V8堆内外内存分配示意图：
![/assets/堆内外内存分配.jpg](/assets/堆内外内存分配.jpg)

V8的新生代和老生代空间都是分页的（关于分页机制参考上一节的内存管理）。新生代采用连续的分页，而老生代采用离散的通过链表连接的分页。

图片来源：alinode

## GC时机

GC的动机来源：

* 分配新内存的需求（已经分配的内存无法满足新的应用内存需求）触发GC
* 当内存使用率达到一定阀值时触发
* 周期性触发

### 造成Jank

![https://1.bp.blogspot.com/-lcY585e2hUM/VjNBt6H0_FI/AAAAAAAAA2g/0OU5dPeHtmE/s640/Figure%2B1%253A%2BGarbage%2Bcollection%2Bperformed%2Bon%2Bthe%2Bmain%2Bthread..png](https://1.bp.blogspot.com/-lcY585e2hUM/VjNBt6H0_FI/AAAAAAAAA2g/0OU5dPeHtmE/s640/Figure%2B1%253A%2BGarbage%2Bcollection%2Bperformed%2Bon%2Bthe%2Bmain%2Bthread..png)


上图中，GC操作和其他任务执行处于同一线程，如果GC阶段比较长，那么会长时间阻塞其他任务的执行。`Chrome`浏览器有一个叫做`Task Scheduler`的调度器，在浏览器处于闲置状态期间执行GC：

![https://res.infoq.com/news/2015/08/Google-V8/zh/resources/1.png](https://res.infoq.com/news/2015/08/Google-V8/zh/resources/1.png)

此种GC执行机制称为`stop the world`，此种处理方式的好处是简单易控制，不会出现GC任务与其他任务同时操作相同对象的情况。但是这会导致用户体验问题，由于GC时期过长导致动画更新频率低于`60 FPS`就是其影响之一。我们要做的是尽量减少`stop the world`的时间。

关于`Jank`指标: [https://www.chromium.org/developers/design-documents/rendering-benchmarks](https://www.chromium.org/developers/design-documents/rendering-benchmarks)

参考: [https://v8project.blogspot.jp/2015/10/jank-busters-part-one.html](https://v8project.blogspot.jp/2015/10/jank-busters-part-one.html)

## Orinoco的优化

代号为`Orinoco`的任务主要是实现V8的并行、并发GC，减少`Jank`来提高吞吐量。V8采用分代的GC机制，在GC过程中存在大量的对象位置移动以及位置移动后指向它们的指针的引用迁移过程，还有老生代空间的紧缩操作等等。在没有`Orinoco`之前，它们的执行情况如下：

![https://2.bp.blogspot.com/-fXj-FC4Vzb8/VwzwXNWmGSI/AAAAAAAABGw/k-98DC5mDb0GIjsXAUmt217oBMkzU8MJQCLcB/s1600/orinoco1.png](https://2.bp.blogspot.com/-fXj-FC4Vzb8/VwzwXNWmGSI/AAAAAAAABGw/k-98DC5mDb0GIjsXAUmt217oBMkzU8MJQCLcB/s1600/orinoco1.png)

可见GC过程会比较复杂，这种执行方式很容易造成严重的`Jank`。

### 页级别的并行

新生代对象的移动(复制）和老生代的内存紧凑过程都实现了页级别的并行优化，然后新生代和老生代的操作互不影响，同样也是并行的：

![https://1.bp.blogspot.com/-cZ0afz-MbiQ/VwzwgOp_zhI/AAAAAAAABG0/Ux5AzNpJdKkj1E8TO29Gjlj_LhPwFVJigCLcB/s1600/orinoco2.png](https://1.bp.blogspot.com/-cZ0afz-MbiQ/VwzwgOp_zhI/AAAAAAAABG0/Ux5AzNpJdKkj1E8TO29Gjlj_LhPwFVJigCLcB/s1600/orinoco2.png)

### 跟踪指针的优化

当对象从某个位置移动到新的位置时，指向它的指针们的指向也要发生变化，如果使用遍历堆的方法找到所有的指针效率很低下，所以V8使用`remembered set`来记录那些可能会产生变化的指针（`intersting`指针)，那些指向新生代对象的指针，以及指向处于碎片化的老生代区某个对象的指针都会被记录。在之前的V8版本中，记录方式如图：

![https://2.bp.blogspot.com/-jUyzVS1LOks/Vwzw4r8u3LI/AAAAAAAABG4/5ByNdD18KrskZfLQScT5J8prY5YYHf7UgCLcB/s1600/oricoco3.png](https://2.bp.blogspot.com/-jUyzVS1LOks/Vwzw4r8u3LI/AAAAAAAABG4/5ByNdD18KrskZfLQScT5J8prY5YYHf7UgCLcB/s1600/oricoco3.png)

通过数组或者`store buffers`的形式记录，且采用`write barrier`机制。

此种方式可能导致`remembered set`里存在重复的记录，会导致对指针操作的优化操作（如并行处理）失效，`Orinoco`采用下图所示的记录方式：

![https://3.bp.blogspot.com/-mqjBA5mAK4Y/Vw0JTqk4_VI/AAAAAAAABHM/3VCmHpe8nwMmcFh8-pIymS2-JxNVvudcQCLcB/s1600/orinoco4.png](https://3.bp.blogspot.com/-mqjBA5mAK4Y/Vw0JTqk4_VI/AAAAAAAABHM/3VCmHpe8nwMmcFh8-pIymS2-JxNVvudcQCLcB/s1600/orinoco4.png)

### black allocation
`black allocation` 将所有新出现在 `Old Space` 的对象（包括`pretentured` 的分配或者晋升）直接标记为黑色，放在特殊的内存页（`black page`）中，这个内存页里只有`black objects`，因此一定能活过下一次 GC。这样做可以一定程度上减轻 marking 的负担，即使猜错了，下下轮 marking 前这些对象又会先刷白，只逃过一次 GC 所以造成的影响也不大。


## GC算法

### 新生代 `Scavenge`

`Scavenge`的基本思想是用空间换时间，将新生代空间平分成两个`semispace`，`live`对象和死亡对象在它们之间不断交换迁移：
![/assets/Scavenge_gc.jpg](/assets/Scavenge_gc.jpg)

新生代的回收过程一般为`stop the world`在这个过程中会有上面提到的优化机制：对象复制迁移的页级别并行和指针指向迁移的并行优化。

此算法不用担心有内存碎片的情况，由于存活的对象比较少，因此`stop the world`的过程比较短。但是不适合大内存对象和老生代对象。

### 老生代 `Mark-Sweep／Mark-Compact`
在`New Space`里存活两轮时，就会晋升到`Old Space`。

老生代的回收过程分为两个部分：标记和清除（紧凑）。

标记过程负责标注出哪些对象是死亡对象将要被回收，

紧凑过程也是一个`stop the world` 的过程。紧凑的过程会用到上面提到的优化机制：对象复制迁移的页级别并行和指针指向迁移的并行优化。

![/assets/sweeping与compacting.jpg](/assets/sweeping与compacting.jpg)

#### 优化一：增量标记
标记死亡对象过程会阻塞主线程，如果采用一次性`stop the world`方式会增加`Jank`，可以采用增量方式进行，分解成多次`stop the world`过程。

#### 优化二：`lazy sweeping, concurrent sweeping, parallel sweeping`

当标记完成后，不立即执行`sweep`操作，即`lazy sweep`，然后由于要执行`sweep`操作的对象是死亡对象，确认不会被主线程其他任务访问，所以此时`sweep`操作可以和其他任务并发执行（因为有少量同步过程，所以不叫做并行），即`concurrent sweeping`，然后可以使用多个线程同时进行`sweep`即为`parallel sweeping`。

这样，整个V8的垃圾回收执行情况如下：

![/assets/node_gc.jpg](/assets/node_gc.jpg)


// todo