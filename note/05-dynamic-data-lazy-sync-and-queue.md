# Dynamic Data - Lazy, Sync and Queue

This article belongs to the series [Read Vue Source Code](https://github.com/numbbbbb/read-vue-source-code).

In this article, we will see:

- Update Watcher in three ways 更新 Watcher 的三种方式
- How to trigger view updating 视图更新
- How to keep the updating order 保证更新顺序

## Update Watcher in Three Ways

In [[04-dynamic-data-observer-dep-and-watcher]] we have learned how Vue builds dynamic data net with Observer, Dep and Watcher. But that's just a glance, there are several important things we need to talk.

Go back to `./watcher.js` again.

In [[04-dynamic-data-observer-dep-and-watcher]] we just simulate the init process, now let's talk about updating.

Remember that when you update a reactive property, the setter will be called and it will call `dep.notify()` which calls `update()` of its subscribers(they are Watchers).

So we can go directly to `update()`.

### update()

```javascript
/**
 * Subscriber interface.
 * Will be called when a dependency changes.
 */
update () {
  /* istanbul ignore else */
  // 三个 if-else-if-else 对应更新 Watcher 的三种方式
  if (this.lazy) {
    this.dirty = true
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

This `if-else` statement has three clauses, let's look through them one by one.

### Lazy

If this watcher is lazy according to the options you pass in during initialization, it just marks itself dirty.

Let's find out where `dirty` is used.

Search `dirty`, you got:

```javascript
/**
 * Evaluate the value of the watcher.
 * This only gets called for lazy watchers.
 */
evaluate () {
  this.value = this.get()
  this.dirty = false
}
```

When `evaluate()` is called, it calls `this.get()` to get the real value and set `dirty` to `false`. Where is `evaluate()` called? Let's do a global search.

If you use Sublime Text, right click on `src` directory and choose `Find in Folder...`.

![](http://i.imgur.com/sO4k7GQ.jpg)

Then enter `evaluate` and click `Find`.

![](http://i.imgur.com/0qtiIU8.jpg)

The first result calls `watcher.evaluate()`, double click that line to jump to that file.

![](http://i.imgur.com/qcP4R3w.jpg)

Code: 

```javascript
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) { // 订阅的对象可能已经改变
        watcher.evaluate() // 获取新值 this.value = this.get()
      } // 非 dirty，那就直接返回旧值
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value // 返回值
    }
  }
}
```

The watcher is dirty, 当且仅当此观察者订阅的某个响应式属性的 `setter` 被调用过（`dep.notify()`）。因为如果要改变响应式属性的值，其 `setter` 必然会被调用，`setter` 被调用，`dep.notify()` 随着被调用，然后调用 `watcher.update()`，`watcher.dirty` 就会被设置为 `true`。

反之，如果订阅的响应式属性没有调用过 `setter`，说明它的值一定没有变，那么就不需要调用 `evaluate()`（`this.value = this.get`） 来更新新值，直接用旧值 `this.value` （`return watcher.value`）就好了。

Got it. When computed property's getter is called, if the watcher is dirty, it will do the evaluation. Use lazy mode can put off the evaluation until you really need the value.

### Sync

Go back to our second `if-else` clause.

```javascript
/**
 * Subscriber interface.
 * Will be called when a dependency changes.
 */
