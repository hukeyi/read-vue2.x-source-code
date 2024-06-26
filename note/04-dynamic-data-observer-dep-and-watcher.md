# Dynamic Data - Observer, Dep and Watcher

In this article, we will learn:

- Observer
- Dep
- Watcher
- How they cooperate

In [[03-init-introduction]], we have learned how Vue does the initialization. After init, many interesting things happen.

For example, if you change one of your properties `name`, then your webpage is automatically updated with the new value.

How to implement that? You will see in this article.

I won't give you the entire structure now, cause I want to show you how I build that through reading source code.

> 本文所有代码文件都位于 `src/core/observer`

## Observer

In the previous article, we have seen `defineReactive` which is used to make a property `reactive`. Let's see its usage in `defineReactive()`.

> 位于 `src/core/observer/index.js`

### defineReactive()

```javascript
/**
 * Define a reactive property on an Object.
 */
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: Function
) {
  const dep = new Dep() // #fixme 这里的 dep 是闭包吗

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  // 获得 val 的 observer 实例；如果存在，则用旧的；如果不存在，则创建新的返回
  // childOb 就是 val 的 observer
  let childOb = observe(val)  // <-- IMPORTANT
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
	// 思考 getter 和 setter 作用的时候，
	// 应当结合「实际情境」想象——什么时候会用到这个 getter/setter？
	// 比如，当有一个组件是 `<span>{{ msg }}</span>` 时，
	// obj 是 $data，val 是 msg ⬅️ #fixme 这个想法对吗
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        // dep.depend(): assign Dep.target(current running watcher) to dep.subs
        // #fixme 每次都要 assign 吗？为什么要在每次获取 reactive 属性时 assgin？
        // 不可以在初始化的时候 assign 然后一劳永逸吗
        dep.depend() // <-- IMPORTANT
        if (childOb) {
        // dep.depend() 表示当前 running 的 watcher 订阅了(depend on) dep
        // 类比社媒的【添加关注】
          childOb.dep.depend() // <-- IMPORTANT
        }
        if (Array.isArray(value)) {
          dependArray(value) // <-- IMPORTANT
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
}
```

Here are the key points:

```javascript
const dep = new Dep()
let childOb = observe(val)

...
  dep.depend()
  childOb.dep.depend()
  dependArray(value)
  
...
  childOb = observe(newVal)
  dep.notify()
```

`getter` 收集依赖关系（`dep.depend()`）；`setter` 通知各个 Watcher 更新（`dep.notify()`）。

Here we meet `Dep`, `observe()`, `dependArray()`, `depend()` and `notify()`.

It's clear that `observe()` and `dependArray()` are helpers, let's read them first.

### observe()

```javascript
/**
 * Attempt to create an observer instance for a value,
 * returns the new observer if successfully observed,
 * or the existing observer if the value already has one.
 */
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value)) {
    return
  }
  let ob: Observer | void
  // hasOwn <=> hasOwnProperty(obj, key)
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) { // 已存在一个 Observer 实例
    ob = value.__ob__
  } else if (
    observerState.shouldConvert &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) { // 无 Observer 实例，new 一个
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob // 返回 Observer 实例
}
```

> 补：[`Object.isExtensible()`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/isExtensible)，判断一个对象是否可扩展/是否可在其上添加新属性。

`observe()` 会返回一个 `Observer` 实例：

- 若实例已存在，直接返回该实例
- 若不存在，new 一个并返回

`observe()` will extract the exist observer or create a new one with `new Observer(value)`. Notice that observe only works for an object, primitive value won't be observed.

```js
// primitive value won't be observed.
if (!isObject(value)){
	return
}
```

#### 补：primitive vs object

