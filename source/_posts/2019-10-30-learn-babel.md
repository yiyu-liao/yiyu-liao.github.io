---
layout: post
title:  "Babel学习笔记"
date:   2019-10-30
tags: Babel
---

### 简介
 
Babel是Javascript编译器，将ECMAScript 2015+版本的代码转换成向后兼容的js代码

基本编译流程：

首先babel对原代码进行词法分析（Tokenize），将原代码分割成Tokens数组, Tokens包括语法片段，位置信息，以及一些类型信息，接下来对Tokens进行词法解释，Parser转换成ATS（Abstract Syntax Tree), 以树结构的形式表示代码结构，AST是babel转译的核心数据结构，后续的操作都依赖于AST。得到AST后，babel进入转换阶段（Transform），babel对AST进行遍历，所有插件的应用都在此阶段进行。最后将AST转换成js, 在此阶段生产Source Map。

简单来说：source code通过babel得到AST, 根据定义的规则修改AST，最终输入。

🌰：`let/const`转化为`var`, `() => {}`箭头函数转化为`function() {}`

### 基本用法

#### 1.安装依赖

```
npm install --save-dev @babel/core @babel/cli @babel/preset-env
npm install --save @babel/polyfill
```

#### 2.在根目录创建配置文件babel.config.js

```js
const presets = [
  [
    "@babel/env",
    {
      targets: {
        edge: "17",
        firefox: "60",
        chrome: "67",
        safari: "11.1",
      },
      useBuiltIns: "usage",
    },
  ],
];

module.exports = { presets };

```

#### 3.命令行操作

```js
./node_modules/.bin/babel src --out-dir lib
```
以上命令的意思是解释scr目录下所有js文件，并且按照定义好的转换规则(babel.config.js)输出到lib目录下


### 实际场景例子

eg: 在development环境，往往需要console.log进行调试，product环境面对的是用户，log记录就不应该存在了，为了避免开发人员遗留的log, 可通过babel插件去除log代码

以下为配置文件：

```js
let plugins = [];

if (process.env.NODE_ENV !== 'development') {
    // https://www.babeljs.cn/docs/babel-plugin-transform-remove-console
    plugins.push('transform-remove-console');
}

module.exports = {
    plugins
}
```

### 开发一个babel插件

babel插件模块，需要暴露一个function, function包含visitor(详细写是不可能的，这辈子都不可能详细写的。因为我菜啊～～)

```js
export default function (babel) {
  var t = babel.types;

  return {
    visitor: { 
      ArrayExpression(path, state) {
        path.replaceWith(
          t.callExpression(
            t.memberExpression(t.identfier('mori'), t.identifier('vector')),
            path.node.elements
          )
        )
      },
      ....
    }
  }
}
```

有几个关键概念需要解释下：

* `babel.type`: 用来操作AST节点，如创建，转换，校验等
* `visitor`：babel采用深度遍历访问每个AST节点，如上述代码，遍历到`ArrayExpression`时，触发定义好到函数；
* `path`: path是指AST节点对象，包含节点的信息，所在位置以及节点间的关系
* `state`: 指插件状态，可以通过state获取插件中的配置参数

需要注意什么

Scope作用域

在Js中，每当你创建一个引用，不管是通过变量（variable）, 函数（function），参数（params）等，
它都属于当前作用域。

在编写一个转换时，我们得确保在改变代码得各个部分不会破坏已经存在的代码。

[demo](https://github.com/Evan-Liao/blog/tree/master/Babel/test.js)

> node demo.js

```
var test = [1, 2, 3]
//transform
demo.init(1,2,3)

```

github传送门：<https://github.com/yiyu-liao/learn-babel>


### 总结

全文写下来，还是有很多陌生的概念，那就当一个学习笔记吧～

### 附录

* [AST在线转换](https://astexplorer.net/#/KJ8AjD6maa)