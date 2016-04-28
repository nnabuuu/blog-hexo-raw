title: '[译]函数响应式编程'
date: 2014-07-05 21:32:48
tags: translation
---
你所期待已久的函数响应式编程简介

原文地址：https://gist.github.com/staltz/868e7e9bc2a7b8c1f754

你一定对学习这个被称为（函数）响应式编程的东西感兴趣。

这东西学起来很难，缺乏好的资料使它变得更难。当我开始学习的时候，我尝试着寻找教程。我只找到少量的实践指南，而它们也仅仅是挠了挠表面，从未挑战去建立整个架构。当你想了解一些函数的时候，库的文档往往对你没什么帮助。我的意思是，老实说，看看这个：

	合并元素的索引，然后把一个可观测序列的可观测序列转化为一个可观测序列，仅仅从最近的可观测序列中取值。通过这种方法将每一个可被观测的序列中的对象投射到一个新的可观测序列的序列上。
	
天呐。。

我读了两本书，其中一本只是描绘了蓝图，另一本则一头钻进“如何使用FRP库”的问题中。最后我通过那种困难的方式学习了响应式编程：一边构造响应式编程项目一边学习。在我在Futurice的工作中，我在一个真正的项目上使用了它，当我遇到难题的时候我的一些同事帮助了我。

在学习的旅途中最困难的是用函数响应式编程进行思考。很多时候需要抛弃那些典型的、有状态的编程习惯，并迫使你的大脑在另一种模式下工作。我没有在网上找到任何这方面相关的内容，我任何需要这样一个“如何用函数响应式编程进行思考”的教程，以便你们可以开始迈出第一步。在那之后，库文档会帮你照亮后面的道路。希望这可以帮到你们。

#什么是函数响应式编程（FRP）？

对于函数相应式编程，网上有很多不好的解释以及定义。维基百科和平常一样说得太空泛以及理论化了。Stackoverflow的规范的答案显然不适合新手。Reactive Manifesto看起来像是你要讲给你公司里的项目经理或者商务人士所听的。微软Rx术语“Rx = Observables + LINQ + Schedulers”过于沉重（那么的“微软化”），我们大多数人都会感到困惑。像“reactive”，“"propagation of change”这种术语和我们典型的MV*以及最爱的编程语言已经做到了。我的框架当然是视图（Views）响应模型（Models）的，变化当然是可以传递的，否则的话什么也不会呈现。

所以，我们就不要继续说上面那些了。

####函数响应式编程就是通过异步数据流进行编程

在某种程度上，这并不是什么新的东西。事件总线或者典型的单击事件就已经是异步事件流了，你在它们上面可以进行观察或者做点其他的事情。函数响应式编程就是那些玩意儿再加上些内固醇。你可以对任何东西创造数据流，而不仅仅是click或者hover事件。流非常的廉价并且无处不在，任何东西都可以是流，变量、用户输入、属性、缓存、数据结构，等等。比如，设想一下你的Twitter feed是一个像单击事件一样的数据流。你可以监听这个是流并且做出相应的反应。

<b>更重要的是，你拥有了一个神奇的工具箱来连接、创造并且过滤任何这些流。</b>这就是“函数式”的魔力。一个流可以作为另一个流的输入，甚至可以是多个流作为一个流的输入。可以合并两个流。你可以过滤一个流而得到你另一个只包含你所感兴趣的事件的流。你可以把数据从一个流映射到另一个流。

如果流对于函数响应式编程如此重要，让我们仔细的来看看它们，从我们熟悉的“点击一个按钮”的事件流开始。