> [primitive value](https://developer.mozilla.org/en-US/docs/Glossary/Primitive)：基本类型，In Javascript, a **primitive** (primitive value, primitive data type) is data that is not an object and has no methods or properties. 
> 
> Diffrence between 'primitive value' with 'literal'? Relationship? #fixme 

vue 检查是否为 primitive 的代码（在 `shared/util`）：

```js
/**
 * Check if value is primitive
 */
export function isPrimitive (value: any): boolean %checks {
  return typeof value === 'string' || typeof value === 'number'
}
```

检查是否为对象：

> plain：朴素的；绝对的；简单的。

```js
/**
 * Quick object check - this is primarily used to tell
 * Objects from primitive values when we know the value
 * is a JSON-compliant type.
 */
export function isObject (obj: mixed): boolean %checks {
  return obj !== null && typeof obj === 'object'
}

const _toString = Object.prototype.toString

/**
 * Strict object type check. Only returns true
 * for plain JavaScript objects.
 */
export function isPlainObject (obj: any): boolean {
  return _toString.call(obj) === '[object Object]'
}
```

> `isPlainObject` 什么时候返回 `false`？
> => `Object.prototype.toString.call(obj)` 什么情况下不返回 `[object Object]`？  #fixme 

If this value is used as root data, it will increments `ob.vmCount++`, we have talked about that in init process.

### dependArray()

Okay, now we have got or created the watcher. Next, `dependArray()`.

```javascript
/**
 * Collect dependencies on array elements when the array is touched, since
 * we cannot intercept array element access like property getters. => 我们可以用 getter 拦截对对象属性的访问，但是没办法拦截对数组元素的访问。
 */
function dependArray (value: Array<any>) {
  for (let e, i = 0, l = value.length; i < l; i++) {
    e = value[i]
    e && e.__ob__ && e.__ob__.dep.depend()
    if (Array.isArray(e)) { // 处理嵌套数组，向下递归
      dependArray(e)
    }
  }
}
```

It just iterates the array recursively and calls `e.__ob__.dep.depend()` which leads us to `depend()` again.

So now we have found the usage of `Dep()`, `Observer()`, `Watcher()`. And `dep.depend()`, `dep.notify()`.

If you use `defineReactive()` to convert a property, that reactive property has one `dep` and one `childOb` set by `observe(val)` if the value is object.

`defineReactive(obj, key)` 后，`obj` 对象的 `key` 属性将会获得：

- 一个 `Dep` 实例 `dep`
- 一个 `Observer` 实例 `childOb`

### class Observer

Let's read `Observer()` now.

```javascript
/**
 * Observer class that are attached to each observed
 * object. Once attached, the observer converts target
 * object's property keys into getter/setters that
 * collect dependencies and dispatches updates.
 */
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    this.value = value // 监听的目标对象
    this.dep = new Dep() // 依赖，用于分发对象的数据更新
    this.vmCount = 0 // 使用该对象作为根数据的 vue 实例个数
    
    // 对象本身会添加一个新属性 `__ob__`
    // 此属性指向监听该对象的 observer 实例
    // 因此，获取对象 value 的 observer 实例的方法就是 value.__ob__
    // 也就是说，通过 let ob = new Observer(value) 这一行代码，value 就绑定上 observer 实例了
    // i.e. 被监听对象存储了监听它的 observer 实例；observer 实例也存储了其监听目标的地址。两者之间是双向的 has-a 关系。
    def(value, '__ob__', this)

	// 增强数组，深度监听数组
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
      this.observeArray(value)
    } else { // 深度监听对象
      this.walk(value)
    }
  }

  /**
   * 深度监听对象各属性
   * Walk through each property and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i], obj[keys[i]])
    }
  }

  /**
   * 深度监听数组元素
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

#### def(obj, key, val) defineProperty 封装

> 把对象 `obj` 的属性 `key` 的值设置为 `val`，同时设置其他的数据属性描述符。

It first defines a `__ob__` property to the value you pass in.

`def(value, '__ob__', this)`，给 `value` 添加一个属性 `__ob__`，该属性的值为当前的 `Observer` 实例。 #util 

> `def()` 声明在 `src/core/util/lang.js`

```js
/**
 * Define a property.
 * 为对象属性设置属性数据描述符
 */
export function def (obj: Object, key: string, val: any, enumerable?: boolean) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable, // #fixme 使用 !! 的原因是？
    writable: true,
    configurable: true
  })
}
```

#### hasProto

`hasProto`，为布尔变量，表示当前环境是否可以使用 `__proto__`。

> `__proto__` 存储的是当前对象的原型地址，记忆为“我的原型“。

为什么要做这个判断？因为如今 `__proto__` 属性[已不再推荐使用](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/proto)，未来可能会从标准中删掉，部分平台仍支持，部分平台不支持。

```js
// can we use __proto__?
const hasProto = '__proto__' in {}
```

#### augment & observeArray() 数组监听

```js
if (Array.isArray(value)) {
  // 数组增强
  const augment = hasProto
	? protoAugment
	: copyAugment
  augment(value, arrayMethods, arrayKeys)
  this.observeArray(value)
}
```

Then if the value is an array, it will intercept array methods(like `push`, `pop`) to make sure Vue can detect array manipulation.  => 拦截数组的成员函数，实现 Vue 对数组操作的监听。

`arrayMethods` 引用自 `observer/array.js`（数组专门的 observer 处理文件）。它是重新定义了 Array 的各种数组操作的克隆的**原型**。（它是 `Array.prototype` 的克隆原型，它存储的各种数组操作比 `Array.prototype` 多了 observe 和 dep.notify，就是会被 vue 监听的数组操作）

打开 `array.js`：

```js
import { def } from '../util/index'

