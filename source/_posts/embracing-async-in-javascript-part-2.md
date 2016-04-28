title: 拥抱JavaScript中的异步2（译）
date: 2014-11-14 11:01:29
tags: translation
---
#拥抱JavaScript中的异步2（译）
[Andy White](https://twitter.com/whitehouse3001)

##本系列的上一期
* [拥抱JavaScript中的异步1](http://io.pellucid.com/blog/embracing-async-in-javascript-part-1) 
* 译文：[http://nnabuuu.github.io/blog-hexo/2014/10/07/embracing-async-in-javascript-part-1/](http://nnabuuu.github.io/blog-hexo/2014/10/07/embracing-async-in-javascript-part-1/)

##简介

在之前的文章中（第一部分），我简要的讨论了一些JavaScript事件循环的基础，函数调用栈，闭包，以及一些基本的回调模式，这些内容都与异步编程相关。在本文中，我想要继续讨论更多JavaScript异步的异步话题。

首先，我想要回应[来自Redditor的对我之前一篇文章的评论](http://www.reddit.com/r/javascript/comments/2hzu7c/embracing_async_in_javascript_part_1/ckxjauu)，该评论拒绝“整个应用应该被构建成为一个异步运行的系统”的思想。这个评论很棒，并且我的确赞同。在我前面的文章中，我并没有在暗示你必须在普通的回调或者其他低级语言特性上构建整个应用来处理你代码中的异步API。但即使你不这样做你也会很快的在其他地方遇到异步代码，而你需要理解并且拥抱其工作方式，这样才能更好的在JavaScript上取得成功。如何拥抱异步代码完全取决于你（以及你的目标平台的支持），有非常多的资源，库，或其他内容可以来帮助你。编写异步代码比起编写同步代码需要更小心以及更多的语言/库的支持，一旦你开始在你的代码中引入异步模式，其异步性往往会不断扩张并且需要越来越多的代码以便维护其一致性以及正确的行为。JavaScript在其核心中并没有对异步代码有太多语言层面的支持，而这个现状正在由新的语言特性改善，例如[原生的promise](http://www.html5rocks.com/en/tutorials/es6/promises/)，[ES6生成器](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*)，[Node.js中的libers](https://github.com/laverdet/node-fibers)或类似的库，还有数以百计的已有的异步模块以及库在类似于[npm](https://www.npmjs.org/search?q=async)的仓库中。

但是，在本文中，我仍然想要定位于低层次，并且谈谈更加底层的JavaScript中的异步代码模式：events和promises。

##Events

在JavaScript中的事件是一种用来在JavaScript的对象之间进行通信的公用订阅机制。事件和回调非常相似：事件的发布者为感兴趣的对象提供一种订阅方式用来在事件发生时接收通知。订阅一个事件代表注册一个回调函数，当事件发生时回调它。当事件发生时，事件发布者简单的调用其注册的任何回调函数。和回调一样，事件可以同步或异步地发生，事件监听回调也可以被同步或异步地调用。

JavaScript原生地将事件用在例如DOM事件的场合，例如点击、鼠标移动、表格提交，等等。即使在非浏览器环境下的JavScript中，事件也被广泛地使用：例如Node.js的EventEmitter。在Node.js中，事件也在stream中出现。

使用事件的主要好处在于他们可以被多个监听器所消费。当事件发生时，事件的发布者可以调用多个被注册的回调函数，因此多个对象可以被通知到。它也可以在某块之间创造松耦合，因为发布者不应该关心“什么”或者“多少”消费者订阅了自己，并且消费者不需要知道发布者内部在做些什么。

大多数的JavaScript框架（浏览器端或非浏览器端）都支持一些事件方式，包括jQuery、AngularJS、Backbone、React、Ember，以及之前提到的Node.js，包含各种各样的`EventEmitter`以及`stream`。

下面是一个简单的使用基于事件的API的示例。这个例子是用Node.js实现的，使用基本的EventEmitter模块。

	// Get the constructor function for the Node.js EventEmitter
	var EventEmitter = require("events").EventEmitter;
	
	// Clock is our event publisher - when started, it will publish a "tick" event 
	// every second.
	function Clock() {
	    this.emitter = new EventEmitter();
	}
	
	// Starts the clock ticking
	Clock.prototype.start = function() {
	    var self = this;
	    this.interval = setInterval(function() {
	        self.emitter.emit("tick", new Date());
	    }, 1000);
	};
	
	// Stops the clock from ticking
	Clock.prototype.stop = function() {
	    if (this.interval) {
	        clearInterval(this.interval);
	        this.interval = null;
	    }
	};
	
	// Register a callback for the "tick" event
	Clock.prototype.onTick = function(callback) {
	    this.emitter.on("tick", callback);
	};
	
	// Create our clock
	var clock = new Clock();
	
	// Register an event for the clock's tick event
	clock.onTick(function(date) {
	    console.log(date);
	});
	
	// Start the clock
	clock.start();
	
这个基础的Node.js程序输出类似如下：

	% node clock.js
	Wed Oct 15 2014 14:08:01 GMT-0600 (MDT)
	Wed Oct 15 2014 14:08:03 GMT-0600 (MDT)
	Wed Oct 15 2014 14:08:04 GMT-0600 (MDT)
	Wed Oct 15 2014 14:08:05 GMT-0600 (MDT)
	Wed Oct 15 2014 14:08:06 GMT-0600 (MDT)
	Wed Oct 15 2014 14:08:07 GMT-0600 (MDT)
	Wed Oct 15 2014 14:08:08 GMT-0600 (MDT)
	...repeats forever...
	
这里，Clock的`onTick`函数允许任意数量的对象注册回调到每一次的时间点上。在示例中，我们只注册了一个订阅者，而实际上我们可以注册更多。

事件是一种又用的同步或异步通信机制，但是他们本身并不有助于解决异步调用的顺寻问题，你可以使用其他的技术来帮助你，例如回调。

## Promises

Promise是另外一种处理JavaScript对象间异步通信的机制。在过去几年中，Promise已经在JavaScript中变得非常流行了，并且现在已经有许多Promise的实现可供挑选，包括即将到来的ECMAScript6的原生Promise实现。

当异步任务完成或失败时通知其他模块方面，Promise与回调十分相似，但是实现的方式与回调以及事件有一些不同。在回调中，一个异步API函数接受一个或多个函数入参，当任务结束或失败时API函数会使用它们，然而一个基于Promise的函数并不接受回调作为参数，而是返回一个其他模块可以注册完成或者失败回调的`Promise`对象。而且，另一个回调与Promise的巨大不同之处在于，Promise对象会在满足条件之后继续持有返回值或错误对象，因此其他模块可以检验Promise的状态，访问其对象，即使Promise已经完成。使用回调以及事件时，回调的调用者以及事件的发布者都不会持有最后一次的值，因此如果一个感兴趣的模块错过了一个事件，它们可能就无法检测到这个事件已经发生，也无法得知随该事件一起被发出的值是什么了。

当我们谈到Promise时，我们引入了一个较为具体的术语，也就是[Promise/A+规范](https://promisesaplus.com/)的描述。当然也有一些其他的Promise规范，但Promise/A+规范似乎是最流行的。网络上有非常多的Promise教学，因此我不会在这里具体讲述，而我的确想提供一个简单的示例来演示Promise是如何被用在顺序的异步函数调用上的。我将使用非常流行的、功能强大的库[Q](https://github.com/kriskowal/q)来进行演示。

这是一个非常“刻意”的例子，但它演示了顺序的异步调用如何能和Promise一起使用。

	function begin() {
	    console.log("begin");
	    return 0;
	}
	
	function end() {
	    console.log("end");
	}
	
	function incrementAsync(i) {
	    var defer = Q.defer();
	
	    setTimeout(function() {
	        i++;
	        console.log(i);
	        defer.resolve(i);
	    }, 0);
	
	    return defer.promise;
	}
	
	Q.fcall(begin)
	    .then(incrementAsync)
	    .then(incrementAsync)
	    .then(incrementAsync)
	    .then(end);
	    
这个例子的输出是：

	begin
	1
	2
	3
	end
	
这个例子的主要驱动方式是Q promise链，由`Q.fcall`以begin为参数开始。`Q.fcall`是一个Q提供的静态方法，用来执行所提供的函数，并返回一个值的Promise。入参函数可以返回一个Promise值也可以返回一个非Promise值，但无论哪种方式，Q将会从`Q.fcall`返回一个Promise。由于`Q.fcall`总是返回一个Promise，你可以使用`then`方法在一个Promise上链接其他函数，`then`函数是Promise的基础方法。返回一个Promise的函数通常被成为"thenable"的函数，意味着你可以使用`.then()`在它之上链接回调函数。

上面的第一个`.then`将`incrementAsync`函数链接到由`Q.fcall(begin)`创造的Promise中。`incrementAsync`函数接受一个数字类型的参数，设置一个超时机制来异步地增加值，然后返回一个增加完结果的值的Promise。`incrementAsync`函数创造了一个Q的`deferred`对象（使用`Q.defer()`），这个对象是Promise的“创造者”进行操作的。Promise的创造者有义务在某一个时间点满足或者拒绝这个Promise，典型的时间点就是异步调用成功或者失败的时刻。在Q里，它是通过在deferred对象上调用`.resolve()`或`reject()`实现的。在`incrementAsync`中，Promise是通过增加i来满足的，然后调用了`.resolve(i)`，也就表示这个Promise被满足了，并且提供了一个值来传递到链接的下一个函数中。传递给`.resolve()`的值被传递到链接的下一个函数中作为函数的第一个参数。在Q的Promise链中，每一个方法都可以为一个值的Promise或者朴素的值，Q会基于成功满足或拒绝的条件顺序执行执行该链。Promise不需要被一个值满足，它可以不使用任何值，仅仅表示异步调用已经成功，没有任何值来提供。

Promise/A+规范要求Promise总是被异步地处理，因此上面例子中的`setTimeout`实际上是多余的，我们用它只是原来强调`incrementAsync`是天然异步的。

Promise是有一点复杂的话题，很难在一篇文章中讲清楚，但有数不胜数的资源用以将来的学习。

## 未来

JavaScript作为一种语言以及生态系统，正在迅速地发展。只有非常多激动人心的语言特性正被开发出来支持异步代码。其中最令人激动的是[ES6 generator](http://davidwalsh.name/es6-generators)，它是一种非常的强大的、JavaScript编程的新方式。我在此不会讲述这个话题，但网络上有非常多好的教程和指南。

## 结论

异步编程是JavaScript中需要理解的重要内容，并且有非常多的方式来拥抱它。对于如何处理异步代码并没有一种定论，但是理解不同的可选项是非常重要的，这样你就可以根据你的需求选取正确的解决方案。