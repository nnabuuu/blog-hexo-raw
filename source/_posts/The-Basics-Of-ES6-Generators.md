title: 'The Basics Of ES6 Generators'
date: 2014-12-21 11:33:48
tags: js, generator, tanslation
---

#ES6Generator基础（译）

原文地址：http://davidwalsh.name/es6-generators

ES6 Generators：全系列

1. [The Basics Of ES6 Generators](http://davidwalsh.name/es6-generators)
2. [Diving Deeper With ES6 Generators](http://davidwalsh.name/es6-generators-dive)
3. [Going Async With ES6 Generators](http://davidwalsh.name/async-generators)
4. [Getting Concurrent With ES6 Generators](http://davidwalsh.name/concurrent-generators)

Javascript ES6 带来的一种最激动人心的新特性是一种新的函数，称为**generator**。它的名字可能有一点奇怪，但是它的行为在第一次看来却*更让人陌生*。这篇文章旨在解释它的是如何工作的基础概念，并且使你明白为什么它们对于JS的未来如此的强大。

##运行至完成

当我们讨论generator时我们所要见到的第一件事情就是：它们与普通函数对于“运行至完成”的概念是多么的不同。

无论你是否意识到，你对于函数的基础已经常常可以断言一些事情：一旦函数开始执行，在其他JS代码可以执行之前，它总会执行到结束。

例：

```js
setTimeout(function(){
    console.log("Hello World");
},1);

function foo() {
    // NOTE: don't ever do crazy long-running loops like this
    for (var i=0; i<=1E10; i++) {
        console.log(i);
    }
}

foo();
// 0..1E10
// "Hello World"
```

在这里，`for`循环会花费非常长的时间直到结束，至少比一毫秒要长，但是我们的计时器回调`console.log(...)`语句无法在`foo()`函数执行时打断它，因此它被堵塞语句之后（在event-loop中）并耐心地等到自己的回合。

如果`foo()`可以被打断，会怎么样呢？那样不会对我们的程序造成极大的破坏吗？

那就是多线程编程的<del>噩梦</del>挑战。但我们非常幸运地处在JavaScript的世界里而不必担心这些事情，因为JS总是单线程的（只有一个命令/函数可以在一个时刻执行）。

注：Web Worker是一种机制，它允许你开启一个完全单独的线程让一部分JS程序在其中执行，该线程完全与你的主JS程序线程并行。此机制并未向我们的程序中引入多线程并发，是因为两个线程只能通过普通的异步事件进行通信，而它总是遵循event-loop的*一次执行一个*的行为，该行为是由“运行至完成”所要求的。

##运行..停止..运行

对于ES6 generator，我们有另一种类型的函数，它可能在中间*暂停*，一次或多次，并在*之后*恢复，允许其他代码在暂停的间隙执行。

如果你曾经阅读过任何有关并发或基于线程的编程的内容，你可能见过“协作”这个词，它基本上意味着一个进程（在我们的例子中，是一个函数）自身选择何时它允许一个中断，这样它可以与其他代码进行**“协作”**。与这个概念相对应的是“抢占”，它表明一个进程/函数可以不依照其意愿被打断。

ES6 generator函数在其并发行为上是“协作”式的。在generator函数体内，你可以使用新的`yield`关键词来从函数自身中暂停。没有任何东西可以从一个generator的外部暂停它，仅当generator（在内部）遇到一个yield时，它才会暂停自己。

但是，一旦generator使用`yield`暂停了自身，它自己无法进行恢复。必须使用一个外部的控制方式重启generator。我们会在下面解释这是如何发生的。

因此，基本上，一个generator函数可以停止以及被重启任意次数。事实上，你可以在一个无限循环（例如臭名昭著的`while (true) { .. }`）中指定一个generator函数，尽管这基本上是非常疯狂的并且在普通的JS程序中是错误的，而使用generator函数这是完全正常的并且有时候正事你所希望做的那样。

甚至更为重要的是，停止和重启并不仅仅是控制generator函数的执行，它也使得执行时的双向的信息传递（出入generator）变为可能。使用普通函数时，你会在开始获取参数并在结束时`return`一个值。使用generator函数，你可以使用每一个`yield`发出消息，并且你可以在每次重启时回传消息。

## 请告诉我语法，谢谢！

让我们看看这些新的激动人心的generator函数的语法。

首先，它有新的声明语法：

```js
function *foo() {
    // ..
}
```

注意到这里的`*`了吗？这是一种新的语法并且看起来有一点奇怪。对于那些来自于其他语言的开发者来说，这可能看起来非常像函数返回值的指针。但是请不要被它迷惑！这仅仅是一个来标志特殊的generator函数类型的方式。

你可能见过其它文章/文档使用`function* foo(){ }` 而不是`function *foo(){ }` （`*`的位置不同）。它们都是有效的，但是我最近决定使用`function *foo() {}`因为这样更加准确，因此我会在这里这样写。

现在，让我们来谈谈我们的generator函数的内容。generator函数在很多方面都只是普通的JS函数。在generator函数内部，新的语法非常少。

我们主要使用的语法，入上面所述，是`yield`关键词。`yield ___`被称为“yield表达式”（不是一个语句），因为当我们重启generator时，我们会回传一个值到内部，并且无论我们返回的是什么它都会被作为`yield ___`表达式的计算结果。

例：

```js
function *foo() {
    var x = 1 + (yield "foo");
    console.log(x);
}
```

这里，当generator函数在`yield "foo"`暂停时，表达式会发送`"foo"`字符串出去，并且无论何时，只要generator被重启，无论什么值被发送，它会被作为这个表达式的结果，这个值然后会加上`1`并且赋值给`x`变量。

看见这里的双向交流了吗？你发送`"foo"`出去，暂停你自己，然后在某一个*之后*的时间点（可能是立即，也可能是距离现在非常长的时间！），generator会被重启并且提供你一个返回值。差不多`yield`关键词是某种创造向外请求一个值的方式。

在任何表达式的地方，你可以仅仅使用`yield`自身，这表明你向外`yield`一个`undefined`。像这样：

```js
// note: `foo(..)` here is NOT a generator!!
function foo(x) {
    console.log("x: " + x);
}

function *bar() {
    yield; // just pause
    foo( yield ); // pause waiting for a parameter to pass into `foo(..)`
}
```

##Generator迭代器

“Generator Iterator”。的确又长又难念，是吧？

迭代器是一种特殊的行为类型，事实上，它是一种设计模式，我们按某种顺序遍历一组值，每次通过调用`next()`获取一个值。例如设想一下对这样一个数组`[1, 2, 3, 4, 5]`使用一个迭代器。第一个`next()`调用会返回`1`，第二个`next()`会返回`2`，并一次类推。当所有的值都被返回之后，最后的`next()`会返回`null`或者`false`，或者其他的信号标志你已经迭代完容器内的所有值。

我们从外部控制generator函数的方式，是使用一个*generator iterator*构造并且交互。这听起来比实际上做起来要复杂。想想一下这个很蠢的例子：

```js
function *foo() {
    yield 1;
    yield 2;
    yield 3;
    yield 4;
    yield 5;
}
```

为了从`*foo()`这个generator函数中遍历值，我们需要构造一个迭代器，如何做呢？非常简单！

```js
var it = foo();
```

噢！所以，用普通方式调用generator函数实际上并没有执行它的任何内容。

把这些塞进你的大脑有一点奇怪。你可能尝试疑问：为什么不是`var it = new foo()`？好吧，这个语法背后的原因很复杂并且不在我们所要讨论的范围内。

那么现在，开始迭代我们自己的generator函数，我们只需：

```js
var message = it.next();
```

这会给我们从`yield 1`语句中提供`1`，但这并不是唯一我们得到的东西。

```js
console.log(message); // { value:1, done:false }
```

我们事实上从每个`next()`调用中获取了一个对象，它包含一个`value`属性用来描述`yield`出的值，而`done`是一个布尔值它用来标志是否generator函数是否已经完全结束。

让我们继续我们的迭代：

```js
console.log( it.next() ); // { value:2, done:false }
console.log( it.next() ); // { value:3, done:false }
console.log( it.next() ); // { value:4, done:false }
console.log( it.next() ); // { value:5, done:false }
```

有趣的是，当我们取出``5`时，done`仍然是`false`。这是因为*严格来说*，generator函数并没有完成。我们仍然需要调用最后一个`next()`，并且我们传入一个值时，它会被设置为`yield 5`表达式的结果。只有**那时**，generator函数才算是结束了。

那么，现在：

```js
console.log( it.next() ); // { value:undefined, done:true }
```

因此，最后我们generator函数的结果是我们完成了这个函数，并且没有提供任何结果（因为我们已经耗尽了所有的`yield ___`表达式）。

你可能在这一点上想：我能不能在generator函数中使用`return`呢，如果我这样做，最后的结果会被设置在`value`属性上吗？

###是...

```js
function *foo() {
    yield 1;
    return 2;
}

var it = foo();

console.log( it.next() ); // { value:1, done:false }
console.log( it.next() ); // { value:2, done:true }
```

###...又不是

依赖来自于generator的`return`值不是一个好主意，因为当使用`for..of`来迭代generator函数（见下）时，最后的`return`值会被扔掉。

为了完整性考虑，我们也来看看早我们遍历一个generator函数时同时向内和向外发送消息。

```js
function *foo(x) {
    var y = 2 * (yield (x + 1));
    var z = yield (y / 3);
    return (x + y + z);
}

var it = foo( 5 );

// note: not sending anything into `next()` here
console.log( it.next() );       // { value:6, done:false }
console.log( it.next( 12 ) );   // { value:8, done:false }
console.log( it.next( 13 ) );   // { value:42, done:true }

```
你可以看到我们仍然可以通过最初的实例化调用`foo(5)`传递初始值，就像普通的函数那样。

第一个`next(..)`调用，我们不传递任何值，为什么？因为没有`yield`表达式接受我们传入的值。

但是如果我们的确向第一个`next(..)`调用传递了值，也不会有任何坏的事情发生，它只会被抛弃掉。对于这种情况，ES6描述了generator函数忽略未使用的值。（**注：**在写作这篇文章时，最新版本的Chrome和FF都表现如此，但是其他浏览器可能不完全兼容并可能不正确地对这种情况抛出一个异常）。

第二个`next(12)`调用将`12`传递给第一个`yield (x + 1)`表达式。第三个`next(13)`调用将`13`传递给第二个`yield (y / 3)`表达式。**反复地重新阅读这段代码**。对于很多人来说，在他们刚开始看这些代码的时候感觉奇怪。

##`for...of`

ES6同样在语法级别拥抱这种迭代器模式，通过直接提供对执行迭代器到结束的支持：`for..of`循环。

例：

```js
function *foo() {
    yield 1;
    yield 2;
    yield 3;
    yield 4;
    yield 5;
    return 6;
}

for (var v of foo()) {
    console.log( v );
}
// 1 2 3 4 5

console.log( v ); // still `5`, not `6` :(
```

如你所见，由`foo()`创建的迭代器被`for..of`循环自动地捕获，并且会为你自动地进行迭代，每轮迭代获取一个值直到得到一个`done:true`。只要`done`是`false`，它就会自动的提取`value`属性并分配给你的迭代变量中（在我们这里是`v`）。一旦`done`已经变为`true`，迭代循环停止（并且不对最后返回的`value`做任何处理，如果有的话）。

如之前所述，你可以看到`for..of`循环忽略并抛弃了`return 6`的值。同事由于没有暴露`next()`调用，`for..of`循环无法用于像我们之前那样向generator传入值的场合。

## 总结

好的，这就是generator的基础。如果你还有一些不明白的话，不必担心。我们每个人开始时都感觉像那样！

想一想这种新的玩具会如何改变你的代码是很自然的。尽管还有**很多**内容，我们刚刚触及了表面。所以在我们了解它们是多么强大之前我们必须要更深入地了解。

在你运行过上面的代码片段之后（试试最新的Chrome,FF或者node0.11+带上`--harmony`标志），你可能会产生以下的疑问：

1. 如何进行错误处理？
2. 一个generator可以调用另一个generator吗？
3. 异步代码如何与generator交互？

这些，或更多的问题，会在接下来的几篇文章中阐述，请继续关注！