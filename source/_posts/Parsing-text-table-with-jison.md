---
title: Parsing-text-table-with-jison
date: 2016-04-21 16:02:41
tags: [NodeJS, jison]
---
# 用jison解析一个文本表格

### 前言

本文受到文章[如何愉快地写个小parser](http://mp.weixin.qq.com/s?__biz=MzA3NDM0ODQwMw==&mid=210542047&idx=1&sn=9c813595c727c0fa028651b9dcdbab12)的启发，尝试用文中所介绍的Jison来进行文本解析处理。

### 什么是Jison

Jison的[官网介绍](http://zaa.ch/jison/about/)是：

```
Parsers help computers derive meaning from arbitrary text. And Jison helps you build parsers!

Jison is essentially a clone of the parser generator Bison (thus Yacc,) but in JavaScript. It includes its own lexical analyzer modeled after Flex.
```
Jison是一个"Bison in JavaScript"。也就是说，首先Jison是一个类似Bison的parser生成器。区别在于，Jison使用JavaScript编写，可以同时用于NodeJS和浏览器执行环境中。

Jison的安装请参照http://zaa.ch/jison/docs/，在NodeJS的开发环境中使用`npm install jison -g`即可完成安装。

### 使用Jison

这里我们用一个简单的例子来介绍如何使用Jison。

最近的工作都在使用CloudFoundry，在使用BOSH的时候经常需要通过`bosh vms`去抓ip地址。这个命令会返回一个类似于下面格式的表格：

```
+---------------------------------+---------+----------------+---------------+
| VM                              | State   | VM Type        | IP            |
+---------------------------------+---------+----------------+---------------+
| a/0                             | running | a              | 192.168.1.159 |
| b/0                             | running | b              | 192.168.1.161 |
| b/1                             | running | b              | 192.168.1.162 |
| b/2                             | running | b              | 192.168.1.163 |
| eeee/0                          | running | iiiiiiiiiiiiii | 192.168.1.164 |
| haaaaaaa/0                      | running | iiiiiiiiiiiiii | 192.168.1.168 |
| c/0                             | running | iiiiiiiiiiiiii | 192.168.1.166 |
| lllllllllloooooooooolllllller/0 | running | iiiiiiiiiiiiii | 192.168.1.167 |
| n1/0                            | running | iiiiiiiiiiiiii | 192.168.1.156 |
| n2/0                            | running | iiiiiiiiiiiiii | 192.168.1.157 |
| n3/0                            | running | iiiiiiiiiiiiii | 192.168.1.158 |
| n4/0                            | running | iiiiiiiiiiiiii | 192.168.1.165 |
| u1/0                            | running | iiiiiiiiiiiiii | 192.168.1.160 |
+---------------------------------+---------+----------------+---------------+
```
对于这样一个格式规整的多行文本，如果用正则表达式去匹配整个表格并提取所有的内容，写出的代码可维护性会比较差。我们下面思考一下如何使用Jison来创建一个parser解析这个表格。

### 思路

首先做词法分析，

![](https://raw.githubusercontent.com/nnabuuu/blog-hexo/gh-pages/img/Parsing-text-table-with-jison/LexicalAnalysis.png)

通过如下的正则表达式来描述所有的词法
```
\s+                                 /* skip whitespace */
<<EOF>>               return 'EOF'  /* End of file*/
"+"                   return 'HEAD' /* Header of the separate line */
("-")+"+"             return 'TAIL' /* Tail of the separate line */
[^+|-]*[^\s+|-]		    return 'CELL' /* Content in the cell, this is what we want */
"|"                   return '|'    /* Vertical bar of the cell */
```

然后是语法分析，如下：

```
%start table

%% /* language grammar */

table
    : separateline headerline separateline content separateline EOF
        {
          console.log({header: $2, body: $4}); // Print out for debug
          return {header: $2, body: $4};       // Separate lines are ignored
        }
    ;

separateline
    : HEAD
        {$$ = $1;}
    | separateline TAIL
        {$$ = $1 + $2;}
    ;

headerline
    : line
		    {$$ = $1}
    ;

content
    :  line
       {$$ = {content: []};$$.content.push($1);}
    |  content + line
       {$1.content.push($2); $$ = $1;}
    ;

line
    : '|'
        {$$ = {content: []};}
    |  line CELL '|'
        {$1.content.push($2); $$ = $1;}
    ;
```

最后我们得到了jison文件[table-v1.jison](https://gist.github.com/nnabuuu/e35074b7b34b2324aa478de74e097203)

### 生成parser

我们使用jison命令生成对应的parser：
```
jison table-v1.jison
```
执行之后，我们会得到名为table-v1.js的parser文件，该文件可以直接使用进行parse。我们首先将表格文本保存为sample.txt。然后执行命令：

```
node table-v1.js sample.txt
```

输出如下：
```
{ header: { content: [ 'VM', 'State', 'VM Type', 'IP' ] },
  body:
   { content:
      [ [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object] ] } }
```
这样，即完成了一个简单的解析。

### 然后
当然，仅仅解析成这样是不够的，我们希望最后能够得到一个这样的“开箱即用”的结果格式：

```
{     [ {'VM': 'a/0', 'State': 'running', 'VM Type': 'a', 'IP': '192.168.1.159'},
        {'VM': 'b/0', 'State': 'running', 'VM Type': 'b', 'IP': '192.168.1.161'},
        ...
      ]
}
```
那么我们要怎么做呢？

且听下回分解。

另：
后面会完成的内容：
table-v1.jison重构为table-v2.jison
通过NodeJS封装parser调用
NodeJS与BOSH的集成
将parser服务化，对外提供service

众人：abuuu同学你列了这么多后面烂尾了怎么办
abuuu：那就删掉啊，不然要怎样（手动白眼）
