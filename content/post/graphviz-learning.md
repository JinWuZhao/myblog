---
title: "学习Graphviz"
date: 2017-12-28T16:59:15+08:00
tags: [ "Graphviz", "DOT" ]
categories:
    - "工具"
    - "程序语言"
draft: false
---
使用go语言开发的朋友应该对pprof不会陌生，pprof可以导出像下面这种图：
![pprof](/graphviz-learning/pprof.png)  
那么问题来了，这是怎么生成的呢？  
答案就是――Graphviz。  

## Graphviz
Graphviz是一个自动化绘图的工具，用于绘制[DOT语言](https://zh.wikipedia.org/wiki/DOT%E8%AF%AD%E8%A8%80)描述的图形。

## QuickStart
废话不多说，先上个例子，安装配置不讲了自己找。  
假设我们描述一个简单的解释器的执行流程：  
```plain
1. 获取输入（getInput)
2. 解析输入字符串（parse），成功转到3，失败转到5
3. 对parse出的AST求值（evaluate），成功转到4，失败转到6
4. 输出求值结果（putResult）
5. 提示语法错误（throwSyntaxError）
6. 求值过程出现异常，比如除0等（throwRuntimeError）
```
创建一个文件，名为interpreter.dot，并写入以下代码：  
```cpp
digraph Interpreter {
  getInput -> parse -> evaluate -> putResult;
  parse -> throwSyntaxError;
  evaluate -> throwRuntimeError;
}
```
在终端执行以下命令：  
```go
dot -Tpng interpreter.dot -o interpreter.png
```
于是就得到了下面这张图：  
![interperter](/graphviz-learning/interpreter.png)  
这里生成的是png格式，也可以生成其它格式的文件，比如pprof生成的就是svg格式的。

## DOT语言介绍
这是Graphviz文档里的DOT语法：  
```haskell
graph: [ strict ] (graph | digraph) [ ID ] '{' stmt_list '}'
stmt_list: [ stmt [ ';' ] stmt_list ]
stmt: node_stmt
    | edge_stmt
    | attr_stmt
    | ID '=' ID
    | subgraph
attr_stmt: (graph | node | edge) attr_list
attr_list: '[' [ a_list ] ']' [ attr_list ]
a_list: ID '=' ID [ (';' | ',') ] [ a_list ]
edge_stmt: (node_id | subgraph) edgeRHS [ attr_list ]
edgeRHS: edgeop (node_id | subgraph) [ edgeRHS ]
node_stmt: node_id [ attr_list ]
node_id: ID [ port ]
port: ':' ID [ ':' compass_pt ]
    | ':' compass_pt
subgraph: [ subgraph [ ID ] ] '{' stmt_list '}'
compass_pt: (n | ne | e | se | s | sw | w | nw | c | _)
```
详情请见：https://graphviz.gitlab.io/_pages/doc/info/lang.html  
看得懂很好，看不懂也没关系😂。马上我们就开始看例子。

## 图形类别
### 无向图
```cpp
graph name {
  a -- b -- c;
  b -- d;
}
```
![graph](/graphviz-learning/graph.png)

### 有向图
```cpp
digraph name {
  a -> b -> c;
  b -> d;
}
```
![graph](/graphviz-learning/digraph.png)

## 属性
```cpp
// 这是一行注释
digraph G {
  size ="4,4"; /* 图的大小 */
  main [shape=box]; /* 节点的形状：盒子 */
  main -> parse [weight=8]; /* 连线的权值，影响长度和曲率 */
  parse -> execute;
  main -> init [style=dotted]; /* 线的风格：点虚线 */
  main -> cleanup;
  execute -> { make_string; printf} /* 相当于：execute -> make_string; execute -> printf; */
  init -> make_string;
  edge [color=red]; /* 全局设置线的颜色 */
  main -> printf [style=bold,label="100 times"]; /* label:在连线上添加说明 */
  make_string [label="make a\nstring"]; /* label:在节点上添加说明（绘制的时候会替换掉节点的名字） */
  node [shape=box,style=filled,color=".7 .3 1.0"]; /* 全局设置节点的形状、风格、背景色 */
  execute -> compare;
}
```
效果如下：  
![attributes](/graphviz-learning/attributes.png)  
更多属性请见：https://graphviz.gitlab.io/_pages/doc/info/attrs.html

## record shape以及label语法
record shape可以依赖label语法构造更复杂的节点和图。  
label语法：
```go
rlabel: field ( '|' field )*
field: boxLabel | '' rlabel ''
boxLabel: [ '<' string '>' ] [ string ]
```
例如我们描述一个二叉树：  
```cpp
digraph Tree {
  node [shape=record];
  root [label="<l> left | <e> element | <r> right"]; // 为left添加一个引用l
  a1 [label="<l> left | <e> element | <r> right"]; // 为element添加一个引用e
  a2 [label="null | <e> element | null"];
  b1 [label="null | <e> element | null"];
  b2 [label="null | <e> element | null"];
  "root":l -> "a1":e; // 连接root节点的l与a1节点的e
  "root":r -> "a2":e;
  "a1":l -> "b1":e;
  "a1":r -> "b2":e;
}
```
效果如下：  
![tree](/graphviz-learning/tree.png)  

## subgraph
subgraph可以在一个图的内部定义子图。  
```cpp
digraph G {
  subgraph cluster0 {
    node [style=filled,color=white];
    style = filled;
    color = lightgrey;
    a0 -> a1 -> a2 -> a3;
    label = "process #1";
  }

  subgraph cluster1 {
    node [style=filled];
    b0 -> b1 -> b2 -> b3;
    label = "process #2";
    color = blue;
  }

  start -> a0;
  start -> b0;
  a1 -> b3;
  b2 -> a3;
  a3 -> a0;
  a3 -> end;
  b3 -> end;

  start [shape=Mdiamond];
  end [shape=Msquare];
}
```
效果如下：  
![subgraph](/graphviz-learning/subgraph.png)  

## 结束
好啦，暂时先介绍到这里。刚刚这些只是冰山一角，更多的功能可以去查阅Graphviz[官方文档](https://graphviz.gitlab.io/documentation/)。
