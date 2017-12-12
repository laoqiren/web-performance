# React基础

参照React官方文档：[https://reactjs.org/docs/](https://reactjs.org/docs/)

## 组件化
React采用组件化的思想构建UI，并提供`jsx`语法创建元素，可以通过两者方式创建React组件：
```js
//函数
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}
// ES6 class
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}

ReactDOM.render(
  <Welcome/>,
  document.getElementById('root')
);
```
组件可以进行嵌套。

### 生命周期
看一个计时的例子:
```js
class Clock extends React.Component {
    constructor(props) {
      super(props);
      this.state = {secondsElapsed: 0};
    }
    tick(){
      this.setState({
        secondsElapsed: this.state.secondsElapsed + 1
      })
    }
    componentDidMount(){
      this.interval = setInterval(() => this.tick(), 1000);
    }
    componentWillUnmount(){
      clearInterval(this.interval);
    }
    render() {
      return (
        <div>
          <h1>Hello, world!</h1>
          <h2>It is {this.state.secondsElapsed}.</h2>
        </div>
      );
    }
  }
  ReactDOM.render(
    <Clock />,
    document.getElementById('root')
  );
```
* 调用`ReactDOM.render()`方法，渲染`Clock`组件，执行其构造函数，初始化`state`
* 组件所有元素渲染完毕，执行`componentDidMount()`生命周期钩子函数，设定计时器
* 浏览器每秒执行一次`tick()`，在其内部改变`state`，状态发生改变，构建新的虚拟DOM，与旧的虚拟DOM进行对比，使用diff算法找到变化的地方，应用到真实DOM
* 组件从DOM中移除时，执行`componentWillUnmount()`钩子函数，在这里移除定时器

所有生命周期：[https://reactjs.org/docs/react-component.html](https://reactjs.org/docs/react-component.html)

## 单向数据流
组件的状态可以通过`props`传递给子组件来改变下层组件：
```js
class App extends React.Component {
    constructor(props) {
      super(props);
      this.state = {secondsElapsed: 0};
    }
    render() {
      return (
        <div>
          <h1>Hello, world!</h1>
          <h2>It is {this.state.secondsElapsed}.</h2>
          <Child number={this.state.secondsElapsed}/>
        </div>
      );
    }
}
class  Child extends React.Component {
    constructor(props) {
      super(props);
    }
    render() {
      return (
        <div>
          <h2>It is {this.props.number}.</h2>
        </div>
      );
    }
}
```
即组件的状态只能向下传递，该组件又可以接受来自它的父组件的`props`，这样一层层地自顶向下传递。

## 虚拟DOM

---
* 用 JavaScript 对象结构表示 DOM 树的结构；然后用这个树构建一个真正的DOM树，插到文档当中
* 当状态变更的时候，重新构造一棵新的对象树。然后用新的树和旧的树进行比较，记录两棵树差异
* 把上面所记录的差异应用到第一步所构建的真正的DOM树上，视图就更新了

网上看到的一篇文章，讲的`preact`的虚拟DOM实现，和React原理差不多：
![](http://efe.baidu.com/blog/the-inner-workings-of-virtual-dom/1.png)

参照原文：[https://medium.com/@rajaraodv/the-inner-workings-of-virtual-dom-666ee7ad47cf](https://medium.com/@rajaraodv/the-inner-workings-of-virtual-dom-666ee7ad47cf)