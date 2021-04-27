![Async Logo](https://raw.githubusercontent.com/caolan/async/master/logo/async-logo_readme.jpg)

[![Build Status via Travis CI](https://travis-ci.org/caolan/async.svg?branch=master)](https://travis-ci.org/caolan/async)
[![NPM version](https://img.shields.io/npm/v/async.svg)](https://www.npmjs.com/package/async)
[![Coverage Status](https://coveralls.io/repos/caolan/async/badge.svg?branch=master)](https://coveralls.io/r/caolan/async?branch=master)
[![Join the chat at https://gitter.im/caolan/async](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/caolan/async?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

*For Async v1.5.x documentation, go [HERE](https://github.com/caolan/async/blob/v1.5.2/README.md)*

Async 是一个功能强大的异步 JavaScript 模块。虽然最初是为了与 [Node.js](https://nodejs.org/) 一起使用，可以通过 `npm i async`安装,
它也可以直接在浏览器中使用。

Async 也能通过 yarn 安装:

- [yarn](https://yarnpkg.com/en/): `yarn add async`

Async提供了大约70个函数。其中包括集合 (`map`, `reduce`, `filter`, `each`…) 的异步扩展。以及常见的异步控制流模式 (`parallel`, `series`, `waterfall`…). 这些函数假定您遵循Node.js约定 (提供一个callback作为异步函数的最后一个参数-- 一个期望错误作为其第一个参数的回调—并调用回调一次)。

你也可以用 `async` 函数代替 callback-accepting 函数提供给 Async 方法.  更多信息,请参考 [AsyncFunction](global.html#AsyncFunction)


## Quick Examples

```js
async.map(['file1','file2','file3'], fs.stat, function(err, results) {
    // results is now an array of stats for each file
});

async.filter(['file1','file2','file3'], function(filePath, callback) {
  fs.access(filePath, function(err) {
    callback(null, !err)
  });
}, function(err, results) {
    // results now equals an array of the existing files
});

async.parallel([
    function(callback) { ... },
    function(callback) { ... }
], function(err, results) {
    // optional callback
});

async.series([
    function(callback) { ... },
    function(callback) { ... }
]);
```

还有更多的功能哦,请查看后面完整功能列表。如果您觉得还缺了点什么，请创建一个GitHub issue 给我们。

## 常见问题 [(StackOverflow)](http://stackoverflow.com/questions/tagged/async.js)

###  同步迭代函数 (Synchronous iteration functions)

在使用async的时候,如果你遇到类似 `RangeError: Maximum call stack size exceeded.`的报错或者其他栈溢出问题 , 你可能使用了同步迭代。使用“同步”，意味着一个方法与它的callback在同一个事件循环(javascript event loop)中执行，使用 I/O 或者 定时器 除外。调用太多的callback会使栈溢出。如果你遇到这个问题，只需要通过`async.setImmediate`方法启动一个新的调用栈在下一次事件循环中运行。



如果在某些情况下提早回调，这也偶然会产生：

```js
async.eachSeries(hugeArray, function iteratee(item, callback) {
    if (inCache(item)) {
        callback(null, cache[item]); // 如果太多的项目被缓存，也会栈溢出
    } else {
        doSomeIO(item, callback);
    }
}, function done() {
    //...
});
```

该成这样:

```js
async.eachSeries(hugeArray, function iteratee(item, callback) {
    if (inCache(item)) {
        async.setImmediate(function() {
            callback(null, cache[item]);
        });
    } else {
        doSomeIO(item, callback);
        //...
    }
});
```
出于性能考虑，Async没有做同步迭代器校验。如果仍然遇到堆栈溢出,可以按照上面的方法进行延迟。或者使用[`async.ensureAsync`](#ensureAsync)方法包装函数，这些函数本质上是异步的，因此不存在此问题，也不需要额外的回调延迟。



如果JavaScript的事件循环仍然有点模糊，请查看[this article](http://blog.carbonfive.com/2013/10/27/the-javascript-event-loop-explained/) or [this talk](http://2014.jsconf.eu/speakers/philip-roberts-what-the-heck-is-the-event-loop-anyway.html) 获取更多详细信息。


### 多次调用回调函数( Multiple callbacks)
确保在调用callback后`return`这个函数,否则在许多情况下会导致多次回调和不可预知的行为。


```js
async.waterfall([
    function(callback) {
        getSomething(options, function (err, result) {
            if (err) {
                callback(new Error("failed getting something:" + err.message));
                // we should return here
            }
            // since we did not return, this callback still will be called and
            // `processData` will be called twice
            callback(null, result);
        });
    },
    processData
], done)
```


每当回调调用不是函数的最后一个语句时，最好使用 `return callback(err, result)`。

### Using ES2017 `async` functions

Async 可以用 `async` functions 代替 Node-风格 回调函数。没有callback形参,通过`return` 替代`callback(null,result)`。通过抛异常`throw new Error()`代替 `callback(err)`。

```js
async.mapLimit(files, 10, async file => { // <- no callback!
    const text = await util.promisify(fs.readFile)(dir + file, 'utf8')
    const body = JSON.parse(text) // <- a parse error here will be caught automatically
    if (!(await checkValidity(body))) {
        throw new Error(`${file} has invalid contents`) // <- this error will also be caught
    }
    return body // <- return a value!
}, (err, contents) => {
    if (err) throw err
    console.log(contents)
})
```

我们只能识别到原生`async` functions，不包括被转移的版本(e.g. with Babel)。另外您可以通过`async.asyncify()`把`async` functions 包装成 node-style callback


### 绑定上下文 Binding a context to an iteratee

传递给Async的异步函数,this的上下文指向会被改变,需要通过`bind`方法指定上下文

```js
// Here is a simple object with an (unnecessarily roundabout) squaring method
var AsyncSquaringLibrary = {
    squareExponent: 2,
    square: function(number, callback){
        var result = Math.pow(number, this.squareExponent);
        setTimeout(function(){
            callback(null, result);
        }, 200);
    }
};

async.map([1, 2, 3], AsyncSquaringLibrary.square, function(err, result) {
    // result is [NaN, NaN, NaN]
    // This fails because the `this.squareExponent` expression in the square
    // function is not evaluated in the context of AsyncSquaringLibrary, and is
    // therefore undefined.
});

async.map([1, 2, 3], AsyncSquaringLibrary.square.bind(AsyncSquaringLibrary), function(err, result) {
    // result is [1, 4, 9]
    // With the help of bind we can attach a context to the iteratee before
    // passing it to Async. Now the square function will be executed in its
    // 'home' AsyncSquaringLibrary context and the value of `this.squareExponent`
    // will be as expected.
});
```

### 内存泄漏 Subtle Memory Leaks

在某些情况下，当您在另一个异步函数中调用Async方法时，您可能想尽早退出异步流：

```javascript
function myFunction (args, outerCallback) {
    async.waterfall([
        //...
        function (arg, next) {
            if (someImportantCondition()) {
                return outerCallback(null)
            }
        },
        function (arg, next) {/*...*/}
    ], function done (err) {
        //...
    })
}
```
有时候你想跳过瀑布流的剩余过程,你调用了外部的回调函数,但是Async还是会等待内部的`next`函数被调用,从而造成函数没有正确结束。

从 3.0版本, 你可以调用任何 `false` 作为 `error` 参数的Async 回调,让Async结束方法。

```javascript
        function (arg, next) {
            if (someImportantCondition()) {
                outerCallback(null)
                return next(false) // ← signal that you called an outer callback
            }
        },
```

### 处理集合时对集合进行改变 Mutating collections while processing them
如果你通过一个数组去调用Async的集合方法(例如 `each`, `mapLimit`, or `filterSeries`),
然后数组被`push`, `pop`, or `splice` 这些方法修改,这可能会导致意外,或者不确定的行为。Async会迭代到满足数组的原始 `length`次数。一些 `push`, `pop`, or `splice` 的索引已经被处理。因此，不建议在异步开始对其进行迭代之后修改该数组。 如果确实需要`push`, `pop`, or `splice`，请改用 `queue`。


## 下载

从
[GitHub](https://raw.githubusercontent.com/caolan/async/master/dist/async.min.js)下载源码.
也可以通过npm安装:

```bash
$ npm i async
```

然后 `require()`引入整个模块:

```js
var async = require("async");
```

或缺部分引入某个方法:

```js
var waterfall = require("async/waterfall");
var map = require("async/map");
```

__开发版:__ [async.js](https://raw.githubusercontent.com/caolan/async/master/dist/async.js) - 29.6kb 未压缩

###  在浏览器 In the Browser

Async 可以运行在任何 ES2015 环境 (Node 6+ and all modern browsers).

如果你想在更老的环境中使用Async, (e.g. Node 4, IE11) 你需要做如下转换.

Usage:

```html
<script type="text/javascript" src="async.js"></script>
<script type="text/javascript">

    async.map(data, asyncProcess, function(err, results) {
        alert(results);
    });

</script>
```


Async 的可移植版本, 包含 `async.js` and `async.min.js`, 在 `/dist` 文件夹下. Async 支持 [jsDelivr CDN](http://www.jsdelivr.com/projects/async).

### ES Modules 支持

Async包含一个 `.mjs`版本，该版本应由兼容的打包工具自动使用，例如Webpack或Rollup等任何使用 `package.json`的`module`字段的东西。

我们还在npm上的另一个`async-es`包中提供Async作为纯ES2015模块的集合。

```bash
$ npm install async-es
```

```js
import waterfall from 'async-es/waterfall';
import async from 'async-es';
```

### Typescript 支持

Async 的第三方类型定义。

```
npm i -D @types/async
```
建议在您的`tsconfig.json`中编译选项中配置ES2017或更高版本，这样会保留`async`函数：

```json
{
  "compilerOptions": {
    "target": "es2017"
  }
}
```

## 其他仓库(友情链接)

* [`limiter`](https://www.npmjs.com/package/limiter) a package for rate-limiting based on requests per sec/hour.
* [`neo-async`](https://www.npmjs.com/package/neo-async) an altername implementation of Async, focusing on speed.
* [`co-async`](https://www.npmjs.com/package/co-async) a library inspired by Async for use with [`co`](https://www.npmjs.com/package/co) and generator functions.
*  [`promise-async`](https://www.npmjs.com/package/promise-async) a version of Async where all the methods are Promisified.

-----------------------
-----------------------

---------华丽的分割线, 后面的译者的理解------------------

-----------------------
-----------------------


## Async 主要包含三个部分的函数

### 1. 对集合的异步拓展,Array.prototype上方法的补充,原生是不支持异步的
   - concat
   - detect find
   - each forEach  --不带index
   - eachOf forEachOf --带index
   - every
   - filter
   - groupBy
   - map
   - mapValues
   - reduce reduceRight transform --都是做累加的
   - reject --filter的补集
   - some
   - sortBy
 - 以上方法又有Limit变体版本
 - Limit是限制并发数量的
 - Series又是Limit并发数为1的变体版本

### 2. 异步工作流
* 工作流
   - queue 队列,执行器只能拿到当前执行的任务
   - priorityQueue 跟queue相同,但是任务带执行优先级
   - cargo 队列,执行器中拿到队列中的所有任务信息
   - waterfall 瀑布,从上往下执行,最后的回调拿到瀑布中最后一个函数的执行结果
   - series 一个一个运行,最后的回调拿到所有函数的执行结果
   - auto 自动运行,根据异步函数的依赖关系自动执行
   - times 执行同一个函数多次
   - retry 执行失败会重试,直到成功(默认重试5次,重试间隔0秒)
   - forever 一直执行这个函数,除非报错

* Promise对应的Async实现
   - parallel 一起执行,任何一个报错就退出,等同于Promise.all [拿到全部正确的结果]
   - Promise.allSettled [拿到全部结果,不论对错],这个Async没有对应的实现,reflect和reflectAll可以阻止错误终端工作流
    ```js
    async.allSettled=async.parallel(async.reflectAll(tasks))
    ```
   - tryEach 任何一个异步函数成功就会退出工作流,等同于Promise.any [拿到第一个正确的结果]
   - race 任何一个异步函数成功或者失败就是退出工作流,等同于Promise.race[拿到第一个执行完成的结果,不论对错]

* 循环工作流
    - whilst while(condition){statement}  条件满足继续循环
    - doWhilst do{statement}while(condition) 至少会执行一次
    -
    - until  loop{}until(condition)  条件满足退出循环
    - doUntil do{}until(condition) 至少会执行一次


* 辅助函数
   - compose 打散管道方法 例如 `a(b(c()))` compose(a,b,c)
   - seq compose的可读版本 seq(a,b,c) 对应的是 `c(b(a()))`
   - applyEach 多个异步方法的参数相同,并发版
   - applyEachSeries 多个异步方法的参数相同,排队版

### 3. 辅助函数
- apply 多参数转换成只有一个callback参数的函数
- asyncify,wrapSync 包装 ES2017 async function 成 Node-style风格AsyncFunction
- constant 包装常量成 AsyncFunction
- dir,log 方便调试输出异步函数的执行结果
- ensureAsync 确保函数是异步执行,防止同步调用迭代栈溢出
- memoize unmemoize 慢函数增加缓存,清除缓存
- nextTick 将函数放入下一次事假循环执行
- reflect reflectAll 返回一个新的函数,及时callback(error)也不会导致工作流结束
- setImmediate 跟nextTick差不多
- timeout 给异步函数增加超时
