### React 中的 setState更新state何时同步何时异步？

React中constructor是唯一可以初始化state的地方，也可以把它理解成一个钩子函数，该函数最先执行且只执行一次。

更新状态不要直接修改this.state。虽然状态可以改变，但不会触发组件的更新。

应当使用this.setState()，该方法接收两种参数：对象或函数。

1. 对象：即想要修改的state
2. 函数：接收两个函数，第一个函数接受两个参数，第一个是当前state，第二个是当前props，该函数返回一个对象，和直接传递对象参数是一样的，就是要修改的state；第二个函数参数是state改变后触发的回调。

回到主题，setState可能是异步的。对此官方有这样一段描述：setState() does not always immediately update the component. It may batch or defer the update until later. This makes reading this.state right after calling setState()a potential pitfall.

关键词：batch、defer、may。

#### 要探究setState为什么可能是异步的，先了解setState执行后会发生什么？

事实上setState内部执行过程是很复杂的，大致过程包括更新state，创建新的VNode，再经过diff算法比对差异，决定渲染哪一部分以及怎么渲染，最终形成最新的UI。这一过程包含组件的四个生命周期函数。

* shouleComponentUpdate
* componentWillUpdate
* render
* componentDidUpdate

此外如果子组件的数据依赖于父组件，还会执行一个钩子函数`componentWillReceiveProps`。

假如setState是同步更新的，每更新一次，这个过程都要完整执行一次，无疑会造成性能问题。事实上这些生命周期为纯函数，对性能还好，但是diff比较、更新DOM总消耗时间和性能吧。

此外为了批次和效能，多个setState有可能在执行过程中还会被合并，所以setState延时异步更新是很合理的。

<!-- 即便CPU和浏览器性能很好，从React的API设计角度考虑，`shouldComponentUpdate`返回true或false，用来决定是否重新渲染组件，即是否继续执行接下来的三个钩子函数，可以避免不必要执行。而`componentWillUpdate`在组件重新渲染之前执行，两个生命周期函数一个可以用来中获取this.state仍然是旧的state。而最新的state是通过函数参数体现出来的。所以 -->



#### setState何时同步何时异步？

由React控制的事件处理程序，以及生命周期函数调用setState不会同步更新state。

React控制之外的事件中调用setState是同步更新的。比如原声绑定事件，setTimeout/setInterval等。

大部分开发中用到的都是React封装的事件，比如onChange、onClick、onTouchMove等，这些事件处理程序中的setState都是异步处理的。

看以下case：

```js

constructor() {
  this.state = {
    count: 10
  }

  this.handleClickOne = this.handleClickOne.bind(this)
  this.handleClickTwo = this.handleClickTwo.bind(this)
}
render() {
  return (
    <button onClick={this.hanldeClickOne}>clickOne</button>
    <button onClick={this.hanldeClickTwo}>clickTwo</button>
    <button id="btn">clickTwo</button>
  )
}

handleClickOne() {
  this.setState({ count: this.state.count + 1})
  console.log(this.state.count)
}

```

输出：10

由此可以看出该事件处理程序中的setState是异步更新state的。

```js
componentDidMount() {
  document.getElementById('btn').addEventListener('clcik', () => {
    this.setState({ count: this.state.count + 1})
    console.log(this.state.count)
  })
}
```

输出： 11

```js
handleClickTwo() {
  setTimeout(() => {
    this.setState({ count: this.state.count + 1})
    console.log(this.state.count)
  }, 10)  
}
```

输出： 11

而绕过React，通过js的事件绑定程序 addEventListener 和使用setTimeout/setInterval 等 React 无法掌控的 APIs情况下，setState是同步更新state。

#### React是怎样控制异步和同步的呢？

在 React 的 setState 函数实现中，会根据一个变量 isBatchingUpdates 判断是直接更新 this.state 还是放到队列中延时更新，而 isBatchingUpdates 默认是 false，表示 setState 会同步更新 this.state；但是，有一个函数 batchedUpdates，该函数会把 isBatchingUpdates 修改为 true，而当 React 在调用事件处理函数之前就会先调用这个 batchedUpdates将isBatchingUpdates修改为true，这样由 React 控制的事件处理过程 setState 不会同步更新 this.state。


#### 多个setState调用会合并处理

```js

render() {
  console.log('render')
}
hanldeClick() {
  this.setState({ name: 'jack' })
  this.setState({ age: 12 })
}
```

在hanldeClick处理程序中调用了两次setState，但是render只执行了一次。因为React会将多个this.setState产生的修改放在一个队列里进行批延时处理。

#### 参数为函数的setState用法

先看以下case：

```js
handleClick() {
  this.setState({
    count: this.state.count + 1
  })
}
```

以上操作存在潜在的陷阱，不应该依靠它们的值来计算下一个状态。


```js
handleClick() {
  this.setState({
    count: this.state.count + 1
  })
  this.setState({
    count: this.state.count + 1
  })
  this.setState({
    count: this.state.count + 1
  })
}
```
最终的结果只加了1

因为调用this.setState时，并没有立即更改this.state，所以this.setState只是在反复设置同一个值而已，上面的代码等同于这样

```js
handleClick() {
  const count = this.state.count

  this.setState({
    count: count + 1
  })
  this.setState({
    count: count + 1
  })
  this.setState({
    count: count + 1
  })
}
```
count相当于一个快照，所以不管重复多少次，结果都是加1。


此外假如setState更新state后我希望做一些事情，而setState可能是异步的，那我怎么知道它什么时候执行完成。所以setState提供了函数式用法，接收两个函数参数，第一个函数调用更新state，第二个函数是更新完之后的回调。

第一个函数接收先前的状态作为第一个参数，将此次更新被应用时的props做为第二个参数。

```js
this.increment(state, props) {
  return {
    count: state.count + 1
  }
}

handleClick() {
  this.setState(this.increment)
  this.setState(this.increment)
  this.setState(this.increment)
}
```

结果: 13

对于多次调用函数式setState的情况，React会保证调用每次increment时，state都已经合并了之前的状态修改结果。

也就是说，第一次调用this.setState(increment)，传给increment的state参数的count是10，第二调用是11，第三次调用是12，最终handleClick执行完成后的结果就是this.state.count变成了13。 

值得注意的是：在increment函数被调用时，this.state并没有被改变，依然要等到render函数被重新执行时（或者shouldComponentUpdate函数返回false之后）才被改变，因为render只执行一次。

让setState接受一个函数的API的设计是相当棒的！不仅符合函数式编程的思想，让开发者写出没有副作用的函数，而且我们并不去修改组件状态，只是把要改变的状态和结果返回给React，维护状态的活完全交给React去做。正是把流程的控制权交给了React，所以React才能协调多个setState调用的关系。

##### 在同一个事件处理程序中不要混用

case:
```js
this.increment(state, props) {
  return {
    count: state.count + 1
  }
}

handleClick() {
  this.setState(this.increment)
  this.setState({ count: this.state.count + 1 })
  this.setState(this.increment)
}
```

结果： 12

第一次执行setState，count为11，第二次执行，this.state仍然是没有更新的状态，所以this.state.count又打回了原形为10，加1以后变成11，最后再执行setState，所以最终count的结果是12。（render依然只执行一次）

setState的第二个回调参数会在更新state，重新触发render后执行。

