title: 用Stream解决编程挑战（译）
date: 2014-10-22 23:32:11
tags:
---
#用Stream解决编程挑战（译）

我第一次使用Node.js解决编程挑战的经验简直是让人焦虑不已。我设计了一个切实可行的解决方法，但我却无法找到一个有效的方法来解析输入。它的格式非常的简单：文本通过stdin的pipe输入。很简单，是吧？但我一般的时间都耗费在这个小小的细节上，最终我得到了一个非常脆弱的、缝缝补补的、处处hack过的代码，这至今仍然让我感到不寒而栗。

这份经验激励着我找到一种管用的方式来完成编程挑战。在解决了更多问题之后我终于找到了一种我希望大家都能觉得又用的方式。

##模式

主要的思想是：创造一个problem的stream，将每一个problem转化为一个solition。这个流程由4个步骤组成：

1. 将输入打散为行的stream。
2. 将这些行转化为和问题相关的数据结构
3. 解决问题
4. 格式化solution并输出

对于那些熟悉stream的人来说，这个模式看起来像这样：

	var split = require("split"); // dominictarr’s helpful line-splitting module

	process.stdin
	    .pipe(split()) // split input into lines
	    .pipe(new ProblemStream()) // transform lines into problem data structures
	    .pipe(new SolutionStream()) // solve each problem
	    .pipe(new FormatStream()) // format the solutions for output
	    .pipe(process.stdout); // write solution to stdout
	    
##我们的问题

