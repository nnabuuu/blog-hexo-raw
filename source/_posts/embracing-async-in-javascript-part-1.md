title: 拥抱JavaScript中的异步（译）
date: 2014-10-07 20:47:48
tags: translation
---
#拥抱JavaScript中的异步（译）

[Andy White](https://twitter.com/whitehouse3001)

##简介

如果你已经写过不计其数的Javascript代码，那么你会意识到，异步编程并不仅仅是一种"nice to have"的能力，而是一种必需品。为了充分利用语言与生态系统，它必须被理解和接受。

##回调

在JavaScript中，最简单的展示异步API的方式之一便是使用一个接受另一个函数作为参数的函数。这些函数参数就是所谓的“回调”或“回调函数”，因为他们给予主函数一个用来回调的钩子 -- 通过在恰当的时机，调用一次或多次你所提供的函数。因为JavaScript将函数视为[一等公民](http://en.wikipedia.org/wiki/First-class_citizen)，你可以将函数实例作为参数传递给其他函数，或者从其他函数中返回函数实例，就像你可以轻松的传递或返回数值，字符串，布尔类型，对象或者数组一样。会点函数可以被主函数在任何时机，以任何的次数，同步或异步的方式，使用任何绑定的上下文以及参数所调用，这样就提供了一种非常灵活以及强大的在JavaScript模块之间进行通信的机制。

## JavaScript的事件循环与回调

在深入了解回调以及异步代码之前，对JavaScript的事件循环与回调有一个基本的认知是非常重要的。在最基本的形式中，JavaScript是一个单线程的运行时，因此它并不支持类似于多线程、多进程或内部进程通信的技术。尽管看起来（实际也是）有这些限制，缺少多线程实际上会让你作为一个javaScript开发者的认识更简单一些，并且允许几种有趣的在编译、压缩、清理（transpilation）的优化技术。

因为只有单线程，你将永远不会遇到基本的来自线程的挑战，例如竞争条件，资源冲突，线程死锁等。当JavaScript解释器开始执行一个代码块时（例如，一次函数调用），它将一直执行这块代码直到同步代码结束（该函数中所包含的最后一句同步代码）。在执行这段同步代码的期间，任何非同步的函数调用（例如外部事件句柄调用，异步函数调用，异步回调等等）将简单的被放置于队列中等待运行时事件循环之后执行。一旦同步代码执行完，事件循环将从队列中获取下一块代码并执行它，知道同步代码的结束，以此类推。这样，你可以安全的断言，你在一个函数中编写的任何一串的同步代码，在其他代码执行之前，将总是不被打断地执行到结束。你也无法使用CPU直到你yield它（或者运行时干掉了你的堵塞或者死循环代码）。这对于应用开发非常有帮助，但也需要一些仔细的考虑与计划，尤其是当你需要用到异步API时，你代码的顺序就非常重要了。

## 是"下一个tick"还是"事件循环的下一个回合"

你经常听到JavaScript开发者提到"下一个tick"还是"事件循环的下一个回合"。基本上，这些概念意味着，当当前同步代码执行完毕后，这些代码将被置于等待执行的队列中，时间循环正准备从队列中获取下一块要执行的代码。所有的异步API都暗示代码将会在之后的的"tick"或者“时间循环的回合”被执行。下一个tick的概念可能更具体的取决于你的JavaScript平台，但是基本上，它仅仅指的是一个函数调用已经被置于队列中为了将来执行。在这个条件下，"之后"这个词可能是也可能不是指向一个确定的时间延迟，但它总是表示代码会在当前同步代码执行完之后被执行。

## 非堵塞操作

由于JavaScript的单线程特性，因此时间敏感的操作，例如IO操作，必须全部都是非堵塞且异步的，这样这些操作就不会堵塞主应用的时间循环。当事件循环被堵塞时，没有其他应用逻辑会被执行，应用程序往往陷于完全停止的情景。长时间的操作全部都应该被异步调用，并且在操作结束（或者失败）时使用一些异步完成的回调进行处理。对于长时间的操作，使用进度回调函数也非常常见，这样进度的增长可以被报告出来（例如大文件的拷贝操作时的百分比通知）。所有这些回调都是简单被加入到事件队列中，并且在事件循环的未来某一个回合被执行，这样就不会有任何代码在任何时刻堵塞事件循环。

## 同步vs异步回调API

对于回调，一个重要的方面是，它可以是同步的，也可以是异步的。通常而言，理解回调将被同步还是异步调用是非常重要的，因为你可能会有一些依次执行的代码，它们不应该被执行直到这些回调函数基于的API调用结束。对于同步的回调API，你基本上不需要做任何事情来实现所期望的顺序，因为回调函数将同步地执行到结束，而对于异步回调API，你往往必须用另一种形式编写代码以确保调用顺序的正确性。

## 基于同步的"each"函数

一个基于"each"函数的同步回调可以很好的阐明这个问题。注意：我现在忽略了回调上下文(`this`)以及`Function.prototype.call`和`apply`。

	// Helper function for logging something to the console
	function logItem(item) {
	    console.log(item);
	}
	
	// Synchronous "each" function - invokes the callback for each item
	// in the array
	function each(arr, callback) {
	    for (var i = 0; i < arr.length; ++i) {
	        // Invoke the callback synchronously for each iteration
	        callback(arr[i]);
	    }
	}
	
	// Try it out!
	console.log("begin");
	each([1, 2, 3], logItem); // "logItem" is our "callback" function here
	console.log("end");

这个示例会在控制台打印：

	begin
	1
	2
	3
	end
	
除了`for`循环以及`each`函数会被同步执行直到到达`console.log("end")`语句，这个示例没有其他特殊的。

## 基于异步的"each"函数 （第一次尝试）

一个异步版本的"each"可能看起来像这样。在这里，我使用setTimeout来强制使回调的执行变得异步。主要setTimeout只是一个帮助函数用来推迟一个函数的调用 -- 从当前同步代码执行结束后算起（也就是，下一个事件循环的回合）。setTimeout也接受一个最小时间延迟的参数，但是在这里，我只是使用延迟0毫秒，仅仅让它异步执行，而不产生任何延迟。

	function asyncEach(arr, callback) {
	    for (var i = 0; i < arr.length; ++i) {
	        // Enqueue a function to be called later
	        // Note: this code does not do what we might expect...
	        setTimeout(function() {
	            callback(arr[i]);
	        }, 0);
	    }
	}
	
	console.log("begin");
	asyncEach([1, 2, 3], logItem);
	console.log("end");
	
由于回调函数现在异步执行，你应该期待代码执行结果为：

	begin
	end
	1
	2
	3
	
但是，令人吃惊（或者，其实并不吃惊）的是，代码结果为：

	begin
	end
	undefined
	undefined
	undefined
	
这个代码错误可能在某一个时刻绊倒过每一个JavaScript开发人员。这里发生了什么？其实，因为我们在循环中使用了`setTimeout`，而不是在每次循环中调用`callback(arr[i])`，我们实际上将一个函数调用延迟了，延迟到当前同步代码块的结束（也就是，`for`循环）。在这个例子中，我们纪录了`begin`，然后延迟3个回调函数的调用，然后纪录`end`，然后释放CPU到事件循环中。事件循环又开始执行我们延迟的回调函数，按顺序执行，而我们期望它的结果应该是纪录`arr[0]`，`arr[1]`和`arr[2]`。

## JavaScript的范围和闭包

为什么结果会是打印三次`undefined`而不是`1`,`2`,`3`呢？这就涉及到另一个重要的内容，JavaScript函数与范围：[闭包](http://en.wikipedia.org/wiki/Closure_(computer_programming))的概念。当你在JavaScript中创建一个函数，这个函数可以访问它被创建的那个范围的所有东西，包括任何你在函数内部创建的新变量。JavaScript不像C,C++,Java,C#那样，没有块级作用域。取而代之的是在函数级别定义作用域。你在函数中定义的任何变量，在函数的其他地方或任何内部函数中都是可以被访问到的。有趣的是，不仅仅一个函数能访问它当前环境作用域的所有变量，函数实例在其整个生命周期同样也“持有”（“关闭”）该作用域，即使其父函数（调用它的函数）已经返回或离开该作用域。只要该函数仍然“存活”（被某些东西引用，还未被回收），那么它就会持有该作用域，即使父函数早已消失。由于函数实例可以从一个函数中被返回，一个函数就可以轻易地在父函数或调用函数生命周期之外存活。这中“持有”作用域有事会导致微小的内存泄露，但我们现在不讨论它。在上面的`asyncEach`示例中，实际发生的事情是：每一个我们在`for`循环中延迟的回调函数保存了到当前作用域的引用（当时这个作用域还存在着），并且持有该作用域即使`for`循环以及`asyncEach`函数已经退出。而回调函数存活在`for`循环之外，因为回调函数实例被通过setTimeout添加至事件队列中，因此作用域变量例如`arr`和`i`还活着，但是现在`i`的值变成了`3`，因为`for`循环在之前已经同步执行到结束了。在每一个回调中，`logItem`函数每次都访问arr[3]，也就是undefined。

有很多种方式可以处理这个问题，但大多数都围绕着围绕我们希望之后捕获的变量周围添加一个额外的函数作用域。

## 基于异步的"each"函数 （第二次尝试）

一个在每个回调函数中获得所期望的`i`值的解决方法是，在我们希望捕获的变量周围引入一个“立即执行的函数表达式”([immediately-invoked function expression, IIFE](http://en.wikipedia.org/wiki/Immediately-invoked_function_expression))。一个IIFE有多种用法，其中一个就是当你没别的办法获取一个作用域时，强制创造一个作用域（例如在`for`循环中）。[这在循环中是不被建议的](http://jslinterrors.com/dont-make-functions-within-a-loop)，但是它可以起到作用：

	// Not recommended
	function asyncEach2(arr, callback) {
	    for (var i = 0; i < arr.length; ++i) {
	        // Use an IIFE wrapper to capture the current value of "i" in "iCopy"
	        // "iCopy" is unique for each iteration of the loop.
	        (function(iCopy) {
	            setTimeout(function() {
	                callback(arr[iCopy]);
	            }, 0);
	        }(i));
	    }
	}
	
	console.log("begin");
	asyncEach([1, 2, 3], logItem);
	console.log("end");

现在我们能得到所期望的结果了：

	begin
	end
	1
	2
	3
	
这之所以能够起作用是因为我们在每一个函数迭代周期创造了一个内部函数作用域，并且我们创建了新的变量`iCopy`并赋予其每个迭代周期中`i`的值。`iCopy`在每一个循环周期都是独一无二的，因此我们不再会遇到在作用域外引用一个变量的问题，在之前的示例中，我们在得到它之前它就变掉了。

## 基于异步的"each"函数 （第三次尝试）

一个更倾向的解决问题的方式不是在循环内部使用IIFE，而是在循环外创建一个函数以创造我们的函数作用于，像这样：

	function asyncEach3(arr, callback) {
	    // Utility inner function to create a wrapper function for the callback
	    function makeCallbackWrapper(arr, i, callback) {
	        // Create our function scope for use inside the loop
	        return function() {
	            callback(arr[i]);
	        }
	    }
	
	    for (var i = 0; i < arr.length; ++i) {
	        setTimeout(makeCallbackWrapper(arr, i, callback), 0);
	    }
	}
	
	console.log("begin");
	asyncEach3([1, 2, 3], logItem);
	console.log("end");
	
这一次，我们使用了一个单独的函数`makeCallbackWrapper`来为每个循环迭代创造我们的函数作用域。这次代码更简洁，容易阅读和维护，并且避免了"循环内IIFE"所带来的性能问题。

## 基于异步的"each"函数 （第四次尝试）

另一个更先进的在`for`循环内部创造作用域的方式是使用函数绑定或者[partial application](http://en.wikipedia.org/wiki/Partial_application)，就像使用原生的[Function.prototype.bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)函数（只有新的浏览器支持），或者使用由[Underscore](http://underscorejs.org/)，[Lo-Dash](http://lodash.com/)，[jQuery](http://jquery.org/)三者任意一个提供的`bind`实现。

函数绑定以及partial application是更大的话题，会在今后的博客文章中讨论。

	function asyncEach4(arr, callback) {
	    for (var i = 0; i < arr.length; ++i) {
	        // boundCallback is a new function which has arr[i] permanently
	        // set (partially applied) as its first argument.  The "null" argument
	        // is the binding for the `this` context variable in the callback, which
	        // we don't care about in this example...
	        var boundCallback = callback.bind(null, arr[i]);
	        setTimeout(boundCallback, 0);
	    }
	}
	
	console.log("begin");
	asyncEach4([1, 2, 3], logItem);
	console.log("end");
	
## 异步代码的执行顺序

我们要如何才能在使用`asyncEach`时保持日志纪录顺序呢，像原先同步的`each`例子一样：

	begin
	1
	2
	3
	end
	
下篇文章讨论。

## 总结

这只是一个JavaScript中基于回调的API以及异步编程的基本介绍。在今后的博客中，我将会探索如何在asyncEach函数中获取你所期望的顺序执行的功能，这样我们仍然能打印`begin, 1, 2, 3, end`即使回调函数是异步的情况。我也会讨论JavaScript中异步编程的其他问题，包括函数上下文变量context是如何与回调一起工作的，“回调地狱”的概念，以及Javascript中的promise是如何解决这些问题的。