const arrayProto = Array.prototype
// arrayMethods 是 Array.prototype 的克隆
// 为什么使用克隆的原型？
// 避免污染原生数组原型的成员函数
// 如果不使用克隆的原型，整个作用域中的数组的 push 等方法都将
// 被替代为当前文件重新定义的方法。
// 这不是我们想要的效果。本文件中重新定义的方法只应当使用在需要 reactive 的数组上
export const arrayMethods = Object.create(arrayProto)

/**
 * Intercept mutating methods and emit events
 */
// 枚举所有数组方法
;['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse']
.forEach(function (method) {
  // cache original method
  // 原生方法用于获得方法返回值
  const original = arrayProto[method]
  // 给重新定义的数组方法设置触发器/响应
  def(arrayMethods, method, function mutator () {
    // avoid leaking arguments:
    // http://jsperf.com/closure-with-arguments

	// 把参数类数组对象转换为 Array
	// arguments 哪来的？
	// [...].push(items)
	// arguments 就是这个 items
	// 后面用到的 this 就是 [...]
    let i = arguments.length
    const args = new Array(i)
    while (i--) {
      args[i] = arguments[i]
    }
    const result = original.apply(this, args)
    const ob = this.__ob__ // 获得当前数组对象的 Observer 实例
    let inserted // 是否有插入的新数组元素
    // 有三个函数可能有插入新数组元素
    switch (method) {
      case 'push':
        inserted = args // push 方法的所有参数都是待插入的新元素
        break
      case 'unshift':
        inserted = args // unshift 同上
        break
      case 'splice':
        inserted = args.slice(2) // splice(pos, delCount, items...)
        // splice 从第三个元素/下标 2 开始才是插入的新元素
        break
    }
    // 如果有待插入的新元素，那么给新元素加上 observe
    if (inserted) ob.observeArray(inserted)
    // notify change 触发更新，这里是 vue 监听数组操作的关键
    ob.dep.notify()
    return result // 返回数组操作的结果
  })
})
```

对于可以使用 `__proto__` 的环境，强行修改原型链。直接把数组的原型强制改变为我们重新定义的 `arrayMethods`：

```js
/**
 * Augment an target Object or Array by intercepting
 * the prototype chain using __proto__
 */
function protoAugment (target, src: Object) {
  /* eslint-disable no-proto */
  target.__proto__ = src
  /* eslint-enable no-proto */
}
```

不能使用 `__proto__` 的环境，无法修改原型链。

于是，利用 `Object.defineProperty` ，把 `arrayMethods` 的重新定义的数组方法添加到数组对象实例 `value` 上。

这样，数组在调用 `push` 等方法时，不需要沿着原型链向上查找 `Array.prototype`，而是直接在实例上就能获得 `push` 方法。相当于拦截了查询过程：

```js
/**
 * Augment an target Object or Array by defining
 * hidden properties.
 */
