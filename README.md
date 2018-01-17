## workerize

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)

在Web Worker中运行一个模块。 

作者是 [Preact- 小型React库的作者](https://github.com/developit/preact)

[github source](https://github.com/developit/workerize)

---

尝试 `workderize`, 在 调试`console`

``` js
let worker = window.workerize(`
	export function add(a, b) {
		// block for half a second to demonstrate asynchronicity
		let start = Date.now();
		while (Date.now()-start < 500);
		return a + b;
	}
`);

(async () => {
	console.log('3 + 9 = ', await worker.add(3, 9));
	console.log('1 + 2 = ', await worker.add(1, 2));
})();
```

在本目录，启动`http`

> python

```
python -m SimpleHTTPServer
```

> javascript

```
npx http-server .
```

---

## workerize

因为只有一个文件

[index.js](./workerize/src/index.js)

---

按照给出的例子,和作者描述

- 将一个小型的专门构建的RPC实现捆绑到您的应用程序中

- 如果导出的模块方法已经是异步的，那么签名是不变的

- 支持同步和异步工作者功能

- 与异步/等待美妙地工作

- 导入的值是可实例化的，只是一个装饰 `Worker`
---

## [整体概括](#整体概括)

- [workerize](#)

- [worker-管理](#)

- [开始运作](#)

- [setup](#)

- [工具函数](#)

---

## workerize

代码 22-34

``` js
export default function workerize(code) {
	let exports = {};
	let exportsObjName = `__EXPORTS_${Math.random().toString().substring(2)}__`; // 随机 id
	if (typeof code==='function') code = `(${toCode(code)})(${exportsObjName})`; 
	// 如果直接输出函数，从我们例子来看，先不管
	code = toCjs(code, exportsObjName, exports);
	// 那么现在先去看 toCjs <-----  1

	code += `\n(${toCode(setup)})(self, ${exportsObjName}, {})`;
	// <--- 2 把 setup 函数加进来
	let blob = new Blob([code], {
			type: 'application/javascript'
		}),
		url = URL.createObjectURL(blob), // 变成了可以被加载的 Url
		worker = new Worker(url),
		counter = 0,
		callbacks = {};
```

- [toCode](#toCode)

> 函数变文本

- [toCjs](#toCjs)

> 改造文本，记录函数

- code += setup

> [到这一步JsBin > code += setip](http://jsbin.com/rufuzeq/5/edit?js,console)

- [Blob](#blob)

> Blob 对象表示不可变的类似文件对象的原始数据。[jsbin 例子](http://jsbin.com/rufuzeq/6/edit?js,console)

- [Worker](#worker)

> `Worker()` 构造函数创建一个 Worker 对象，该对象执行指定的URL脚本。这个脚本必须遵守 同源策略 。

---

### worker-管理

代码 35-44

``` js
	worker.kill = signal => {
		worker.postMessage({ type: 'KILL', signal });
		setTimeout(worker.terminate);
	}; // 关闭
	let term = worker.terminate;
	worker.terminate = () => {
		URL.revokeObjectURL(url); // 丢 url
		term(); // 触发本身结束命令
    }; // 
    worker.rpcMethods = {};
```

---

## 开始运作

代码 84-95

``` js
	setup(worker, worker.rpcMethods, callbacks); // <---- 1
	worker.call = (method, params) => new Promise( (resolve, reject) => {
		let id = `rpc${++counter}`;
		callbacks[id] = { method, resolve, reject };
		worker.postMessage({ type: 'RPC', id, method, params });
	});
	// exports 通过 toCjs 函数 获取到了 函数名
	// exports = {
	// 	'add' : true
	// }
	for (let i in exports) {
		// i == 'add'
		if (exports.hasOwnProperty(i) && !(i in worker)) {
			worker[i] = (...args) => worker.call(i, args);
		}
	}
	return worker;
```

- [setup](#setup)

- `wroker[i]`

还记得 我们例子的使用 `worker.add`

``` js
// usage
(async () => {
	console.log('3 + 9 = ', await worker.add(3, 9)); // args 3,9
	console.log('1 + 2 = ', await worker.add(1, 2)); // args 1,2
})();
// index.js
worker[i] = (...args) => worker.call(i, args); // args=
```

- `worker.call`

> 使用函数，通知 worker 内部, 

``` js
	worker.call = (method, params) => new Promise( (resolve, reject) => { // 变成了 异步/Promise
		let id = `rpc${++counter}`; // 创建 唯一id
		callbacks[id] = { method, resolve, reject }; // 放入本地缓存函数集
		worker.postMessage({ type: 'RPC', id, method, params }); // 第一次触发 onmessage 事件
		// type 自定义类型, id 唯一标签, method 函数, params 变量
	});
```
---

### setup

代码 45-83

> ctx == worker , rpcMethods == {}, callbacks == {}
``` js

	function setup(ctx, rpcMethods, callbacks) {
		/*
		ctx.expose = (methods, replace) => {
			if (typeof methods==='string') {
				rpcMethods[methods] = replace;
			}
			else {
				if (replace===true) rpcMethods = {};
				Object.assign(rpcMethods, methods);
			}
		};
		*/
		ctx.addEventListener('message', ({ data }) => {
			if (data.type==='RPC') {
				let id = data.id;
				if (id!=null) { 
					if (data.method) { 
						let method = rpcMethods[data.method]; // 本地缓存-rpcMethods-中找出 method
						if (method==null) {
							ctx.postMessage({ type: 'RPC', id, error: 'NO_SUCH_METHOD' }); 
						}
						else {
							Promise.resolve()
								.then( () => method.apply(null, data.params) )
								.then( result => { ctx.postMessage({ type: 'RPC', id, result }); }) 
								// 触发ctx.onmessage 事件 
								// data == { type: 'RPC', id, result }); } 
								.catch( error => { ctx.postMessage({ type: 'RPC', id, error }); }); 
								// 触发ctx.onmessage 事件 
								// data == { type: 'RPC', id, error }); } 
						}
					}
					else {
						let callback = callbacks[id];  // 本地缓存-函数集-id
						if (callback==null) throw Error(`Unknown callback ${id}`);
						delete callbacks[id];
						if (data.error) callback.reject(Error(data.error));
						else callback.resolve(data.result);
					}
				}
			}
		});
	}
```

- `addEventListener('message', //...)` 

[`Worker`](#worker) - 属性 

> onmessage 一个事件监听函数，每当拥有 message 属性的 MessageEvent 从 worker 中冒泡出来时就会执行该函数。

> 事件的 data 属性存有消息内容。

- `data.type === RPC`

- ctx.postMessage

> 向 worker 的内部作用域内传递消息。触发 `ctx.onmessage` 事件

因为 `ctx.postMessage` 在 `onmessage` 触发事件内部，所以[第一次触发](./README.md#L178)，不会来自内部

---



---

## 工具函数

### toCode

``` js
function toCode(func) {
	return Function.prototype.toString.call(func);
}
```

``` js
function add(){
}

```

> Function.prototype.toString.call(add) // 变成 String

< "function add(){}"

---

### toCjs

> - code : String

``` js
	// String
	export function add(a, b) {
		// block for half a second to demonstrate asynchronicity
		let start = Date.now();
		while (Date.now()-start < 500);
		return a + b;
	}
```

> - exportsObjName : String

`__EXPORTS_${Math.random().toString().substring(2)}__`

>> "__EXPORTS_4706166142920267__"

> - exports : Object `{}`


``` js
function toCjs(code, exportsObjName, exports) {
	exportsObjName = exportsObjName || 'exports';
	exports = exports || {};
	code = code.replace(/^(\s*)export\s+default\s+/m, (s, before) => {
		// export default function
		exports.default = true; // 具有默认导出函数
		// before == ''
		return `${before}${exportsObjName}.default = `;
		// code 将变成 __EXPORTS_4706166142920267__.default = function ...
		// 但是 这个例子 没有 default
	});
	code = code.replace(/^(\s*)export\s+(function|const|let|var)(\s+)([a-zA-Z$_][a-zA-Z0-9$_]*)/m, (s, before, type, ws, name) => {
		exports[name] = true;
		return `${before}${exportsObjName}.${name} = ${type}${ws}${name}`;
	});
	// code ==
	// `__EXPORTS_4706166142920267__.add = function add(a, b) {
	// 	// block for half a second to demonstrate asynchronicity
	// 	let start = Date.now();
	// 	while (Date.now()-start < 500);
	// 	return a + b;
	// }`

	return `var ${exportsObjName} = {};\n${code}\n${exportsObjName};`;

	// return 
	// var __EXPORTS_4706166142920267__ = {};
	// `__EXPORTS_4706166142920267__.add = function add(a, b) {
	// 	// block for half a second to demonstrate asynchronicity
	// 	let start = Date.now();
	// 	while (Date.now()-start < 500);
	// 	return a + b;
	// }`
	// __EXPORTS_4706166142920267__
}
```

[本例子的-JsBin-](http://jsbin.com/rufuzeq/3/edit?js,console)

### `code.replace`

> `replace()` 方法返回一个由替换值替换一些或所有匹配的模式后的新字符串。

> 模式可以是一个字符串或者一个正则表达式, 替换值可以是一个字符串或者一个每次匹配都要调用的函数。

[mdn文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/String/replace)

### 第一个正则式说明

``` js
code.replace(/^(\s*)export\s+default\s+/m,)

``` 

> `^` 

开头

> `\s`

匹配一个空白符，包括空格、制表符、换页符、换行符和其他 `Unicode` 空格。

> `( )`  (\s)

匹配 `\s` 并且捕获匹配项。 这被称为捕获括号（capturing parentheses）。

> `*` 

匹配前面的模式 `\s` 0 或多次。

> `export`

直白匹配 `export`  文本

> `+` 

匹配前面的模式 `\s` 1 或多次。

> /m

多行; 将开始和结束字符（^和$）视为在多行上工作（也就是，分别匹配每一行的开始和结束（由 \n 或 \r 分割），而不只是只匹配整个输入字符串的最开始和最末尾处。

---


### 第二个正则式说明

``` js
code.replace(/^(\s*)export\s+(function|const|let|var)(\s+)([a-zA-Z$_][a-zA-Z0-9$_]*)
```

> `(function|const|)`

匹配 `function` 或 `const`

> [a-zA-Z$_]

一个字符集合，也叫字符组。匹配集合中的任意一个字符。你可以使用连字符'-'指定一个范围。

比如这个就是 小写的 `a到z` 大写 的`A到Z` `$` 和 `_`

#### [更多正则式内容](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/RegExp)

### return

`var ${exportsObjName} = {};\n${code}\n${exportsObjName};`

---

## Blob

[mdn文档](https://developer.mozilla.org/zh-CN/docs/Web/API/Blob)

## worker

[mdn文档](https://developer.mozilla.org/zh-CN/docs/Web/API/Worker)

## 整体概括