update () {
  /* istanbul ignore else */
  if (this.lazy) {
    this.dirty = true
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

If this watcher's `sync` is `true`, it will call `this.run()`. Search `run`.

```javascript
/**
 * Scheduler job interface.
 * Will be called by the scheduler.
 */
run () {
  if (this.active) {
    const value = this.get()
    if (
      value !== this.value ||
      // Deep watchers and watchers on Object/Arrays should fire even
      // when the value is the same, because the value may
      // have mutated.
      isObject(value) ||
      this.deep
    ) {
      // set new value
      const oldValue = this.value
      this.value = value
      if (this.user) { // 若不是 render watcher i.e. computed watcher or watch watcher
        try {
          this.cb.call(this.vm, value, oldValue) // 调用用户定义的回调函数
        } catch (e) {
          handleError(e, this.vm, `callback for watcher "${this.expression}"`)
        }
      } else {
        this.cb.call(this.vm, value, oldValue)
      }
    }
  }
}
```

This function calls `this.get()`. If the value changes or it's an object or this watcher is deep, the old value will be replaced and the callback function will be called.

> 关于 `this.active` 多说一嘴：这个变量表示当前 Watcher 实例是否被激活。什么叫被激活？我们可以在源码中查找 `this.active` 的用法：在 destroy 生命周期中，当前组件的所有 `watcher` 会调用 `teardown()`，which will 把 watcher 从其所有依赖关系 dep 中删掉，同时 watcher 的 `active` 会被设置为 `false`。
> 
> i.e. 只有属于正在“运行”的组件的 `watcher` 才是被激活状态。这样才合理——根本没有加载在用户页面上的组件的 `watcher` 们当然不需要响应任何更新。

In [previous article](https://github.com/numbbbbb/read-vue-source-code/blob/master/04-dynamic-data-observer-dep-and-watcher.md) we have learned how `this.get()` works, you can read it again if you forget.

Sync mode is easy to understand, but unfortunately, it's `false` by default. The most frequently used mode is Async.

### Queue

```javascript
/**
 * Subscriber interface.
 * Will be called when a dependency changes.
 */
update () {
  /* istanbul ignore else */
  if (this.lazy) {
    this.dirty = true
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

If your watcher is neither lazy nor sync, the execution will flow to `queueWatcher(this)`.

> `./src/core/observer/scheduler.js`

```javascript
/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 */
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) { // flushing a queue means 把当前队列的待办事项全部处理掉
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) { // 保证 queue id 升序
        i--
      } // i 会停在小于等于 watcher 的最右边元素的位置，watcher 在 queue 
      // 中的正确位置，就是 i 的右边一位
      // e.g. queue = 1,3,5 watcher = 2
      // i 将等于 0, watcher 将插入下标 0+1=1，queue 变为 1,2,3,5
      queue.splice(i + 1, 0, watcher) // 按顺序插入，queue 保持 id 升序
    }
    // queue the flush
    // waiting flags whether current flushSchedulerQueue already in the nextTick
    if (!waiting) { // queueWatcher 可能被 call 多次
      waiting = true // waiting flag variable 用于：
      nextTick(flushSchedulerQueue) // 保证只 call 一次 nextTick
    }
  }
}
```

If the queue isn't flushing now, it simply pushes the watcher into the queue.

If the queue is flushing, it will find the right position of this watcher based on its id. 

Finally, if we are not waiting, calls `flushSchedulerQueue()` at nextTick.

Here we meet two flags: `flushing` and `waiting`. Seems they are very similar, why should we use two flags?

We can do a reverse think. What if we only have `flushing`? 

`flushing` will be set to `true` when `flushSchedulerQueue()` is executed. Oh, notice that `flushSchedulerQueue()` is called with `nextTick()`, thus it won't be executed now. If we call `queueWatcher()` multiple times, there will be duplicated `flushSchedulerQueue()` at nextTick!

That's it. **`flushing` marks whether the tasks in the queue is executing, `waiting` marks whether the flush operation is placed at nextTick.**

## How to Trigger View Updating

Now we know how watchers update their value, but hey, watchers are used for computed properties and watch callbacks, how do our views update when reactive properties change?

There are no reasons for Vue to implement another dynamic data process, it should reuse Watcher for view updating. But we haven't seen any watchers created for view updating.

Let's use global searching again. What's the keyword? Remember we have met `_update` and `_render` in init process, let's try `_update` first.

![](http://i.imgur.com/SCB7qkC.jpg)

Seems the `updateComponent` in the first result is what we need, double click it.

> `./src/core/instance/lifecycle.js`，以下代码片段截自  `mountComponent()`

![](http://i.imgur.com/2V83kfm.jpg)

Code:

```javascript
  } else {
    updateComponent = () => {
    // vm._render() 返回 vnode
    // 以下代码的执行顺序：
    // 先 _render() 获得 vnode
    // 然后执行 vm._update(vnode, hydrating)
    vm._update(vm._render(), hydrating)
  }
}

vm._watcher = new Watcher(vm, updateComponent, noop)
hydrating = false
```

Here it is! We are right, Vue creates a Watcher for `updateComponent`. These lines are inside `mountComponent`, and `mountComponent` is the core of `$mount`. So after initializing the component, Vue will call `$mount` and inside it, the Watcher is created.

When a new Watcher is created, it's `lazy` is `false` by default, so at the end of the constructor, it will call `get()` and build the whole dynamic data net.

Notice that `updateComponent` is the second parameter, and it will become the `getter` of the Watcher. So when Vue tries to get this Watcher's value【因为 `Watcher` 的 `get` 函数每次都会调用 `this.getter`，which in this case is `updateComponent`】, it will update the view. If it's hard to understand, you can treat this getter as a wrapper of the real getter, it updates the view after calling the real getter(the `vm._render()`).

A little tricky, but it works well.

## How to Keep The Updating Order

After learning initialization and data updating process, we can try to solve a complicated problem.

How to make sure all data and views update in the correct order?

Let's see a small example:

```js
<div id="app">
  {{ newName }}