/* istanbul ignore next */
function copyAugment (target: Object, src: Object, keys: Array<string>) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}
```

After that(`augment(value, arrayMethods, arrayKeys)`), it calls `observeArray()` which will iterates items and call `observe()`.

#### walk(obj)

枚举 `obj` 上的所有属性，在每一个属性上调用 `defineReactive`。给 obj 上所有属性绑上双向绑定。

If the value is not an array, this function just walks through all keys and use `defineReactive()` to convert all values into a reactive property.

### defineReactive 作用总结

As you can see, `defineReactive()` calls `new Observer()`, `Observer()` may also call `defineReactive()`. Thus, when you want to convert a property with `defineReactive()`, it will recursively converts all sub properties into reactive property.

**To be clear, we use `defineReactive()` to create reactive PROPERTY, and we use `observe()` to create Observer for the VALUE of that PROPERTY(if the value is object).**

`defineReactive()` 用于监听对象的**属性**；
`observe()` 用于深度监听对象上**属性值类型为 Object 的属性值**。

因为，当对象属性值为 Object，实际存储的是内存地址，只要内存地址不改变，无论地址上的内容（i.e. 属性值）如何改变，都不会触发更新响应。

The reason is simple. If the value is an object, change the property of that object won't trigger the setter of property. Property only save the reference to that object in memory, change the content of that object won't affect it's memory address, thus won't really change the property's value.

If we have a `data` like this:

> `data.parents` 就是一个 `data` 对象上类型为 Object 的属性
> `{ mon: 'foomon', dad: 'foodad' }` 就是 `data.parents` 的值

```javascript
data: {
  name: 'foo',
  parents: {
    mom: 'foomom',
    dad: 'foodad'
  }
}
```

After calling `defineReactive(vm._data)`, we got this:

![](https://i.imgur.com/TM5j5GL.jpg)

Give yourself some time to fully understand it.

Our next target is `Dep()`.

## Dep

补：注释解释 dep

```css
/**
 * A dep is an observable that can have multiple
 * directives subscribing to it.
 */
```

`Dep` 实例是用户（Observer 对应的响应属性）的「订阅列表」。订阅列表可以拥有多个 `Watcher` 实例。类似社媒「用户-粉丝列表-订阅者」的关系，「observer-dep-subs（ `Watcher` 实例数组）」。

`Dep` 类有三个数据成员：

```js
static target: ?Watcher; // Dep.target 的解释在下面，代表正在被 evaluated 的 Watcher 实例对象
id: number; // dep 的 id 标识
subs: Array<Watcher>; // 订阅了 dep 的订阅者 watcher 列表
```

Open `./dep.js`, you can see this class has only four methods.

```javascript
addSub (sub: Watcher) { // 添加新订阅者
  this.subs.push(sub)
}

removeSub (sub: Watcher) { // 删除指定订阅者
  remove(this.subs, sub)
}

depend () { // 把当前 dep 对象加入正在 evaluted 的 watcher 中
  if (Dep.target) { // 当前正在 evaluated 的 Watcher 实例对象
    Dep.target.addDep(this) // 在上述 watcher 实例中添加新的订阅对象
  }
}

