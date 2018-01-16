## workerize

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)

在Web Worker中运行一个模块。 

作者是 [Preact- 小型React库的作者](https://github.com/developit/preact)

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

代码 22-34

``` js
export default function workerize(code) {
	let exports = {};
	let exportsObjName = `__EXPORTS_${Math.random().toString().substring(2)}__`; // 随机 id
	if (typeof code==='function') code = `(${toCode(code)})(${exportsObjName})`; 
	code = toCjs(code, exportsObjName, exports);
	code += `\n(${toCode(setup)})(self, ${exportsObjName}, {})`;
	let blob = new Blob([code], {
			type: 'application/javascript'
		}),
		url = URL.createObjectURL(blob),
		worker = new Worker(url),
		counter = 0,
		callbacks = {};
```

- [toCode](#toCode)

- [toCjs](#toCjs)

---

代码 35-44

``` js
	worker.kill = signal => {
		worker.postMessage({ type: 'KILL', signal });
		setTimeout(worker.terminate);
	};
	let term = worker.terminate;
	worker.terminate = () => {
		URL.revokeObjectURL(url);
		term();
    };
    worker.rpcMethods = {};
```

---

代码 45-83

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
						let method = rpcMethods[data.method];
						if (method==null) {
							ctx.postMessage({ type: 'RPC', id, error: 'NO_SUCH_METHOD' });
						}
						else {
							Promise.resolve()
								.then( () => method.apply(null, data.params) )
								.then( result => { ctx.postMessage({ type: 'RPC', id, result }); })
								.catch( error => { ctx.postMessage({ type: 'RPC', id, error }); });
						}
					}
					else {
						let callback = callbacks[id];
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
---

代码 84-95

``` js
	setup(worker, worker.rpcMethods, callbacks);
	worker.call = (method, params) => new Promise( (resolve, reject) => {
		let id = `rpc${++counter}`;
		callbacks[id] = { method, resolve, reject };
		worker.postMessage({ type: 'RPC', id, method, params });
	});
	for (let i in exports) {
		if (exports.hasOwnProperty(i) && !(i in worker)) {
			worker[i] = (...args) => worker.call(i, args);
		}
	}
	return worker;
```

---

## 工具函数

### toCode

``` js
function toCode(func) {
	return Function.prototype.toString.call(func);
}

```
### toCjs

``` js
function toCjs(code, exportsObjName, exports) {
	exportsObjName = exportsObjName || 'exports';
	exports = exports || {};
	code = code.replace(/^(\s*)export\s+default\s+/m, (s, before) => {
		exports.default = true;
		return `${before}${exportsObjName}.default = `;
	});
	code = code.replace(/^(\s*)export\s+(function|const|let|var)(\s+)([a-zA-Z$_][a-zA-Z0-9$_]*)/m, (s, before, type, ws, name) => {
		exports[name] = true;
		return `${before}${exportsObjName}.${name} = ${type}${ws}${name}`;
	});
	return `var ${exportsObjName} = {};\n${code}\n${exportsObjName};`;
}
```