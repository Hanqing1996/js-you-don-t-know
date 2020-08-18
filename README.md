## 异步
#### 在ES6以前，事件调度是由环境完成的
JavaScript 引擎本身并没有时间的概念，只是一个按需执行 JavaScript 任意代码片段的环境。“事件”（JavaScript 代码执行）调度总是由包含它的环境进行。

所以，举例来说，如果你的 JavaScript 程序发出一个 Ajax 请求，从服务器获取一些数据，那你就在一个函数（通常称为回调函数）中设置好响应代码，然后 JavaScript 引擎会通知宿主环境：“嘿，现在我要暂停执行，你一旦完成网络请求，拿到了数据，就请调用这个函数。”

然后浏览器就会设置侦听来自网络的响应，拿到要给你的数据之后，就会把回调函数插入到事件循环，以此实现对这个回调的调度执行。

---
#### 完整运行特性
> 函数一旦开始执行，在它执行完毕前，其它函数不会执行。

由于<strong> JavaScript 的单线程特性 </strong>， foo() （以及 bar() ）中的代码具有原子性。也就是说，一旦 foo() 开始运行，它的所有代码都会在 bar() 中的任意代码运行之前完成，或者相反。这称为完整运行（run-to-completion）特性。
```
var a = 1;
var b = 2;
function foo() {
a++;
b = b * a;
a = b + 3;
}
function bar() {
b--;
a = 8 + b;
b = a * 2;
}
// ajax(..)是某个库中提供的某个Ajax函数
ajax( "http://some.url.1", foo );
ajax( "http://some.url.2", bar );
```
由于 foo() 不会被 bar() 中断， bar() 也不会被 foo() 中断，所以这个程序只有两个可能的输出，取决于这两个函数哪个先运行——如果存在多线程，且 foo() 和 bar() 中的语句可以交替运行的话，可能输出的数目将会增加不少！

---
#### 竞态条件
对于
```
var a = 1;
var b = 2;
function foo() {
a++;
b = b * a;
a = b + 3;
}
function bar() {
b--;
a = 8 + b;
b = a * 2;
}
// ajax(..)是某个库中提供的某个Ajax函数
ajax( "http://some.url.1", foo );
ajax( "http://some.url.2", bar );
```
由于不能确定两个 ajax 任务谁先完成，所以 foo 和 bar 的调用先后顺序是不确定的。
* 输出 1：
```
var a = 1;
var b = 2;
// foo()
a++;
b = b * a;
a = b + 3;
// bar()
b--;
a = 8 + b;
b = a * 2;
a; // 11
b; // 22
```
* 输出 2：
```
var a = 1;
var b = 2;
// bar()
b--;
a = 8 + b;
b = a * 2;
// foo()
a++;
b = b * a;
a = b + 3;
a; // 183
b; // 180
```
同一段代码有两个可能输出意味着还是存在不确定性！但是，这种不确定性是在函数（事件）顺序级别上，而不是多线程情况下的语句顺序级别（或者说，表达式运算顺序级别）。换句话说，这一确定性要高于多线程情况。

在 JavaScript 的特性中，这种函数顺序的不确定性就是通常所说的竞态条件（race condition）， foo() 和 bar() 相互竞争，看谁先运行。具体来说，因为无法可靠预测 a 和 b的最终结果，所以才是竞态条件。

---
## 回调
#### 控制反转
> 回调最大的问题是控制反转，它会导致信任链的完全断裂。

```
xxxAjax( "..", function(..){} );
```
xxxAjax 往往是一个第三方的函数（比如 axios），这意味着我们不知道它怎么调用我们传入的回调函数。比如：

* 调用回调过早
* 调用回调过晚（或没有调用）；
* 调用回调的次数太少或太多
* 没有把所需的环境 / 参数成功传给你的回调函数；
* 吞掉可能出现的错误或异常；
* ……

这感觉就像是一个麻烦列表，实际上它就是。你可能已经开始慢慢意识到，对于被传给你
无法信任的工具的每个回调，你都将不得不创建大量的混乱逻辑。

---
## Promise
#### 反转控制反转
> 回调最大的问题在于控制反转，即将回调函数的调用权限交给了不能完全信任的第三方API。要想解决这个问题，就要思考第三方API到底做了什么。显然，第三方API在得到某些结果（比如服务器返回的数据）后，将这些结果作为参数传入了回调函数。。等等，也就是说，其实作为参数的结果其实才是最重要的，如果我们能拿到结果，完全可以自己控制怎么调用回调函数。Promise 就此应运而生。

> 总结一下，Promise 通过”拿到结果“和”处理结果“的逻辑分离，将回调函数的控制权抢了回来。
```
const fetch = (api, method, data) => {

  return new Promise((resolve, reject) => {
    wx.request({
      url: url[api], 
      method: method == 'POST' ? 'POST' : 'GET',
      data: data ? data : {},
      header: {
        'content-type': 'application/x-www-form-urlencoded'
      },
      success: (res) => {
        resolve(res) // 拿到结果，但是并不做处理
      },
      fail: (err) => {
        reject(err)
      }
    })
  })

}

// 对结果做处理，即执行回调函数，注意回调函数是由我执行的，控制权在我
const res = await fetch('login', 'GET', {
    code
})

const data = parseResponse(res)
```

#### 鸭子类型
识别 Promise（或者行为类似于 Promise 的东西）就是定义某种称为 thenable 的东西，将其定义为任何具有 then(..) 方法的对象和函数。我们认为，任何这样的值就是Promise 一致的 thenable。

根据一个值的形态（具有哪些属性）对这个值的类型做出一些假定。这种类型检查（type check）一般用术语鸭子类型（duck typing）来表示——“如果它看起来像只鸭子，叫起来像只鸭子，那它一定就是只鸭子”

---
## ES6 生成器（generator）

生成器是一类特殊的函数，可以一次或多次启动和停止，而不遵从之前提到的”完整运行特性“。
```
var x = 1;
function *foo() {
x++;
yield; // 暂停！
console.log( "x:", x );
}
function bar() {
x++;
}
```
```
// 构造一个迭代器it来控制这个生成器
var it = foo();
// 这里启动foo()！
it.next();
x; // 2
bar();
x; // 3
it.next(); // x: 3
```


