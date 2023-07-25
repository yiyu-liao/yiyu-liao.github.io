---
layout: post
title:  "webpack编译主流程初窥"
date:   2019-11-20
tags: webpack
---

> 本文基于webpack5.0.0-beta.7, 参考一些优秀的文章，通过debug源码的方式，了解webpack的编译主流程，仅供参考。


## 准备工作

```
git clone git@github.com:webpack/webpack.git
```

为了不污染源码，在根目录新建debug文件夹📃，并创建如下文件: 


* 编译代码：

```js
// debug/src/index.js

const name = "webpack";
console.log("很高兴认识你,", name);
```

* webpack配置：

```js
// debug/webpack.config.js

const path = require("path");
module.exports = {
	context: __dirname,
	mode: "development",
	devtool: "source-map",
	entry: "./src/index.js",
	output: {
		path: path.join(__dirname, "./dist")
	},
	module: {
		rules: [
			{
				test: /\.js$/,
				use: ["babel-loader"],
				exclude: /node_modules/
			}
		]
	}
};
```

* 启动文件：

```js
// debug/start.js

const webpack = require("../lib/index.js"); // 直接使用源码中的webpack函数
const config = require("./webpack.config");
const compiler = webpack(config);
compiler.run((err, stats) => {
	if (err) {
		console.error(err);
	} else {
		console.log(stats);
	}
});

```

* 在vscode配置debug入口，如下：

```js
{
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "启动webpack调试程序",
      "program": "${workspaceFolder}/debug/start.js"
    }
  ]
}
```

至此，点击▶️，看看调试程序是否正常运行（若正常运行，会在debug/dist自动生成打包后的js文件）

接下来，就可以在源码进行断点调试啦～～ 🤔

----

## 源码解读

### compiler.run()

cd webpack, 打开package.json, 找到main字段，可以看到webpack的主入口在lib/index.js

当调用了`const compiler = webpack(config)`，其实就是创建一个Compiler实例，在这里我们发现，Compiler在webpack运行过程中，仅此创建一次，贯穿整个打包过程，下面为主要逻辑代码。


```js
const createCompiler = options => {
    // ...
	const compiler = new Compiler(options.context);
	compiler.options = options;
	if (Array.isArray(options.plugins)) {
		for (const plugin of options.plugins) {
			if (typeof plugin === "function") {
				plugin.call(compiler, compiler);
			} else {
				plugin.apply(compiler);
			}
		}
	}
    compiler.hooks.environment.call();
	compiler.hooks.afterEnvironment.call();
	compiler.options = new WebpackOptionsApply().process(options, compiler);    
	return compiler;
};
```

当准备工作做好，内部钩子相关联的事件处于蓄势待发状态，接下来把我们进入Compiler, 看里面的run方法是什么回事🤔？


最终定位到`this.compile()`, 通过`callAsync`触发钩子，逻辑代码如下：

```js
this.hooks.make.callAsync(compilation, err => {
    //...
	compilation.finish(err => {
		compilation.seal(err => {
            //...
			this.hooks.afterCompile.callAsync(compilation, err => {
				return callback(null, compilation);
			});
		});
	});
});
```

