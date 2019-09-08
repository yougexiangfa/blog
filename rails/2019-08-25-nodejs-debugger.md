---
title: Node.js debugger
category: rails
layout: post
---

在软件开发的过程中总免不了 debug，学会正确的debug姿势 能够有效的提高效率，接下来就科普下 Javascript 中的 dedug 姿势。

## Node.js inspect

Nodejs 中提供了一个进程外的 dubug 工具，要使用debug的话，通过 inspect 参数启动 nodejs 即可。

假定我们要调试一个js文件
```js
// code/debug.js
let a = 1
```

在命令行执行命令：
```shell
node inspect code/debug.js
```

会进入到如下环境：

```
  1
> 2 let a = 1
debug>
```
可以通过 help 查看帮助信息。

此时断点位于文件开头的第一行代码，由于此时断点处还未执行首行的赋值表达式：`let a = 1`，执行 next 进入下一行代码，然后执行 repl 可以进入交互式解释环境（Read Eval Print Loop），方便查看变量、执行表达式等调试操作；

此时输入 `a` 的结果是 1，如果在执行 next 之前进入 repl 环境，输入 `a` 的结果则是 undefined，注意并非是报 `a is not defined`的错误。

在 repl 环境下，可以通过输入 `.help` 查看帮助信息。

要退出 repl 环境回到 debug 环境，执行 `Ctrl + C`.

## Debugger

使用 node inspect 的方式虽然实现了 debug, 但是从第一行代码就开始断点实用性显然优先，要想随心所欲的设置断点，可以使用debugger 作为标记。

```js
// code/debug.js
let a = 1
debugger
let b = 2
```
我们在上述js文件中加入debugger，然后再运行 node inspect，断点依然默认停留在第一行代码处，此时执行 cont, 则直接跳转到 debugger 所标记的断点处。

## shebang line nodejs script with debug

对于 shebang line 的 node.js 应用，可以在命令前加上 node inspect, 如：

```shell
node inspect webpack --config config.js
```

## dev console
在浏览器环境下，debug 可以使用浏览器自带的开发者工具。

在 chrome 浏览器地址栏输入 `about:blank`，可以得到一个非常纯净的页面，然后右键菜单，选择检查即可打开开发者工具。

点击 source 菜单，就可以在对应的js代码里设置断点并进行调试了。

## queryOjbect

Chrome devTools console 控制台上有一个小众的 API 叫 queryObjects()，它可以从原型树上反查所有直接或间接的继承了某个对象的其它对象，比如 queryObjects(Array.prototype)可以拿到所有的数组对象，queryObjects(Object.prototype)则基本上可以拿到页面里的所有对象了（除了继承自Object.create(null)的对象之外）。而且关键是这个 API 会在内存里搜索对象前先进行一次垃圾回收。


## 参考阅读
* [Shebang Line](https://github.com/fish-shell/fish-shell/blob/master/sphinx_doc_src/index.rst#shebang-line)