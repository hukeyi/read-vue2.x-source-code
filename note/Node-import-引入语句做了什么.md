[toc]

在读 vue 源码时发现的疑惑：[[03-init-introduction#What those mixins do]]。为什么要在 `Vue` 的声明文件中写以下几句调用代码：

```js
// 这些函数在这里被调用，并传入 Vue 作为参数，问题是，这个文件什么时候被执行？被谁执行？难道在 new Vue 之后接下来五个函数自动就执行吗？
initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)
```

我原本以为在 `import` 时，只有我们需要的那个被引入对象的函数被“插入”引用它的代码文件中。这个“插入”只是单纯的代码复制粘贴过程，不涉及任何解析执行。结果看源码才发现我的认识是错误的。

## 实例测试

首先，写待引用文件 `test3.js`：

```js
// Node 的声明
function Node() {
	console.log("I have been initialized. ");
}

// 一些代码
console.log("Will I be executed? I am outside any function block. ");

// 测试这个函数是否会作用到 Node 上
function InitNode(Node) {
	console.log('InitNode executed. ');
	Node.prototype.print1 = function () {
		console.log("I am print1() added by InitNode. ");
	};
}
InitNode(Node);

export default Node;
```

然后，另一个用于引用 `test3.js` 的文件 `test4.js`：

```js
import Node from './test3.js';

let node = new Node();
node.print1();
```

执行 `test4.js`，控制台会输出：

```zsh
Will I be executed? I am outside any function block. 
InitNode executed. 
I have been initialized. 
I am print1() added by InitNode. 
```

See？`InitNode` 为 `Node` 原型加入的新成员函数 `print1()` 生效了。

那么，这个代码生效的更精准的位置在哪？是在 `import` 时就生效，还是在 `new Node()` 时生效的呢？

我们注释掉 `new Node()` 和 `print1()`：

```js
import Node from './test3.js';

// let node = new Node();
// node.print1();
```

再执行 `test4.js`，控制台输出：

```shell
Will I be executed? I am outside any function block. 
InitNode executed. 
```

`InitNode` 依然执行了！说明 `import` 语句所做的工作不只是把 `Node` 声明的代码复制粘贴到引入文件（e.g. test4.js）中，它还会执行被引入的文件（test3.js）。