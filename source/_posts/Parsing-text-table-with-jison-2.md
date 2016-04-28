---
title: Parsing-text-table-with-jison-2
date: 2016-04-28 00:11:16
tags: [NodeJS, jison]
---
# 用jison解析一个文本表格2

在上一篇文章中，我们完成了一个最初版本的的Parser，它可以用来解析一个基本的表格文本。接下来，我们将以此为基础逐步完善更多内容。

### 将表格内容存入对象 Store cells in an object instead of array

我们知道在bison中，使用者可以通过yylval保存全局变量。类似的，在jison中，我们可以通过全局变量yy来储存。
见[官网描述：http://zaa.ch/jison/docs/#sharing-scope](http://zaa.ch/jison/docs/#sharing-scope)

那么，首先我们好奇的是，这个变量yy到底包含哪些内容。我们在执行时将它输出到控制台一探究竟：

```
{ lexer:
  {
    yy: [Circular],
    _input: '',
    done: true,
    _backtrack: false,
    _more: false,
    yyleng: 0,
    yylineno: 16,
    match: '',
    matched: '+---------------------------------+---------+----------------+---------------+\r\n| VM | State | VM Type | IP |\r\n+---------------------------------+---------+----------------+---------------+\r\n| a/0 | running | a | 192.168.1.159 |\r\n| b/0 | running | b | 192.168.1.161 |\r\n| b/1 | running | b | 192.168.1.162 |\r\n| b/2 | running | b | 192.168.1.163 |\r\n| eeee/0 | running | iiiiiiiiiiiiii | 192.168.1.164 |\r\n| haaaaaaa/0 | running | iiiiiiiiiiiiii | 192.168.1.168 |\r\n| c/0 | running | iiiiiiiiiiiiii | 192.168.1.166 |\r\n| lllllllllloooooooooolllllller/0 | running | iiiiiiiiiiiiii | 192.168.1.167 |\r\n| n1/0 | running | iiiiiiiiiiiiii | 192.168.1.156 |\r\n| n2/0 | running | iiiiiiiiiiiiii | 192.168.1.157 |\r\n| n3/0 | running | iiiiiiiiiiiiii | 192.168.1.158 |\r\n| n4/0 | running | iiiiiiiiiiiiii | 192.168.1.165 |\r\n| u1/0 | running | iiiiiiiiiiiiii | 192.168.1.160 |\r\n+---------------------------------+---------+----------------+---------------+',
    yytext: '',
    conditionStack: [ 'INITIAL' ],
    yylloc:
    {
      first_line: 17,
      last_line: 17,
      first_column: 78,
      last_column: 78 },
    offset: 0,
    matches: [ '', index: 0, input: '' ] },
    parser: { yy: {}, parseError: [Function: parseError] } }
```
我们看到，yy中储存了多个运行时用到的变量。我们很容易在里面找到bison中其他变量的对应，例如yytext和yyleng。
这些变量在语法解析的过程中均可以被访问到，因此，我们大胆猜想一下，如果我们想要保存/使用自己的变量，或许可以直接通过yy.myVarible进行访问。
我们修改jison文件如下（仅标出修改部分），并保存为table-v2.jison：

```
%lex

%{
yy.convert = function(obj, header){
  for(var i=0;i<obj._content.length;i++)
  {
    obj[header[i]]=obj._content[i];
  }
  delete obj._content;
  return obj;
}
%}

%%

...

headerline
  : line
    {
      yy.header = [];
      for(var i=0;i<$1._content.length;i++){
        yy.header.push($1._content[i]);
      }
      $$ = $1;
    }
  ;

...

content
  : line
    {
      $$ = {_content: []};
      $$._content.push(yy.convert($1,yy.header));
    }
  | content line
    {
      $1._content.push(yy.convert($2,yy.header));
      $$ = $1;
    }
  ;

```
再运行下面的代码看看：

```
jison table-v2.jison
node table-v2.js sample.txt
```

控制台输出：

```
{ _content:
  [ { VM: 'a/0',
  State: 'running',
  'VM Type': 'a',
  IP: '192.168.1.159' },
    { VM: 'b/0',
    State: 'running',
    'VM Type': 'b',
    IP: '192.168.1.161' },
    { VM: 'b/1',
    State: 'running',
    'VM Type': 'b',
    IP: '192.168.1.162' },
    { VM: 'b/2',
    State: 'running',
    'VM Type': 'b',
    IP: '192.168.1.163' },
    { VM: 'eeee/0',
    State: 'running',
    'VM Type': 'iiiiiiiiiiiiii',
    IP: '192.168.1.164' },
    { VM: 'haaaaaaa/0',
    State: 'running',
    'VM Type': 'iiiiiiiiiiiiii',
    IP: '192.168.1.168' },
    { VM: 'c/0',
    State: 'running',
    'VM Type': 'iiiiiiiiiiiiii',
    IP: '192.168.1.166' },
    { VM: 'lllllllllloooooooooolllllller/0',
    State: 'running',
    'VM Type': 'iiiiiiiiiiiiii',
    IP: '192.168.1.167' },
    { VM: 'n1/0',
    State: 'running',
    'VM Type': 'iiiiiiiiiiiiii',
    IP: '192.168.1.156' },
    { VM: 'n2/0',
    State: 'running',
    'VM Type': 'iiiiiiiiiiiiii',
    IP: '192.168.1.157' },
    { VM: 'n3/0',
    State: 'running',
    'VM Type': 'iiiiiiiiiiiiii',
    IP: '192.168.1.158' },
    { VM: 'n4/0',
    State: 'running',
    'VM Type': 'iiiiiiiiiiiiii',
    IP: '192.168.1.165' },
    { VM: 'u1/0',
    State: 'running',
    'VM Type': 'iiiiiiiiiiiiii',
    IP: '192.168.1.160' } ] }
```

干的不错，我们成功的将表格内容存入了对应的对象中，收工！

### 一些jison的坑

从上面的代码中可以看到，我在预编译阶段声明并定义了yy.convert函数。我们可以在语法分析阶段直接使用该函数。
但是，如果我们在预编译阶段声明我们要用到的变量例如`yy.header = [];`，则会出现严重问题。Jison在每一个语法匹配完成之后将重置yy.header为最初声明的空的Array。导致在匹配`content`语法时无法拿到正确的yy.header。
因此，我将yy.header的初始化放到了headerline的处理逻辑中，以此绕过这个坑。
我已经在github题了一个[issue](https://github.com/zaach/jison/issues/327)，暂时没有人回复。。。

### 然后

我想到可以继续扩展的功能点包括：

1. 多行Header
2. 表格中有空的Cell

这些会在接下来的文章中一一补充。

众人：才刚刚填了一个坑立马又挖了两个，眼看着就要烂尾了。。
abuuu：就算烂尾我也要挖-。-！
