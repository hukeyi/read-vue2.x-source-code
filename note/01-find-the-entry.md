# Find the Entry

This article belongs to the series [Read Vue Source Code](https://github.com/numbbbbb/read-vue-source-code).

In this article, we will:

- Get the code
- Open your editor
- Find the entry

> 入口文件是：`src/platforms/web/entry-runtime-with-compiler.js`

## Get the Code

You can read code using GitHub, but it's slow and hard to see the directory structure. So let's download the code first.

Open [Vue's GitHub page](https://github.com/vuejs/vue), click "Clone or download" button and then click "Download ZIP".

![](http://i.imgur.com/Fshkk3Z.jpg)

You can also use `git clone git@github.com:vuejs/vue.git` if you like using console.

> I use the latest dev code I can get, so it may have some differences with the code you download.
> 
> Don't worry, this series is aimed at telling you **how** to understand the source code. You can use the same methods even if the code is different.
> 
> You can also download [the version I use](https://github.com/numbbbbb/read-vue-source-code/blob/master/vue.zip) if you like.

## Open Your Editor

I prefer to use Sublime Text, you can use your favorite editor.

Open your editor, double click the zip file you downloaded and drag the folder to your editor.

![](http://i.imgur.com/WgediMc.jpg)

## Find the Entry

Now we meet our first question: where should we start?

It's a common question for big open source projects. Vue is a npm package, so we can ==open `package.json` first.==

> [[package.json 属性总结]]

```json
{
  "name": "vue",
  "version": "2.3.3",
  "description": "Reactive, component-oriented view layer for modern web interfaces.",
  "main": "dist/vue.runtime.common.js",
  "module": "dist/vue.runtime.esm.js",
  "unpkg": "dist/vue.js",
  "typings": "types/index.d.ts",
  "files": [
    "src",
    "dist/*.js",
    "types/*.d.ts"
  ],
  "scripts": {
    "dev": "rollup -w -c build/config.js --environment TARGET:web-full-dev",
    "dev:cjs": "rollup -w -c build/config.js --environment TARGET:web-runtime-cjs",
    "dev:esm": "rollup -w -c build/config.js --environment TARGET:web-runtime-esm",
    "dev:test": "karma start build/karma.dev.config.js",
    "dev:ssr": "rollup -w -c build/config.js --environment TARGET:web-server-renderer",
    ...
```

==First three keys are `name`, `version` and `description`,== no need to explain.

There is a `dist/` in `main`, `module` and `unpkg`, that implies they are related to generated files. Since we are looking for the entry, just ignore them.

Next is `typings`. After Googling, we know it's a TypeScript definition. We can see many type definitions in `types/index.d.ts`.

Go on.

`files` contains three paths. First one is `src`, it's the source code directory. This narrows the range, but we still don't know which file to start with.

==Next key is `scripts`. Oh, the `dev` script!== This command must know where to start.

```
rollup -w -c build/config.js --environment TARGET:web-full-dev
```

> #fixme 学习 linux 常用命令。`TARGET` 是什么

Now we have `build/config.js` and `TARGET:web-full-dev`. Open `build/config.js` and search `web-full-dev`:

```javascript
// Runtime+compiler development build (Browser)
'web-full-dev': {
  entry: resolve('web/entry-runtime-with-compiler.js'),
  dest: resolve('dist/vue.js'),
  format: 'umd',
  env: 'development',
  alias: { he: './entity-decoder' },
  banner
},
```

Great! 

This is the config of dev building. It's entry is `web/entry-runtime-with-compiler.js`. But wait, where is the `web/` directory?

==Check the entry value again, there is a `resolve`. Search `resolve`:==

```javascript
const aliases = require('./alias')
const resolve = p => {
  const base = p.split('/')[0]
  if (aliases[base]) {
    return path.resolve(aliases[base], p.slice(base.length + 1))
  } else {
    return path.resolve(__dirname, '../', p)
  }
}
```


Go through `resolve('web/entry-runtime-with-compiler.js')` with the parameters we have:

- p is `web/entry-runtime-with-compiler.js`
- base is `web`
- convert the directory to `alias['web']`

Let's jump to `alias` to find `alias['web']`.

```javascript
module.exports = {
  vue: path.resolve(__dirname, '../src/platforms/web/entry-runtime-with-compiler'),
  compiler: path.resolve(__dirname, '../src/compiler'),
  core: path.resolve(__dirname, '../src/core'),
  shared: path.resolve(__dirname, '../src/shared'),
  web: path.resolve(__dirname, '../src/platforms/web'),
  weex: path.resolve(__dirname, '../src/platforms/weex'),
  server: path.resolve(__dirname, '../src/server'),
  entries: path.resolve(__dirname, '../src/entries'),
  sfc: path.resolve(__dirname, '../src/sfc')
}
```

Okay, it's `src/platforms/web`. Concat it with the input file name, ==we get `src/platforms/web/entry-runtime-with-compiler.js`.==

```javascript
/* @flow */

import config from 'core/config'
import { warn, cached } from 'core/util/index'
import { mark, measure } from 'core/util/perf'

import Vue from './runtime/index'
import { query } from './util/index'
import { shouldDecodeNewlines } from './util/compat'
import { compileToFunctions } from './compiler/index'

const idToTemplate = cached(id => {
  const el = query(id)
  return el && el.innerHTML
})

const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

  const options = this.$options
  // resolve template/el and convert to render function
  if (!options.render) {
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
  ...
```

Cool! You have found the entry!

> Have you noticed the `/* flow */` comment at the top? After Google, we know it's a type checker. It reminds me of `typings`, I search again and find out that `flow` can use `typings` definitions to validate your code.

> Maybe next time I can use `flow` and `typings` in my own project. You see, even if we haven't really started reading the code, we have learned some useful and practical things.

Let's go through this file step-by-step:

- import config
- import some util functions
- import Vue(What? Another Vue?)
- define `idToTemplate`
- define `getOuterHTML`
- define `Vue.prototype.$mount` which use `idTotemplate` and `getOuterHTML`
- define `Vue.compile`

The two important points are:

1. This is **NOT** the real Vue code, we should know that from the filename, it's just an entry
2. This file extracts the `$mount` function and defines a new `$mount`. After reading the new definition, we know it just add some validations before calling the real mount

> 补充：new `$mount` 加了一些验证。比如第一段 if 验证 el 是否被挂载到 `<html>` 或者 `<body>` 上，如果是且在非生产环境，则发出警告。

## Next Step

Now you know how to find the entry of a brand new project. In next article, we will go on to find the core code of Vue.

Read next chapter: [Dig into the Core](https://github.com/numbbbbb/read-vue-source-code/blob/master/02-dig-into-the-core.md).

## Practice ✅

Remember those "util functions"? Read their codes and tell what they do. It's not difficult, but you need to be patient. 

Don't miss `shouldDecodeNewlines`, you can see how they fight with IE.

---

首先找到 util 函数的文件路径：

```js
import { warn, cached } from 'core/util/index'
import { mark, measure } from 'core/util/perf'
import { query } from './util/index'
import { shouldDecodeNewlines } from './util/compat'
```

### warn & cached

先打开 `core/util/index`：

```js
export * from 'shared/util'
export * from './lang'
export * from './env'
export * from './options'
export * from './debug'
export * from './props'
export * from './error'
export { defineReactive } from '../observer/index'
```

在 `./debug` 中找到了 `warn()`：

```js
warn = (msg, vm) => {
    if (hasConsole && (!config.silent)) {
      console.error(`[Vue warn]: ${msg}` + (
        vm ? generateComponentTrace(vm) : ''
      ))
    }
  }
```

就是在 console 打印警告信息。调用的`generateComponentTrace` 函数也在同一文件声明，作用就是追溯 `vm` 在 DOM 树中的位置。

继续找 `cached`，在 `shared/util` 找到了：

```js
/**
 * Create a cached version of a pure function.
 */
export function cached<F: Function> (fn: F): F {
  const cache = Object.create(null)
  return (function cachedFn (str: string) {
    const hit = cache[str]
    return hit || (cache[str] = fn(str))
  }: any)
}
```

发现一个闭包。

`Object.create(null)` 如何理解？MDN 的解释：

> With `Object.create()`, we can create an object [with `null` as prototype](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object#null-prototype_objects). The equivalent syntax in object initializers would be the [`__proto__`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Object_initializer#prototype_setter) key.
>
> ```js
> o = Object.create(null);
> // Is equivalent to:
> o = { __proto__: null };
> ```

`return hit || (cache[str] = fn(str))` 如何理解？`return` 后接赋值语句？赋值语句返回的是什么？

[MDN: 表达式和运算符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Expressions_and_Operators#Return_value_and_chaining)指南中有一段话讲，赋值表达式本身等价于赋值运算符 `=` 右边的运算结果：

> The expression `x = 7` is an example of the first type. This expression uses the `=` *operator* to assign the value seven to the variable `x`. The expression itself evaluates to `7`.

那么上面这句 `return` 语句就好理解了：如果 `hit` 存在，就返回它；如果不存在，就先把 `cache[str]` 赋值为 `fn(str)`，然后返回 `fn(str)`。

`hit` 存在是什么情况？

`cache` 使用闭包手段来缓存工具函数的缓存区。`hit` 的含义为「缓存命中」。

以上函数可以解释为：

- 调用 `cached(fn)`，把函数 `fn` 缓存进入缓存区 `cache`
- `cached(fn)` 的返回值为已经缓存进入缓存区的 `cache` 的函数 `fn` ，把返回值保存到某个变量 `foo` 中
- 当需要使用函数 `fn` 时，直接调用 `foo` ，从缓存区中拿出 `fn` 执行

作用是将 `fn` 保存为私有函数。

### mark & measure

然后，转去看 `core/util/perf`：

```js
import { inBrowser } from './env'

export let mark
export let measure

if (process.env.NODE_ENV !== 'production') {
  const perf = inBrowser && window.performance
  /* istanbul ignore if */
  if (
    perf &&
    perf.mark &&
    perf.measure &&
    perf.clearMarks &&
    perf.clearMeasures
  ) {
    mark = tag => perf.mark(tag)
    measure = (name, startTag, endTag) => {
      perf.measure(name, startTag, endTag)
      perf.clearMarks(startTag)
      perf.clearMarks(endTag)
      perf.clearMeasures(name)
    }
  }
}
```

这个容易多了。

第一行 `isBrowser`，判断全局对象 `window` 是否存在，判断环境是否为浏览器：

```js
export const inBrowser = typeof window !== 'undefined'
```

接下来是重点。

`perf` 是性能 performance 的缩写。[`window.performance`](https://developer.mozilla.org/zh-CN/docs/Web/API/Performance?qs=perfor) 是 web api，记录当前页面性能的相关信息接口。如果 `isBrowser` 为 `true`，则 `perf` 就赋值为这个浏览器全局对象的性能属性。

> 备注：Performance 是专用于性能测试的 web api！用于优化网页速度时测试指标。
>
> ‼️参考：[Performance 有什么用？怎么用？](https://www.digitalocean.com/community/tutorials/js-js-performance-api)
>
> > There are a lot of tools which can help you understand how your application works locally. The Performance API is here to help us have a granular understanding of our web pages in the wild. You can get real data and see how your site works in different browsers, networks, parts of the world and more!

`mark(markName)` 是 `window.performance.mark()` 函数的重写。`measure(measureName, startMark, endMark)` 定义了一个 measure 后，把相关的 measure 和两个 mark 从浏览器的性能输入缓冲区中全部清空。

它俩调用的四个函数全部是[ `Performance` 对象](https://developer.mozilla.org/en-US/docs/Web/API/Performance)提供的 api：

```js
perf.mark(tag)
perf.measure(name, startTag, endTag)
perf.clearMarks(startTag)
perf.clearMarks(endTag) // same as above one
perf.clearMeasures(name)
```

mark 理解为某个时间点的时间戳，measure 理解为两个时间点的时间间隔。

比如，「开始编译」和「结束编译」为两个时间点，它俩之间的时间间隔被命名为「编译」。

第一次调用 `mark()`：

```js
if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile') // 标记编译开始
      }
```

第二次调用 `mark()` 和第一次调用 `measure(measureName, startMark, endMark)` ：

```js
if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile end') // 标记编译结束
        measure(`${this._name} compile`, 'compile', 'compile end') // 标记编译开始与结束之间的时间间隔
      }
```

总而言之，这两个函数的作用就是：记录运行日志和性能测试日志。

### query

在 `./util/index.js` 中：

```js
/**
 * Query an element selector if it's not an element already.
 */
export function query (el: string | Element): Element {
  if (typeof el === 'string') {
    const selected = document.querySelector(el)
    if (!selected) {
      process.env.NODE_ENV !== 'production' && warn(
        'Cannot find element: ' + el
      )
      return document.createElement('div')
    }
    return selected
  } else {
    return el
  }
}
```

查找选择器（el）对应的 DOM 元素。

疑问 #fixme ：el 不存在，抛出 warning 的同时返回一个新创建的空的 `div` 元素，意义在？

### shouldDecodeNewlines

```js
// check whether current browser encodes a char inside attribute values
function shouldDecode (content: string, encoded: string): boolean {
  const div = document.createElement('div')
  div.innerHTML = `<div a="${content}">`
  return div.innerHTML.indexOf(encoded) > 0
}

// #3663
// IE encodes newlines inside attribute values while other browsers don't
export const shouldDecodeNewlines = inBrowser ? shouldDecode('\n', '&#10;') : false
```

专门为 IE 做的验证函数。

IE 会把 html 标签的属性值中的 `\n` 编码为 `&#10;`，这个函数特地做这个判断，检查浏览器是否会做这个编码。

