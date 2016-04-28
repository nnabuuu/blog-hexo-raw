title: '神经网络骇客指南（翻译中）'
date: 2014-11-16 13:51:07
tags: translation
---
原文见：http://karpathy.github.io/neuralnets/
##神经网络骇客指南（译）

各位好，我是一名[斯坦福的计算机科学的博士](http://cs.stanford.edu/people/karpathy/)。作为我研究的一部分，我已经在深度学习上研究了好几年，我有几个“pet project”，其中一个是[ConvNetJS](http://convnetjs.com/) - 一个用来训练神经网络的Javascript库。Javascript允许一个人轻松地将现在所发生的事情可视化，并且可以实现多样的参数选择设置，但是我仍然经常听到人们想要一些更加彻底的话题。这篇文章（我打算慢慢地写到几个章节那么长）是我一份谦逊地尝试。我把它放在网上而不是以一个PDF文件的形式呈现是因为，所有的图书都应该这样，并且最终希望它能包括一些动画和演示。

我对神经网络的个人经验是：当我抛开一切整篇、密集的反向传播方程的推导，而仅仅开始写代码时，一切都清晰多了。因此这个教程会包含**非常少的数学**(我不认为这是有必要的，而且有些时候会混淆一些简单的概念)。由于我的背景是计算机科学以及物理，我会以**骇客的角度**来看待问题。我会围绕着代码以及物理直觉而不是数学推导来展示。基本上我会以一种“我刚开始学习时希望被那样教导”的方式努力地呈现算法。

	“...当我开始编写代码时一切都清晰多了。”
	
你可能会想急切地跳进去学习神经网络、反向传播、它们如何能应用于数据集上、等等。但是在我们到达那里之前，我想先让我们忘掉这一切。让我们后退一步，明白什么是真正的核心。让我们从元电路开始谈起。

###第一章：元电路

在我看来，思考神经网络的最佳方式是将其比作元电路。在这里，实际的值（而不是布尔值`{0,1}`）沿着边沿“流动”并且在门出交汇。但是不同于门电路的`与`、`或`、`非`等，我们的二进制门包含例如`*`（乘）、`+`（加）、`max`或者一元门例如`exp`等等。不同于基本的布尔电路，我们最终也会有**gradients**在同样的边沿流动，但是是向相反的方向。这已经有点超前了，我们还是先专注一下，从简单的开始。

#### 基本情况：电路中的单门

让我们先考虑一个单一的、简单的、包含一个门的电路。示例如下：

![simple circuit with one gate](/img/Hacker-s-guide-to-Neural-Networks/1.png)

这个电路接受两个实际值`x`和`y`并且在`*`门中计算`x * y`。 Javascript的版本会非常简单，看起来像这样：
```js
var forwardMultiplyGate = function(x, y) {
  return x * y;
};
forwardMultiplyGate(-2, 3); // returns -6. Exciting.
```	
以数学的形式我们可以认为这个门实现了函数：

	f(x, y) = x * y

在这个例子中，我们所有的门都会接受一个或两个输入，并产生一个**单一**的输出值。

##### 目标

我们在学习时所感兴趣的问题看起来像下面这样：

1.	我们为一个已知电路提供一些具体的输入（例如 `x = -2`，`y = 3`）
2.	电路计算出一个输出值（例如 `-6`）
3.	那么问题的核心变为：我们如何轻微的改变输入以便增加输出？

在这个例子中，我们应该向什么方向改变`x,y`以便得到一个比`-6`更大的数字呢？注意到，例如`x = -1.99`以及`y = 2.99`时`x * y = -5.95` 这是一个比`-6.0`更大的数。别被它搞晕了：`-5.95`是比`-6.0`更大的。这个增量为`0.05`，尽管`-5.95`的大小（到0的距离）更小一些。

##### 策略#1：本地随机搜索

好，等一下，现在我们有一个电路，我们有一些输入并且我们希望轻微地改变它们以便增加输出？为什么这个很难？我们可以简单的“转发”电路来计算对于任何给定的`x`和`y`的输出，所以这不是很简单吗？我们为什么不随机调整`x`和`y`来跟踪效果最好的策略呢：
```js
// circuit with single gate for now
var forwardMultiplyGate = function(x, y) { return x * y; };
var x = -2, y = 3; // some input values
	
// try changing x,y randomly small amounts and keep track of what works best
var tweak_amount = 0.01;
var best_out = -Infinity;
var best_x = x, best_y = y;
for(var k = 0; k < 100; k++) {
  var x_try = x + tweak_amount * (Math.random() * 2 - 1); // tweak x a bit
  var y_try = y + tweak_amount * (Math.random() * 2 - 1); // tweak y a bit
  var out = forwardMultiplyGate(x_try, y_try);
  if(out > best_out) {
    // best improvement yet! Keep track of the x and y
    best_out = out; 
    best_x = x_try, best_y = y_try;
  }
}
```
当我执行它时，我的到了`best_x = -1.9928`，`best_y = 2.9901`，以及`best_out = -5.9588`。因为`-5.9588`比`-6.0`更大，所以我们就搞定了，是吗？并不是这样：如果你能负担得起时间的话，它对于仅包含几个门的微小的问题来说是一个完美的策略。但是对于接受上百万输入的大量的电路来说，并非如此。结果是，我们可以做的更好。

##### 策略#2：数值梯度

这里就有一个更好的方法。再次记住，在我们前期我们被提供了一个电路（例如，我们的电路是一个单一的`*`门）和一些特殊的输入（例如`x = -2, y = 3`）。门会计算出结果(`-6`)，而现在我们希望对`x`和`y`进行微调以便得到更高的输出。

对于我们接下来要做的事情，一个很棒的直觉是这样：想象一下获取来自于门电路的输出值输出值，并且对其正向加压。正向电压会反过来通过们进行传输并且引起对输入`x`和`y`的推动。这个推动就告诉了我们应该如何改变`x`和`y`以便增加输出值。

在我们这个特定的例子中，这个推动力可能是什么样子的呢？考虑一下，我们可以凭直觉知道施加在`x`上的力应该是正向的，因为使`x`轻微地增大会增加电路的输出。例如，将`x`从`x = -2`增加到`x = -1`会使我们得到`-3` - 远大于`-6`。在另一方面，我们会希望在`y`上施加负向的力，而使它变得更小(因为更小的`y`，例如从`y = 3`降到`y = 2`会使我们的结果更高：`2 * -2 = -4`，同样比`-6`更大)。但这毕竟是我们脑中的直觉。随着我们的深入，我们会了解到，我这里提到的牵引力实际上是基于输入值(`x`和`y`)的**导数**输出值。你可以已经听说过这些：

	导数可以被认为是一种施加于各个输入值的力，用于使输出变得更高。
	
所以我们如何来精确地评价这个牵引力（导数）呢？实际上有一个非常简单的方法。我们反向地来操作：不同于增加电路的输出值，我们一个接一个地迭代每个输入值，轻微地增加它并检测输出值如何改变。输出的改变就是导数。尽管我们到现在为止还是凭借直觉。我们还是看一下数学定义。我们可以写出函数关于某个输入的导数，例如对`x`的导数可以写成：

<div>
<span class="MathJax_Preview"></span><div class="MathJax_Display" role="textbox" aria-readonly="true" style="text-align: center;"><span class="MathJax" id="MathJax-Element-2-Frame"><nobr><span class="math" id="MathJax-Span-12" style="width: 15.419em; display: inline-block;"><span style="display: inline-block; position: relative; width: 12.815em; height: 0px; font-size: 120%;"><span style="position: absolute; clip: rect(0.992em 1000.003em 3.336em -0.466em); top: -2.497em; left: 0.003em;"><span class="mrow" id="MathJax-Span-13"><span class="mfrac" id="MathJax-Span-14"><span style="display: inline-block; position: relative; width: 3.076em; height: 0px; margin-right: 0.107em; margin-left: 0.107em;"><span style="position: absolute; clip: rect(1.669em 1000.003em 2.867em -0.414em); top: -3.174em; left: 50%; margin-left: -1.456em;"><span class="mrow" id="MathJax-Span-15"><span class="mi" id="MathJax-Span-16" style="font-family: STIXGeneral-Regular;">∂</span><span class="mi" id="MathJax-Span-17" style="font-family: STIXGeneral-Italic;">f<span style="display: inline-block; overflow: hidden; height: 1px; width: 0.159em;"></span></span><span class="mo" id="MathJax-Span-18" style="font-family: STIXGeneral-Regular;">(</span><span class="mi" id="MathJax-Span-19" style="font-family: STIXGeneral-Italic;">x<span style="display: inline-block; overflow: hidden; height: 1px; width: 0.003em;"></span></span><span class="mo" id="MathJax-Span-20" style="font-family: STIXGeneral-Regular;">,</span><span class="mi" id="MathJax-Span-21" style="font-family: STIXGeneral-Italic; padding-left: 0.211em;">y</span><span class="mo" id="MathJax-Span-22" style="font-family: STIXGeneral-Regular;">)</span></span><span style="display: inline-block; width: 0px; height: 2.503em;"></span></span><span style="position: absolute; clip: rect(1.669em 1000.003em 2.659em -0.414em); top: -1.82em; left: 50%; margin-left: -0.466em;"><span class="mrow" id="MathJax-Span-23"><span class="mi" id="MathJax-Span-24" style="font-family: STIXGeneral-Regular;">∂</span><span class="mi" id="MathJax-Span-25" style="font-family: STIXGeneral-Italic;">x<span style="display: inline-block; overflow: hidden; height: 1px; width: 0.003em;"></span></span></span><span style="display: inline-block; width: 0px; height: 2.503em;"></span></span><span style="position: absolute; clip: rect(0.836em 1000.003em 1.201em -0.466em); top: -1.247em; left: 0.003em;"><span style="border-left-width: 3.076em; border-left-style: solid; display: inline-block; overflow: hidden; width: 0px; height: 1.25px; vertical-align: 0.003em;"></span><span style="display: inline-block; width: 0px; height: 1.044em;"></span></span></span></span><span class="mo" id="MathJax-Span-26" style="font-family: STIXGeneral-Regular; padding-left: 0.315em;">=</span><span class="mfrac" id="MathJax-Span-27" style="padding-left: 0.315em;"><span style="display: inline-block; position: relative; width: 7.971em; height: 0px; margin-right: 0.107em; margin-left: 0.107em;"><span style="position: absolute; clip: rect(1.669em 1000.003em 2.867em -0.622em); top: -3.174em; left: 50%; margin-left: -3.904em;"><span class="mrow" id="MathJax-Span-28"><span class="mi" id="MathJax-Span-29" style="font-family: STIXGeneral-Italic;">f<span style="display: inline-block; overflow: hidden; height: 1px; width: 0.159em;"></span></span><span class="mo" id="MathJax-Span-30" style="font-family: STIXGeneral-Regular;">(</span><span class="mi" id="MathJax-Span-31" style="font-family: STIXGeneral-Italic;">x<span style="display: inline-block; overflow: hidden; height: 1px; width: 0.003em;"></span></span><span class="mo" id="MathJax-Span-32" style="font-family: STIXGeneral-Regular; padding-left: 0.263em;">+</span><span class="mi" id="MathJax-Span-33" style="font-family: STIXGeneral-Italic; padding-left: 0.263em;">h</span><span class="mo" id="MathJax-Span-34" style="font-family: STIXGeneral-Regular;">,</span><span class="mi" id="MathJax-Span-35" style="font-family: STIXGeneral-Italic; padding-left: 0.211em;">y</span><span class="mo" id="MathJax-Span-36" style="font-family: STIXGeneral-Regular;">)</span><span class="mo" id="MathJax-Span-37" style="font-family: STIXGeneral-Regular; padding-left: 0.263em;">−</span><span class="mi" id="MathJax-Span-38" style="font-family: STIXGeneral-Italic; padding-left: 0.263em;">f<span style="display: inline-block; overflow: hidden; height: 1px; width: 0.159em;"></span></span><span class="mo" id="MathJax-Span-39" style="font-family: STIXGeneral-Regular;">(</span><span class="mi" id="MathJax-Span-40" style="font-family: STIXGeneral-Italic;">x<span style="display: inline-block; overflow: hidden; height: 1px; width: 0.003em;"></span></span><span class="mo" id="MathJax-Span-41" style="font-family: STIXGeneral-Regular;">,</span><span class="mi" id="MathJax-Span-42" style="font-family: STIXGeneral-Italic; padding-left: 0.211em;">y</span><span class="mo" id="MathJax-Span-43" style="font-family: STIXGeneral-Regular;">)</span></span><span style="display: inline-block; width: 0px; height: 2.503em;"></span></span><span style="position: absolute; clip: rect(1.669em 1000.003em 2.659em -0.466em); top: -1.82em; left: 50%; margin-left: -0.258em;"><span class="mi" id="MathJax-Span-44" style="font-family: STIXGeneral-Italic;">h</span><span style="display: inline-block; width: 0px; height: 2.503em;"></span></span><span style="position: absolute; clip: rect(0.836em 1000.003em 1.201em -0.466em); top: -1.247em; left: 0.003em;"><span style="border-left-width: 7.971em; border-left-style: solid; display: inline-block; overflow: hidden; width: 0px; height: 1.25px; vertical-align: 0.003em;"></span><span style="display: inline-block; width: 0px; height: 1.044em;"></span></span></span></span></span><span style="display: inline-block; width: 0px; height: 2.503em;"></span></span></span><span style="border-left-width: 0.003em; border-left-style: solid; display: inline-block; overflow: hidden; width: 0px; height: 2.566em; vertical-align: -0.872em;"></span></span></nobr></span></div><script type="math/tex; mode=display" id="MathJax-Element-2">
\frac{\partial f(x,y)}{\partial x} = \frac{f(x+h,y) - f(x,y)}{h}
</script>
</div>

在这里，h是非常小的，这是你改变的总量。并且，如果你不太熟悉计算的话，必须要注意的是，在等式的左边，横线**并不**表示除法。这个符号∂f(x,y)/∂x是一个整体：函数 f(x,y) 对x的编导。等式右边的横线表示除法。我知道这非常让人迷惑，但这是一个标准符号。无论怎样，我希望这并没有太吓人，因为它的确很简单：电路被赋予一些f(x,y)的初始值，然后我们将其中的一个输入改变一个非常小的量h，并且获取新的输出f(x+h,y)。将二者相减我们能得到改变量，然后除以h我们就得到了对于任意改变量的标准量。从另一方面说，下面的代码这正反应了我上面阐述：

```js
var x = -2, y = 3;
var out = forwardMultiplyGate(x, y); // -6
var h = 0.0001;

// compute derivative with respect to x
var xph = x + h; // -1.9999
var out2 = forwardMultiplyGate(xph, y); // -5.9997
var x_derivative = (out2 - out) / h; // 3.0

// compute derivative with respect to y
var yph = y + h; // 3.0001
var out3 = forwardMultiplyGate(x, yph); // -6.0002
var y_derivative = (out3 - out) / h; // -2.0
```
让我们对`x`进行研究，我们将输入从`x`变为`x + h`，然后电路反馈