这里需要提下[Tapable](https://github.com/webpack/tapable)，Tapable的核心功能是根据不同钩子将注册的事件在触发时按序进行，是典型的发布者/订阅模式，在Tapable的帮助下，原本杂乱无章的事件，能够得到有效的控制。

简单实现一个SyncHook

```js
class Hooks {
	constructor(args) {
		this.taps = [];
		this.interceptors = [];
		this._args = args;
	}

	tab(name, fn) {
		this.taps.push({ name, fn })
	}

}

class SyncHooks extends Hooks {
	call(name, fn) {
		try {
			this.taps.forEach(tap => tap.fn(name));
			fn(null, name);
		}catch(error) {
			fn(error)
		}
	}
}
```

### process(options, compiler)

当调用`this.hooks.make.callAsync`其实触发了什么？为了找到答案，我们在vscode全局搜索`hooks.make.tapAsync`, 发现`lib/EntryPlugin.js`下有如下逻辑：
> 成熟的开源库，往往作者大佬进行了非常优雅的封装，代码的模块依赖也非常错中复杂，依靠搜索关键词可以快速找到答案。

```js
compiler.hooks.make.tapAsync("EntryPlugin", (compilation, callback) => {
	const { entry, name, context } = this;

	const dep = EntryPlugin.createDependency(entry, name);
	compilation.addEntry(context, dep, name, err => {
		callback(err);
	});
});
```

先不管这里的逻辑是怎样的，思考下`EntryPlugin`是什么时候调用的？我们继续通过关键词`new EntryPlugin`,
在`lib/EntryOptionPlugin.js`找到`EntryPlugin`在此进行了实例化，继续回溯搜索`new EntryOptionPlugin`, 最终发现在`lib/WebpackOptionsApply.js`发现它的身影。

回到最初createCompiler方法，如下，`compiler.options = new WebpackOptionsApply().process(options, compiler)`这行代码，make钩子在process函数中就已经注册好了, 把一切准备工作都关联了起来。

```js
const createCompiler = options => {
    // ...
	const compiler = new Compiler(options.context);
	compiler.options = options;
	if (Array.isArray(options.plugins)) {
		for (const plugin of options.plugins) {
			if (typeof plugin === "function") {
				plugin.call(compiler, compiler);
			} else {
				plugin.apply(compiler);
			}
		}
	}
    compiler.hooks.environment.call();
	compiler.hooks.afterEnvironment.call();
	compiler.options = new WebpackOptionsApply().process(options, compiler);    
	return compiler;
};
```

回到lib/EntryPlugin.js看看`compiler.hooks.make.tapAsync`都干了啥。其实就是运行compiliation.addEntry方法，继续探索compiliation.addEntry。

```js
addEntry(context, entry, name, callback) {
    this.hooks.addEntry.call(entry, name);
    // entryDependencies中的每一项都代表了一个入口，打包输出就会有多个文件
    let entriesArray = this.entryDependencies.get(name)
	entriesArray.push(entry)
    this.addModuleChain(context, entry, (err, module) => {
        this.hooks.succeedEntry.call(entry, name, module);
        return callback(null, module);
    })
}
```

通过debug找出函数的调用顺序

this.addEntry --> this.addModuleChain --> this.handleModuleCreation --> this.addModule --> this.buildModule --> this._buildModule --> module.build(this指代compiliation)

在/lib/NormalModule.js下找到build方法，返回doBuild方法

```js
build(options, compilation, resolver, fs, callback) {

		return this.doBuild(options, compilation, resolver, fs, err => {
			try {
				const result = this.parser.parse(
					this._ast || this._source.source(),
					{
						current: this,
						module: this,
						compilation: compilation,
						options: options
					},
					(err, result) => {
						if (err) {

						} else {
							handleParseResult(result);
						}
					}
				);
				if (result !== undefined) {
					// parse is sync
					handleParseResult(result);
				}
			} catch (e) {
				
			}
		});
	}
```

doBuild方法其实就是用适合的loader去加载模块资源，webpack只能识别js文件，通过doBuild后，
所有文件都转换了js文件。

接着通过传入的callback，执行this.parser.parse，this.parser是JavascriptParser的实例，
最终调用parse调用第三方库acorn对js源代码进行词法解析, 这个过程就是收集模块之间的依赖。

```js
parse(code, options){
    // 调用第三方插件`acorn`解析JS模块
    let ast = acorn.parse(code)
    // 省略部分代码
    if (this.hooks.program.call(ast, comments) === undefined) {
        this.detectStrictMode(ast.body)
        this.prewalkStatements(ast.body)
        this.blockPrewalkStatements(ast.body)
        // 这里webpack会遍历一次ast.body，其中会收集这个模块的所有依赖项，最后写入到`module.dependencies`中
        this.walkStatements(ast.body)
    }
}
```

### compilation.seal()

至此从入口文件，webpack已经收集了所有的模块依赖，就等着打包输入啦～～（真累🥺）

compilation.seal最终将资源保存在compilation.assets, compilation.chunks中，所以在编写
webpack插件时候，经常会看到这两个的字段的身影。

### compiler.hooks.emit.callAsync()

执行seal后，所以资源都保存到内存中了，最后调用emit钩子根据webpack.config的配置将文件输出到指定路径。

## 总结

写得比较模糊仓促，其实对于webpack loader/tapable这一块还可以继续深入探究，可能只有自己才能看得懂哈哈哈，anyway～～ 有空再回头补充下，感谢开源～～









