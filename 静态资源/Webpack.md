# Webpack

Webpack看名字就知道它是一个Web应用打包工具。看下面官方的一张图就大概知道其设计理念：

![http://7xsi10.com1.z0.glb.clouddn.com/webpack-intro.png](http://7xsi10.com1.z0.glb.clouddn.com/webpack-intro.png)

通过前面的网络资源章节，我们知道资源请求的开销多大，我们要减少网络请求，而Webpack将多个模块打包到少数几个bundle文件以减少资源请求开销。但是否粗暴地将所有资源都打包到一个文件就完美了？非也，事实上我们要考虑的问题更多，我们要充分利用前面讲到的缓存提高性能；我们要考虑到单个bundle文件的大小，不能太大，而是考虑将大的单个文件拆分成几个小的文件从而充分利用浏览器的并行下载能力；我们还要考虑减少不必要的资源请求，使用动态加载（懒加载）来提高性能。

下面我将分别从上面几个方面介绍Webpack。
## 多个模块打包减少资源请求数

这是Webpack提供的基本能力，其将Web应用的各种资源对象都当作模块，通过模块依赖图，将大量的模块打包到最终的几个少数目标bundle文件。

举个例子，下面是配置：

```js
const HtmlWebpackPlugin = require('html-webpack-plugin');

const config = {
     entry: {
         app: __dirname + '/src/app.js'
     },
     output: {
         path: __dirname + '/public',
         filename: '[name].[chunkhash].js'
     },
     module: {
         rules: [
            {
                test:/\.scss$/,
                loaders: ['style-loader','css-loader','sass-loader']
            },
            {
                test: /(\.jsx|\.js)$/,
                use: {
                    loader: "babel-loader",
                    options: {
                        presets: [
                            "env"
                        ]
                    }
                },
                exclude: /node_modules/
            }
         ]
     },
     plugins: [
        new HtmlWebpackPlugin({
            filename: 'index.html',
            template: __dirname + '/public/template.html'
        })
    ]
}

module.exports = config;
```

下面是业务代码:

```js
// app.js
import $ from 'jquery';
import _ from 'lodash';
import Person from './Person';
import './style/main.scss';

const person1 = new Person("Luo Xia",1996,[98,99,98]);
const person2 = new Person("Jack",1992,[98,96,99]);

console.log(_.zipWith(person1.scores,person2.scores,(a,b)=>a-b));

$("#container").html(`
    年龄：<span id="year">${person1.getYear()}</span>岁
`);

// Person.js
class Person {
    constructor(name,birth){
        this.name = name;
        this.birth = birth;
    }
    getYear(){
        return new Date().getFullYear() - this.birth;
    }
}

export default Person;
```

我们使用`chunkhash`来命名最后的打包文件，其根据文件内容计算hash值，我们每次修改任意模块（包括js和scss文件)，都会使得该值发生变化。

还记得我们之前的缓存章节的`ETag`字段吗？我们想要达到的目的是根据文件内容变化合理调节缓存机制，对于业务代码我们需要通过hash值来管理资源版本，但是对于那些不常变化的库文件（如这里引用的`jquery`和`lodash`)，我们需要充分利用缓存策略来提高页面加载速度。如果像我们这里将所有文件打包到一起，那些库就算没有变化，但是业务代码发生变化后也会引起库文件代码重新加载，性能不可观。

```
Hash: bcdb73ac672157c01b11
Version: webpack 3.10.0
Time: 1697ms
                      Asset       Size  Chunks                    Chunk Names
app.15a96f713232f7ab2557.js     835 kB       0  [emitted]  [big]  app
```

## 拆分代码和提取公共部分

首先为什么要拆分代码呢？

* 按照之前的做法我们将所有模块打包到一个bundle文件，我们会得到一个很大的文件，然而我们浏览器是可以并行下载多个文件的，这样下载一个大文件没法利用并行下载能力，导致资源加载速度较慢；
* 在多页应用当中，不同页面可能需要引用的模块并不相同，我们需要针对不同页面打包不同文件，如上面例子中，假设我们有两个页面，第一个页面引用`app.js`，而第二个页面只需引用`Person.js`，我们需要打包两个不同的文件；
* 对于一些不经常变化的库文件，如上面的`lodash`和`jquery`，我们要将它们分离出来以采取不同的缓存策略。

### 多个入口文件
修改上面的配置:
```js
entry: {
    app: __dirname + '/src/app.js',
    Person: __dirname + '/src/Person.js'  //添加一个入口文件
}
```
打包结果：
```
Hash: 27e2057d8543e1a2a7ae
Version: webpack 3.10.0
Time: 1147ms
                         Asset       Size  Chunks                    Chunk Names
   app.1b8b3bd9ea8fe644623c.js     835 kB    0, 1  [emitted]  [big]  app
Person.6b9b0bed699a1861b41a.js     545 kB       1  [emitted]  [big]  Person
```

我们发现，虽然`Person.js`被单独打包，但是`app.js`的打包文件大小没变，它依旧包含了`Person.js`的代码，而我们想要其不包含两个打包文件的公共部分。

