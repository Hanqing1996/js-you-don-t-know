#### try... catch 是同步的
```
function foo() {
    setTimeout(function () {
        let n=0
        console.log(n.splice(0,1))
    }, 100);
}
try {
    foo();
    // 在 setTimeout 的回调函数执行过程中抛出全局错误
}
catch (err) {
    // 永远不会到达这里，即错误无法被捕捉
}
```
准确来说，对于
```
try{
  fn()
}
catch(err){

}
只要 err 发生在一个异步任务中，就无法被捕捉
```


---
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

每次执行一个生成器函数，就会构建一个迭代器（实际上就隐式构建了生成器的一个实例），我们可以通过这个迭代器来控制这个生成器实例。
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
// 构造一个迭代器it来控制一个生成器的实例
var it = foo();
// 这里启动foo()！
it.next();
x; // 2
bar();
x; // 3
it.next(); // x: 3
```
* 提问和回答机制
> 注意 yield xxx 的意思是“生成器给迭代器一个 xxx”,不是"yield 的值是 xxx"

> it.next() 意思是“生成器要给我的下一个值是什么？”没有对 yield 进行赋值。

> it.next( 7 ) 的意思是“yield 的值被赋为 7,生成器要给我的下一个值是什么？”
```
function *foo(x) {
var y = x * (yield "Hello"); //
return y;
}
var it = foo( 6 );
var res = it.next(); //
res.value; // "Hello"
res = it.next( 7 ); 
res.value; // 42
```
```
it.next() 是在说：生成器 *foo(..) 要给我的下一个值是什么？

yield "Hello" 是在说：生成器 *foo(..) 给你"hello"

it.next( 7 ) 是在说： yield 的值被赋为 7,生成器 *foo(..) 要给我的下一个值是什么？
```
> 但是，稍等！与 yield 语句的数量相比，还是多出了一个额外的 next() 。所以，最后一个 it.next(7) 调用再次提出了这样的问题：生成器将要产生的下一个值是什么。但是，再没有 yield 语句来回答这个问题了，是不是？那么谁来回答呢？

> return 语句回答这个问题！如果你的生成器中没有 return 的话——在生成器中和在普通函数中一样， return 当然不是必需的——总有一个假定的 / 隐式的 return; （也就是 return undefined; ），它会在默认情况下回答最后的 it.next(7) 调用提出的问题。

> 注意启动生成器时（也就是第一个 next）一定要用不带参数的 next()。
---
 #### iterable

 iterable，是指一个对象，这个对象有一个 Symbol.iterator 函数，其返回值是一个迭代器。
 
 比如数组，就是一个 iterable。
 
 #### 迭代器
 * 对于一个 iterable，我们调用它的 Symbol.iterator 函数，就会得到一个迭代器
 ```
  var a = [1,3,5,7,9]; // 数组 a 就是一个 iterable
  var it = a[Symbol.iterator](); // 调用 a 的 Symbol.iterator 方法，得到一个迭代器
  it.next().value; // 1
  it.next().value; // 3
  it.next().value; // 5
  ..
 ```
 * 还记得我们前面提过的吗，执行一个生成器也会得到一个迭代器
 ```
 function * fn(){}
 var iterator=fn()
 ```
#### for..of
for..of 循环会自动调用一个 iterable 的 Symbol.iterator 
```
var a = [1,3,5,7,9];
for (var v of a) {
console.log( v );
}
// 1 3 5 7 9
```
#### 生成器与异步
```
function foo(x,y) {
    setTimeout(()=>{
        it.next( 'libai' );
    },5000)
    
    return 2
}
    
function *main() {
    var text = yield foo(1,3);
    console.log(text)

}
var it = main();
// 这里启动！
var res=it.next();
```
* 流程解析
1. it.next() 导致 main 函数开始执行 foo(1,3)
2. 执行 foo(1,3),返回2,setTimeout 的回调函数被放入事件队列;
3. 执行语句 yield 2,导致 2 被生成器交给迭代器 it;（注意不是 yield 的值被赋为 2！！！）
4. 执行语句 var res=it.next();console.log(res.value),打印出 2
5. 同步代码执行完毕，5s后执行回调函数（在这5s内,生成器函数 main 不继续执行，也就是说语句 var text = yield;console.log(text) 被阻塞），执行 it.next( 'libai' ),导致var text = ‘libai’;console.log(text) 被执行。

#### 生成器和 Promise 的共同之处
```
function foo(x,y) {
    setTimeout(()=>{
        it.next( 'libai' );
    },5000)
}
    
function *main() {
    var text = yield foo(1,3);
    console.log(text)
}
var it = main();
// 这里启动！
it.next();
```
改用 Promise 实现大概是这样的
```

function foo(){
    return new Promise(resolve=>{
        setTimeout(()=>{
            resolve('libai')
        },5000)
    })
}

async function main(){
    var text=await foo()
    console.log(text)
}

await main()
```
 * 都会阻塞代码，直到执行完某个语句（生成器是 it.next(xxx),Promise 是 resolve(xxx)）才会继续执行
 * 都可以在异步任务结束时捕捉到异步任务返回的结果（生成器用 yield xxx ,Promise 用 resolve(xxx)），也可以理解成都能将“获取结果”和“处理结果”的逻辑分开。

#### 同步错误处理 
生成器 yield 暂停的特性意味着我们不仅能够从异步函数调用得到看似同步的返回值，还可以同步捕获来自这些异步函数调用的错误！
```
function foo(x,y) {
    setTimeout(()=>{

        // 这里不是我们关心的重点，下面只是一个简单的同步代码中捕捉错误的过程
        try{
            let n=2
            n.splice(0,1)
        }
        catch(err){
            it.next(err)
        }

    },5000)
}
    
function *main() {
    var text = yield foo(1,3);
    console.log(text)
}

// 下面是我们关心的重点
try{
    var it = main();
    it.next();
}catch(err){
    console.log(err)
}
```
5s 后打印 TypeError: n.splice is not a function

事实上这里的错误捕捉仍然是同步的，只是 yield 的暂停机制保证了在错误被拿到并赋值给 yield 后才继续执行生成器函数 main。

