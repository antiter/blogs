# Promise 执行过程的正确理解姿势

本文一步一步深入解读 Promise 的注册和执行过程，读懂这几段代码，Promise 的注册和执行过程都将所向披靡，不再话下~~~~。你就是 Promise 大神！~~。当然了，就没人能误导你了~

## 前言

本文已代码解读的方式来学习整个过程。这里提供了四段代码，如果你都能理解清楚，正确的说出output过程，那么厉害大牛如你，Promise 的执行过程了如指掌。    

好多大牛已经熟悉了，当然了，并不是所有的人都能理解这几段代码，那么我们就一起来看看吧~   

本文会依次解析这几段代码，最后给出 Promise 的终极执行过程。   

## 第一段代码   
```js
new Promise((resolve, reject) => {
  console.log("外部promise");
  resolve();
})
  .then(() => {
    console.log("外部第一个then");
    return new Promise((resolve, reject) => {
      console.log("内部promise");
      resolve();
    })
    .then(() => {
    console.log("内部第一个then");
    })
    .then(() => {
    console.log("内部第二个then");
    });
  })
  .then(() => {
    console.log("外部第二个then");
  });

```
这个输出还是比较简单的，外部第一个 new Promise 执行，执行完 resolve ，然后执行外部第一个 then 。外部第一个 then 方法里面 return 一个 Promise，这个return ，代表 外部的第二个 then 的执行需要等待 return 之后的结果。当然会先执行完内部两个 then 之后，再执行 外部的第二个 then ，机智如你，完全正确。       

output:   
外部promise
外部第一个then
内部promise
内部第一个then
内部第二个then
外部第二个then

## 这是第二段代码
```js
new Promise((resolve, reject) => {
  console.log("外部promise");
  resolve();
})
  .then(() => {
    console.log("外部第一个then");
    new Promise((resolve, reject) => {
      console.log("内部promise");
      resolve();
    })
      .then(() => {
        console.log("内部第一个then");
      })
      .then(() => {
        console.log("内部第二个then");
      });
  })
  .then(() => {
    console.log("外部第二个then");
  });
```
这段代码和第一段代码就相差一个 return ，然而结果确是不一样的。   

那这个怎么理解呢？    

我们核心要看 then 的回调函数是啥时候注册的，我们知道，事件机制是 “先注册先执行”，即数据结构中的 “栈” 的模式，first in first out。那么重点我们来看下他们谁先注册的。   

**外部的第二个 then 的注册，需要等待 外部的第一个 then 的同步代码执行完成。**  当执行内部的 new Promise 的时候，然后碰到 resolve，resolve 之后需要执行的动作是一个微任务异步执行，那么自然要先执行完同步任务，比如如下：  
```js
new Promise((resolve, reject) => {
    resolve();
    console.log(1111);
})
.then(() => {
    consle.log(2222);
})
```
这个代码显然优先执行 111，在执行 2222。   

同理回到上面的代码，内部的 resolve 之后，当然是先执行内部的 new Promise 的第一个 then 的注册，这个 new Promise执行完成，立即同步执行后面的 .then 的注册。

然而这个内部的第二个 then 是需要第一个 then 的的执行来决定的，而第一个 then 的回调是没有执行，仅仅只是执行了同步的 .then 方法的注册，所以会进入等待状态。   

这个时候，外部的第一个 then 的同步操作已经完成了，然后开始注册外部的第二个 then，此时外部的同步任务也都完成了。同步操作完成之后，那么开始执行微任务，我们发现 内部的第一个 then 是优先于外部的第二个 then 的注册，所以会执行完内部的第一个 then 之后，然后注册内部的第二个 then ，然后执行外部的第二个 then ,然后再执行内部的第二个 then。   

输出结果如下：   
output:   
外部promise
外部第一个then
内部promise
内部第一个then
外部第二个then
内部第二个then

## 再看第三段代码
```js
new Promise((resolve, reject) => {
  console.log("外部promise");
  resolve();
})
  .then(() => {
    console.log("外部第一个then");
    let p = new Promise((resolve, reject) => {
      console.log("内部promise");
      resolve();
    })
    p.then(() => {
        console.log("内部第一个then");
      })
    p.then(() => {
        console.log("内部第二个then");
      });
  })
  .then(() => {
    console.log("外部第二个then");
  });
```
这段代码的差异，就是内部的 Promise 的代码的写法变了，不再是链式调用。   

这里怎么理解呢？   

这里在执行内部的 new Promise 的 resolve 执行完成之后，new Promise 之后的两个同步 p.then 是两个执行代码语句，都是同步执行，自然是会同步注册完。  

两种方式的最主要的区别是：   

- 链式调用的注册是前后依赖的
   比如上面的外部的第二个 then 的注册，是需要外部的第一个的 then 的执行。
- 变量定义的方式，注册都是同步的
   比如这里的 p.then 和 var p = new Promise 都是同步执行的。   

所以这里的代码执行就比较清晰了，内部都执行完成之后（因为都优先于外部的第二个 then 的注册）,再执行外部的第二个 then ：

output:  
外部promise
外部第一个then
内部promise
内部第一个then
内部第二个then
外部第二个then

## 最后第四段代码
```js
let p = new Promise((resolve, reject) => {
  console.log("外部promise");
  resolve();
})
p.then(() => {
    console.log("外部第一个then");
    new Promise((resolve, reject) => {
      console.log("内部promise");
      resolve();
    })
      .then(() => {
        console.log("内部第一个then");
      })
      .then(() => {
        console.log("内部第二个then");
      });
  })
p.then(() => {
    console.log("外部第二个then");
  });
```
这段代码中，外部的注册采用了非链式调用的写法，根据上面的讲解，我们知道了外部代码的 p.then 是并列同步注册的。所以代码在内部的 new Promise 执行完，p.then 就都同步注册完了。   

内部的第一个 then 注册之后，就开始执行外部的第二个 then 了（外部的第二个 then 和 外部的第一个 then 都是同步注册完了）。然后再依次执行内部的第一个 then ,内部的第二个 then。   

output:  
外部promise
外部第一个then
内部promise
外部第二个then
内部第一个then
内部第二个then


我相信，如果能看懂上面的四段代码之后，对 Promise 的执行和注册非常了解了。     

如果还是不太懂，麻烦多看几遍，相信你一定能懂~~~~~~~~   

核心思想：   

Promise 的 then 的 注册 和 执行 是分离的。      
注册 : 是完全遵循 JS 的代码的执行过程。    
执行 : 先 同步，再 微任务 ，再 宏观任务。   

只有分开理解上述，才能真正理解它们的执行顺序~~~~~~~~~~~~~~~~


最后出一道题：   
```js
new Promise((resolve, reject) => {
  console.log("外部promise");
  resolve();
})
  .then(() => {
    console.log("外部第一个then");
    new Promise((resolve, reject) => {
      console.log("内部promise");
      resolve();
    })
      .then(() => {
        console.log("内部第一个then");
      })
      .then(() => {
        console.log("内部第二个then");
      });
    return new Promise((resolve, reject) => {
      console.log("内部promise2");
      resolve();
    })
      .then(() => {
        console.log("内部第一个then2");
      })
      .then(() => {
        console.log("内部第二个then2");
      });
  })
  .then(() => {
    console.log("外部第二个then");
  });
```

你能知道输出结果，且能解释清楚吗？  