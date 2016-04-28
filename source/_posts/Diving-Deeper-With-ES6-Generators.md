title: 'Diving Deeper With ES6 Generators'
date: 2014-12-24 22:29:28
tags:
---

#深入ES6Generators（译）

原文地址：[http://davidwalsh.name/es6-generators-dive](http://davidwalsh.name/es6-generators-dive)

ES6 Generators：全系列

1. [The Basics Of ES6 Generators](http://davidwalsh.name/es6-generators)
2. [Diving Deeper With ES6 Generators](http://davidwalsh.name/es6-generators-dive)
3. [Going Async With ES6 Generators](http://davidwalsh.name/async-generators)
4. [Getting Concurrent With ES6 Generators](http://davidwalsh.name/concurrent-generators)

如果你仍然不熟悉ES6 generator，首先去读一读并理解“[第一部分：ES6Generator基础](http://davidwalsh.name/es6-generators)”。一旦你认为你已经了解了这些基础，现在我们就可以深入到一些细节之中了。

## 错误处理

ES6 generator设计时一个最为强大的部分就是它内部的代码语义是同步的，即使外部迭代控制是异步执行的。

这是个很棒而且复杂的方式，意味着你可以使用简单的错误处理技术，这个技术你可能已经非常熟悉了 -- 也就是`try..catch`机制

例如：

```js
    try {
        var x = yield 3;
        console.log( "x: " + x ); // may never get here!
    }
    catch (err) {
        console.log( "Error: " + err );
    }
}
```

尽管函数会在`yield 3`表达式处停止，并且可能随意暂停一段时间，如果一个错误没回传到generator，那个`try..catch`会捕获它！尝试用普通的异步能力（例如callback）来做做看？：）

但是，怎样才能使一个错误返回到generator中呢？

```js
var it = foo();

var res = it.next(); // { value:3, done:false }

// instead of resuming normally with another `next(..)` call,
// let's throw a wrench (an error) into the gears:
it.throw( "Oops!" ); // Error: Oops!
```

这里，你可以看到我们使用了iterator上的另一个方法 -- `throw(..) ` -- 它会“抛”一个异常到generator中，就像它正好在generator当前`yield`暂停处发生一样。`try..catch`捕获如你所期望的那样这个错误！

注：如果你`throw(..)`一个错误进入generator，而没有`try..catch`住它，那么这个错误会（和普通错误一样）传播会来（如果最终没有被捕获，则会成为一个未被捕获的异常）。因此：

```js
function *foo() { }

var it = foo();
try {
    it.throw( "Oops!" );
}
catch (err) {
    console.log( "Error: " + err ); // Error: Oops!
}
```

显然，反方向的错误处理方式同样可行：

```js
function *foo() {
    var x = yield 3;
    var y = x.toUpperCase(); // could be a TypeError error!
    yield y;
}

var it = foo();

it.next(); // { value:3, done:false }

try {
    it.next( 42 ); // `42` won't have `toUpperCase()`
}
catch (err) {
    console.log( err ); // TypeError (from `toUpperCase()` call)
}
```

## generator代理

你可能想做的另一件事情是从你的generator函数内部调用另一个generator。我指的并不仅仅是普通方式实例化一个generator，而是代理你自己的的迭代控制到那一个另外的generator上。

例：

```js
function *foo() {
    yield 3;
    yield 4;
}

function *bar() {
    yield 1;
    yield 2;
    yield *foo(); // `yield *` delegates iteration control to `foo()`
    yield 5;
}

for (var v of bar()) {
    console.log( v );
}
// 1 2 3 4 5
```

如在第一部分解释的那样（我使用`function *foo(){ }`而不是`function* foo(){ }`），我这里同样使用`yield *foo()`来代替很多其他文章/文档使用的`yield* foo()`。我认为这样可以更加准确/清晰地说明代码正在做些什么。

让我们分解一下看看它是如何工作的。`yield 1`和`yield 2`直接发送他们的值到`for..of`循环外的`next()`的（隐式）调用中，就像我们已经理解/期望的那样。

但是接着程序就走到了`yield*`，然后你会注意到，我们正在通过实例化它（foo()）yield到另一个generator。因此我们基本上是在yield或代理到另一个generator的iterator上 -- 这可能是最准确的想象它的方式。

一旦`yield*`被（临时）从`*bar()`代理到`*foo()`，那么现在`for..of`循环的`next()`调用实际上是在控制`foo()`，因此`yield 3`和`yield 4`发送他们的值到外面的`for..of`循环中。

一旦`*foo()`结束，控制权会返回到最初的generator上，它最终调用到`yield 5`

为了简化起见，这个例子仅仅`yield`值出去，但是当然如果你不适用一个`for..of`循环，而是就手动调用iterator的`next(..)`并且通过消息传递，这些消息会通过同样的行为穿过`yield*`代理：

```js
function *foo() {
    var z = yield 3;
    var w = yield 4;
    console.log( "z: " + z + ", w: " + w );
}

function *bar() {
    var x = yield 1;
    var y = yield 2;
    yield *foo(); // `yield*` delegates iteration control to `foo()`
    var v = yield 5;
    console.log( "x: " + x + ", y: " + y + ", v: " + v );
}

var it = bar();

it.next();      // { value:1, done:false }
it.next( "X" ); // { value:2, done:false }
it.next( "Y" ); // { value:3, done:false }
it.next( "Z" ); // { value:4, done:false }
it.next( "W" ); // { value:5, done:false }
// z: Z, w: W

it.next( "V" ); // { value:undefined, done:true }
// x: X, y: Y, v: V
```

尽管我们在这里只显示了一层代理，而`*foo()`去`yield*`代理到另一个、另一个、另一个generator也是可以的。

```js
function *foo() {
    yield 2;
    yield 3;
    return "foo"; // return value back to `yield*` expression
}

function *bar() {
    yield 1;
    var v = yield *foo();
    console.log( "v: " + v );
    yield 4;
}

var it = bar();

it.next(); // { value:1, done:false }
it.next(); // { value:2, done:false }
it.next(); // { value:3, done:false }
it.next(); // "v: foo"   { value:4, done:false }
it.next(); // { value:undefined, done:true }
```

如你所见，`yield *foo()`代理迭代循环（这些`next()`调用）知道结束，然后一旦它这样做，任何从`foo()`中`return`的值（在本例里：字符串`"foo"`）会被设置为`yield*`表达式的结果值，然后被赋值给本地变量`v`。

`yield`和`yield*`之间有一个有趣的区别：对于`yiled`表达式，其结果是任何同事子序列`next(..)`传入的值，但是对于`yield*`表达式，它仅仅接收被代理的generator的`return`值（因为`next(..)`发送的值是对代理透明的）。

你也可以通过`yield*`代理从两个方向进行错误处理（如前所示）：

```js
function *foo() {
    try {
        yield 2;
    }
    catch (err) {
        console.log( "foo caught: " + err );
    }

    yield; // pause

    // now, throw another error
    throw "Oops!";
}

function *bar() {
    yield 1;
    try {
        yield *foo();
    }
    catch (err) {
        console.log( "bar caught: " + err );
    }
}

var it = bar();

it.next(); // { value:1, done:false }
it.next(); // { value:2, done:false }

it.throw( "Uh oh!" ); // will be caught inside `foo()`
// foo caught: Uh oh!

it.next(); // { value:undefined, done:true }  --> No error here!
// bar caught: Oops!
```

如你所见，`throw("Uh oh!")`通过`yield*`代理抛出错误到`*foo()`内部的`try..catch`中。同样的，`*foo()`内部的`throw "Oops!"`抛回到`*bar()`中，然后它在另一个`try..catch`中捕获这个错误，如果我们没有捕获其中的某一个，那么这些错误会继续向外传播像你平常期望的那样。

## 总结

Generator具有同步的语法，这表示你可以穿过`yield`表达式使用`try..catch`错误处理机制。generator同时也有一个`throw(..)`方法将一个错误抛入generator的暂停位置，这当然也可以被generator内部的`try..catch`捕获。

`yield*`允许你从当前的generator代理迭代控制到另一个generator。其结果是`yield*`表现得像一个连接两个方向的通道，同时可供传递消息以及错误。

但是，到现在位置，我们还有一个基本的问题没有解决：generator是如何帮助我们处理异步模式的呢？我们现在在这两篇文章中所见的一切都是generator函数的同步迭代。

其中的关键就是建立一种机制，generator暂停以开始一个异步任务，然后在异步任务结束时恢复（通过他的iterator的`next()`调用）。我们将在下一篇文章中探索各种关于使用generator创造这样的异步控制机制的方式。请继续关注！