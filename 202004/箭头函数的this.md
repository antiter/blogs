# 关于阮一峰（ES6入门）中箭头函数 this 问题的纠正？

Dear 阮一峰：

今日，我发了一篇公众号文章 [别低估自己，但，这道题，真的有点难](https://mp.weixin.qq.com/s/1iw1MBfitockO5U0ZJIeXQ)

在部分群里引起了一些讨论，其中有一点是关于箭头函数的 this 指针的问题。  

有朋友在群里有引用了您的文章来反驳：

![ryf_group](https://img12.360buyimg.com/img/s2064x996_jfs/t1/116914/3/1932/342627/5e9d7124E7be097af/aa0d6f675e406c14.png)

这个内容，出自您的超高赞誉的 es6 入门文章(ES6 必读必参考书籍)：

https://github.com/ruanyf/es6tutorial/blob/2e1c10776a9350debfec47de8ddeaf1c2c5a80cc/docs/function.md。

有一段文字描述箭头函数的 this 的。

![this](https://img12.360buyimg.com/img/s1598x562_jfs/t1/101392/28/19115/177302/5e9d6d17E506d415b/53227e5d5634b28d.png)

经过验证，貌似有点描述不对。  

查询 [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions) 的 描述是这样的：

<pre>
Since arrow functions do not have their own this, the methods call() and apply() can only pass in parameters. Any this argument is ignored.
</pre>

这里描述的是 call，apply 函数会忽略掉 this 参数。

## 我的理解为如下两点：

- call，apply 忽略掉 this 参数是表象，实际上是因为箭头函数的 this 是指向父级的 this，自身没有 this，所以无法修改自身的 this。

- 箭头函数的 this ，也是可变的，不是定义时候所在的对象，而是动态执行的时候获取父级的 this 来作为自身的 this 指向。（当前箭头函数逐级向上查找 this，如果找到了，则作为自己的 this 指向，如果没有则继续向上查找）

箭头函数的 this 变更，可以采用变更父级的 this 来达到变更子箭头函数的 this。

## 验证例子

```js
function outer(){
    var inner = function(){
        var obj = {};
        obj.getVal=()=>{
            console.log("*******");
            console.log(this);
            console.log("*******");
        }
        return obj;
    };
    return inner; 
}

```
## 直接执行方法，读取 getVal 方法。

```js

outer()().getVal();

VM2169:5 *******
VM2169:6 Window {parent: Window, opener: null, top: Window, length: 0, frames: Window, …}
VM2169:7 *******

```
getVal 函数是箭头函数，方法里面的 this 是跟着父级的 this 上下文。

在 outer() 执行后，返回闭包函数 inner

然后执行闭包函数 inner，而闭包函数的 inner 是这里的 window 调用的，所以 inner 的 this 是指向 window。输出 this，指向 window。

验证例子：

```js
var age = 444;
var age2 = 555;
function test(){
    var age = 123;
    return function(){
        console.log(this);
        console.log(age,age2);
    }
}

test()();
// 输出如下
VM2593:6 Window {parent: Window, opener: null, top: Window, length: 0, frames: Window, …}
VM2593:7 123 555
```

这里的 this 是指向 window 的，因为闭包函数的的作用域已经跳出父依赖函数，是父函数 test 执行之后返回的函数，已经共享调用父方法 test 的作用域。

验证例子2：

```js
var age = 444;
var age2 = 555;
function test(){
    this.age = 123;
    return function(){
        console.log(this.age,this.age2);
    }
}
(new test())(); // 输出： 444 555
console.log(age);// 输出：444
```
这里先 new 一个 test 实例，用来锁住 this，然后在返回的闭包函数执行，this 仍然是指向 window。

当然这里是普通函数，完全可以改变 this 指针。

最上面例子中 getVal 是箭头函数，this 指针再往上看一级即可。

## 改变箭头函数的 this

题目不变，方便阅读，再拷贝一下。
```js
function outer(){
    var inner = function(){
        var obj = {};
        obj.getVal=()=>{
            console.log("*******");
            console.log(this);
            console.log("*******");
        }
        return obj;
    };
    return inner; 
}
```
使用 Bind 改变父级 inner 函数的 this，来达到改变子箭头函数 getVal 的 this 指向，

```js
outer().bind(outer)().getVal();

VM2169:5 *******
VM2169:6 ƒ outer(){
    var inner = function(){
        var obj = {};
        obj.getVal=()=>{
            console.log("*******");
            console.log(this);
            console.log("*******");
        }
  …
VM2169:7 *******
```

执行 outer 方法，返回 inner 函数，然后改变 inner 的 上下文，使用 bind 将 inner 的 this 指向到 outer。

这里成功改变了 getVal 的 this 指向，this 已经随同父级元素的 this 的改变而改变。