### 提取重复部分
针对上面的问题，我们使用`CommonsChunkPlugin`插件来提取重复部分

```js
plugins: [
    new HtmlWebpackPlugin({
        filename: 'index.html',
        template: __dirname + '/public/template.html'
    }),
    new webpack.optimize.CommonsChunkPlugin({
        name: 'common' // 指定公共 bundle 的名称。
    })
]
```
打包结果：
```
Hash: 36bf65c32b9a5ba7d095
Version: webpack 3.10.0
Time: 1109ms
                         Asset       Size  Chunks                    Chunk Names
   app.5ddc3d289fdd16c6b4b4.js     290 kB       0  [emitted]  [big]  app
Person.1b8c2aef957c8cdb3732.js   25 bytes       1  [emitted]         Person
common.fa9e9dad5c5f6a690ed1.js     549 kB       2  [emitted]  [big]  common
```

这个时候我们发现，在两个模块中都引入的`Person.js`和`lodash`的代码被单独打包到了`common.[chunkhash].js`里了。

### 分离vendor和业务代码
上面的例子中，我们的`jquery`还是包含在了`app.js`里，和业务代码包含在了一起，而我们需要将它们分开。

```js
entry: {
    app: __dirname + '/src/app.js',
    Person: __dirname + '/src/Person.js',
    vendor: ['lodash','jquery'] //添加入口文件，以抽离出库文件
}
```
打包结果：
```
Hash: 09e8171214a6d9bd80b8
Version: webpack 3.10.0
Time: 1123ms
                           Asset       Size  Chunks                    Chunk Names
     app.87181a36ed57cd1b01d8.js     293 kB    0, 3  [emitted]  [big]  app
  common.88e0331c5d3b500cec0e.js     541 kB       1  [emitted]  [big]  common
  vendor.9aa18535edeb21d84c5a.js     272 kB       2  [emitted]  [big]  vendor
  Person.b5a494c4325215143bac.js    2.98 kB       3  [emitted]         Person
```
测试发现`jquery`成功地分离到了`vendor.[chunkhash].js`里，但是`lodash`因为是重复部分所以被打包到了`common.[chunkhash].js`里。

而当我们在这个配置基础上做如下修改：
```js
 plugins: [
    new HtmlWebpackPlugin({
        filename: 'index.html',
        template: __dirname + '/public/template.html'
    }),
    new webpack.optimize.CommonsChunkPlugin({
        name: 'vendor' // 修改common为vendor即和入口定义的vendor相同
    })
]
```
我们会发现所有的库文件都被打包到同一个`vendor.[chunkhash].js`，但是`Person.js`的代码由于是重复的，也被打包到了这里面去。这个问题暂时还没解决 >_<||| 

### 动态导入与懒加载

有的时候，我们并不需要在首页渲染时就引用全部所需要的静态文件，我们只在未来的某些时候需要用到的模块可以通过动态导入模块技术来实现懒加载。

我们在前面例子的基础上，再增加一个`City.js`模块，`app.js`里边会动态地加载这个模块，即只有当用户点击某个按钮后，在事件处理程序里引用这个模块：

```js
//app.js
import $ from 'jquery';
import _ from 'lodash';
import Person from './Person';
import './style/main.scss';

const person1 = new Person("Luo Xia",1996,[98,99,98]);
const person2 = new Person("Jack",1992,[98,96,99]);

console.log(_.zipWith(person1.scores,person2.scores,(a,b)=>a-b));

$("#container").html(`
    年龄：<span id="year">${person1.getYear()}</span>岁
`);

let button = $('#btn');

button.on('click',e => import(/* webpackChunkName: "City" */ './City').then(module => {
    const City = module.default;

    let city1 = new City('Chongqing','China');
    $('#container').append(city1.sayCountry());
}));

//City.js
class City {
    constructor(name,country){
        this.name = name;
        this.country = country;
    }
    sayCountry(){
        return this.country;
    }
}

export default City;
```

打包结果：
```
Hash: e542951fc739c858041a
Version: webpack 3.10.0
Time: 1195ms
                           Asset       Size  Chunks                    Chunk Names
       0.e86cda345d7499709c39.js    1.26 kB       0  [emitted]         City
     app.6de4abc067bb830a59a3.js     290 kB       1  [emitted]  [big]  app
  common.ad3692e54127021bf1cf.js     543 kB       2  [emitted]  [big]  common
  Person.8ef9b8c6bcc2c0567d3f.js   25 bytes       3  [emitted]         Person
```

我们可以看到，`City.js`被单独打包到`0.[chunkhash].js`。然后我们通过devtools网络工具可以查看到首页渲染时并没有加载这个文件，而是当用于点击按钮后再加载这个文件。


## 其他

* 上面的例子中，我们看到有一些打包文件最终还是很大，这就需要我们在生产环境下对打包文件做进一步的压缩，这方面可以结合前面的资源压缩章节，Webpack也提供了相关的插件，可以替代gulp的工作
* 资源版本管理的过程中，可能会发现内容不变情况下，chunkhash发生变化，或者增加新模块导致其他模块的chunkhash发生变化等，这方面可以参考官方文档的`manifest`和模块标识符部分，这里不详述