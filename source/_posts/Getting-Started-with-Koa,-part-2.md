title: 'Getting Started with Koa, part 2'
date: 2014-12-06 22:37:41
tags:
---

#开始使用Koa，第二部分（译）

http://blog.risingstack.com/getting-started-with-koa-part-2/

原作者： Gellért Hegyi [https://twitter.com/native_cat](https://twitter.com/native_cat)

在[上一节](http://blog.risingstack.com/introduction-to-koa-generators/)我们掌握了generators并且明白了一点，那就是我们可以**以同步的形式编写代码而异步的执行**。这非常棒，因为同步代码非常简单，优雅而且具有更好的可读性，而异步代码可能会导致“尖叫”与“哭泣”（回调地狱）。

这一章将讲述解决这个痛苦的工具，这样我们就可以只编写那些有趣的部分。它会给予基本Koa特性和机制的介绍。

## 前述

```js
// First part
var thunkify = require('thunkify');
var fs = require('fs');
var read = thunkify(fs.readFile);
 
// Second part
function *bar () {
  try {
    var x = yield read('input.txt');
  } catch (err) {
    console.log(err);
  }
  console.log(x);
}
 
// Third part
var gen = bar();
gen.next().value(function (err, data) {
  if (err) {
    gen.throw(err);
  }
  gen.next(data.toString());
});
```
在上一篇文章的最后一个例子里，如你所见，我们可以将其分为3个重要地部分。第一是我们必须创造我们的**thunkified functions**，它会被用在一个generator中。然后我们必须使用thunkified functions编写我们的**generator functions**。最后一个部分是我们调用并遍历generators，处理异常以及其他。如果你想一想你会发现，最有一部分和我们程序的本质并没有什么关系，基本上它让我们运行一个generator。幸运地是，有一个模块帮我们这样做。来见一见**co**。

## CO

[**Co**](https://github.com/visionmedia/co)是一个node的基于generator的流程控制模块。下面的代码和上面做的事情完全一样，但是我们摆脱了调用generator的代码。唯一我们需要做的事情，是**将generator传递给一个函数`co`**,**调用它**，然后魔力就发生了。好吧，其实并不是什么魔力，它只是为你处理所有的generator调用代码，因此我们不需要为那操心了。

```js
var co = require('co');
var thunkify = require('thunkify');
var fs = require('fs');
 
var read = thunkify(fs.readFile);
 
co(function *bar () {
  try {
    var x = yield read('input.txt');
  } catch (err) {
    console.log(err);
  }
  console.log(x);
})();
```

就像我们已经知道的那样，你可以防止一个`yield`z在任何东西之前来**对一些东西进行评价**。因此并不是只有*trunks*可以被`yield`。因为`co`想要创建一个简单的控制流，它对一些特殊的类型进行yield。当前**支持yield的类型**：

* thunks（函数)
* array （并发执行）
* objects （并发执行）
* generators （代理）
* generator functions （代理）
* promises

我们已经讨论过**thunks**是如何工作的了，因此让我们去看看其他的。

## 并发执行

```js
var read = thunkify(fs.readFile);
 
co(function *() {
  // 3 concurrent reads
  var reads = yield [read('input.txt'), read('input.txt'), read('input.txt')];
  console.log(reads);
 
  // 2 concurrent reads
  reads = yield { a: read('input.txt'), b: read('input.txt') };
  console.log(reads);
})();
```

如果你`yield`一个数组或一个对象，它将**并行地评估它的内容**。当然你的集合也可以是`thunks`、`generators`。你可以**nest**，它会穿过数组或者对象并发执行你所有的函数。**注意**：被yield的结果不会是展开的，它会保留同样的结构。

```js
var read = thunkify(fs.readFile);
 
co(function *() {
  var a = [read('input.txt'), read('input.txt')];
  var b = [read('input.txt'), read('input.txt')];
 
  // 4 concurrent reads
  var files = yield [a, b];
  
  console.log(files);
})();
```

你也可以通过在thunk的调用**之后**进行yield实现并发。

```js
var read = thunkify(fs.readFile);
 
co(function *() {
  var a = read('input.txt');
  var b = read('input.txt');
 
  // 2 concurrent reads
  console.log([yield a, yield b]);
 
  // or
 
  // 2 concurrent reads
  console.log(yield [a, b]);
})();
```

## 代理

你当然也可以yield generators。注意我们并不需要使用`yield *`。

```js
var stat = thunkify(fs.stat);
 
function *size (file) {
  var s = yield stat(file);
 
  return s.size;
}
 
co(function *() {
  var f = yield size('input.txt');
 
  console.log(f);
})();
```

我们过一下产不多所有你使用`co`时进行yield的可能性。这里有一个最新的示例（取自于co的[github页面](https://github.com/visionmedia/co)）把它们结合起来。

```js
var co = require('co');
var fs = require('fs');
 
function size (file) {
  return function (fn) {
    fs.stat(file, function(err, stat) {
      if (err) return fn(err);
      fn(null, stat.size);
    });
  }
}
 
function *foo () {
  var a = yield size('un.txt');
  var b = yield size('deux.txt');
  var c = yield size('trois.txt');
  return [a, b, c];
}
 
function *bar () {
  var a = yield size('quatre.txt');
  var b = yield size('cinq.txt');
  var c = yield size('six.txt');
  return [a, b, c];
}
 
co(function *() {
  var results = yield [foo(), bar()];
  console.log(results);
})();
```

我相信现在你已经足够掌握generators了，你已有有了一个关于如何使用这些工具操作**异步控制流**的很好的概念。

现在是时候转到我们整个系列的主题，**Koa**了！

## Koa

你所需要知道的，关于`koa`模块自身的信息并没有多少。你甚至可以看它的源码，只有4个文件，每个文件约300行。Koa遵循你写的每个程序都只做并且做好一件事情的传统。因此你会看到，每个好的koa模块都是（并且每个node模块都应该是）简短的。只做一件事情并且重度依赖于其他模块。你应该记住这些并且依照它进行开发。它会有利于每个人，你以及其他阅读你源代码的人。记住这一点然后让我们移动到Koa的主要特性。

## 应用

```js
var koa = require('koa');
var app = koa();
```

创建一个Koa应用只是简单的引入模块函数。这提供给你一个对象，它包含了一个generators（中间件）的数组，对一个新的请求以类似栈的方式进行执行。

## 级联

一个重要的术语，当使用Koa时，它指的是**中间件**。因此让我们先弄清楚这个。

Koa中的中间件是处理请求的函数。一个由Koa创建的服务器可以有一个与它关联的中间件的栈。

Koa中的级联的意思是，控制流穿过一系列的中间件。在Web开发中这是非常重要的，你可以简单地用这种方式构造复杂的行为。Koa使用generator非常直观并且简洁地实现它。**它yield下游，然后控制流程回到上游**。要向流程中添加一个generator，调用`use`函数并传入一个generator。试着猜猜看为什么下面的代码对每一个到达的请求产生`A, B, C, D, E`的输出！

这是一个服务器，因此`listen`函数进行你所认为的行为，它监听一个特殊的端口。（它的参数和纯node的`listen`方法是一样的）。

```js
app.use(function *(next) {
  console.log('A');
  yield next;
  console.log('E');
});
 
app.use(function *(next) {
  console.log('B');
  yield next;
  console.log('D');
});
 
app.use(function *(next) {
  console.log('C');
});
 
app.listen(3000);
```

当一个新的请求到达时，它开始流经中间件，以你所编写的顺序执行。因此在示例中，请求触发了第一个中间件，它输出`A`，然后到达`yield next`。**当一个中间件到达`yield next`，它会走向下一个中间件直到离开。**因此我们到达第二个中间件它输出`B`。然后又跳转到最后一个`C`。没有更多的中间件了，我们完成了**下游流程**，现在我们开始返回到之前的中间件（就像一个栈），`D`。然后第一个中间件结束，`F`，然后我们成功的完成了上游。

在这一点上，`koa`模块自身并没有包含任何其他的复杂性 - 因此我们不拷贝/粘贴已经很完备的[Koa站点](http://koajs.com/)的文档，你可以再那里阅读。这里是这些部分的链接：

[Context](http://koajs.com/#context)

[Request](http://koajs.com/#request)

[Response](http://koajs.com/#response)

[Others](http://koajs.com/#settings)

让我们看一个例子（也是来自于Koa的站点），它使用而来HTTP功能。第一个中间件计算响应时间。看看你能够多么容易地接触到响应的开始和结束，多么优雅地分离这些函数行为。

```js
app.use(function *(next) {
  var start = new Date;
  yield next;
  var ms = new Date - start;
  this.set('X-Response-Time', ms + 'ms');
});
 
app.use(function *(next) {
  var start = new Date;
  yield next;
  var ms = new Date - start;
  console.log('%s %s - %s', this.method, this.url, ms);
});
 
app.use(function *() {
  this.body = 'Hello World';
});
 
app.listen(3000);
```

## 完成

现在你已经熟悉Koa的核心了，你可以说你的旧web框架已经做了所有其他有用的事情并且你需要它们。但是也请记住那里还有数以万计的功能你从未使用过，或者还有一些工作并未按照你想象的方式工作。这就是Koa以及现代node框架好的方面。**你可以从`npm`中以简短模块的方式向你的应用添加需要的功能，并且它会完全按照你需要的方式运行。**

在下一章，我们会看到，使用Koa以上述哲学创建一个网站是多么简单。再那之后，再使用这些你已经了解的知识来感受一下Koa和它的自然感吧。
