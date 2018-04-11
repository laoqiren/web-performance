# Pub/Sub方式

## 介绍
发布订阅者模式是一种非常常用的设计模式。在基于事件驱动的程序开发中尤为重要。

自定义事件：
```js
$('#btn').on('myevent',function(data){
    // 此回调函数为订阅者
});
$('#btn').trigger('myevent,'hello'); //发布消息
```
上面的`DOM`元素就充当了发布订阅调度中心。

在`Node`中通过`EventEmitter`实现类似功能：
```js
const event = require('event');

let emitter = new events.EventEmitter();
emmitter.on('data',function(data){...});//订阅
emmitter.emit('data','hhhh');//发布
```

pub/sub插件：
```js
jQuery.pubsub.subscribe('c49.filter.change', function(topic, msg){
    //Do something on filter change
});
jQuery.pubsub.publish('c49.filter.change', {
    "action":"add",
    "filter":{"value":1,"label":"The price filter"}
});
```
**tips:`Pub/Sub`模式与观察者模式并不一样，在下一节`Vue.js`方式中会详细提到**

## 使用`Pub/Sub`模式实现响应式编程

我们要实现：

```js
// js
let user = new MVVM("user");
user.set("name",888);
```

```html
<input type="number" data-bind-user="name" />  
<p data-bind-user="name"></p>
```
每次调用`user.name=xxx`时，`input`和`p`中内容自动变化到最新值，`input`中输入新值后，反映到`user`和界面其他绑定该值的地方。即数据的双向绑定。

### 订阅

我们在解析模板过程做订阅的步骤：

```js
function DataBinder(object_id){
    // 使用一个jQuery对象作为简单的订阅者发布者
    var pubSub = $({});

    // 我们希望一个data元素可以在表单中指明绑定：data-bind-<object_id>="<property_name>"        

    var data_attr = "bind-" + object_id,
            message = object_id + ":change";

    // 订阅数据变化
    pubSub.on(message,function(evt,prop_name,new_val){
        $("[data-" + data_attr + "=" + prop_name + "]").each(function(){
            var $bound = $(this);

            if($bound.is("input,text area,select")){
                $bound.val(new_val);
            }else{
                $bound.html(new_val);
            }
        });
    });

    return pubSub;
}
```
当然这里只是处理`data-`属性方式的数据绑定，复杂的还可以实现完整的模板解析。

### 发布

当我们调用`model`层的数据改变方法时，在该方法内部发布`object_id + ":change"`事件：
```js
set: function(attr_name,val){
    this.atttibutes[attr_name] = val;
    binder.trigger(uid + ":change", [attr_name, val, this]);
}
```

### 数据的双向绑定

要实现双向绑定不过是在之前的基础上监听`UI`事件，在处理程序里发布相关事件通知与之相关联的对象并修改`model`。

```js
$(document).on("change","[data-" + data_attr + "]",function(evt){
    var $input = $(this);
    pubSub.trigger(message, [ $input.data(data_attr),$input.val()]);
});
```

完整的代码：（参考芋头的博客）

```js
function DataBinder(object_id){
    //使用一个jQuery对象作为简单的订阅者发布者
    var pubSub = $({});

    //我们希望一个data元素可以在表单中指明绑定：data-bind-<object_id>="<property_name>"        

    var data_attr = "bind-" + object_id,
            message = object_id + ":change";

    //使用data-binding属性和代理来监听那个元素上的变化事件
    // 以便变化能够“广播”到所有的关联对象   

    $(document).on("change","[data-" + data_attr + "]",function(evt){
        var $input = $(this);
        pubSub.trigger(message, [ $input.data(data_attr),$input.val()]);
    });

    //PubSub将变化传播到所有的绑定元素，设置input标签的值或者其他标签的HTML内容   

    pubSub.on(message,function(evt,prop_name,new_val){
        $("[data-" + data_attr + "=" + prop_name + "]").each(function(){
            var $bound = $(this);

            if($bound.is("input,text area,select")){
                $bound.val(new_val);
            }else{
                $bound.html(new_val);
            }
        });
    });

    return pubSub;
}

function MVVM(uid){
    var binder = new DataBinder(uid),

        user = {
            atttibutes: {},

            //属性设置器使用数据绑定器PubSub来发布变化   

            set: function(attr_name,val){
                this.atttibutes[attr_name] = val;
                binder.trigger(uid + ":change", [attr_name, val, this]);
            },

            get: function(attr_name){
                return this.attributes[attr_name];
            },

            _binder: binder
        };

        binder.on(uid +":change",function(vet,attr_name,new_val,initiator){
            if(initiator !== user){
                user.set(attr_name,new_val);
            }
        })
    return user;
}

//JavaScript

var user = new MVVM("user");
user.set("name",888);

// html
<input type="number" data-bind-user="name" />  
<p data-bind-user="name"></p>
```

这里是以整个`mvvm`实例的数据为事件通知粒度的，当然也可以以数据中的每个属性为粒度，不同属性用不同事件管理，然后每个不同属性在模板中的绑定都可以看作是该事件的订阅者，`set`方法里根据传入的属性名发布不同的事件，也可以完成同样的功能。

## 存在的问题