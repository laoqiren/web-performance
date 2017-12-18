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

### 布局技术最佳实践
* 少使用内联样式
* 表格数据使用：`table-layout: fixed`
* `flexbox`布局可能带来性能问题

### 减少样式规则
许多时候我们在页面上引入的规则很多并没有用到，例如使用bootstrap等，除了按需引入外，我们还可以通过一些自动化工具去除页面中没有使用到的规则。如`gulp-uncss`:

```js
var gulp = require('gulp');
var postcss = require('gulp-postcss');
var uncss = require('postcss-uncss');

gulp.task('default', function () {
    var plugins = [
        uncss({
            html: ['index.html', 'posts/**/*.html', 'http://example.com']
        }),
    ];
    return gulp.src('./src/*.css')
        .pipe(postcss(plugins))
        .pipe(gulp.dest('./dest'));
});
```
详情参考文档：[https://github.com/ben-eb/gulp-uncss](https://github.com/ben-eb/gulp-uncss)

### 减少DOM树嵌套层次
减少不必要的嵌套层可以加快reflow过程。

### 批量操作
如果以下面的方式更新元素：
```js
var myelement = document.getElementById('myelement');
myelement.width = '100px';
myelement.height = '200px';
myelement.style.margin = '10px';
```
会造成三次reflow(不考虑浏览器优化)

改善方式是通过改变元素的`class`属性来应用样式表里不同规则：
```js
var myelement = document.getElementById('myelement');
myelement.classList.add('newstyles');
```
```css
.newstyles {
	width: 100px;
	height: 200px;
	margin: 10px;
}

```
或者直接修改`cssText`:
```js
myelement.cssText += 'width:100px;height:200px;margin:10px';
```

### 离线操作

* 使用`createDocumentFragment()`等创建节点，再将其他所有节点加入其子节点，最后才加入到真实DOM里去
* 先将元素设置为隐藏即：`display:none`，做一系列更改后再显示回来

### 减少访问会引起flush的属性
尽量减少访问上面提到的一系列会引起reflow队列flush的属性，如果要多次访问，最好设置缓存变量。

### 使进行复杂动画的元素脱离文档流
通过设置动画元素`position: absolute;` 或者 `position: fixed;`可以使元素脱离文档流来避免在动画的过程中影响其他元素。

```js
// block1是position:absolute 定位的元素，它移动会影响到它父元素下的所有子元素。
// 因为在它移动过程中，所有子元素需要判断block1的z-index是否在自己的上面，
// 如果是在自己的上面,则需要重绘,这里不会引起回流
$("#block1").animate({left:50});
// block2是相对定位的元素,这个影响的元素与block1一样，但是因为block2非绝对定位
// 而且改变的是marginLeft属性，所以这里每次改变不但会影响重绘，
// 还会引起父元素及其下元素的回流
$("#block2").animate({marginLeft:50});
```

### 动画平滑度

动画每一帧元素移动1个像素和4个像素平滑程度比起来，两者平滑程度感受起来差别不大，但是后者会大大减少reflow次数。

### 动画硬件加速
// todo


**参考**

[https://www.sitepoint.com/10-ways-minimize-reflows-improve-performance/](https://www.sitepoint.com/10-ways-minimize-reflows-improve-performance/)

[http://www.blogjava.net/BearRui/archive/2010/05/10/web_performance_repaint_relow.html](http://www.blogjava.net/BearRui/archive/2010/05/10/web_performance_repaint_relow.html)