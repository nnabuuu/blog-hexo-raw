title: Design-of-Q
date: 2014-08-18 19:58:48
tags:
---
/*
本文档的目的是通过增量的、回顾其主要设计决策的方式来构建一个promise库，以此解释promises如何工作以及为何采用这样独特的方式实现。旨在让用户可以自由的体验到不同的实现方式以便满足其自身的需求，而不必错过任何重要的细节。

-

设想一下你正在编写一个无法立即返回一个值的函数。最显而易见的API是：将最终结果传递到一个作为参数的回调函数中，来代替将其返回。
*/

	var oneOneSecondLater = function (callback) {
	    setTimeout(function () {
	        callback(1);
	    }, 1000);
	};

/*
这是一个解决这种琐碎问题的一个非常简单的方案，但它还有很多进步的空间。

一个更通用的解决方案会同时为返回值以及被抛出的异常提供同样的工具（即回调函数）。有几个显而易见的方式来扩展回调模式以处理异常。其中一个是同时提供一个普通的回调函数(callback)以及一个处理错误的回调函数(errback)。
*/

	var maybeOneOneSecondLater = function (callback, errback) {
	    setTimeout(function () {
	        if (Math.random() < .5) {
	            callback(1);
	        } else {
	            errback(new Error("Can't provide one."));
	        }
	    }, 1000);
	};
/*
还有其他的解决方案，区别在于将错误作为回调函数的一个参数传入，以位置或者“哨兵值“进行区分。但是，这样的解决方案没有一个事实上考虑了被抛出的异常。异常以及try/catch块的目的是延后明确处理异常的时间直到程序已经回到一个有意义尝试从异常中恢复的点。如果异常没有被处理，我们需要一些机制来隐式地传播异常。


Promises
========

考虑一个更普遍的解决方式，代替返回值或者抛出异常，函数返回一个表示函数最终结果的对象，要么成功，要么失败。这个对象，无论是从打比方还是命名的角度来说，就是一个最终要被处理的promise，我们可以在promise之上调用函数来观测它是被满足还是被拒绝。如果这个promise被拒绝而并没有被主动的观测到，那么其他衍生的promise会被因为一些原因被隐式地拒绝。

在下面这个特殊的设计迭代中，我们将promise设计为一个带有以回调函数为参数的"then"函数的对象。
*/

	var maybeOneOneSecondLater = function () {
	    var callback;
	    setTimeout(function () {
	        callback(1);
	    }, 1000);
	    return {
	        then: function (_callback) {
	            callback = _callback;
	        }
	    };
	};

/*
这种设计有两个不足：

- 第一个then的调用者决定被使用的回调函数。如果每一个被注册的回调函数都会被resolution通知到的话会更好。
- 如果回调函数在promise被构造多于一秒之后注册，那么它不会被调用。

一个更普遍的解决方式在于接受任意数量的回调函数并且允许他们无论在超时（更普遍的说
，resolution事件被触发）前或后均可注册。我们通过将promise设置为一个两种状态的对象来实现。

一个promise最初处于unresolved状态，并且所有的回调函数都被加入一个pending状态观测者的数组中。当promise被resolve之后，所有的观测者都会被通知到。我们通过判断是否这个pending回调函数的队列存在的方式来区分状态是否被转换，我们在resolution之后会扔掉它（这个队列）。
*/

	var maybeOneOneSecondLater = function () {
	    var pending = [], value;
	    setTimeout(function () {
	        value = 1;
	        for (var i = 0, ii = pending.length; i < ii; i++) {
	            var callback = pending[i];
	            callback(value);
	        }
	        pending = undefined;
	    }, 1000);
	    return {
	        then: function (callback) {
	            if (pending) {
	                pending.push(callback);
	            } else {
	                callback(value);
	            }
	        }
	    };
	};
	
/*
这样已经足够好了，如果将其改为一个功能函数的话会非常有用处。一个deferred是一个拥有两个部分的对象：一个用来注册观测者，另一个用来将resolution通知观测者。
（见 design/q0.js）
*/

	var defer = function () {
	    var pending = [], value;
	    return {
	        resolve: function (_value) {
	            value = _value;
	            for (var i = 0, ii = pending.length; i < ii; i++) {
	                var callback = pending[i];
	                callback(value);
	            }
	            pending = undefined;
	        },
	        then: function (callback) {
	            if (pending) {
	                pending.push(callback);
	            } else {
	                callback(value);
	            }
	        }
	    }
	};
	
	var oneOneSecondLater = function () {
	    var result = defer();
	    setTimeout(function () {
	        result.resolve(1);
	    }, 1000);
	    return result;
	};
	
	oneOneSecondLater().then(callback);