notify () { // 通知 subs 订阅列表中的所有订阅者，当前 dep 实例更新
  // stabilize the subscriber list first
  const subs = this.subs.slice()
  for (let i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}
```

`addSub()`, `removeSub()` and `notify()` deal with watchers. Each `Dep` instance has an array to store its watchers and tell them to `update()` during `notify()`. We have seen that `notify()` will be called in a setter, so if you change a reactive property, it will trigger watchers' updating.（`notify()` 被调用后，会触发相应订阅者的更新）

`depend()` is strange, it first checks `Dep.target`, if it exists, call `Dep.target.addDep(this)`. What is `Dep.target`?

In the comments below this class, we can learn that **`Dep.target` is globally unique. It's the watcher being evaluated now.**

`Dep.target` 是**全局变量**。它代表**当前正在被 evaluated 的 the Watcher 实例对象**（同一时间只能有一个 watcher 实例被 evaluated）：

> #todo remember to link to `coding` repo [[23-03-05-design-pattern-vue-observer-pattern-new-bing-vue#w#why we need Dep.target]] 详细解释了 Dep.target 的用处和必要性

```js
// the current target watcher being evaluated.
// this is globally unique because there could be only one
// watcher being evaluated at any time.
Dep.target = null
const targetStack = []
// 如果当前 watcher 需要另一个 watcher2 的值，那么
// 先将 watcher 入栈暂存；然后处理 watcher2，并把当前执行观察者（Dep.target）设置为 watcher2；
// 处理完 watcher 2 后，watcher 出栈，并且当前执行观察者被重置为 watcher
// 「后来者先执行」，因为最新数据应具有最高优先级；后来的数据覆盖先来的

export function pushTarget (_target: Watcher) {
  if (Dep.target) targetStack.push(Dep.target)
  Dep.target = _target
}

export function popTarget () {
  Dep.target = targetStack.pop()
}
```

Next to it are two functions for stack operations. It's easy to understand if one watcher wants to get another watcher's value during evaluation, we need to store current target, switch to the new target and come back after it finishes.

`Dep.target` must be a watcher, so `Dep.target.addDep(this)` inside `depend()` tells us watcher has a method named `addDep()`. Its name implies that each watcher also has a list of `Dep`s it's watching.

Let's turn to watcher now.

## Watcher

Open `./watcher.js`, it's a little long but...hey, we are right, Watcher has the list to store it's `Dep`s.

The `constructor` simply initials some variables, set your computed function or watch expression to `this.getter` and try to get the value if it's not lazy.

```js
export default class Watcher{
	constructor(vm, expOrFn, cb, options){
		...
		/* 一些变量初始化 */
		this.getter = expOrFn // JS 表达式 or 函数
		...
		/* 调用 get() 获得当前值 */
		this.value = this.lazy? undefined: this.get()
	}
}
```

根据初始化函数可知，下文的 `this.getter` （记为获取更新值的函数）指传入 Vue 实例的变量对应的函数。举例说明：

```js
var vm = new Vue({
	data: {
		a: 1
	},
	watch: {
		a: function(newVal, oldVal){
			console.log(`Change of varable a from ${oldVal} to ${newVal}. `);
		}
	}
})
```

上一段代码表示，`data['a']` 的 Watcher 实例就是 `vm` 的 `watch` 对象中的 `a` 传入的函数 `function(newVal, oldVal){...}`。用于获取 Watcher 观察的订阅对象更新后的新值。

> 注意 `getter` 函数千万**不能使用箭头函数**，原因： #todo note 库合并到 coding 库后在这里引入：22-12-30-js-this-箭头函数-call-apply-bind 。

Let's go on with `get()`, this is the only thing we get from `constructor()`.

```javascript
/**
 * Evaluate the getter, and re-collect dependencies.
 */
get () {
  pushTarget(this) // 把当前 watcher push 入 Dep.target 栈；
  let value
  const vm = this.vm // 当前 vue 实例
  if (this.user) { // whats this: 代表当前 watcher 是否为用户创建的 watcher
  // watcher 有三种：1) render watcher; 2) computed watcher; 3) watch watcher
  // 第一种是 vue 给的更新视图的 watcher；后两种都是用户创建的 watcher
  // 第一种 this.user = false, 后两种 this.user = true
    try {
      value = this.getter.call(vm, vm) // 获取订阅对象更新后的新值
    } catch (e) {
      handleError(e, vm, `getter for watcher "${this.expression}"`)
    }
  } else {
    value = this.getter.call(vm, vm)
  }
  // "touch" every property so they are all tracked as
  // dependencies for deep watching
  if (this.deep) {
    traverse(value) // 递归遍历 value(订阅对象的新值)
    // 为深度观察收集新的 dep
  }
  popTarget()
  this.cleanupDeps()
  return value
}
```

Remember `Dep.target`? Here it calls `pushTarget()` and `popTarget()`, and do the evaluation between them!

Imagine we have a component like this:

```javascript
{
  data: {
    name: 'foo'
  },
  computed: {
    newName () {
      return this.name + 'new!'
    }
  }
}
```

We know that `data` will be converted to reactive property, it's value, the object【i.e. `{ name: 'foo' }`】 will be observed. If you get data use `this.foo` it will be proxied to `this._data['foo']`.

Now let's try to build a watcher step-by-step【创建 watcher 实例的步骤】:

- assign our input function to getter【input function: `newName()`】
- call `this.get()`【`get()` 执行 getter 并重新获取依赖关系 deps】
- call `pushTarget(this)` which changes `Dep.target` to this watcher
- call `this.getter.call(vm, vm)`【执行 `vm.newName()`】
- **run `return this.foo + 'new!'`【执行 `vm.newName()`】**
- **because `this.foo` is proxied to `this._data[foo]`, the reactive property `_data`'s getter is triggered**
- **inside the getter, it calls `dep.depend()`**
	- inside `depend()`, it calls `Dep.target.addDep(this)`, here `this` refers to the const `dep`【i.e. `dep.depend()` 里的 `dep`】, it's `_data`'s dep
- then it calls `childOb.dep.depend()` which add the dep of `childOb` to our target. Notice this time the `this` of `Dep.target.addDep(this)` refers to `childOb.__ob__.dep`
- inside `addDep()`, the watcher add this dep to it's `this.newDepIds` and `this.newDeps`
- because the default value of `this.depIds` is `[]`, the watcher calls `dep.addSub(this)`
- inside `addSub`, the dep add the watcher to it's `this.subs`
- now the watcher has gotten the value, it will `traverse()` the value to collect dependencies, calls `popTarget()` and `this.cleanupDeps()`

After this complex process, the watcher knows its dependencies, the dep knows its subscribers, the dynamic data net is built. With this net, Dep can `notify()` its subscribers when the reactive property gets the new value, which may trigger the `get()` again and refresh the value and relations.

And what `cleanupDeps` does? After reading the code, you can tell how it works to refresh the dependence relations.

---

![](http://i.imgur.com/5BRYgfi.jpg)

Above is the initialization of dynamic data net, this can help you understand the process.

If reactive property changes, it just triggers this process again to refresh computed property value and rebuild the dynamic data net.

## Observer, Dep, Watcher 三者关系

 #important 

Observer 对应着整个 reactive system。通常是一个【响应式对象】，which holds some 响应式属性。

Dep 对应响应式对象的各个【响应式属性】。

Watcher 对应订阅了这些响应式属性的【订阅者】。Watcher 不只负责发现属性更新，还负责发现更新后，调用订阅者的更新函数。

订阅者有哪些类型？

- `computed` property
- `watch` property
- render watcher（视图更新，即更新 DOM）

用一个例子说明：

```js
const vm = new Vue({
  data: {
    showText: true,
    text: "Hello World!",
  },
  computed: {
    displayText() {
      return this.showText ? this.text : "";
    },
  },
});

vm.showText = false;
```

Vue 在初始化时，会为 `data` 创建一个 `Observer` 实例对象。初始化这个 `Observer` 实例对象时，同时又分别为 `showText` 和 `text` 两个属性创建各自的 `Dep` 实例对象（一个响应式属性对应一个）。

然后，初始化到 `computed` 时，会为 `displayText()` 创建一个 `Watcher` 实例对象（this is when things get tricky）。

创建 `Watcher` 对象，将会调用 `this.get()`。`this.get()` 内部调用 `pushTarget(this)`，将当前 `Watcher` 对象加到 `Dep.target` 调用栈中。

当前 `Watcher` 对象成为 `Dep.target` 时，就调用 `this.getter.call(vm, vm)`（就上述例子来说，`this.getter` 就是 `displayText()`）。`displayText()` 将会 `return this.showText? this.text : "";`。

重点来了！

无论 `this.showText` 为 true 还是 false，`this.showText` 都会被调用，`showText` 属性的 `getter` 函数必然会被调用（[[#defineReactive()]]），`getter` 函数除了返回属性当前值外，另一个重要功能是把 `Dep.target` 加入到当前属性（which in this case, `showText`）的 `Dep` 实例对象的 `this.subs` 订阅列表中，同时，把当前属性的 `Dep` 实例对象加入到 `Dep.target` 的 `this.depIds` 中。

而对于 `this.text`，当且仅当 `this.showText` 为 true 时，它才会被调用！也就是说，它与 `displayText()` 是否有订阅关系，是动态变化的（由 `this.showText` 的值决定）。也就是说，`displayText()` 的 `Watcher` 实例，订阅的 `Dep` 实例对象，是动态变化的！也就是说，`Watcher` 绑定的依赖关系是动态变化的！对于 `displayText()` 的 `Watcher` 来说，当 `this.showText = true`，则同时订阅 `this.showText` 和 `this.text`；当 `this.showText = false`，就只需要订阅 `this.showText`。

为什么要这样？因为当 `this.showText` 为 `false` 时，若 `this.text` 属性值变化，就不需要更新 `displayText()`。

⬆️ ==这就是为什么每一次 `Watcher` 的 `get` 或 `update` 函数被调用，都需要重新收集其依赖关系。==

虽然存在一定程度的浪费（每次 `this.get` 都要收集依赖，依赖并不是每次都变化），但这是为了维护一个可行的响应式系统所需的必要牺牲。

## Next Step

Now we know how to build the dynamic data net. Next article will focus on three watcher updating ways and discuss how Vue keeps the correct updating order.

## Practice

Read the `cleanupDeps` method in `./watcher.js` and tell how this method updates the dependency during `get()` process.

Hint: the key is those two arrays: `this.newDepIds` and `this.depIds`. You may want to read `addDep()` first.


