# View Render - Introduction

This article belongs to the series [Read Vue Source Code](https://github.com/numbbbbb/read-vue-source-code).

In this article, we will learn:

- How to find render functions
- The structure of view render

## Find Render Functions

After understanding the initialization and data updating, now we turn to the view render part to see how Vue converts our data to the real DOM nodes.

> Notice: the word "render" refers to the whole view rendering process, while the formatted `render()` or `_render()` refers to the real function.

First of all, we need to find functions related to render process.

In [[05-dynamic-data-lazy-sync-and-queue]] , we have learned that `mountComponent()` will use `_update()` and `_render()` to update views. Let's do a global search to find `_update`.

> `./src/core/instance/lifecycle.js`

```javascript
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  if (vm._isMounted) {
    callHook(vm, 'beforeUpdate')
  }
  const prevEl = vm.$el
  const prevVnode = vm._vnode
  const prevActiveInstance = activeInstance
  activeInstance = vm
  vm._vnode = vnode
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(
      vm.$el, vnode, hydrating, false /* removeOnly */,
      vm.$options._parentElm,
      vm.$options._refElm
    )
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  activeInstance = prevActiveInstance
  // update __vue__ reference
  if (prevEl) {
    prevEl.__vue__ = null
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el
  }
  // updated hook is called by the scheduler to ensure that children are
  // updated in a parent's updated hook.
}
```

`_update()` uses `__patch__()` to calculate which part needs updating and manipulate corresponding DOMs. We will talk about `__patch__()` later.

Go back to `mountComponent()`, our next target is `_render()`.

We have seen `_render()` is defined in `core/instance/render.js`, let's look at it again.

```javascript
Vue.prototype._render = function (): VNode {
  const vm: Component = this
  const {
    render,  // <-- IMPORTANT
    staticRenderFns,
    _parentVnode
  } = vm.$options // <-- IMPORTANT

  if (vm._isMounted) {
    // clone slot nodes on re-renders
    for (const key in vm.$slots) {
      vm.$slots[key] = cloneVNodes(vm.$slots[key])
    }
  }

  vm.$scopedSlots = (_parentVnode && _parentVnode.data.scopedSlots) || emptyObject

  if (staticRenderFns && !vm._staticTrees) {
    vm._staticTrees = []
  }
  // set parent vnode. this allows render functions to have access
  // to the data on the placeholder node.
  vm.$vnode = _parentVnode
  // render self
  let vnode
  try {
    vnode = render.call(vm._renderProxy, vm.$createElement) // <-- IMPORTANT
  } catch (e) {
  ...
```

Inside `_render()`, it calls `render()` which is extracted from `vm.$options`.

After global searching `$option`, we find:

```javascript
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

`options` is the parameter passed in when you call `new Vue({...})`, so `render()` must comes from `vm.constructor`. `vm` is the instance of Vue, so we try to global search `options.render`, maybe we can find where it's set.

Look through the search results, this one set a value.

![](http://i.imgur.com/RvT8WgO.jpg)

Double click it.

![](http://i.imgur.com/wVMfCcr.jpg)

Code:

```javascript
...
const { render, staticRenderFns } = compileToFunctions(template, {
  shouldDecodeNewlines,
  delimiters: options.delimiters
}, this)
options.render = render
options.staticRenderFns = staticRenderFns
...
```

Cool! This is the outmost wrapper of `$mount()`, it calls `compileToFunctions()` with your `template`, gets `render` and `staticRenderFns` as return value and set it to `options.render`.

Go on, `compileToFunctions()` comes from `./compiler/index.js`:

```javascript
const { compile, compileToFunctions } = createCompiler(baseOptions)

export { compile, compileToFunctions }
```

`createCompiler()` comes from `compiler/index.js`:

```javascript
// `createCompilerCreator` allows creating compilers that use alternative
// parser/optimizer/codegen, e.g the SSR optimizing compiler.
// Here we just export a default compiler using the default parts.
export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  const ast = parse(template.trim(), options)
  optimize(ast, options)
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})
```

`createCompilerCreator()`, good name! It's a high order function which creates the `createCompiler()` function which is used to create the compiler. Later our template will be compiled using this compiler.

After reading the code of `createCompilerCreator()`, we learn that it's just a wrapper. The core function is `baseCompile()` which uses `parse()`, `optimize()` and `generate()` to do the real jobs.

Let's make it clear:

- first, you choose your own parser, optimizer and codegen, use them to build the core compile function(or use the default one)
- then, pass your compile function to `createCompilerCreator()`, it will return a function which can be used to create the compiler, thus its name is `createCompiler()`
- next, call `createCompiler()` with options, it will return the real compiler
- finally, use that compiler to compile the input template

> 这属于抽象工厂设计模式吗？还是策略模式？ #fixme 

The purpose of this complicated process is extracting the core compiler and options. If you treat **core compiler function** and **options** as two parameters for `createCompiler()`, you can see it's similar with currying. Two parameters come at different time, so in `createCompilerCreator()` it has to create a new function to store **core compiler function** and return that function(this function is `createCompiler()`). When **options** is passed in, `createCompiler()` combines them and create the final compiler.

Now we have found all functions related to render process: `_update()`, `__patch__()`, `_render()`, `createCompiler()`, `parse()`, `optimize()`, `generate()`. Let's organize them.

## The Structure of Render

Render process starts when `mountComponent()` is called. It will call `_render()` to compile template into `render` and `staticRenderFns`, then pass them to `_update()` to calculate the operations and apply them to DOMs.

![Structure of Render](http://i.imgur.com/NM77eiy.jpg)

## Next Step

Next two articles will talk about **compiler** and **patch**, you will see how VDom is designed and how to do the diff calculation fastly.

Read next chapter: [View Rendering - Compiler](https://github.com/numbbbbb/read-vue-source-code/blob/master/07-view-render-compiler.md).

## Practice ✅

```javascript
vnode = render.call(vm._renderProxy, vm.$createElement)
```

This line is located in `_render()`, try to figure out what is `vm._renderProxy` and it's role.

`vm._renderProxy` 定义在 `./src/core/instance/proxy.js` 中。

```js
/* not type checking this file because flow doesn't play well with Proxy */