/*
现在这个resolve有一个瑕疵：它能够被调用多次，从而改变被promised了的结果。它没有能够符合“一个函数只能要么返回一个值要么抛出一个错误”的事实要求。我们可以通过只允许第一次调用来设定resolution的方式保护结果免于意外或者恶意的重置。
*/

	var defer = function () {
	    var pending = [], value;
	    return {
	        resolve: function (_value) {
	            if (pending) {
	                value = _value;
	                for (var i = 0, ii = pending.length; i < ii; i++) {
	                    var callback = pending[i];
	                    callback(value);
	                }
	                pending = undefined;
	            } else {
	                throw new Error("A promise can only be resolved once.");
	            }
	        },
	        then: function (callback) {
	            if (pending) {
	                pending.push(callback);
	            } else {
	                callback(value);
	            }
	        }
	    }
	};
	
/*
你可以设置一个参数，在这种情况下要么抛出一个错误或者忽略之后所有其他resolution。一个测试就是给予resolver一串worker，并竞争resolve该promise，而剩余的resolution会被忽略。如果你不想让这些worker知道谁获胜了，这也是可行的。下文中，所有的示例都会忽略多重resolution而不是抛出异常。

现在，defer可以同时处理多resolution和多observation的情况。（见 design/q1.js）

--------------------------------

源于两个单独的立场，这个设计衍生出了几种不同的变化。第一种立场是：将promise和deferred中的resolver部分分离或者结合都是有用的。通过某种方式从其他值中识别promise也是有用的。

-

将promise从resolver中分离开允许我们在最小特权原则下进行编码。给予某人一个promise，应该仅仅给予他观测resolution的权力，而给予某人一个resolver，应该仅仅给予他决定resolition的权力。一方的权力绝不不应该被授予另一方。通过时间的检验我们发现，任何过度的授权都会不可避免地被滥用，并且这将会非常难以编写。

然而，分离的不好之处在于，快速地废除promise对象会给垃圾回收器带来额外负担。

-

以外，有非常多的方式来区分promise以及其他值。最显而易见并且最重要的区分方式是使用原型继承（design/q2.js）
*/

	var Promise = function () {
	};
	
	var isPromise = function (value) {
	    return value instanceof Promise;
	};
	
	var defer = function () {
	    var pending = [], value;
	    var promise = new Promise();
	    promise.then = function (callback) {
	        if (pending) {
	            pending.push(callback);
	        } else {
	            callback(value);
	        }
	    };
	    return {
	        resolve: function (_value) {
	            if (pending) {
	                value = _value;
	                for (var i = 0, ii = pending.length; i < ii; i++) {
	                    var callback = pending[i];
	                    callback(value);
	                }
	                pending = undefined;
	            }
	        },
	        promise: promise
	    };
	};
	
/*
使用原型继承的缺点在于它使得一个项目中只能使用一种promise库。这会难以实施，使实施依赖变成一种灾难。

另一种实现方式是使用duck-typing，通过是否存在某种约定命名的方法来区分promise以及其他对象。在我们的案例中，CommonJS/Promises/A创立了通过是否使用了"then"的方式来区分promise以及其他值。这个方式的缺点是无法判断那些只是碰巧有一个"then"方法的对象。在实际状况中，这并不是一个问题，并且这种实现"可以then"的微小差异是可以被管理的。
*/

	var isPromise = function (value) {
	    return value && typeof value.then === "function";
	};
	
	var defer = function () {
	    var pending = [], value;
	    return {
	        resolve: function (_value) {
	            if (pending) {
	                value = _value;
	                for (var i = 0, ii = pending.length; i < ii; i++) {
	                    var callback = pending[i];
	                    callback(value);
	                }
	                pending = undefined;
	            }
	        },
	        promise: {
	            then: function (callback) {
	                if (pending) {
	                    pending.push(callback);
	                } else {
	                    callback(value);
	                }
	            }
	        }
	    };
	};
	
/*
下一个大的步骤是使它可以简单的生成promise，使用从旧的promise中获取的值来构造新的promise。假设你收到从数个函数调用中得到的两个数字的promise，我们应该可以创建他们的和的promise。考虑一下这用callback是怎样实现的。
*/

	var twoOneSecondLater = function (callback) {
	    var a, b;
	    var consider = function () {
	        if (a === undefined || b === undefined)
	            return;
	        callback(a + b);
	    };
	    oneOneSecondLater(function (_a) {
	        a = _a;
	        consider();
	    });
	    oneOneSecondLater(function (_b) {
	        b = _b;
	        consider();
	    });
	};
	
	twoOneSecondLater(function (c) {
	    // c === 2
	});
	
有非常多的原因证明，这种实现是非常脆弱的，特别是它需要明确的编码来进行通知（在本例中是使用一个哨兵值），是否是一个回调函数被调用了。此外，我们必须注意考虑到在事件循环结束前被发出的条件：`consider`函数需要在它被使用之前出现。

在下面的几个步骤中，我们会能够用promise实现它，使用更少的代码以及隐式地处理错误传递。
*/

	var a = oneOneSecondLater();
	var b = oneOneSecondLater();
	var c = a.then(function (a) {
	    return b.then(function (b) {
	        return a + b;
	    });
	});