![image](https://camo.githubusercontent.com/28787087e17e13046655c0f71d6e4080c3508b10/68747470733a2f2f676973742e67697468756275736572636f6e74656e742e636f6d2f7374616c747a2f38363865376539626332613762386331663735342f7261772f343964613639346232343839663965376237323736646633316131646362323036313739613439362f7a636c69636b73747265616d2e706e67)

流是正在进行的事件按时间排序得到的序列。它可以广播三种不同的东西：一个值（属于某种类型）、一个错误，或者一个“已完成”的信号。设想一下，比如“已完成”会在当前包含此按钮的窗口或者视图被关闭时发生。

我们只采用<b>异步</b>的方式捕获这些被广播的事件，通过定义一个“当某个值被广播时执行”的函数，一个“当错误被广播时执行”的函数以及一个“当完成被广播时执行”的函数来完成。有时候后两个可以被省略，你可以只专注于定义那个捕获某个值的函数。对这个流进行“监听”被成为订阅。我们所定义的函数是观察者。流是被观测的主体（或被称为“可观测对象”）。这正是[<b>观察者设计模式</b>](https://en.wikipedia.org/wiki/Observer_pattern)

另一个画这张图的方式是使用ASCII，我们将在本教程的某些部分使用：

	--a---b-c---d---X---|->

	a, b, c, d 是被广播的值
	X 是一个错误
	| 是“已完成”信号
	---> 是时间轴
	
既然这已经感觉如此熟悉了，并且我不想让你们感到厌烦，让我们来做点新的事情：我们接下来要用原有的单击事件流构建出一些新的单击事件流。

首先，让我们创造一个计数事件流用来表明这个按钮被按了多少次。在函数响应式编程的公共库里，每一个流都有很多种函数附在它的上面，例如`map`, `filter`, `scan` 等等。当你调用其中一个函数时，比如`clickStream.map(f)`，它会基于当前的点击流返回一个新的流。它并不对原先的点击流做任何修改。这就是所谓的不变性，它和FRP流的关系就像煎饼和糖浆一样如此的美好。这允许我们进行链式调用，比如：`clickStream.map(f).scan(g)`:

	点击流: ---c----c--c----c------c-->
           vvvvv map(c 变为 1)    vvvv
           ---1----1--1----1------1-->
           vvvvvvvvv scan(+) vvvvvvvvv
	计数流: ---1----2--3----4------5-->
	
函数`map(f)`根据你所提供的函数`f`，将每一个被广播的值替换成新的值，并放入新的流中。在我们的例子里，我们把每一次点击映射为数字1.函数`scan(f)`汇集这个流上前面所有的值，产生`x = g(accumulated, current)`，而`g`在本例中只是一个简单的相加函数。这样，每当点击发生的时候，`counterStream`就会广播一次点击总数。

为了展现FRP的真正威力，我们考虑这样一个场景：你想要一个双击的事件流，为了让它更有趣一点，我们希望这个流把“三击”（更一般的情况，大于两次的点击）也考虑为双击。做一次深呼吸，想象一下在传统的、有状态的方式下你会怎样做？我敢打赌这一定相当麻烦并且涉及到许多用来保持状态和纪录时间间隔的变量。

好吧，在FRP中这非常简单。事实上，它的逻辑只需要[4行代码](http://jsfiddle.net/staltz/4gGgs/27/)。但首先，让我们忽略代码，用思维图是最好的理解以及构建流的方式。

![image](https://camo.githubusercontent.com/74d215aac2e23ae940cf5d1f4e08cc8878c9fecf/68747470733a2f2f676973742e67697468756275736572636f6e74656e742e636f6d2f7374616c747a2f38363865376539626332613762386331663735342f7261772f623538306164346133336236336163623263656439623865356539306661616238636137656632362f7a6d756c7469636c69636b73747265616d2e706e67)

灰色方格里面是讲一个流转化为另一个流的函数。简言之，我们首先把单击聚集到一个列表中，无论何时250毫秒的“事件沉默”发生（这正是`buffer(stream.throttle(250ms)`所做的）。现在不用担心理解细节，我们现在只是在演示使用FRP。它的结果是一个列表的流，在此基础上我们通过`apply()`将每一个列表映射为表示它的长度的数字。最后我们通过`filter(x >= 2)`忽略数字`1`，这样就完成了用3个步骤创建我们所需要的流。我们接下来就可以订阅（监听）这个流并按我们所希望的进行处理。

我希望你喜欢这种方法的美丽之处。这个例子仅仅是冰山一角：你可以用这种方法处理不同类型的流。例如：在API返回结果的流中，有很多其他可用的函数。

#“为什么我应该考虑采用FRP？”
FRP提高了你的代码的抽象层次，这样你就可以专注于业务逻辑所互相依存的事件中，而不必摆弄大量的实现细节。用FRP写出的代码可能会更简洁。

在现代网络以及移动应用程序中，它的好处更明显，这些应用往往与数据事件相关的UI事件有着高度的交互。10年前，与网页进行交互基本上就是向后端提交一个长的表单，并且在前端进行简单的呈现。而现在，应用程序已经进化到更实时：修改一个变淡字段可以自动触发保存到后端，“喜欢”一些内容可以实时反映给其他已连接的用户，等等。

当今的应用程序都有非常多各式各样的实时事件的使得与用户的高度交互体验成为可能。我们需要适当的工具来妥善处理它，而函数响应式编程就是答案。

#Thinking in FRP，例
让我们来考虑一个真正的场景，一个现实世界的例子来一步步引导你如何用FRP进行思考。没有集合的示例，没有解释不完全的内容。在本教程的最后我们将能够产生真正的功能代码，同时我们也会知道为什么我们这样做。

我选择了<b>JavaScript</b>和[RxJS](https://github.com/Reactive-Extensions/RxJS)作为这例子的工具，只有一个原因：JavaScript是最熟悉的语言，同时[Rx*库家族](https://rx.codeplex.com/)被广泛用于许多语言和平台。(.NET, Java, Scala, Clojure, JavaScript, Ruby, Python, C++, Objective-C/Cocoa, Groovy, etc).因此，无论你使用什么样的工具，你都可以从接下来的教程中获益。

#实现一个“你可能感兴趣的人”推荐框
在Twitter上有一个UI元素用来建议你可以关注的账号
![image](https://camo.githubusercontent.com/c30617647900cd2f1e03b2455fc31f72c000f0d2/68747470733a2f2f676973742e67697468756275736572636f6e74656e742e636f6d2f7374616c747a2f38363865376539626332613762386331663735342f7261772f323133303335373061316338633539396535396638663866626461326430336162393433616339622f7a74776974746572626f782e706e67)

我们要关注模仿其核心功能，它们是：

* 在开启时，从API读取账号数据并展示3个建议
* 在点击“刷新”时，向3行读取3个其他账号建议
* 在点击 "X" 按钮时，关闭其当前对应的账号并显示另外一个
* 每一行显示账号的头像并链接到它们的主页

我们可以省去其他功能和按钮因为它们是次要的。并且Twitter最近关闭了未经授权的公共API。让我们来为Github来构建一个类似的UI。Github提供了获取用户的API。

如果你想要快速浏览的话，完整的代码已经放在http://jsfiddle.net/staltz/8jFJH/48/了。

#请求和响应

<b>如何用FRP解决这个问题？</b>让我们开始，（几乎）所有的东西都可以是一个流。那就是FRP的咒语。让我们从最简单的功能开始：“当启动时，从API获取加载3个账户信息”。这没有任何特别之处，仅仅是(1) 发送一个请求， (2)获取响应 (3)呈现响应。所以我们继续，让我们的请求成为一个流。起初这会觉得有点过于简单了，但是我们需要从最基本的开始，不是吗？

在启动时我们只需要执行一个请求，所以如果我们将其建模为一个数据流，它会是一个只广播一个值的流。之后，我们知道我们会发出很多请求，但现在，只有一个。

	--a------|->

	这里a是一个字符串 'https://api.github.com/users'
	
这是一个我们想要请求的URL，当请求事件发生时，它告诉我们两件事情：“什么时候”与“什么”。“什么时候”请求应该被执行，也就是什么时候事件应该被广播。以及“什么”应该被响应，也就是被广播的值是什么：一个包含URL的字符串。

创建一个带有单一值这样的流在Rx*是非常简单的。流的官方术语是“可观测对象”，因为它可以被观测，但是我觉得这是个非常愚蠢的名字，所以我把它成为一个流。

	var requestStream = Rx.Observable.returnValue('https://api.github.com/users');
	
但是现在，这只是一个字符串的流，没法做其他操作，因此，我们要在这个值被广播的时候触发一些事情。这就是通过[描述](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservableprototypesubscribeobserver--onnext-onerror-oncompleted)这个流做到的。

	requestStream.subscribe(function(requestUrl) {
	  // 执行请求
	  jQuery.getJSON(requestUrl, function(responseData) {
	    // ...
	  });
	}
	
注意我们在使用JQuery Ajax回调（我们假设你应该[已经知道了](http://devdocs.io/jquery/jquery.getjson)）来处理异步请求。但是等一下，FRP是用来处理异步数据流的。那个请求对应的相应不能是一个包含“将来一段时间会到达数据”的流吗？嗯，在概念的层面上，看起来的确是这样，所以让我们来试一下。
	requestStream.subscribe(function(requestUrl) {
	  // 执行请求
	  var responseStream = Rx.Observable.create(function (observer) {
	    jQuery.getJSON(requestUrl)
	    .done(function(response) { observer.onNext(response); })
	    .fail(function(jqXHR, status, error) { observer.onError(error); })
	 	.always(function() { observer.onCompleted(); });
	  });
	
	  responseStream.subscribe(function(response) {
	    // 处理响应
	  });
	}
	
`Rx.Observable.create()`所做的就是通过显式通知每一个观察者（或者说是“订阅者”）数据事件(`onNext()`)或者错误(`onError()`)来创造你自己的流。我们刚刚所做的只是包装JQuery Ajax Promise。<b>等一下，这是否说明一个Promise就是一个可观测对象呢？</b>

![image](https://camo.githubusercontent.com/4df519edd2d527bf5e90b7d00e22cdc3c3be00d4/687474703a2f2f7777772e6d79666163657768656e2e6e65742f75706c6f6164732f333332342d616d617a65642d666163652e676966)

是的。

可观测对象是一个Promise++。在Rx中你可以通过`var stream = Rx.Observable.fromPromise(promise)`很容易的把一个Promise转化为一个可观测对象，所以我们就这样用吧。唯一的区别在于，可观测对象并不与Promise/A+兼容，但是在概念上是没有冲突的。Promise是一个简单的带有单一广播值的可观测对象。FRP流相比起Promise而言允许很多返回值。

这样很棒，说明了FRP至少和Promise一样强大。如果你相信Promise有点大肆炒作了，那么留意一下FRP的能力。

现在回到我们的例子，如果你注意到了的话，我们在`subscribe()`中调用了另一个，这看起来就像是传说中的回调地狱。而且，创建`responseStream`是基于`requestStream`的。就像你之前听到的那样，在FRP中我们有在其他流以外简单的转换以及创建新的流的方式。我们正应该那样做。

你现在应该知道的一个基本的函数是`map(f)`，它获取流A中的所有值，对其调用`f()`，然后产生出流B的一个对应的值。如果我们对我们的请求和响应那样做的话，我们就可以把请求URL映射到响应Promise（伪装成流）中。

	var responseMetastream = requestStream
	  .map(function(requestUrl) {
	    return Rx.Observable.fromPromise(jQuery.getJSON(requestUrl));
	  });
	  
这样我们就创建了一个名为"metaStream"的野兽：一个流的流。不用惊慌，metastream就是一个流，并且它所广播的值是另一个流。你可以把它想象成是指针：每一个被广播的值是一个指向另一个流的指针。在我们的例子中，每一个请求URL被映射到一个指向包含了对应的响应Promise流的指针。

![image](https://camo.githubusercontent.com/96ffbf34242769e3ffa585594312e6da8a5ab099/68747470733a2f2f676973742e67697468756275736572636f6e74656e742e636f6d2f7374616c747a2f38363865376539626332613762386331663735342f7261772f653866643162623662643933353333636638616661653432626466313962646666393266626332632f7a726573706f6e73656d65746173747265616d2e706e67)

一个为响应创建的metaStream看起来让人疑惑，似乎并没有帮助到我们。我们只是想要一个简单的响应流，其中每一个广播的值是一个JSON对象，而不是一个JSON对象的Promise。过来和Flatmap先生问声好吧：它是一种将metaStream“平坦化”的`map()`，它通过把所有广播给“分支”流的东西广播给“主干”流来实现。Flatmap不是一种“修复”，metaStream也不是一个bug，他们都是用来处理FRP中异步响应的真正的工具。

	var responseStream = requestStream
	  .flatMap(function(requestUrl) {
	    return Rx.Observable.fromPromise(jQuery.getJSON(requestUrl));
	  });
	  
![image](https://camo.githubusercontent.com/56bb9263de95fdecdcb4fa75c7c8da63cd80f6dc/68747470733a2f2f676973742e67697468756275736572636f6e74656e742e636f6d2f7374616c747a2f38363865376539626332613762386331663735342f7261772f373436613565313733323833363862636261356462643339376238346665383037396565663764642f7a726573706f6e736573747265616d2e706e67)

好了，因为我们是根据请求流来定义响应流的，因此如果我们之后在请求流上有更多的的事件发生，那么我们会有对应的事件如预期一样在响应流上发生：

	requestStream:  --a-----b--c------------|->
	responseStream: -----A--------B-----C---|->

	(小写字母是一个请求, 大写字母是其对应的响应)
	
这样最终我们就得到了一个响应流，我们可以用它来呈现我们所接收到的数据。

	responseStream.subscribe(function(response) {
	  // render `response` to the DOM however you wish
	});
	
把所有的代码连接起来，现在我们有了：

	var requestStream = Rx.Observable.returnValue('https://api.github.com/users');
	
	var responseStream = requestStream
	  .flatMap(function(requestUrl) {
	    return Rx.Observable.fromPromise(jQuery.getJSON(requestUrl));
	  });
	
	responseStream.subscribe(function(response) {
	  // render `response` to the DOM however you wish
	});
	
#刷新按钮

我还没有提交，返回的JSON是一个100用户的列表。API只允许我们指定页的偏移量，不允许指定页的大小，因此我们仅仅使用了3个数据而浪费了其他97个。我们现在暂时可以忽略这个问题，因为之后我们会看到我们是如何缓存响应的。

每次刷新按钮被点击，请求流应该广播一个新的URL，以便于我们得到一个新的响应。我们需要两个东西：一个单击事件的流（任何东西都可以成为流），并且我们需要根据刷新点击流来改变请求流。好在RxJS自带了从事件监听器构造可观测对象的工具。

	var refreshButton = document.querySelector('.refresh');
	var refreshClickStream = Rx.Observable.fromEvent(refreshButton, 'click');
	
既然刷新点击事件自身并不包含任何API URL，我们需要把每一个点击映射到一个实际的URL。现在我们将请求流改变为映射到API端和随机偏移量参数的刷新点击流。

	var requestStream = refreshClickStream
	  .map(function() {
	    var randomOffset = Math.floor(Math.random()*500);
	    return 'https://api.github.com/users?since=' + randomOffset;
	  });
	  
因为我是如此的愚蠢，我没有任何自动化的测试，我刚刚弄坏了我们之前构造的功能。现在在加载时不会有请求发出了，它仅仅在刷新按钮单击时才会发出。呃！我需要这两个行为：请求会在刷新单击或者加载时被发出。

我们知道如何为我们两个功能构造不同的流：

	var requestOnRefreshStream = refreshClickStream
	  .map(function() {
	    var randomOffset = Math.floor(Math.random()*500);
	    return 'https://api.github.com/users?since=' + randomOffset;
	  });
	
	var startupRequestStream = Rx.Observable.returnValue('https://api.github.com/users');
	
但是现在我们如何才能把它们合二为一呢？嗯，有`merge()`这个函数。用图来解释，它是这样工作的：

	stream A: ---a--------e-----o----->
	stream B: -----B---C-----D-------->
	          vvvvvvvvv merge vvvvvvvvv
	          ---a-B---C--e--D--o----->
	          
现在它应该变得简单了：

	var requestOnRefreshStream = refreshClickStream
	  .map(function() {
	    var randomOffset = Math.floor(Math.random()*500);
	    return 'https://api.github.com/users?since=' + randomOffset;
	  });
	
	var startupRequestStream = Rx.Observable.returnValue('https://api.github.com/users');
	
	var requestStream = Rx.Observable.merge(
	  requestOnRefreshStream, startupRequestStream
	);
	
这里有一个更干净的方式，不需要中间流：

	var requestStream = refreshClickStream
	  .map(function() {
	    var randomOffset = Math.floor(Math.random()*500);
	    return 'https://api.github.com/users?since=' + randomOffset;
	  })
	  .merge(Rx.Observable.returnValue('https://api.github.com/users'));
	  
更短一些，可读性更高一些：

	var requestStream = refreshClickStream
	  .map(function() {
	    var randomOffset = Math.floor(Math.random()*500);
	    return 'https://api.github.com/users?since=' + randomOffset;
	  })
	  .startWith('https://api.github.com/users');
	  
`startWith()`方法按你所想象的那样工作。无论你的输入流看起来什么样，`startWith(x)`的输出流会在开始包含`x`，但是我并没有足够的DRY（Don't Repeat Yourself），我重复了API的字符串。一种修复它的方式是将`startWith()`移动到`refreshClickStream`的附近，其本质就是在加载时“模拟”一次刷新操作。

	var requestStream = refreshClickStream.startWith('startup click')
	  .map(function() {
	    var randomOffset = Math.floor(Math.random()*500);
	    return 'https://api.github.com/users?since=' + randomOffset;
	  });

好的，如果你现在返回我们之前“弄坏自动化测试”的地方，你应该看到两者唯一的区别在于我添加了`startWith()`。

#用流为3个关注推荐建模
直到现在，我们仅仅在发生responseStream的`subscribe()`的呈现这一步触及到推荐UI元素。现在对于刷新按钮，我们有一个问题：只要你点击了“刷新”，当前的3个推荐并没有被清除。新的推荐仅仅在响应到达之后才能被获取，但是为了让UI看起来能好一些，我们需要在刷新按钮单击时清除当前的推荐。

	refreshClickStream.subscribe(function() {
	  // clear the 3 suggestion DOM elements 
	});
	
不，别那么快，伙计。这样是不好的，因为饿哦们现在有两个能影响推荐DOM元素的订阅者（另一个是`responseStream.subscribe()`），并且听起来这并没有真正做到关注点隔离。还记得FRP的咒语吗？

![image](https://camo.githubusercontent.com/348b1eb60fa6c0d44d7f091240f0ff462b539619/68747470733a2f2f676973742e67697468756275736572636f6e74656e742e636f6d2f7374616c747a2f38363865376539626332613762386331663735342f7261772f373936626539623636666637636535386239306536356134396533623938333262383632646564642f7a6d616e7472612e6a7067)

因此让我们将推荐建模成一个流，其中每一个被广播的值是一个包含推荐数据的JSON对象。我们会分别为3个推荐做这件事情。我们为推荐1所做的流看起来像是这样：

	var suggestion1Stream = responseStream
	  .map(function(listUsers) {
	    // get one random user from the list
	    return listUsers[Math.floor(Math.random()*listUsers.length)];
	  });

其他的`suggestion2Stream`和`suggestion3Stream`可以简单的从`suggestion1Stream`拷贝过来。这违反了DRY原则，但是这能够让我们的教程示例更简单。而且我任何思考如何避免这种情况是一个很好的锻炼。

不像原来那样在responseStream的`subscribe()`中进行呈现，我们在这里做：

	suggestion1Stream.subscribe(function(suggestion) {
	  // render the 1st suggestion to the DOM
	});
	
回到“刷新时，清空推荐”，我们可以将刷新点击映射到空的推荐数据，并在`suggestion1Stream`包含它，像这样：


	var suggestion1Stream = responseStream
	  .map(function(listUsers) {
	    // get one random user from the list
	    return listUsers[Math.floor(Math.random()*listUsers.length)];
	  })
	  .merge(
	    refreshClickStream.map(function(){ return null; })
	  );
	  
当我们呈现时，我们将空解释为“没有数据”，这样来隐藏其UI元素。

	suggestion1Stream.subscribe(function(suggestion) {
	  if (suggestion === null) {
	    // hide the first suggestion DOM element
	  }
	  else {
	    // show the first suggestion DOM element
	    // and render the data
	  }
	});
	
现在蓝图是这样：

	refreshClickStream: ----------o--------o---->
	     requestStream: -r--------r--------r---->
	    responseStream: ----R---------R------R-->   
	 suggestion1Stream: ----s-----N---s----N-s-->
	 suggestion2Stream: ----q-----N---q----N-q-->
	 suggestion3Stream: ----t-----N---t----N-t-->
	 
其中`N`是`null`的意思

为了做得更好，我们也可以在启动时呈现“空”的推荐。这是通过为推荐流添加`startWith(null)`来做到的：

	var suggestion1Stream = responseStream
	  .map(function(listUsers) {
	    // get one random user from the list
	    return listUsers[Math.floor(Math.random()*listUsers.length)];
	  })
	  .merge(
	    refreshClickStream.map(function(){ return null; })
	  )
	  .startWith(null);
	  
其结果是：

	refreshClickStream: ----------o---------o---->
	     requestStream: -r--------r---------r---->
	    responseStream: ----R----------R------R-->   
	 suggestion1Stream: -N--s-----N----s----N-s-->
	 suggestion2Stream: -N--q-----N----q----N-q-->
	 suggestion3Stream: -N--t-----N----t----N-t-->
	 
#关闭推荐并使用被缓存的响应
我们还有一个功能需要实现：每一个推荐都应该有一个`x`按钮来关闭它，并在原地加载另一个推荐。乍一想，你可能想说，每个按钮被点击时发送一个新的请求没什么问题：

	var close1Button = document.querySelector('.close1');
	var close1ClickStream = Rx.Observable.fromEvent(close1Button, 'click');
	// and the same for close2Button and close3Button
	
	var requestStream = refreshClickStream.startWith('startup click')
	  .merge(close1ClickStream) // we added this
	  .map(function() {
	    var randomOffset = Math.floor(Math.random()*500);
	    return 'https://api.github.com/users?since=' + randomOffset;
	  });

这没办法工作。它会关闭并重新加载所有的推荐，而不只是我们点击的那一个。我们有非常多的方式解决这个问题，为了保持这件事情很有趣，我们通过重用之前的响应来解决这个问题。API的页大小为100而我们之前仅仅使用了3个，因此我们还有非常多的新鲜数据可以使用，而不需要更多的请求。

再一次的，让我们用流来思考。当一个`close1`点击事件发生，我们希望使用最近一次被`responseStream`广播的响应来从响应列表中得到一个随机的用户。像这样：

	    requestStream: --r--------------->
	   responseStream: ------R----------->
	close1ClickStream: ------------c----->
	suggestion1Stream: ------s-----s----->
	
在Rx*中有一个连接符函数被称为`combineLatest`似乎能解决我们的需求。它包含两个流A和B作为输入，当其中任何一个流广播一个值，`combineLatest`把两个流中最近被广播的值连接起来，并输出一个结果`c = f(x,y)`，其中f是一个你定义的函数。用图表来看更容易解释：

	stream A: --a-----------e--------i-------->
	stream B: -----b----c--------d-------q---->
	          vvvvvvvv combineLatest(f) vvvvvvv
	          ----AB---AC--EC---ED--ID--IQ---->
	
	f是大写函数
	
我们可以在`close1ClickStream`和`responseStream`上使用combineLatest()，因此每当第一个关闭按钮被点击时，我们能广播最近一次的响应并为`suggestion1Stream`创建一个新的值。另一方面，combineLatest()是对称的：每当一个新的响应被广播到`responseStream`时，它会结合最近一次的“close 1”点击事件来创建一个新的推荐。这非常有趣，因为它允许我们简化之前`suggestion1Stream`的代码，如下：

	var suggestion1Stream = close1ClickStream
	  .combineLatest(responseStream,             
	    function(click, listUsers) {
	      return listUsers[Math.floor(Math.random()*listUsers.length)];
	    }
	  )
	  .merge(
	    refreshClickStream.map(function(){ return null; })
	  )
	  .startWith(null);
	  
还有一个未解的难题。combineLatest()使用最近的两个来源，但是如果其中一个源还未广播任何值，那么combineLatest()不能向输出流创建数据事件。如果你看看上面的ASCII表格，你会发现当第一个流广播值a的时候，我们没有任何输出。直到第二个流广播了一个值b之后我们才得到第一个输出值。

解决之道有很多，我们使用最简单的。即，在开始时模拟一次对"close 1"的点击：

	var suggestion1Stream = close1ClickStream.startWith('startup click') // we added this
	  .combineLatest(responseStream,             
	    function(click, listUsers) {l
	      return listUsers[Math.floor(Math.random()*listUsers.length)];
	    }
	  )
	  .merge(
	    refreshClickStream.map(function(){ return null; })
	  )
	  .startWith(null);

#总结

你得到的所有代码如下：

	var refreshButton = document.querySelector('.refresh');
	var refreshClickStream = Rx.Observable.fromEvent(refreshButton, 'click');
	
	var closeButton1 = document.querySelector('.close1');
	var close1ClickStream = Rx.Observable.fromEvent(closeButton1, 'click');
	// and the same logic for close2 and close3
	
	var requestStream = refreshClickStream.startWith('startup click')
	  .map(function() {
	    var randomOffset = Math.floor(Math.random()*500);
	    return 'https://api.github.com/users?since=' + randomOffset;
	  });
	
	var responseStream = requestStream
	  .flatMap(function (requestUrl) {
	    return Rx.Observable.fromPromise($.ajax({url: requestUrl}));
	  });
	
	var suggestion1Stream = close1ClickStream.startWith('startup click')
	  .combineLatest(responseStream,             
	    function(click, listUsers) {
      return listUsers[Math.floor(Math.random()*listUsers.length)];
    }
	  )
	  .merge(
	    refreshClickStream.map(function(){ return null; })
	  )
	  .startWith(null);
	// and the same logic for suggestion2Stream and suggestion3Stream
	
	suggestion1Stream.subscribe(function(suggestion) {
	  if (suggestion === null) {
	    // hide the first suggestion DOM element
	  }
	  else {
	    // show the first suggestion DOM element
	    // and render the data	
	  }
	});
	
<b>你可以在这里找到这个工作示例：http://jsfiddle.net/staltz/8jFJH/48/</b>

这段代码虽短但是很密集：它包括通过适当分析关注点来管理多种事件，甚至还包括了响应的缓存。函数式风格使代码看起来更像陈述而非命令。我们并不是给定一串指令去执行，我们只是通过定义流之间的关系来完成。比如，我们使用FRP告诉程序`suggestion1Stream`是`close 1`流与最近一次响应用户的结合，同时，当刷新或者程序开始发生时将其置为`null`。

同时，还要注意，代码中包含极少量的逻辑控制语句例如`if`, `for`, `while`，同时也没有JavaScript应用中常见的回调风格的控制流，这非常令人印象深刻。如果你想的话，你甚至可以在上述`subscribe()`中完全移除`if`和`else`而使用`filter()`（我将把实现细节留给你做练习）。在FRP中，我们有与流相关的函数例如`map`, `filter`, `scan`, `merge`, `combineLatest`, `startWith`，以及更多的控制事件驱动程序流程的函数。这个函数工具集允许你完成更多功能而使用更少的代码。

#接下来的事情

如果你认为Rx*会成为你FRP的首选库，你需要花点时间来了解非常长的函数列表函数列表包括：变形、合并以及创建Observable。如果你想要通过流的图表来理解这些函数，[看一看RxJava的非常有用的文档](https://github.com/Netflix/RxJava/wiki/Creating-Observables)。无论什么时候你陷入困境，试着画出这些图，思考他们，看看这一长串的函数列表，然后更多的思考。以我的经验来看这个工作方法非常有效。

一旦你开始使用Rx\*开始编程，你绝对需要理解[Cold vs Hot Observables](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/gettingstarted/creating.md#cold-vs-hot-observables)的内容。如果你忽略了它，它早晚会回来痛咬你的。我已经警告过你了。通过学习函数式编程进一步提高你的技巧，并熟悉影响Rx\*的副作用。

然而函数响应式编程并不仅仅是Rx\*。你可以使用Bacon.js来直观的使用它，而不需要理会那些在使用Rx\*时会遇到的奇怪的问题。Elm语言，则是自成一派：它是一个可以编译成JavaScript + HTML + CSS的FRP语言，还有一个时间浏览的调试器。它非常棒。

FRP非常适合重事件的前端程序或应用程序。但是这并不仅仅是客户端的事情。它在后端以及贴近数据库的场景也能有很好的发挥。事实上，RxJava在Netflix的API中是一个允许服务器端进行并发执行的重要组件。FRP不是一个局限于特定类型的应用程序或者语言的框架。它是一个你在编写任何事件驱动软件时都可以使用的范例。

如果这篇文章帮到了你，记得[来Twitter转发](https://twitter.com/intent/tweet?original_referer=https%3A%2F%2Fgist.github.com%2Fstaltz%2F868e7e9bc2a7b8c1f754%2F&amp;text=The%20introduction%20to%20Reactive%20Programming%20you%27ve%20been%20missing&amp;tw_p=tweetbutton&amp;url=https%3A%2F%2Fgist.github.com%2Fstaltz%2F868e7e9bc2a7b8c1f754&amp;via=andrestaltz)~