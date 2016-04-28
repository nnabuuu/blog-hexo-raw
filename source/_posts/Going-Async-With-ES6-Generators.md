title: 'Going Async With ES6 Generators'
date: 2014-12-26 21:01:21
tags: generator async translation
---
#用Generator进行异步编程（译）

原文地址：[http://davidwalsh.name/async-generators](http://davidwalsh.name/async-generators)

ES6 Generators：全系列

1. [The Basics Of ES6 Generators](http://davidwalsh.name/es6-generators)
2. [Diving Deeper With ES6 Generators](http://davidwalsh.name/es6-generators-dive)
3. [Going Async With ES6 Generators](http://davidwalsh.name/async-generators)
4. [Getting Concurrent With ES6 Generators](http://davidwalsh.name/concurrent-generators)

现在你已经[见识过了ES6 generator](http://davidwalsh.name/es6-generators/)并且已经对它已经[有所熟悉](http://davidwalsh.name/es6-generators-dive/)了，现在是时候开始使用它们来增强我们真实的代码了。

Generator的主要唱出在于它们提供了一个单线程的，同步样式的代码风格，**同时允许你把异步隐藏为实现细节**。这使得我们用一种非常自然的方式表达，专注于我们程序的步骤/声明的流程，而不必同时不得不遵循异步语法并避免陷阱。

换句话说，我们通过隔离对值的消费（我们的generator逻辑）与异步得到这些值的细节（generator迭代器中的`next(..)`），实现了**能力与缺点的完美分离**。

结果呢？我们获得了异步代码的强大能力，同时也获得了（看上去是）同步代码的可读性以及可维护性。

那么我们如何实现这个非凡的能力呢？

## 最简单的异步

在最简单的场景下，generator不需要任何额外的操作来实现你的程序中并没有的异步操作。

例如，让我们设想你已经有了这样的代码：

```js
function makeAjaxCall(url,cb) {
    // do some ajax fun
    // call `cb(result)` when complete
}

makeAjaxCall( "http://some.url.1", function(result1){
    var data = JSON.parse( result1 );

    makeAjaxCall( "http://some.url.2/?id=" + data.id, function(result2){
        var resp = JSON.parse( result2 );
        console.log( "The value you asked for: " + resp.value );
    });
} );
```

要使用一个generator来表现同样的程序，你需要这样做：

```js
function request(url) {
    // this is where we're hiding the asynchronicity,
    // away from the main code of our generator
    // `it.next(..)` is the generator's iterator-resume
    // call
    makeAjaxCall( url, function(response){
        it.next( response );
    } );
    // Note: nothing returned here!
}

function *main() {
    var result1 = yield request( "http://some.url.1" );
    var data = JSON.parse( result1 );

    var result2 = yield request( "http://some.url.2?id=" + data.id );
    var resp = JSON.parse( result2 );
    console.log( "The value you asked for: " + resp.value );
}

var it = main();
it.next(); // get it all started
```

让我们来解释一下它是如何工作的：

`request(..)`功能函数基本上包装我们普通的`makeAjaxCall(..)`功能类以保证它的回调函数能调用generator iterator的`next(..)`方法。

对于`request("..")`调用，你会注意到它*没有返回值*（换句话说，它是`undefined`）。这不是什么大问题，但是它和我们在本文之后的实现方式有所不同：我们在这里实际上是调用了`yield undefined`。

因此我们调用`yield ..`（和这个`undefined`值），它实际上什么也没做，它只是在这一点上暂停了我们的generator。它将会等待直到`it.next(..)`被调用来恢复它，这个调用我们已经排列在队列中（作为回调函数），在Ajax调用结束后发生。

但是`yield .. `表达式的*结果*又发生了什么？我们将它赋值到变量`result1`上。它是如何得到第一个Ajax调用的内部的值的呢？

因为当`it.next(..)`被作为Ajax回调函数调用时，它实际是在给它传递Ajax的响应结果，这表明值在那个当前暂停的时间点被发送回我们的generator内部，也就是`result1 = yield ..`表达式的中间！

这的确非常的酷并且超级强大。本质上，`result1 = yield reequest(..)`是在**请求这个值**，但是它（几乎！）完全对我们隐藏了 -- 至少我们不需要在这里担心它 -- 外表之下的实际实现是异步的。它通过*隐藏*`yield`中的暂停能力实现了异步，并且分离出generator的*恢复*能力到另外一个函数中，因此我们的main代码只需要进行一个**（看起来是）同步的值的请求**。

对于第二个`result2 = yield result(..)`表达式也是一样：它对于暂停和恢复是透明的，并且提供了我们所需求的值，所有这些都没有让任何异步细节打扰到我们我代码。

当然`yield`出现了，因此那里的确有一个细微的提示“一些神奇的东西（异步）*可能发生*在那个时间点”。但是`yield`比起回调地狱（或者甚至是promise链的API冗余！）来已经是一个简单的语法信号/冗余了。

注意到我刚刚说了*可能发生*。这是一个相当强大的事情。上面的程序总是发出一个Ajax请求，但是**如果它不这样呢**？如果我们之后将我们的程序改为读取内存中之前得到的Ajax响应呢？或者一些程序中的复杂URL rouer可能在某些条件下立即响应一个Ajax请求而不需要真的从一个外部服务器获取呢？

我们可以改变`request(..)`的实现使它变成这样：

```js
var cache = {};

function request(url) {
    if (cache[url]) {
        // "defer" cached response long enough for current
        // execution thread to complete
        setTimeout( function(){
            it.next( cache[url] );
        }, 0 );
    }
    else {
        makeAjaxCall( url, function(resp){
            cache[url] = resp;
            it.next( resp );
        } );
    }
}
```

注意：这里有一个小技巧是需要使用`setTimeout(..0)`进行延迟以防cache已经在结果里面了。如果我们刚刚立即调用`it.next(..)`，它会产生一个错误，因为（这就是那个技巧）generator*尚未*处于暂停状态。我们的函数调用`request(..)`*首先*被评估，然后`yield`暂停。因此我们不能再次在`request(..)`内部调用`it.next(..)`，因为在那个时刻generator扔在执行（`yield`还没有被进行）。但是我们可以”之后“调用`it.next(..)`，在当前线程执行完的一瞬间，也就是我们的`setTimeout(..0)`”伪造“的一个实现。**我们会在下面有一个更好的实现**。

现在我们的main generator代码仍然看起来像：

```js
var result1 = yield request( "http://some.url.1" );
var data = JSON.parse( result1 );
..
```

看到了吧？！我们的generator逻辑（也就是控制流）和不加cache的版本比起来**完全**不需要变化。

`*main()`中的代码仍然请求一个值，然后暂停直到它得到值。在我们当前的情境下，”暂停“可以非常长（发送一个真实的请求到服务器，一般为300-800ms）或者可能几乎立即结束（`setTimeout(..0)`进行延迟处理）。而我们的控制流并不关心。

这就是**将异步行为抽象为实现细节**真正的强大之处。

## 更好的异步

对于一个单独的异步generator工作，上面的实现已经相当不错了。但是它马上会到达局限，所以我们需要一个更强大的异步机制来和我们的generator做搭配，它能够承担更多的负担。这个机制是什么呢？就是**Promise**。

如果你对于ES6的Promise还有点模糊不清，我写了一个[5篇文章的系列](http://blog.getify.com/promises-part-1/)，去读一读吧。我会在这里wait直到你回来的（偷笑，哈哈）。这只是个老掉牙的异步的笑话啦！

本文早先的Ajax代码都有同样的[控制反转](http://blog.getify.com/promises-part-2/)的问题（也就是”回调地狱“），就下你给我们最初的那个充满了回调的例子一样。到目前为止，我们缺乏这样一些东西：

1. 没有明确的异常处理的方式。我们已经[从上篇文章中学到](http://davidwalsh.name/es6-generators-dive/#error-handling)，我们可以探测到一个Ajax调用时的异常（通过某种方式），通过`it.throw(..)`传递回我们的generator，然后使用`try..catch`在我们的generator逻辑中处理它。但是那只是更多的手动任务来接通“后端”（我们处理generator iterator的代码），并且如果我们需要非常多的generator是，它可能无法重复使用。

2. 如果`makeAjaxCall(..)`工具类不受控制，并且它调用了多次的callback，或者信号同时成功与失败，等等。那么我们的generator会出故障（未捕获的异常，不期待的值，等等）。处理并且阻止这些问题很多都是手动工作，并且同样无法重用。

3. 经常的，我们并不仅仅”并发“执行任务（例如两个并行的Ajax调用那样）。由于generator`yield`表达式是一个单一暂停点，两个或两个以上的generator不可以在同时运行 -- 它们不得不一次一个的执行，按顺序。因此，对于如何在单独的generator `yield`点发送多个任务，而不在表面之下进行大量的人工编码，是尚不可知的。

如你所见，所有这些问题都是*可以被解决的*，但是谁又希望每次都重新发明这些解决方法呢？我们需要一个更强大的模式，设计为专为基于generator的异步编码的[可信的，可重用的解决方案](http://blog.getify.com/promises-part-3/)。

那个模式是？**`yield` out promises**，并且当他们被fulfill时让它们恢复generator。

回想一下上面我们所做的`yield request(..)`，以及`request(..)`功能方法没有任何返回值，仅仅`yield undefined`是有效的吗？

让我们小小的对他进行调整。让我们改变我们的`request(..)`功能方法使其成为一个基于promise的方法，这样它会返回一个promise，并且这样的话我们`yield` out的东西**实际上是一个promise**（而不是`undefined`）。

```js
function request(url) {
    // Note: returning a promise now!
    return new Promise( function(resolve,reject){
        makeAjaxCall( url, resolve );
    } );
}
```

现在，`request(..)`会构造一个Ajax调用结束后被处理的promise，并返回这个promise，所以它可以被`yield`出去，下一步呢？

我们会需要一个功能方法来控制我们的generator iterator，它接收这些被`yield`的promise然后将他们与恢复generator联通(通过`next(..)`)。我现在会调用下面这个`runGenerator(..)`功能类：

```js
// run (async) a generator to completion
// Note: simplified approach: no error handling here
function runGenerator(g) {
    var it = g(), ret;

    // asynchronously iterate over generator
    (function iterate(val){
        ret = it.next( val );

        if (!ret.done) {
            // poor man's "is it a promise?" test
            if ("then" in ret.value) {
                // wait on the promise
                ret.value.then( iterate );
            }
            // immediate value: just send right back in
            else {
                // avoid synchronous recursion
                setTimeout( function(){
                    iterate( ret.value );
                }, 0 );
            }
        }
    })();
}
```

值得注意的关键点：

1. 我们自动的对generator进行初始化（创建它的`it`迭代器），然后我们异步地运行`it`直到结束（`done: ture`）。

2. 我们寻找要被`yield`出去（即在每个`it.next(..)`调用时的返回值`value`）的promise。如果有的话，我们通过在promise之上注册`then(..)`等待直到它结束。

3. 如果任何立即的（即非promise）值被返回，我们简单地发送这个值到generator中以便它继续立即执行。

现在，我们怎么使用它呢？

```js
runGenerator( function *main(){
    var result1 = yield request( "http://some.url.1" );
    var data = JSON.parse( result1 );

    var result2 = yield request( "http://some.url.2?id=" + data.id );
    var resp = JSON.parse( result2 );
    console.log( "The value you asked for: " + resp.value );
} );
```

Bam！等一等...这不**和我们之前的generator代码一样**吗？是的。再一次地，generator的强大之处显示出来了。实际上是，我们现在在创建promise，将他们`yield`出去，然后在generator结束时恢复他们 -- **所有这些都隐藏了实现细节！** 当然并不是完全隐藏，只是从消费代码（我们generator内部的控制流）中分离出来了。

通过等待被`yield`出去的promise，并且发送完成结果回`it.next(..)`，代码`result1 = yield request()..`得到了和之前完全相同的值。

但是现在我们在使用promise来管理generator代码中的异步部分，我们解决所有的来自于回调风格解决方案的倒转/信任问题。我们通过使用generator + promise得到所有上面的解决方案。

1. 我们现在有了便于使用的内嵌的异常处理。我们在上面的`runGenerator(..)`中并没有显示它，但是从promise中监听一个异常并发送至`it.throw(..)`并不困难 -- ranh9ou我们可以在我们的generator代码中使用`try..catch`来捕获并处理这些异常。

2. 我们拥有了所有由promise提供的[控制/可信性解决方案](http://blog.getify.com/promises-part-2/#uninversion)。不需要更多的关心。

3. Promise拥有非常多位于上层的强大的抽象，它可以自动地处理复杂的多“并发”任务，等等。

例如`yield Promise.all([ .. ])`可以接受一个prmose的数组来“并发执行”任务，然后`yield`出一个单一的promise（给generator来处理），它在处理前等待所有的子promise结束（无论以何种顺序）。你从`yield`表达式返回的（当promise结束时）是一个所有子promise的响应数组，按照它们请求的顺序（无论它们的结束顺序是如何）。

首先让我们来看看异常处理：

```js
// assume: `makeAjaxCall(..)` now expects an "error-first style" callback (omitted for brevity)
// assume: `runGenerator(..)` now also handles error handling (omitted for brevity)

function request(url) {
    return new Promise( function(resolve,reject){
        // pass an error-first style callback
        makeAjaxCall( url, function(err,text){
            if (err) reject( err );
            else resolve( text );
        } );
    } );
}

runGenerator( function *main(){
    try {
        var result1 = yield request( "http://some.url.1" );
    }
    catch (err) {
        console.log( "Error: " + err );
        return;
    }
    var data = JSON.parse( result1 );

    try {
        var result2 = yield request( "http://some.url.2?id=" + data.id );
    } catch (err) {
        console.log( "Error: " + err );
        return;
    }
    var resp = JSON.parse( result2 );
    console.log( "The value you asked for: " + resp.value );
} );
```

当获取URL时promise被拒绝（或者任何形式的错误/异常），promise rejection会被映射为一个generator错误（使用我们之前没有描述的`runGenerator(..)`中的`it.throw(..)`），它会被`try..catch`语句捕获住。

现在，让我们来看一个更下复杂的例子，它使用了promise来管理更多异步的复杂问题：

```js
function request(url) {
    return new Promise( function(resolve,reject){
        makeAjaxCall( url, resolve );
    } )
    // do some post-processing on the returned text
    .then( function(text){
        // did we just get a (redirect) URL back?
        if (/^https?:\/\/.+/.test( text )) {
            // make another sub-request to the new URL
            return request( text );
        }
        // otherwise, assume text is what we expected to get back
        else {
            return text;
        }
    } );
}

runGenerator( function *main(){
    var search_terms = yield Promise.all( [
        request( "http://some.url.1" ),
        request( "http://some.url.2" ),
        request( "http://some.url.3" )
    ] );

    var search_results = yield request(
        "http://some.url.4?search=" + search_terms.join( "+" )
    );
    var resp = JSON.parse( search_results );

    console.log( "Search results: " + resp.value );
} );
```

`Promise.all([ .. ])`构造一个promise，它等待3个子promise。并且，被`yield`出提供给`runGenerator(..)`功能函数的主promise会被监听作为generator的恢复。子promise可以接收一个响应，它看起来像另一个URL，并且以链式连接另一个子promise到达新的地点。如果要学习更多promise链式表达，[阅读这篇文章](http://blog.getify.com/promises-part-5/#the-chains-that-bind-us)。

任何异步的功能性/复杂性问题都可以由promise解决，而同步风格代码则可以通过使用generator `yield`出promise（的promise的promise...）来实现。**这真是两全其美**

## runGenerator(..): 功能库

我们已经定义了我们自己的`runGenerator(..)`来启用这个强大的generator+promise组合。我们省略了这个功能函数的完全实现（为了简单起见），因为还有很多细节上和异常处理相关的内容需要完成。

但是，你并不想编写你自己的`runGenerator(..)`是吧？

我认为是的。

有非常多的promise/异步库提供了这样的功能。我在这里不会讲述，但你可以看一看`Q.spawn(..)`，`co(..)`库，等等。

我会简单的介绍一下我自己的功能库：[asynquence](http://github.com/getify/asynquence)的[`runner(..)`插件](https://github.com/getify/asynquence/tree/master/contrib#runner-plugin)，我认为它比上面的那些库提供了一些特殊的适配性。我写了深入的[两部分的blog关于asynquence的系列文章](http://davidwalsh.name/asynquence-part-1/)如果你感兴趣学到更多的话你可以去看一看。

首先，*asynquence*提供了功能类自动处理“首参数为错误风格”的回调：

```js
function request(url) {
    return ASQ( function(done){
        // pass an error-first style callback
        makeAjaxCall( url, done.errfcb );
    } );
}
```

这**更加的友好**了，不是吗！？

下一步，asynquence的`runner(..)`插件在*aynquence*序列（异步序列步骤）的中途消耗一个generator，所以你可以从之前的步骤向内传递消息，而你的generator可以向外或向下一步传递消息，而所有的错误会自动地如你期望的那样传播。

```js
// first call `getSomeValues()` which produces a sequence/promise,
// then chain off that sequence for more async steps
getSomeValues()

// now use a generator to process the retrieved values
.runner( function*(token){
    // token.messages will be prefilled with any messages
    // from the previous step
    var value1 = token.messages[0];
    var value2 = token.messages[1];
    var value3 = token.messages[2];

    // make all 3 Ajax requests in parallel, wait for
    // all of them to finish (in whatever order)
    // Note: `ASQ().all(..)` is like `Promise.all(..)`
    var msgs = yield ASQ().all(
        request( "http://some.url.1?v=" + value1 ),
        request( "http://some.url.2?v=" + value2 ),
        request( "http://some.url.3?v=" + value3 )
    );

    // send this message onto the next step
    yield (msgs[0] + msgs[1] + msgs[2]);
} )

// now, send the final result of previous generator
// off to another request
.seq( function(msg){
    return request( "http://some.url.4?msg=" + msg );
} )

// now we're finally all done!
.val( function(result){
    console.log( result ); // success, all done!
} )

// or, we had some error!
.or( function(err) {
    console.log( "Error: " + err );
} );
```

asynquence `runner(..)`功能类接受一个可选的消息来开始generator，这个消息往往是由之前的步骤而来，并且在generator的`token.messages`数组中是可见的。

然后，和我们之前示范使用`runGenerator(..)`功能类一样，`runner(..)`监听一个被`yield`的poromise或*asynquence*序列（在这种情况下使用`ASQ().all(..)`序列来并发执行），并且等待它的结束然后恢复generator。

当generator结束时，最后`yield`出的值会传递给序列的下一个步骤。

并且，如果有任何错误在这个序列的任何地方发生，甚至是在generator内部发生，它会被传播给单独的`or(..)`被注册的错误处理者。

*asynquence*尝试将promise和generator尽可能简单的结合。你可以随心所欲的构造任何generator流和基于promise的序列步骤流。

## ES7 `async`

ES7的时间轴上有一个提案，它看起来会被接受，来创建另一种函数：`async function`，它看起来是使用generator自动地包装一个像`runGenerator(..)`（或*asynquence*的`runner(..)`）功能类。那样的话你可以发送promise并且`async function`会自动地将其包装并且在结束时恢复promise（甚至不需要使用iterator！）

所以它看起来可能会像这样：

```js
async function main() {
    var result1 = await request( "http://some.url.1" );
    var data = JSON.parse( result1 );

    var result2 = await request( "http://some.url.2?id=" + data.id );
    var resp = JSON.parse( result2 );
    console.log( "The value you asked for: " + resp.value );
}


main();
```

如你所见，一个`async function`可以被直接调用（就像`main()`一样），而不需要像`runGenerator(..)`或`ASQ().runner(..)`的包装功能类来包装它。在内部，有别于使用`yield`，你将会使用`await`（另一个新的关键词）来告知`async function`在继续执行前等待promise的结束。

基本上，我们会拥有大多数包装库包装后的generator的能力，但是**直接由原生语法支持**

酷！是吧！

在同时，像*asynquence*这样的库给予我们这些执行功能函数来让我们使用异步generator更容易！


## 总结

简单的说：generator + `yield`ed promise组合了双方最好的部分让我们得到了强大而优雅的同步语法+异步流程控制的能力。使用简单的包装功能函数（有非常多的功能库已经提供了这一点），我们可以自动的运行我们的generator到结束，包括正常结果以及出错的处理。

在ES7的大陆上，我们很可能会见到`async function`让我们可以不依靠功能库来达到（至少对基本的case可以这样实现）。

**JavaScript中异步的未来是光明的**，并且只会变得更光明！我应该戴上太阳镜。

但是我们还没有结束，我们还有最后一个部分想要发掘一下：

如果你可以将两个或多个generator连接在一起会怎么样呢？让他们单独但“并发”的执行，并且让他们在执行的过程中互相发送消息？那会是一种更强大的能力，不是吗？这个模式被称为"CSP"（communicating sequential processes）。我们会在下一篇文章中解锁CSP的强大能力。请继续关注！