为了让这个教程更接地气一点，让我们来解决一个[Google Code Jam challenge](https://code.google.com/codejam/contest/2929486/dashboard)。这个问题是让我们验证数独游戏的解答。输入看起来像这样：

	2                  // number of puzzles to verify
	3                  // dimensions of first puzzle (3 * 3 = 9)
	7 6 5 1 9 8 4 3 2  // first puzzle
	8 1 9 2 4 3 5 7 6
	3 2 4 6 5 7 9 8 1
	1 9 8 4 3 2 7 6 5
	2 4 3 5 7 6 8 1 9
	6 5 7 9 8 1 3 2 4
	4 3 2 7 6 5 1 9 8
	5 7 6 8 1 9 2 4 3
	9 8 1 3 2 4 6 5 7
	3                  // dimensions of second puzzle
	7 9 5 1 3 8 4 6 2  // second puzzle
	2 1 3 5 4 6 8 7 9
	6 8 4 9 2 7 4 5 1
	1 3 8 4 6 2 7 9 5
	5 4 6 8 7 9 2 1 3
	9 2 7 3 5 1 6 8 4
	4 6 2 7 9 5 1 3 8
	8 7 9 2 1 3 5 4 6
	3 5 1 6 8 4 9 2 7
	
我们输出的格式应该是：

	Case #1: Yes
	Case #2: No
	
其中“Yes”表明解答是正确的。

让我们开始吧。

##建立

我们的第一步就是要从stdin中提取输入。在Node中，stdin是一个可读的stream。基本上，一个可读stream会在数据可读后立刻发送数据（更多的解释，参见[readable stream docs](http://nodejs.org/api/stream.html#stream_class_stream_readable)）。下面这行代码会输出所有输入到stdin中的内容：

	process.stdin.pipe(process.stdout);
	
`pipe`方法从可读stream中获取所有的数据并写入一个可写stream。

可能从这份代码中并不显而易见，但是`process.stdin`会以大块byte的形式pipe数据，而我们感兴趣的是以行为分隔的文本。为了将这种大块数据分解成行，我们可以将`process.stdin` pipe进入dominictarr所写的`split`模块中。首先`npm install split`，然后：

	var split = require("split");
	
	process.stdin.setEncoding("utf8"); // convert bytes to utf8 characters
	
	process.stdin
	     .pipe(split())
	     .pipe(process.stdout);
	     
##使用transform stream构造问题

现在我们有了由行组成的序列，我们可以开始进行我们真正的工作了。我们会将这些行转化为一串代表数独问题的二维数组中。然后，我们pipe每个数独问题到另一个流并用它来检验它是否是一个正确的解答。

Node的原生transform stream提供了我们所需要的抽象。一个transform stream将写入它的数据进行转化，并将结果以一个可读stream的方式输出。有点疑惑？我们下面会让你清楚一些。

为了创建一个transform stream，我们要继承stream.Transform并调用它的构造函数。

	var Transform = require("stream").Transform;
	var util = require("util");
	
	util.inherits(ProblemStream, Transform); // inherit Transform
	
	function ProblemStream () {
	    Transform.call(this, { "objectMode": true }); // invoke Transform's constructor
	}
	
你会注意到，我们传递了`objectMode`的flag到`Transform`的构造函数中。原始的Stream上只接受string和buffer。而我们希望输出一个二维数组，所以我们需要打开object模式。

Transform stream有两个重要的方法：`_transform`和`_flush`。`_transform`在每当有数据写入stream时被调用。我们使用这个方法来将一系列的行转化为一个数组解答。`_flush`将在transform stream被通知没有更多的数据会被写入时被调用。这个函数有助于我们结束任何尚未结束的任务。

让我们草拟我们的transform函数：

	ProblemStream.prototype._transform = function (line, encoding, processed) {
	     // TODO
	}
	
`_transform`接受3个参数。第一个是写入stream的数据。在我们这个情况下，就是一行文本。第二个参数是stream编码，在此我们设为utf8。最后一个参数是一个无参的回调函数用来提供已经结束输入处理的信号。

当你在实现`_transform`函数的时候要牢记两点：

1. 调用`processed`回调函数并不向output stream中添加任何内容。它仅仅是一个信号，标志着我们已经完成了传递给`_transform`的内容的处理
2. 如果要输出一个值，使用`this.push(value)`

记住这些，让我们再来看看输入。

	2
	3
	7 6 5 1 9 8 4 3 2
	8 1 9 2 4 3 5 7 6
	3 2 4 6 5 7 9 8 1
	1 9 8 4 3 2 7 6 5
	2 4 3 5 7 6 8 1 9
	6 5 7 9 8 1 3 2 4
	4 3 2 7 6 5 1 9 8
	5 7 6 8 1 9 2 4 3
	9 8 1 3 2 4 6 5 7
	3
	7 9 5 1 3 8 4 6 2
	2 1 3 5 4 6 8 7 9
	6 8 4 9 2 7 4 5 1
	1 3 8 4 6 2 7 9 5
	5 4 6 8 7 9 2 1 3
	9 2 7 3 5 1 6 8 4
	4 6 2 7 9 5 1 3 8
	8 7 9 2 1 3 5 4 6
	3 5 1 6 8 4 9 2 7
	
我们马上就遇到了一个问题：我们的`_transform`方法每行被调用一次，但是前面三行每一行都代表不同的意义。第一行描述了要解决多少个问题，第二行是接下来的解答由几行组成，第三行是解答内容。我们的stream需要用不同的方式处理每一行。

幸运的是，我们可以将状态保存在transform stream内部：

	var Transform = require("stream").Transform;
	var util = require("util");
	
	util.inherits(ProblemStream, Transform);
	
	function ProblemStream () {
	    Transform.call(this, { "objectMode": true });
	
	    this.numProblemsToSolve = null;
	    this.puzzleSize = null;
	    this.currentPuzzle = null;
	}
	
通过这些变量，我们就可以追踪到我们正处在行序列的何处。

	ProblemStream.prototype._transform = function (line, encoding, processed) {
	    if (this.numProblemsToSolve === null) { // handle first line
	        this.numProblemsToSolve = +line;
	    }
	    else if (this.puzzleSize === null) { // start a new puzzle
	        this.puzzleSize = (+line) * (+line); // a size of 3 means the puzzle will be 9 lines long
	        this.currentPuzzle = [];
	    }
	    else {
	        var numbers = line.match(/\d+/g); // break line into an array of numbers
	        this.currentPuzzle.push(numbers); // add a new row to the puzzle
	        this.puzzleSize--; // decrement number of remaining lines to parse for puzzle
	
	        if (this.puzzleSize === 0) {
	            this.push(this.currentPuzzle); // we've parsed the full puzzle; add it to the output stream
	            this.puzzleSize = null; // reset; ready for next puzzle
	        }
	    }
	    processed(); // we're done processing the current line
	};
	
	process.stdin
	    .pipe(split())
	    .pipe(new ProblemStream())
	    .pipe(new SolutionStream()) // TODO
	    .pipe(new FormatStream()) // TODO
	    .pipe(process.stdout); 
	    
让我们花点时间来回顾一下代码。记住`_transform`会为每行所调用。第一行_transform接收到对应的需要解决问题的数目。由于`numProblemsToSolve`是null，所以这个逻辑分支会被执行。被传递到`_transform`的第二行是解答的尺寸。由于我们已经知道解答的尺寸，第三行是构造数据结构的开始。一旦解答被构造，我们会将一个完整的解答推送到transform stream的输出端，然后准备创建一个新的解答。循环此过程直到我们读完所有行。

##解决所有的问题吧！

解析完并构造出数独解答的数据结构之后，我们终于可以开始解答这个问题了。

“解答问题”的任务，可以被解释为“将一个问题转化为一个解答”。这就是我们的下一个stream所要做的。

	util.inherits(SolutionStream, Transform);
	
	function SolutionStream () {
	    Transform.call(this, { "objectMode": true });
	}
	
然后，我们定义一个`_transform`方法，它接受一个problem参数，并返回一个布尔值。

	SolutionStream.prototype._transform = function (problem, encoding, processed) {
	    var solution = solve(problem);
	    this.push(solution);
	    processed();
	
	    function solve (problem) {
	        // TODO
	        return false;
	    }
	};
	
	process.stdin
	    .pipe(split())
	    .pipe(new ProblemStream())
	    .pipe(new SolutionStream())
	    .pipe(new FormatStream()) // TODO
	    .pipe(process.stdout);
	    
不像`ProblemStream`一样，这个stream会为每一个输入构造一个输出，`_transform`会为每个问题执行一次，我们需要解决所有的问题。

我们所有要做的就是写一个函数来决定是否数独问题被解决了，我把这个留给你自己来解答。

## 修饰输出

现在我们解决了这个问题，我们的最后一步是格式化输出。如你所料，我们又将使用一个transform stream。

我们的`FormatStream`接受一个解答并转化为一个字符串传递到`process.stdout`中。

还记得输出格式吗？

	Case #1: Yes
	Case #2: No
	
我们需要纪录问题号，并将布尔值转化为"Yes"或者"No"。

	util.inherits(FormatStream, Transform);
	function FormatStream () {
	    Transform.call(this, { "objectMode": true });
	
	    this.caseNumber = 0;
	}
	
	FormatStream.prototype._transform = function (solution, encoding, processed) {
	    this.caseNumber++;
	
	    var result = solution ? "Yes" : "No";
	
	    var formatted = "Case #" + this.caseNumber + ": " + result + "\n";
	
	    this.push(formatted);
	    processed();
	};
	
现在，将`FormatStream`连接到我们的pipeline中，我们就完成了

	process.stdin
	    .pipe(split())
	    .pipe(new ProblemStream())
	    .pipe(new SolutionStream())
	    .pipe(new FormatStream())
	    .pipe(process.stdout);
	    
[从Github上获取完整代码。](https://github.com/nimbus154/node-coding-challenge-pattern)

## 最后一条提示

使用`pipe`的最大好处是你可以在任何可读/可写流中重用你的代码。如果你需要解答一个来自网络的问题，将`process.stdin`和`process.stdout`改为网络stream，所有的一切应该可以直接使用。

对于每一个问题你可以需要做相应的微调，但我希望它给出了一个好的开始。

