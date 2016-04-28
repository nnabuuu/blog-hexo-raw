title: 'Getting Concurrent With ES6 Generators'
date: 2015-01-04 21:44:36
tags: generator translation
---
#用generator实现并发（译）

原文地址：[http://davidwalsh.name/concurrent-generators](http://davidwalsh.name/concurrent-generators)

ES6 Generators：全系列

1. [The Basics Of ES6 Generators](http://davidwalsh.name/es6-generators)
2. [Diving Deeper With ES6 Generators](http://davidwalsh.name/es6-generators-dive)
3. [Going Async With ES6 Generators](http://davidwalsh.name/async-generators)
4. [Getting Concurrent With ES6 Generators](http://davidwalsh.name/concurrent-generators)

如果你已经阅读并消化了本系列的第一、第二和第三章节，你应该对ES6 generator已经相当有把握了。希望你真正的被激发来开始用它们做一些事情。

我们要探索的最后的话题是比较前沿的东西，它可能会让你的大脑有点混乱（老实的说，我的大脑仍然处于混乱状态）。所以慢慢的看完并且思考这些概念以及例子。并且还需要读一读其他关于这个话题的文章。

你在这里所做的投入，会从一个长远的角度给你带来回报。我完全确信JS完美的异步能力的未来会来自于这些概念。

## 标准CSP（Communicating Sequential Processes）
受限，我受到这个话题的激发完全是来自于[David Nolen](http://github.com/swannodette) [@swannodette](http://twitter.com/swannodette)。严肃的说，他所写得一切文章都值得一读。这里有一些可以让你起步的链接：

* ["Communicating Sequential Processes"](http://swannodette.github.io/2013/07/12/communicating-sequential-processes/)
* ["ES6 Generators Deliver Go Style Concurrency"](http://swannodette.github.io/2013/08/24/es6-generators-and-csp/)
* ["Extracting Processes"](http://swannodette.github.io/2013/07/31/extracting-processes/)

好的，下面我就来讲一讲我在这个题目上的探索。我并不是从一个正式的Clojure背景转向JS的，我也没有GO或者ClojureScript相关的经验。我很快发现我迷失在这些文章中，并且我必须要做大量的实验和猜测来收集这些文章中有用的部分。

在探索的过程中，我认为我已经学到了一些目标以及精神都想通的东西，但来自于并不是那么古板的思维方式。

我试图要做的是简历一个简单的Go-风格的CSP（已经ClojureScript core.async）API，同时（我希望）保留大部分潜在的能力。当然，那些比我聪明的人完全有可能迅速地看到我目前为止探索过程中错过的部分。如果这牙膏的话，我希望我的探索能够继续进行下去，并且我会继续和我们分享我的启示！

## （部分）分解CSP理论

CSP是讲些什么的？它提到的“communicating”、“Sequential”是什么？“Processes”又是什么？

首先，CSP来自于[Tony Hoare的书"Communicating Sequential Processes"](http://www.usingcsp.com/)。这是一些厚重的CS理论的东西，但如果你有兴趣在学术范围有所建树的话，那么这是最好的开始的地方。当然我并不是要用一种深奥的、令人头痛的、CS的方式来讲述它。我会用一种非正式的方式来描述。

所以，我们先从“sequential”这个词开始。这个部分你应该已经非常熟悉了。它所谈论的是另一种单线程行为以及我们用ES6 generators实现的同步风格代码的方式。

回忆一下，generator是的语法是像这样的：

```js
function *main() {
    var x = yield 1;
    var y = yield x;
    var z = yield (y * 2);
}
```

每一条语句都是有序的（按照顺序）执行，每次执行一条。`yield`关键词表明代码中有可能发生堵塞暂停（只堵塞当前generator自己代码，并不堵塞外部程序！）的地方，但是这并不改变`*main()`内部自上而下的代码处理顺序。很简单，是吧？

下面，让我们谈谈“processes”，这又是关于些什么的呢？

从本质上来说，一个generator就像是一个虚拟“进程”。它是一块自我包含的程序，它可以，如果JS允许这样做的话，完全和程序剩余部分并发执行。

事实上，这样说有一点点问题。如果generator访问共享内存（也就是说，如果它访问它自身内部本地变量中的“free viriables”），这就不是独立的了。但是让我们先假设一下我们的generator函数并不访问外部变量（因此在函数式编程中我们会把它叫做一个“combinator”）。这样它就可以在理论上以自己单独进程的方式运行。

但是上面我们所说的是“processes” -- 复数形式 -- 因为有一点很重要的是，同时有两个或多个进程在运行。换句话说，两个或更多的generator进行配对，一般来说是进行协作以便完成更大的任务。

为什么要分离generators而不仅仅用一个呢？最重要的原因是：**内容隔离**。如果你可以将任务XYZ分解为各个子任务X、Y和Z，那么它们的实现代码会更加容易看懂和维护。

当你将一个函数`function XYZ()`分解为`X()`，`Y()`和`Z()`也是同样的道理，这里`X()`会调用`Y()`，`Y()`会调用`Z()`。我们将函数分解为单独的子函数来进行更好的代码隔离，使我们的代码更容易维护。

**我们可以对多个generator做同样的事情。**

最后我们来说一说“communicating”。这又是什么呢？这是从上面两个词推论出来的 -- 协作 -- 如果generator需要一起工作，它们需要一种通信渠道（不仅仅是访问共享词法作用域中的内容，而是实实在在的能够进行排他性访问的共享通信渠道）。

这个通信渠道是用来发送什么的呢？可以发送无论任何你需要发送的东西（数字、字符串、等等）。事实上，你甚至不需要真正的往通道中发送数据以便使用该通道进行通信。“Communication”可以是简单的协调 -- 就像将控制权从一个地方转移到另一个地方。

为什么我们要转移控制权？主要的原因是JS是单线程的，并且在任何时刻它们中只有一个可以处于活动状态。其余的都处在暂停执行的状态，也就意味着它们处在任务的中途阶段，而只是暂停了，它们在等待必要时恢复。

任意独立的“进程”可以神奇的合作和交流，这似乎并不现实。松散耦合的目标是相当令人敬佩的，但它其实不切实际。

相反，似乎所有成功的CSP实现都是一种对某个特定领域内现有已知逻辑集合的因式分解，它的每一个部分都是特地为其他部分的协作而设计的。

或许我完全错了，但是我还没有看到任何方式让两个随机的generator函数可以非常容易的粘在一起变成一个CSP配对。他们都需要被设计成为另一方工作，接受同样的通信协议，等等。

## JS中的CSP

这里有几个已经应用在JS中了的CSP理论。

我们之前提到的Dabid Nolen创建了几个有趣的项目，包括[Om](https://github.com/swannodette/om)以及[core.async](http://www.hakkalabs.co/articles/core-async-a-clojure-library/)。[Koa](http://koajs.com/)库(为node.js创建)有一些非常有趣的实现，主要是通过它的`use(..)`方法。另一个对core.async/Go CSP API非常“忠诚”的库是[js-csp](https://github.com/ubolonton/js-csp)。

你应该确切的研究一下这些伟大的项目来看看不同的实现方式以及关于如何探索JS中的CSP的例子。

## asynquence的`runner(..)`: 设计CSP
由于我一直以来都在积极地探索如何将CSP模式的并发性应用到我自己的JS代码中，因此为我的异步流控制库[asynquence](http://github.com/getify/asynquence)扩展CSP能力是非常自然的。

我已经有了`runner(..)`功能插件来处理generator的异步运行（见 ["Part 3: Going Async With Generators"](http://davidwalsh.name/async-generators/#rungenerator-library-utility)），所以在我看来，它可以很容易地扩展为用[类-CSP的方式](https://github.com/getify/asynquence/tree/master/contrib#csp-style-concurrency)同时处理多个generator。

我所要解决的第一个问题：你怎么知道下面是哪一个generator要获得控制权？

给每个generator赋予一个ID并让其他generator知晓以便它们将消息或者控制权传递给另一个进程的方式未免太繁琐和笨重了。在经过多次试验之后，我选定了一个简单的round-robin的调度方法。因此如果你将三个generator A、B和C匹配起来之后，A会先获得控制权，然后B在A进行yield时接管，然后C在B进行yield时接管，然后再到达A，以此类推。

但是我们要怎样才能真正的传递控制呢？需要一个明确的API来描述它吗？再一次地，经过多次试验之后，我使用了一个更加隐式的实现方式，它和[Koa的实现方式](http://koajs.com/#cascading)（碰巧）很相似：每个generator获得一个共享“token”的引用 -- `yield它以便传递控制权转移的信号。

另一个问题则是消息通道应该是什么样的。一方面，你有一个非常确定了的API就像core.async以及js-csp中的那些（`put(..)`以及`take(..)`）。在我自己的经验中，我则倾向于另一个方面：一个不那么正式的方法（它甚至不是一个API，只是一个共享的数据结构就像`array`这样），这看起来就够了。

我决定使用一个数组(称为`messages`)这样你便可以大刀阔斧的决定你要如何来使用它。可以将消息`push()`进入这个数组，也可以从这个数组中`pop()`消息，指定数组中会话相关的元素组建不同消息，并可以在这些空间内创建更加复杂的数据结构，等等。

我的设想是有些任务只需要非常简单的消息传递，而另外一些会非常的复杂，因此以其将简单的内容变得复杂，不如将消息通道变得正式而使其成为一个`array`（这样就不需要除了`array`自身以外的API了）。将消息传递机制转化为其他形式是非常容易的，你会发现它的妙用（见我们接下来会瘫倒的状态机的例子）。

最后，我留意到这些generator“进程”仍然可以从普通generator的异步能力中获益。换句话说，如果你`yield`出一个Promise（或asynquence sequence）而不是一个控制token，`runner(..)`机制将会暂停并等待将来的结果值而不会**移交控制权** -- 相反，它会将结果返回给当前的进程（generator）并使它继续享有控制权。

因此最后一点备受争议的应该是（如果我的想法都是正确的话），它和空间内的其它库都不一样。看起来真正的CSP对我这样的实现方式嗤之以鼻。但是我认为我的提议最终会变得非常非常有用。

## 一个简单的FooBar示例

好的，我们说够了理论，现在让我们来看看代码：

```js
// Note: omitting fictional `multBy20(..)` and
// `addTo2(..)` asynchronous-math functions, for brevity

function *foo(token) {
    // grab message off the top of the channel
    var value = token.messages.pop(); // 2

    // put another message onto the channel
    // `multBy20(..)` is a promise-generating function
    // that multiplies a value by `20` after some delay
    token.messages.push( yield multBy20( value ) );

    // transfer control
    yield token;

    // a final message from the CSP run
    yield "meaning of life: " + token.messages[0];
}

function *bar(token) {
    // grab message off the top of the channel
    var value = token.messages.pop(); // 40

    // put another message onto the channel
    // `addTo2(..)` is a promise-generating function
    // that adds value to `2` after some delay
    token.messages.push( yield addTo2( value ) );

    // transfer control
    yield token;
}
```

OK，现在我们有两个generator“进程”， `*foo()`和`*bar()`。你能留意到他们都处理了`token`对象（当然你也可以给它们任意命名）。`token`的`messages`熟悉是我们的共享消息通道。在初始时，它们充满了我们在初始化CSP运行时所产生的消息（见下文）。

`yield token`显示地将控制权传递给“下一个”generator（通过round-robin策略）。然而，`yield multiBy20(value)`以及`yield addTo2(value)`都在yield promise（从这些虚构的延迟数学函数中），这表明generator在这一点会暂停知道promise结束。在promise结束时，当前处于控制的generator会获取返回值并继续下去。

无论最后的`yield`返回值是什么，在本例中是`yield "meaning of...`表达式，它都是我们的CSP运行完成的消息（见下文）。

现在我们有了两个CSP进程generator，我们要如何执行它们呢？使用asynquence吧：

```js
// start out a sequence with the initial message value of `2`
ASQ( 2 )

// run the two CSP processes paired together
.runner(
    foo,
    bar
)

// whatever message we get out, pass it onto the next
// step in our sequence
.val( function(msg){
    console.log( msg ); // "meaning of life: 42"
} );
```

显然，这只是一个简单的示例。但是我认为它准确的解释了这个概念。

现在或许你该[自己试一试](http://jsbin.com/tunec/2/edit?js,console)了（试着将返回值进行链式处理！）来确保你已经理解了这些概念并且能够编写你自己的代码了！

## 另一个玩具Demo的例子

让我们用一个经典的CSP例子来试一试，但是让我们从我所观测到的简单部分开始，而不是像通常人们做的那样从一个学术的角度开始。

**Ping-pong**。很有趣的运动是吧？这是我最喜欢的运动。

让我们设想一下你已经实现了进行一次乒乓球比赛的代码。你用一个循环来运行，然后你有两块代码（例如，用一个`if`或者`switch`实现的分支）每一块代表者一名选手。

你的代码运行得很好，你的游戏就看起来像是一个真正的乒乓球比赛一样！

但是我们之前看到的是CSP在什么方面非常有用？**逻辑隔离**。我们在乒乓球运动中要隔离的逻辑是？这两名选手！

因此，我们可以，站在一个非常高的角度，来给两个“进程”（generators）建模，每一个代表一名选手。随着我们的深入，我们会发现这些用来在两名选手之间传递控制权的“胶水代码”其自身是一个task，并且它的代码可以被放到第三个generator中，这里我们可以将其建模为一个裁判。

我们会过滤掉所有的领域相关问题，例如得分、赛制、物理学、策略、AI、控制、等等。我们唯一关心的只是模拟来回击球（这也就是我们的CSP控制权转移的描述）。

想看看Demo吗？[点击这里运行](http://jsbin.com/qutabu/1/edit?js,output)（注：你需要使用最新版本的FF或者Chrome，它们支持ES6 JavaScript，来看到generator是如何运作的）。

好，下面我们逐条讲解我们的代码。

受限，asynquence sequence看起来长什么样呢？

```js
ASQ(
    ["ping","pong"], // player names
    { hits: 0 } // the ball
)
.runner(
    referee,
    player,
    player
)
.val( function(msg){
    message( "referee", msg );
} );
```

我们将序列初始化为两条消息：`["ping", "pong"]`和`{hits: 0}`。我们马上就会用到它们。

然后，我们设置CSP运行3个进程（通过轮询的方式）：一个`*referee()`和两个`*player()`实例。

比赛最后的消息在我们的序列中不停传输，并最后作为裁判的消息被打印出来。

裁判的实现：

```js
function *referee(table){
    var alarm = false;

    // referee sets an alarm timer for the game on
    // his stopwatch (10 seconds)
    setTimeout( function(){ alarm = true; }, 10000 );

    // keep the game going until the stopwatch
    // alarm sounds
    while (!alarm) {
        // let the players keep playing
        yield table;
    }

    // signal to players that the game is over
    table.messages[2] = "CLOSED";

    // what does the referee say?
    yield "Time's up!";
}
```
我把控制token称为`table`以便和问题的domain（乒乓球）相匹配。语义上让一个选手击球时"yield the table"给另一个选手相当恰当，不是吗？

`while`循环`*referee()`只是不停的yield `table`回给选手只要它的手表还没有计时结束。当结束时，它会宣布`"Time's up!"`

现在，让我们来看看`*player()` generator（我们使用了两个实例）。

```js
function *player(table) {
    var name = table.messages[0].shift();
    var ball = table.messages[1];

    while (table.messages[2] !== "CLOSED") {
        // hit the ball
        ball.hits++;
        message( name, ball.hits );

        // artificial delay as ball goes back to other player
        yield ASQ.after( 500 );

        // game still going?
        if (table.messages[2] !== "CLOSED") {
            // ball's now back in other player's court
            yield table;
        }
    }

    message( name, "Game over!" );
}
```

第一个选手从数组中获取他自己的名字（`"ping"`），然后第二个选手获取他的名字（`“pong”`），因此他们可以同时辨别双方。双方都保留一个对共享`ball`对象的引用（包含了`hits`计数器）。

当选手还未听到裁判结束的消息时，他们通过将计数器`hits`增加1的方式"击打"`ball`（并且输出一条消息进行公告），然后他们等待500毫秒（只是来模拟一下球速并没有达到光速啦！）

如果比赛仍在继续，他们会"yield table"回到另一名选手中。

就是这样啦！

[看看这里的Demo代码](http://jsbin.com/qutabu/1/edit?js,output)，通过将所有的代码放在一起看看它们是如何共同工作的。

## 状态机：Generator协作

我们最后一个例子：定义一个[状态机](http://en.wikipedia.org/wiki/Finite-state_machine)作为一系列generator的协同程序，它们有一个简单的helper驱动。

[Demo](http://jsbin.com/luron/1/edit?js,console)（用最新的FF或者Chrome打开）

首先，让我们顶一个一个help函数来控制我们的有限状态句柄：

```js
function state(val,handler) {
    // make a coroutine handler (wrapper) for this state
    return function*(token) {
        // state transition handler
        function transition(to) {
            token.messages[0] = to;
        }

        // default initial state (if none set yet)
        if (token.messages.length < 1) {
            token.messages[0] = val;
        }

        // keep going until final state (false) is reached
        while (token.messages[0] !== false) {
            // current state matches this handler?
            if (token.messages[0] === val) {
                // delegate to state handler
                yield *handler( transition );
            }

            // transfer control to another state handler?
            if (token.messages[0] !== false) {
                yield token;
            }
        }
    };
}
```

该`state(..)` helper工具方法创建了一个为一个具体的状态值准备的[delegating-generator](http://davidwalsh.name/es6-generators-dive#delegating-generators)包装器，它会自动的运行状态机，并且将控制权在每个状态间进行传递。

出于惯例，我决定共享`token.messages[0]`位置来保存当前状态机的状态。这表明你可以从之前的步骤中传递一条消息作为初始状态。但是如果没有这样的初始消息被传递的话，我们会简单的将第一个状态定义为我们的初始状态。同样出于惯例，最终状态被断言为`false`。这很容易实现：

状态值可以是任何种类的值：`number`s, `string`s，等等。只要这个值能被`===`处理，你就可以使用它来做你的状态。

在下面这个例子中，我展示了一个状态机从4个`number`形式的状态值之间进行转化，采用这种顺序`1 -> 4 -> 3 -> 2`。为了demo的目的，它也会进行计数以便进行不止一次的转换。当我们的generator状态机最终到达终止位（`false`）时，*asynquence*序列会移动到下一个步骤，如你所期待的那样。

```js
// counter (for demo purposes only)
var counter = 0;

ASQ( /* optional: initial state value */ )

// run our state machine, transitions: 1 -> 4 -> 3 -> 2
.runner(

    // state `1` handler
    state( 1, function*(transition){
        console.log( "in state 1" );
        yield ASQ.after( 1000 ); // pause state for 1s
        yield transition( 4 ); // goto state `4`
    } ),

    // state `2` handler
    state( 2, function*(transition){
        console.log( "in state 2" );
        yield ASQ.after( 1000 ); // pause state for 1s

        // for demo purposes only, keep going in a
        // state loop?
        if (++counter < 2) {
            yield transition( 1 ); // goto state `1`
        }
        // all done!
        else {
            yield "That's all folks!";
            yield transition( false ); // goto terminal state
        }
    } ),

    // state `3` handler
    state( 3, function*(transition){
        console.log( "in state 3" );
        yield ASQ.after( 1000 ); // pause state for 1s
        yield transition( 2 ); // goto state `2`
    } ),

    // state `4` handler
    state( 4, function*(transition){
        console.log( "in state 4" );
        yield ASQ.after( 1000 ); // pause state for 1s
        yield transition( 3 ); // goto state `3`
    } )

)

// state machine complete, so move on
.val(function(msg){
    console.log( msg );
});
```

应该很容易看出这里将发生什么。

`yield ASQ.after(1000)`表明这些generator可以做任何基于异步流的promise/sequence操作，就像我们之前看到的那样。`yield transition(..)`是我们如何转移到一个新的状态上。

我们前面的`state(..)` helper实际上做了处理`yield*`[代理](http://davidwalsh.name/es6-generators-dive#delegating-generators)以及状态转换的复杂工作，是的我们的状态句柄可以以一种非常简单和自然的格式进行编写。

## 总结

CSP的关键在于将两个或多个generator“进程”结合在一起，给予他们一个共享的通信渠道，以及一个它们之前传递控制权的方法。

JS中非常多的库都或多或少的有一些正式的实现，它们基本符合Go/Clojure/ClojureScript的API和语法。所有这些库的背后都有一些非常聪明的开发者，并且他们对于今后的研究都是一笔巨大的财富。

[asynquence](http://github.com/getify/asynquence)尝试了一种非正式的实现方式但仍然希望它能满足这个机制。如果没有别的，asynquence的`runner(..)`使上手使用CSP风格的generator非常简单，你可以进行尝试并且学习。

然而最好的部分仍然是asynquence CSP可以和其他异步功能（promise、generators、控制流等等）无缝结合，你可以使用你所拥有的任何工具，都在这样一个小小的库里。

在过去的4篇文章中我们已经讨论了相当多的关于generator的细节。我希望你对此感到兴奋并受到启发去探索如何改变你自己的JS代码！你会用generator来做些什么呢？