import config from 'core/config'
import { warn, makeMap } from '../util/index'

let initProxy

if (process.env.NODE_ENV !== 'production') {
  // 一些允许全局调用的全局对象，基本是些浏览器或 Node 内置的对象
  const allowedGlobals = makeMap(
    'Infinity,undefined,NaN,isFinite,isNaN,' +
    'parseFloat,parseInt,decodeURI,decodeURIComponent,encodeURI,encodeURIComponent,' +
    'Math,Number,Date,Array,Object,Boolean,String,RegExp,Map,Set,JSON,Intl,' +
    'require' // for Webpack/Browserify
  )

  const warnNonPresent = (target, key) => {
    warn(
      `Property or method "${key}" is not defined on the instance but ` +
      `referenced during render. Make sure to declare reactive data ` +
      `properties in the data option.`,
      target
    )
  }

  const hasProxy =
    typeof Proxy !== 'undefined' &&
    Proxy.toString().match(/native code/)

  if (hasProxy) {
    const isBuiltInModifier = makeMap('stop,prevent,self,ctrl,shift,alt,meta')
    config.keyCodes = new Proxy(config.keyCodes, {
      set (target, key, value) {
        if (isBuiltInModifier(key)) {
          warn(`Avoid overwriting built-in modifier in config.keyCodes: .${key}`)
          return false
        } else {
          target[key] = value
          return true
        }
      }
    })
  }

  const hasHandler = {
    has (target, key) {
      // has，target 对象中是否含有 key 属性
      const has = key in target
      // isAllowed 检查 key 是否允许直接调用
      // 允许直接调用的前提：1）浏览器或 node 的内置对象；2）vue 实例的全局属性，以 `_` 开头的
      const isAllowed = allowedGlobals(key) || key.charAt(0) === '_'
      if (!has && !isAllowed) { // 假设 key 既不在对象 target 中，又不是全局属性，则警告
        warnNonPresent(target, key)
      }
      // 1）target 中存在 key 属性 或者 2）key 属性不是全局属性
      // 就返回 true
      return has || !isAllowed
      // target 不存在 key 且 key 是全局属性，则会返回 false
    }
  }

  const getHandler = {
    get (target, key) {
      if (typeof key === 'string' && !(key in target)) {
      // target 中不存在 key 属性，则警告
        warnNonPresent(target, key)
      }
      // 否则，返回 key 的属性值
      return target[key]
    }
  }

  initProxy = function initProxy (vm) {
    if (hasProxy) {
      // determine which proxy handler to use
      const options = vm.$options
      const handlers = options.render && options.render._withStripped
        ? getHandler
        : hasHandler
      vm._renderProxy = new Proxy(vm, handlers)
    } else {
      vm._renderProxy = vm
    }
  }
}

export { initProxy }
```

上述文件导出的 `initProxy` 将在 `initMixin` 中被调用（也就是说，它在 vue 的初始化过程中）：

> `./src/core/instance/init.js`

```js
/* istanbul ignore else */
if (process.env.NODE_ENV !== 'production') {
  initProxy(vm)
} else {
  vm._renderProxy = vm
}
```

回答 Practice 的问题“what is `vm._renderProxy` and it's role”：

`vm._renderProxy` 是 `vm` 实例的拦截器，在 `get` 获取 `vm` 的属性之前，增加安全检查，检查查询的属性是否存在（`has`），或者是否允许直接调用（`isAllowed`）。

它的角色就是，调用对象前的安全检查员。在对象属性被调用前，它检查两个问题：1）被调用对象的目标属性 key 是否存在？2）目标属性 key 是否可以被调用？

两个问题都要返回 true，才会返回属性值。
