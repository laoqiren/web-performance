# Vue方式

本文介绍`Vue.js`响应式的实现原理。在`vue`官方文档中关于其响应式原理有一个简单的介绍: [https://cn.vuejs.org/v2/guide/reactivity.html](https://cn.vuejs.org/v2/guide/reactivity.html)，其中还给出一张图来解释：

![https://cn.vuejs.org/images/data.png](https://cn.vuejs.org/images/data.png)

可以看到`Vue`的思想也是类似`MVVM`，在应用程序员看来，`model`层和`view`层解耦，开发者改变`model`数据，`UI`会自动响应数据的变化。我们将从内部原理细节去立即它的响应式原理。

我们要实现：

```js
let vm = new MVVM({
  el: '#app',
  data: {
    name: "luoxia",
    info: {
      year: 21,
      addr: "China"
    }
  }
});

vm.$watch("name",newName=>{
  console.log("name变化了哦：",newName)
});
```

```html
<div id="app">
    hello,{{name}}
    <input type="text" l-model="name" />
</div>
```

原理中主要涉及到`Observe`、`Watcher`、`Dep`几个概念，用一张图解释它们的关系：

![https://github.com/DMQ/mvvm/raw/master/img/2.png](https://github.com/DMQ/mvvm/raw/master/img/2.png)

## `Observe`的过程

在创建`vm`实例的步骤中，有一个很重要的过程是对`data`中的属性进行`getter/setter`化，对`data`中的每个属性进行监听，处理目标对属性的绑定，以及属性值变化后通知订阅者。`Vue`使用`Object.defineProperty()`来实现。

```js
class MVVM {
  constructor(options={}){
    this.$options = options;
    this._data = options.data;

    observe(this._data);
    // ...
  }
}

class Observer {
  constructor(data){
    let dep = this.dep = new Dep(); // dep理解为Pub/Sub中心，也是data中属性的观察者
    const keys = Object.keys(data);
    for(let key of keys){
      let value = data[key];

      observe(value); //递归observe

      Object.defineProperty(data,key,{
        configurable: true,
        get(){
          if(Dep.target){
            console.log("初始绑定:",key,value);
            dep.addSub(Dep.target); // 通知观察者dep，让dep加入watcher到dep维护的订阅者队列
          }
          return value;
        },
        set(newVal){
          if(newVal === value){
            return;
          }
          value = newVal;
          observe(newVal);
          dep.notify(); // 通知观察者dep属性变化
        }
      })
    }
  }
}

function observe(data){
  if(!data || typeof data !== 'object'){
    return;
  }
  return new Observer(data);
}
```

## `Watcher`

在模板编译过程中，每遇到一个数据绑定，就会生成一个订阅该属性变化的`watcher`，它具有`update()`方法，`pub/sub`中心接到来自于`setter`的属性变化通知，调用所有订阅者的`update()`方法，在`watcher`的`update()`方法中，回去调用更新`UI`的方法:

```js
class Watcher {
  constructor(vm,exp,cb){
    this.cb = cb; // 将用于更新UI
    this.vm = vm; // vm实例
    this.exp = exp; // 正则匹配出来的属性名
    let arr = exp.split('.');
    let val = vm;
    Dep.target = this; // 用于识别初次访问该属性的getter，用于加入观察者到订阅者队列
    arr.forEach(key => {
      val = val[key]; //会访问该属性的getter
    });
    this.oldValue = val; // 记录第一次绑定时的属性值
    Dep.target = null; // 防止后续访问该属性的getter重复加入订阅者队列
  }
  update(){
    // 获取属性的最新值
    let arr = this.exp.split('.');
    let val = this.vm;
    arr.forEach(key => {
      val = val[key];
    });
    
    if(this.oldValue !== val){
      this.oldValue = val;
      this.cb(val); // 更新UI的回调
    }
  }
}
```

## `Compiler`

`Compiler`用于编译模板，在模板中会有对`vm`实例中`data`相应属性的绑定，在编译过程中主要做的事情：

* 匹配出绑定属性的地方
* 为该次绑定添加相应的`watcher`，`watcher`回调中更新UI
* 初始绑定值的替换
* 嵌入到`DOM`结构中

```js
compile(){
    let vm = this;

    vm.$el = document.querySelector(this.$options.el);
    let fragment = document.createDocumentFragment();

    var child;
    while (child = vm.$el.firstChild) {
        fragment.appendChild(child);    // 此时将el中的内容放入内存中
    }
    // 对el里面的内容进行替换
    function replace(frag) {
        Array.from(frag.childNodes).forEach(node => {
            let txt = node.textContent;
            let reg = /\{\{(.*?)\}\}/g;   // 正则匹配{{}}

            if (node.nodeType === 3 && reg.test(txt)) { // 即是文本节点又有大括号的情况{{}}

                    // 初次数据的替换
                    node.textContent = txt.replace(reg,vm.getExpVal(RegExp.$1)).trim();

                    // 添加观察者
                    new Watcher(vm,RegExp.$1,newVal=>{
                        // 更新UI，用trim方法去除一下首尾空格
                    node.textContent = txt.replace(reg, newVal).trim();
                    });
                
            }

            if (node.nodeType === 1) {  // 元素节点
                let nodeAttr = node.attributes; // 获取dom上的所有属性,是个类数组
                Array.from(nodeAttr).forEach(attr => {
                    let name = attr.name;   // v-model  type
                    let exp = attr.value;   // c        text
                    if (name.includes('l-')){
                        node.value = vm[exp];   // this.c 为 2
                        // 数据双向绑定
                        node.addEventListener('input', e => {
                        let newVal = e.target.value;
                        // 相当于给this.c赋了一个新值
                        // 而值的改变会调用set，set中又会调用notify，notify中调用watcher的update方法实现了更新
                        vm[exp] = newVal;   
                    });
                    }
                    // 监听变化
                    new Watcher(vm, exp, function(newVal) {
                        node.value = newVal;   // 当watcher触发时会自动将内容放进输入框中
                    });
                    
                    
                });
            }
            // 如果还有子节点，继续递归replace
            if (node.childNodes && node.childNodes.length) {
                replace(node);
            }
        });
        }

    replace(fragment);
    vm.$el.appendChild(fragment);
}
```

## `Dep`

在我的理解当中，`Dep`充当两种角色：

* 与`Observer`的联系：在创建`Observer`时，会实例化一个`Dep`对象，这个`dep`实例充当属性的观察者，属性变化时，`setter`内部会调用`dep.notify()`用于将属性变化通知给这个`dep`，而有模板对该属性进行绑定时，会调用`dep.addsub()`用于将观察者加入到`dep`维护的`watcher`队列当中。此处体现观察者模式。
* 充当`Pub/Sub`的调度中心，内部维护`watcher`队列，每次属性变化，`setter`调用`dep.notify()`，`dep`接受到通知后，会调用每个`watcher`的`update()`方法。此处体现发布订阅者模式。

```js
class Dep {
  constructor(){
    this.subs = [];
  }
  addSub(sub){
    this.subs.push(sub);
    return this;
  }
  notify(){
    this.subs.map(sub=>sub.update());
  }
}
```

总的来说，完整的思路：

* 使用`Object.defineProperty()`为属性添加观察者，而这里的观察者是`Dep`，它又充当`Pub/Sub`调度中心的角色
* 编译模版，每次匹配到属性的绑定，实例化一个`watcher`，实例化的时候会去访问属性的值，所以会访问属性的`getter`，该`getter`再通知`Dep`实例将这个`watcher`加入到`Dep`维护的`watcher`队列中
* 当属性变化时，首先会调用属性的`setter`，`setter`会通知`dep`实例，`dep`实例接到通知后再广播给所有`watcher`
* 所有`watcher`接到通知后，调用自己的`update()`方法，如果该`watcher`监听的属性发生了变化，就要调用相应的方法更新`UI`

## 存在的问题