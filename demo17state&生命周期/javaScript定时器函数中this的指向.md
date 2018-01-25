### javaScript定时器/延时器函数中this的指向

先看一个栗子：

```js
var name = 'from window';

var obj = {
    name: 'from obj',
    func: function(){
        setInterval(function () {
            console.log(this.name)
        },1000)
    }
}

obj.func()

```
输出结果：`"from window"`

在`setInterval`和`setTimeout`中，如果回调函数是普通函数并且this没有特别指向，那么回调中的this默认指向window。


但是一般在开发中，很多场景都需要改变`this`的指向。以下三中方式可以修改回调中`this`的指向。

#### 1、最常用的方法：在外部函数中将`this`存为一个变量，在回调函数中使用变量，而不是直接使用this。

```js
var name = 'from window';

var obj = {
    name: 'from obj',
    func: function(){
        var _this = this;
        setInterval(function () {
            console.log(_this.name)
        },1000)
    }
}

obj.func()
```

输出结果： `from obj`

#### 2、使用es6中的`bind()`方法修改函数中this的指向。

`bind()`和`call()`/`apply()`的作用一样，都可以修改函数内部this的指向，但是`call`和`apply`是修改`this`指向后函数会立即执行，而`bind`则是返回一个新的函数，它会创建一个与原来函数体相同的新函数，新函数中的`this`指向传入的对象。如果接触过`react`，这下就知道在类组件的类构造函数`constructor`中为什么经常会见到`this.click = this.click.bind(this)`这种写法了。

```js
var name = 'from window';

var obj = {
    name: 'from obj',
    func: function(){
        setInterval(function () {
            console.log(_this.name)
        }.bind(this),1000)
    }
}

obj.func()
```

输出结果： `from obj`

这里不使用call和apply的原因是call、apply不会返回新函数，而是调用函数时使用。

#### 3、使用es6的箭头函数：箭头函数的最大作用就是`this`指向。

```js
var name = 'from window';

var obj = {
    name: 'from obj',
    func: function(){
        setInterval(()=>{
            console.log(this.name)
        },1000)
    }
}

obj.func()

```
输出结果： `from obj`

箭头函数没有自己的`this`，它的`this`继承自外部函数的作用域。所以定时器回调中的`this`指向func函数中的`this`，又因为func由obj调用，所以func中的this指向obj。
