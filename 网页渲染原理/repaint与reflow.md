# repaint与reflow
在前面小节，我们对网页渲染过程做了介绍，其中最后两步就是layout与paint，当渲染对象被创建并添加到树中，它们并没有位置和大小，计算这些值的过程称为layout或reflow。绘制阶段，遍历渲染树并调用渲染对象的paint方法将它们的内容显示在屏幕上，绘制使用UI基础组件。
## 何时发生？
由两者的定义我们可以知道在元素的大小和位置发生变化时都会触发reflow，且也会引起repaint，当元素的位置和大小没有变化，如改变背景颜色等等，只会进行重新repaint,如修改`opacity`, `background-color`, `visibility`, `outline`等。

reflow比repaint开销更大，修改一个元素的大小或位置会影响它的子节点，父节点，兄弟节点甚至是整个文档。例如修改页面字体大小，将会导致整个文档reflow，在body最前面添加一个元素会导致后面所有元素reflow。

具体来说：
* 添加、删除元素
* 更改元素的display:none或显示引起reflow
* 增加或删除元素style规则，不同规则影响不同，如修改width会引起reflow
* 移动元素位置
* CSS3动画，每一帧都会引起reflow(可能有硬件加速)
* 用户行为如改变浏览器字体大小、改变窗口大小、在输入框输入文字、触发`:hover`等等

## 浏览器优化策略

* 采用Dirty bit系统，只针对具有dirty标记的元素进行重新layout，即增量layout，且此过程是异步的.
* Firefox为增量layout生成了reflow队列，以及一个调度执行这些批处理命令。Webkit也有一个计时器用来执行增量layout－遍历树，为dirty状态的渲染对象重新布局。意思是会将多个reflow操作汇集到一定数量后再一起处理，减轻压力
* 如果元素只是位置发生变化，其大小从缓存里读取，就不用再次计算

针对上面的reflow队列，当访问以下style属性时，浏览器为了给你最精确的值，需要flush队列，因为队列中可能会有影响到这些值的操作：

* offsetTop, offsetLeft, offsetWidth, offsetHeight
* scrollTop/Left/Width/Height
* clientTop/Left/Width/Height
* width,height
* 请求了getComputedStyle(), 或者 ie的 currentStyle

## 减少reflow和repaint

// todo