</div>

var app = new Vue({
  el: '#app',
  data: {
    name: 'foo'
  },
  computed: {
    newName () {
      return this.name + 'new!'
    }
  }
})
```

In this example, we have one property, one computed property. And we display the computed property in view.

After initialization, we have one reactive property【`$data.name`】 and two watchers【1）render watcher for `#app`；2）computed watcher for `newName`】 subscribe to it. Notice the view doesn't subscribe to the computed property because the computed property is a watcher, not a reactive property.

Now we change the value of `name`, we know that the computed property and view will both update. But would them update in the correct order? If view updates first, it will show the old computed property value.

How Vue solves this problem?

Let's simulate the updating process.

When `name` changes【`setter` 被调用】, it will call `dep.notify()`. `notify()` will iterate it's subscriber array and call their `update()`. By default, the watcher's `lazy` and `sync` are both `false`, so two tasks will be pushed to queue.

Okay, the key is the task order.

Read `flushSchedulerQueue` again, we can find there are a sort call and some comments【showing below】. Before running all tasks, the queue will sort them based on their `id`. Recall that `$mount` is the last operation during initialization, we know that **the computed watcher is created before rendering watcher**, so it's `id` is smaller, thus it's executed early.

```js
 // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
```

> Queue introduces a new problem: if you use the queue and read computed property right after changing the data it depends, you will get old value. However, after global searching, I found that `sync` is always `true`, so seems the queue is never used.【what？不是吧？ #fixme 】

You see, **the updating order is set based on initialization order**, now you know why we have to learn init process first.

You may ask, what would happen if the computed watcher is `lazy`? I will leave this to you.

## Next Step

We have learned three updating ways and how to keep the correct updating order. But these all happen "inside", how does Vue apply the updating to DOM? How to convert your `.vue` files into browser executable code? Next several articles will talk about the entire render process.

Read next chapter: [[06-view-render-introduction]]。

## Practice ✅

Try to tell how Vue keeps the correct updating order if the computed watcher is `lazy`.

Hint: simulate the updating process by yourself.

回答：以计算属性为例。

computed watcher 的 `lazy` 默认为 `true`（from: [Computed Properties and Watchers — Vue.js (vuejs.org)](https://v2.vuejs.org/v2/guide/computed.html))：

> **computed properties are cached based on their reactive dependencies.** A computed property will only re-evaluate when some of its reactive dependencies have changed. This means as long as `message` has not changed, multiple access to the `reversedMessage` computed property will immediately return the previously computed result without having to run the function again.

举例说明：

```html
<template>
  <div>
    <ul>
      <li>
        <span>{{ item.name }}</span>
        <span>{{ item.quantity }}</span>
        <span>{{ item.unitPrice }}</span>
        <span>{{ item.totalPrice }}</span>
      </li>
    </ul>
  </div>
</template>

<script>
export default {
  data() {
    return {
      item: { name: 'Item 1', quantity: 2, unitPrice: 10 },
    };
  },
  computed: {
    totalPrice() {
      return this.item.quantity * this.item.unitPrice;
    },
  },
};
</script>
```

在上面的例子中，总价格 `totalPrice` 是一个计算属性，更新它的最佳时机应当是物品的价格或数量发生变化时。

假设计算属性没有懒更新（`lazy = false`），想象一下会发生什么情况？假设物品的 `name` 改变，没必要更新的 `totalPrice` 也会随着更新。

如果 `lazy = true` 就不会出现这种情况了。当物品的 `name` 改变，`totalPrice` 的 `watcher` 实例的 `dirty` 仍然等于 `false`（因为 `price` 的 `setter` 没有被调用），执行计算属性 `price` 的 `getter` 时：

```js
function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) { // 订阅的对象可能已经改变
        watcher.evaluate() // 获取新值 this.value = this.get()
      } // 非 dirty，那就直接返回旧值
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value // 返回值
    }
  }
```

不会进入 `if(watcher.diry)` 代码块，不会执行 `watcher.evaluate()`，而是直接返回 `watcher.value`。

什么时候会执行 `watcher.evaluate()` 呢？

当物品的 `unitPrice` 或 `quantity` 改变，关联的 `watcher` 会被通知（`dep.notify`），然后 `watcher.dirty` 被设置为 `true`。然后，再调用计算属性的 `getter`，就会执行 `watcher.evaluate()`，在其内部调用 `watcher.get()`，获取计算属性 `totalPrice` 的新值。