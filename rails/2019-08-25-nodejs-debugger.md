---
title: Node.js debugger
category: rails
layout: post
---

## 基本用法

```shell
node inspect test.js
```

## webpack
```shell
node inspect webpack --config config.js

```

## queryOjbect

Chrome devTools console 控制台上有一个小众的 API 叫 queryObjects()，它可以从原型树上反查所有直接或间接的继承了某个对象的其它对象，比如 queryObjects(Array.prototype)可以拿到所有的数组对象，queryObjects(Object.prototype)则基本上可以拿到页面里的所有对象了（除了继承自Object.create(null)的对象之外）。而且关键是这个 API 会在内存里搜索对象前先进行一次垃圾回收。