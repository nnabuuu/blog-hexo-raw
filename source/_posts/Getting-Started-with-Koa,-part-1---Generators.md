title: 'Getting Started with Koa, part 1 - Generators'
date: 2014-12-05 23:58:10
tags: translation
---

#开始使用Koa，第一部分 - Generators （译）

原文地址：http://blog.risingstack.com/introduction-to-koa-generators/

原作者： Gellért Hegyi [https://twitter.com/native_cat](https://twitter.com/native_cat)


[Koa](http://koajs.com/)是一个小巧而简单的web框架，由[Express](http://expressjs.com/)的team带来，其目标是创造一个更加现代的开发web的方式。

在本系列中，你会了解Koa的机制，学习如何高效正确地使用它来编写web应用。第一部分包含一些基本概念（generators, thunks）

## 为什么是Koa？

Koa包含的关键特性允许你简单并且快速地编写web应用（并且不包含callback）。它使用来自于[ES6](http://wiki.ecmascript.org/doku.php?id=harmony:specification_drafts)的新语言元素使得控制逻辑管理在Node中比其他框架更加容易。

Koa自身非常小巧，这是是因为不同于试下流行的web框架（例如express），Koa追随高度模块化的原则，意味着每一个模块只做一件事情。将这句话记在脑中，让我们开始吧！

## 你好Koa

```js
var koa = require('koa');  
var app = koa();

app.use(function *() {  
  this.body = 'Hello World';
});

app.listen(3000);  
```
在我们开始之前，为了运行示例以及你自己的ES6代码，你需要使用0.11.9或更高的版本并设置`--harmony`标志位。

你可以从上面的示例中看到，这里没有什么让人感兴趣的点，除了在函数声明之后比较陌生的`*`号。这样，就使这个函数变成了一个generator函数。

## Generators

	当你执行函数时，如果能够在任何点暂停它，进行一些其他计算，做一些其他操作，再返回到这个函数里，并带有一些值，然后继续，这样不好吗？

这是另外一种迭代器（像循环一样）。那就是一个generator做的最好的事情，它在ES6中被实现，因此我们可以轻松使用它。

让我们来构造一些generators！首先我们需要创建你的generator函数，它看起来和普通的函数完全一样，除了一点，你需要在`function`关键词后放置一个`*`符号。

```js
function *foo () { }  
```

现在我们就有了一个*generator函数*。当我们调用这个函数时它会返回一个迭代对象，因此不像普通的函数调用，当我们调用一个generator时，代码并没有开始执行，其原因我们会在之后讲述，我们将手动地遍历它。

```js
function *foo (arg) { }  // generator function  
var bar = foo(123);      // iterator  object  
```
通过它返回的对象`bar`，我们可以遍历这个函数。为了开始并且迭代到下一个generator步骤我们可以简单的调用`bar`的`next()`方法，当`next()`被调用时，函数开始执行，或从上一次停止的地方执行，直到它到达一个暂停点。

但是除了继续，它也返回一个对象，该对象给予有关generator的信息。它有一个`value`属性，标识当我们暂停generator时当前迭代的值。另外一个布尔值是`done`，它用来标识generator已经结束执行。

```js
function *foo (arg) { return arg }  
var bar = foo(123);  
bar.next();          // { value: 123, done: true }  
```
像我们看到的那样，实际上示例那里并没有任何暂停，因此当它返回一个对象的时候`done`就变成了`true`。如果你在generator中指定一个return值，它会在最后一个迭代对象（当`done`是`true`的时候）中被返回。现在我们要做的只是来暂停一个generator。和我们所说的那样，它就像是遍历一个函数并且在每个迭代周期它产出（yield）一个值（在我们暂停的地方），因此我们使用`yield`关键词进行暂停。

## yield

	yield [[expression]];
	
调用`next()`会使generator开始执行并且它会运行直到遇到一个`yield`。然后它返回一个带有`value`和`done`属性的对象，这里`value`持有表达式的值。该表达式可以是任何形式。

```js
function* foo () {  
  var index = 0;
  while (index < 2) {
    yield index++;
  }
}
var bar =  foo();

console.log(bar.next());    // { value: 0, done: false }  
console.log(bar.next());    // { value: 1, done: false }  
console.log(bar.next());    // { value: undefined, done: true }  
```
当我们再一次调用`next()`时，*yield值*会在generator中返回并且它会继续执行。也可以在generator中从*迭代对象*接受一个值（使用`next(val)`），然后当它geneator继续时它会被返回到generator中。

## 错误处理

如果你在迭代对象的值中发现了错误，你可以使用`throw()`方法并在generator中捕获这个错误。这使得错误处理在generator中是非常友好的。

```js
function *foo () {  
  try {
    x = yield 'asd B';   // Error will be thrown
  } catch (err) {
    throw err;
  }
}

var bar =  foo();  
if (bar.next().value == 'B') {  
  bar.throw(new Error("it's B!"));
}
```
## for ... of

在ES6中有一个循环类别，可以用来在generator中进行遍历，即`for...of`循环。该遍历器会进行执行直到`done`是`false`。留意一点，如果你使用这个循环，你将无法在一个`next()`的调用中中传递值，并且该循环会抛弃返回值。

```js
function *foo () {  
  yield 1;
  yield 2;
  yield 3;
}

for (v of foo()) {  
  console.log(v);
}
```

## yeild *

如之前所说，你可以yield几乎任何东西，甚至可以yield一个generator，但是那样你就必须使用`yield *`，这被称为代理。你正在代理到另一个generator上，因此你可以使用一个迭代对象对多个generator进行遍历。

```js
function *bar () {  
  yield 'b';
}

function *foo () {  
  yield 'a'; 
  yield *bar();
  yield 'c';
}

for (v of foo()) {  
  console.log(v);
}
```
## Thunks

为了完全理解Koa，thunks是另外一种你必须掌握的概念。它们主要用来帮助调用另外一个函数。你可以把它和*lazy evaluation*联系起来。对我们来说，最重要的是它们可以用来在一个函数调用的外部从参数列表中移动node的回调函数。

```js
var read = function (file) {  
  return function (cb) {
    require('fs').readFile(file, cb);
  }
}

read('package.json')(function (err, str) { })  
```
有一个小型的模块叫做[thunkify](https://github.com/visionmedia/node-thunkify)，它将一个普通的node函数转化为一个thunk。你可以质疑它的使用，但是其结果是它可以很好的移除generator中的回调。

首先，我们需要将想要在generator中使用的node函数装换为一个*thunk*。然后在generator中使用这个thunk，就像它会返回一个值一样，否则我们就必须进入到回调中了。当调用起始`next()`时，它的value会是一个函数，它的参数是*被thunkified的*函数的回调。在回调中我们可以检验错误（并且进行`throw`，如果需要的话）或者用接收到的值调用`next()`。

```js
var thunkify = require('thunkify');  
var fs = require('fs');  
var read = thunkify(fs.readFile);

function *bar () {  
  try {
    var x = yield read('input.txt');
  } catch (err) {
    throw err;
  }
  console.log(x);
}
var gen = bar();  
gen.next().value(function (err, data) {  
  if (err) gen.throw(err);
  gen.next(data.toString());
})
```
慢慢花时间了解这个例子的每一个细节，因为这对于理解Koa非常重要。如果你专注于示例的generator部分，你会看到它非常棒。它有同步代码的简洁，优秀的错误捕获，而它仍然是异步执行的。

## 待续...

最后的例子看起来很繁琐，但是在下一部分我们会找到一些工具来处理繁琐的部分，仅剩优美的部分。并且我们最终会明白Koa以及它流畅的机制，正事该机制使得web开发如此简单。

