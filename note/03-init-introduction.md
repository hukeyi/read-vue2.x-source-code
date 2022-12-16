# Init - Introduction

This article belongs to the series [Read Vue Source Code](https://github.com/numbbbbb/read-vue-source-code).

In this article, we will learn:

- What those mixins do
- Understand the init process
- what  happened when you do `const app = new Vue(options);`

#### `options` 到底包含哪些属性？

见 [Vue 2.x Docs](https://v2.vuejs.org/v2/api/#Options-Data)。

#### `new Vue(options)` 返回的 `Vue` 实例是什么？含义是？ 

返回的实例就是一个「组件 component」，它对应页面上某一个组件。

CSS 的视角中，HTML 页面上由一个个「盒子」组成；而 Vue.js 视角中，HTML 页面由一个个「组件」构成。每一个盒子都有内外边距+边界+内容区几大区域；每一个组件都由 `Vue` 类定义，并通过 `new Vue(options)` 实例化出组件。

比如，有一个组件定义在文件 `myComponent.vue` 中，文件大致长这样：

```vue
<template>
	<!-- 略 -->
</template>
<script>
	export default {
		props: { /* 略 */},
		data(){
			return { /* 略 */ }
		}
	}
</script>
<style>
	/* 略 */
<style>
```

假设同文件夹的 `myPage.vue` 要引入这个组件，则 `myPage.vue` 的 `<script>` 标签中会写：

```vue
<template>
	<!-- 在模版中引用时，直接把 myComponent 当一个 html 标签一样使用 -->
	<myComponent></myComponent>
</template>
<script>
	import myComponent from './myComponent.vue';

	export default { /* 略，这里是 myPage 的 options */ }
<script>
```

`options` 就是组件 `.vue` 文件中 `export default { /* 我就是 options */ }` 导出的 `Object` 对象，是一些描述组件特性（`data, props` 等，以及 `mount()` 等生命周期钩子）的属性及其值。

`options` 就类似于「类的数据成员和成员函数」，更具体的说，这个「类」就是 `Vue` 类，它是页面的组件模版类。用 Vue 框架写的页面的所有组件元素全部以此为模版构建出来。刚开始 `new Vue(options)` 输入的 `options` 即初始化类时输入的初始参数。

## What those mixins do

We are inside `src/core/instance/index.js` now.

> 补：这个文件是 `Vue` 声明所在的文件。

```javascript
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

// Vue 的声明
function Vue (options) {
	// 检测 Vue 是否通过 new 关键字调用
	// 不是则抛出警告
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  // 初始化
  // options 为 const app = new Vue({...}) 时输入的参数
  // this._init 为 initMixin(Vue) 时创建的 Vue 原型上的成员函数
  this._init(options)
}

// 这些函数在这里被调用，并传入 Vue 作为参数，问题是，这个文件什么时候被执行？被谁执行？A: 在 import Vue 时就执行，即在 new Vue() 之前就已经执行
initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```

#### `!(this instanceof Vue)` 的含义是什么？
 
我们可以从 if 的代码块中打印的警告获得提示：Vue 是一个构造函数，应当通过 new 关键字调用。 
这个警告告诉我们，`Vue()` 应当且只应当用 `new Vue()` 的方式调用。
问题来了：为什么 `!(this instanceof Vue)`  可以用于判断其调用方式呢？
 
从 new 关键字的起作用的过程说起：
1.  创建一个空的简单 JavaScript 对象（即 **`{}`**）；
2.  为步骤 1 新创建的对象添加属性 **`__proto__`**，将该属性链接至构造函数的原型对象；
3.  将步骤 1 新创建的对象作为 **`this`** 的上下文；
4.  如果该函数没有返回对象，则返回 **`this`**。
 
 ```js
 funtion simulateNew(Vue){
 	let newObj = {};
 	// 下句代码把 newObj 的原型设置为 Vue.prototype，即 newObj instanceof Vue == true
 	newObj.__proto__ = Vue.prototype;
 	// 下句代码将会把 Vue 函数内部的 this 绑定为刚刚创建的 newObj
 	// 因此，用 new 调用 Vue 构造函数时，Vue 内部的 this 将会等于 newObj，即 this instanceof Vue == true
 	return Vue.call(newObj, ...arguments);
 }
 ```

#### 另一个疑惑，为什么在 Vue 的声明之后调用这几个函数？这些函数什么时候会执行？
 
简答：在 `import Vue from 'core/index'` 时就会执行。注意，不是在 `new Vue()` 时执行，而是在 `import` 语句执行时，被引入的文件 `core/index` 就会整体被执行。
 
更详细的答案在：[[Node-import-引入语句做了什么]]

First, we will walk through those five mixins, find out what they do. Then we go into the `_init` function and see what happens when you execute `var app = new Vue({...})`.

### initMixin

> `initMixin` 给 `Vue` 的原型（`Vue.prototype`，相当于加在类上，即每一个实例上）加了一个成员函数 `_init()`

Open `./init.js`, scroll to the bottom and read definitions.

> 忘了 `mark()` 是干嘛的就看：[[01-find-the-entry#mark & measure]]

> 注意：以下的 `vm.$xx` 都是 `Vue` **实例**的属性或函数，要与加在 `Vue.prototype` 上的属性或函数（e.g. `Vue.prototype._init`）区分开。`Vue` 实例的属性或函数不与其他 `Vue` 实例共享。

```js
let uid = 0;

export function initMixin(Vue: Class<Component>) {
	// initMixin 给 Vue 的原型加了一个成员函数 _init()
	Vue.prototype._init = function (options?: Object) {
		const vm: Component = this;
		// a uid
		vm._uid = uid++;

		let startTag, endTag;
		/* istanbul ignore if */
		if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
			startTag = `vue-perf-init:${vm._uid}`;
			endTag = `vue-perf-end:${vm._uid}`;
			mark(startTag); // 记录 init 开始时间戳
		}

		// a flag to avoid this being observed
		vm._isVue = true; // ? 【#fixme 这个属性干啥的，啥叫避免被 observed】
		// merge options 合并 options
		// 噢！把 this._init(options) 输入的新的 options 与原本 Vue 已经有的旧的 options 合并
		if (options && options._isComponent) {
		// 优化内部组件的实例化过程？
		// 因为动态 options 合并很慢，所以使用 initInternalComponent 用于优化合并过程？ 
		// 【#fixme 什么叫动态 options 合并】
			// optimize internal component instantiation
			// since dynamic options merging is pretty slow, and none of the
			// internal component options needs special treatment.
			initInternalComponent(vm, options);
		} else {
			vm.$options = mergeOptions(
				resolveConstructorOptions(vm.constructor),
				options || {},
				vm
			);
		}
		/* istanbul ignore else */
		if (process.env.NODE_ENV !== 'production') {
			initProxy(vm);
		} else {
			vm._renderProxy = vm;
		}
		// expose real self
		vm._self = vm;
		initLifecycle(vm);
		initEvents(vm);
		initRender(vm);
		callHook(vm, 'beforeCreate');
		initInjections(vm); // resolve injections before data/props
		initState(vm);
		initProvide(vm); // resolve provide after data/props
		callHook(vm, 'created');

		/* istanbul ignore if */
		if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
			vm._name = formatComponentName(vm, false);
			mark(endTag);
			measure(`${vm._name} init`, startTag, endTag);
		}

		if (vm.$options.el) {
			vm.$mount(vm.$options.el);
		}
	};
}
```

This file defines:

- function `initMixin()`, it defines `Vue.prototype._init`, we will come back at next section
- function `initInternalComponent()`, its comments implies that this function can speed up internal component instantiation because dynamic options merging is pretty slow

```js
function initInternalComponent(
	vm: Component,
	options: InternalComponentOptions
) {
	// 首先是用 Object.create 克隆了 vm.constructor（就是 Vue()）
	// 【 #fixme vm.constructor.options 是啥？vm.constructor 不是构造函数吗？它哪里来的 options？】
	// opts 是 vm.$options 的指针，也就是说，下面更改 opts 就是更改 vm.$options
	const opts = (vm.$options = Object.create(vm.constructor.options));
	// doing this because it's faster than dynamic enumeration.
	// 【 #fixme dynamic enumeration 难道是说 for (let prop in opts)？是说直接一个一个写比用 for 循环枚举更快？】
	opts.parent = options.parent;
	opts.propsData = options.propsData;
	opts._parentVnode = options._parentVnode;
	opts._parentListeners = options._parentListeners;
	opts._renderChildren = options._renderChildren;
	opts._componentTag = options._componentTag;
	opts._parentElm = options._parentElm;
	opts._refElm = options._refElm;
	if (options.render) {
		opts.render = options.render;
		opts.staticRenderFns = options.staticRenderFns;
	}
}
```

- function `resolveConstructorOptions()`, it collects options
- function `resolveModifiedOptions()`, this is related to [a bug](https://github.com/vuejs/vue/issues/4976). In short, it can let you modify or attach options during hot-reload
- function `dedupe()`, used by `resolveModifiedOptions` to ensure lifecycle hooks won't be duplicated

### stateMixin

Open `./state,js`, this is a long file, search `statemixin`.

```javascript
export function stateMixin (Vue: Class<Component>) {
  // flow somehow has problems with directly declared definition object
  // when using Object.defineProperty, so we have to procedurally build up
  // the object here.
  const dataDef = {}
  dataDef.get = function () { return this._data }
  const propsDef = {}
  propsDef.get = function () { return this._props }
  if (process.env.NODE_ENV !== 'production') {
    dataDef.set = function (newData: Object) {
      warn(
        'Avoid replacing instance root $data. ' +
        'Use nested data properties instead.',
        this
      )
    }
    propsDef.set = function () {
      warn(`$props is readonly.`, this)
    }
  }
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)

  Vue.prototype.$set = set
  Vue.prototype.$delete = del

  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: Function,
    options?: Object
  ): Function {
    const vm: Component = this
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if (options.immediate) {
      cb.call(vm, watcher.value)
    }
    return function unwatchFn () {
      watcher.teardown()
    }
  }
}
```

This function defines:

- `dataDef` and it's getter
- `propsDef` and it's getter
- setters for `dataDef` and `propsDef` if not built for production, which just log two warnings
- `dataDef` to `Vue.prototype` as `$data`
- `propsDef` to `Vue.prototype` as `$props`
- `Vue.prototype.$set` and `Vue.prototype.$delete`
- `Vue.prototype.$watch`

Sounds familiar? Yes, here we get `$data`, `$props`, `$set`, `$delete` and `$watch`. Read this file carefully, you can learn some coding skills and use them in your own projects.

Have you noticed the `Watcher`? Seems like an important class! You are right, in later articles we will explain `Observer`, `Dep` and `Watcher`. They cooperate to implement data and view synchronization.【数据-视图双向绑定】

### eventsMixin

Open `./events.js` and search `eventsMixin`, it's too long to put the screenshot here, so read it by yourself.

This function defines:

- `Vue.prototype.$on`
- `Vue.prototype.$once`
- `Vue.prototype.$off`
- `Vue.prototype.$emit`

You must have used them for many times, just read the code and learn how to implement event manipulation elegantly.

### lifecycleMixin

Open `lifecycle.js`, scroll down to find `lifecycleMixin`.

This function defines:

- `Vue.prototype._update()`, DOM updating happens here! Will be covered in later articles
- `Vue.prototype.$forceUpdate()`
- `Vue.prototype.$destroy()`

Hey, what's that below `$destroy`? `mountComponent`! We have seen it before, it's the core of `$mount` and is wrapped twice.

Keep going, we get several functions about the component. They are used in DOM updating, just ignore them now.

### renderMixin

Open `./render.js`, it defines `Vue.prototype._render()` and some helpers. They will also appear in later articles, just keep in mind that we meet `_render` here.

---

Okay, so now we understand what those mixins do, they just set some functions to `Vue.prototype`.

![](http://i.imgur.com/MhqgVXP.jpg)

The important thing here is how to divide and organize a bunch of functions. How many parts would you make if you are the author? Which part should one function go? Think from the point of author's view, it's very interesting and helpful.

## Understand the init process

After looking the static parts, now we go back to the core.

```javascript
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
```

Here we have a small demo from Vue's official document:

```javascript
var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
})
```

Now load them to your mind and let's begin the execution.

First, we call `new Vue({...})`, which means:

```javascript
options = {
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
}
```

Then `this._init(options)`. Remember where `_init()` is defined? Yes, in `./init.js`, open it and read the function codes.

`_init()` do:

- set `_uid`
- mark performance start tag 
- set `_isVue`
- set options
- set `_renderProxy`. Proxy is used during development to show you more render information
- set `_self`
- call a bunch of init functions
- mark performance end tag
- call `$mount()` to update DOM

Now we focus on those init functions.

```javascript
initLifecycle(vm)
initEvents(vm)
initRender(vm)
callHook(vm, 'beforeCreate')
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
callHook(vm, 'created')
```

`callHook()` is easy to understand, it just calls your hook functions. Next, we will explain the other six init functions in detail.

### initLifecycle

It's located in `./lifecycle`. 

This function connects this component with its parent, initializes some variables used in lifecycle methods.

### initEvents

It's located in `./events.js`.

This function initializes variables and updates them with its parent's listeners.

### initRender

It's located in `./render.js`.

This function initializes `_vnode`, `_staticTrees` and some other variables and methods.

Here we meet VNode for the first time. 

What is VNode? It's used to build the VDom. VNode and VDom correspond to real Node and DOM. The reason Vue chose these two representations is performance. 

When your data changes, Vue needs to update the webpage. The simplest way is refreshing the whole page. But it costs a lot of browser resources and much of them are just wasted. Normally you just update a few properties, why not just update the parts that change? Thus Vue adds a layer of VNode and VDom between data and view, implements an algorithm to calculate the best DOM manipulation strategy and apply that to the DOM.

We will talk about render and update later.

### initInjections

It's located in `./inject.js`.

This function is short and simple, it just resolves the injections in options and set them to your component.

But wait, what's that, is it `defineProperty`? No, it's `defineReactive`. The word `reactive` must remind you of something, Vue can update view automatically while data change, maybe we can find something related to that inside this function, let's go.

Open `../observer/index.js` and search `defineReactive`.

This function first defines `const dep = new Dep()`, does some validation, then extracts the getter and setter.

```javascript
let childOb = observe(val) // <-- IMPORTANT
Object.defineProperty(obj, key, {
  enumerable: true,
  configurable: true,
  get: function reactiveGetter () {
    const value = getter ? getter.call(obj) : val
    if (Dep.target) {
      dep.depend() // <-- IMPORTANT
      if (childOb) {
        childOb.dep.depend() // <-- IMPORTANT
      }
      if (Array.isArray(value)) {
        dependArray(value)
      }
    }
    return value
  },
  set: function reactiveSetter (newVal) {
    const value = getter ? getter.call(obj) : val
    /* eslint-disable no-self-compare */
    if (newVal === value || (newVal !== newVal && value !== value)) {
      return
    }
    /* eslint-enable no-self-compare */
    if (process.env.NODE_ENV !== 'production' && customSetter) {
      customSetter()
    }
    if (setter) {
      setter.call(obj, newVal)
    } else {
      val = newVal
    }
    childOb = observe(newVal) // <-- IMPORTANT
    dep.notify() // <-- IMPORTANT
  }
})
```

Next it defines `childOb = observe(val)`, set a new property to our component.

I have marked the key parts in comment. Even if you haven't read the related code, you can tell how Vue do view updating while data changes. It just wraps value with getter and setter inside which it constructs dependency and sends notify.

This `defineReactive` function is used in many places, not only in initInjections, we will talk about `Observer`, `Dep` and `Watcher` in later articles, just go back to our initialization and go on.

### initState

It's located in `./state.js`.

```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch) initWatch(vm, opts.watch)
}
```

Old friends again. Here we get our props, methods, data, computed properties and watch functions. Let's check them one by one.

#### initProps

Do some validations and use `defineReactive` to wrap props and set them to the component.

#### initMethods

Just set methods to the component.

#### initData

More validations, but here it uses `proxy` to set data, search `proxy` and read it, you know it will proxy operations like `this.name` to `this._data['name']`. 

At the end, this function calls `observe(data, true) /* asRootData */`. We will talk about Observer in later articles, here I just want to explain that `true`. Each observed object has a property called `vmCount` which means how many components use this object as root data. If you call `observe` with `true`, it will call `vmCount++`. The default value of `vmCount` is `0`.

Maybe this property will be used in other process, but after a global search, I just find Vue uses it as a sign for root data. If `ob && ob.vmCount`, this object must be a root data.

#### initComputed

This function first extracts the function you put in as a getter, then creates a Watcher with it and store in `watchers` array. Finally, it calls `defineComputed` to set this computed property to the component.

You can guess what the name Watcher means, but we will leave it to later articles.

#### initWatch

This function creates watcher for each input uses `createWatcher()`. `createWatcher()` calls `vm.$watch()` which is defined in `stateMixin`. Scroll to the end to see it. `vm.$watch()` will create Watcher and...What?! It may call `createWatcher()` again! What the hell is it doing?

Read it carefully, if `cb` is plain object, `$watch()` will call `createWatcher()`. And inside `createWatcher()`, it will extract options and handler if the handler(in `$watch()` it's named `cb`) is plain object. Alright, `$watch()` just through it back because he doesn't want to do extra jobs.

### initProvider

It's located also in `./inject.js`.

This function extracts providers in options and calls them on the component.

Have you noticed the comments after `initInjections` and `initProvider`? It says:

```javascript
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
```

Why do they have this order? I'll let you answer this.

---

That's all. Hard to remember all inits? I made a picture for you:

![](http://i.imgur.com/DImNrXn.jpg)

This article is a little long and contains many details. Init process is the basement of later articles, so make sure you have understood all contents.

I'm not intend to tell you everything, so I suggest you read the whole init process again and go into the implementation of unfamiliar functions to see how they work.

## Next Step

This article shows the whole initialization process. After initialization, data can be modified, views will also sync with data. How does Vue implement the data updating process? Next article will reveal it.

Read next chapter: [Dynamic Data - Observer, Dep and Watcher](https://github.com/numbbbbb/read-vue-source-code/blob/master/04-dynamic-data-observer-dep-and-watcher.md).

## Practice

```javascript
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
```

Why do they have this order? I'll let you answer this.

Hint: think from another side, what will happen if you change their order?

