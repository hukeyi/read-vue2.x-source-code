[toc]

> 参考： [npm 官方文档：package.json 各属性含义介绍](https://docs.npmjs.com/cli/v9/configuring-npm/package-json)

## package.json 是什么

简单讲，package.json 就是 npm 包的描述文件。看别人源码时，如果源码项目是一个 npm 包，就应当从 package.json 开始，这相当于看 github 项目首先看 README.md，看书先看目录，它们都是项目和书籍内容的概括和索引。

## 属性介绍

### files

`"files"`：指定别人下载你的包时（`npm install`）会下载的文件们，这是一个数组。

如果项目文件夹中有 `.npmignore` ，此文件中列出的所有文件/文件夹都会被忽略（不会被下载）。作用类似于 `.gitignore`。

如果项目文件中没有 `.npmignore` 但有 `.gitignore`，`.gitignore` 会被当做 `.npmignore` 来使用。

有一些无论如何都会被忽略的文件，比如 `.gitignore`；和一些无论如何都会被包含的文件，比如 `package.json`。具体是哪些直接看开头给的官方文档。

### main

项目的主文件入口。官方的解释很清楚了：

> if your package is named `foo`, and a user installs it, and then does `require("foo")`, then your main module's exports object will be returned.

`main` 指定的文件一定是一个 module，它指定的就是别人用这个包时，会获得东西：`const something = require('foo');`，`something` 就是 `main` 指向的文件 export 的 module。

### bin

> bin is short for **binary** . It's the location of binary (or executable) files.

终于知道 `bin` 是啥了。原来是二进制文件！已经编译好能够被机器直接执行的文件。

怪不得学 css 预处理器（ #fixme 合并库的时候记得在这里链接到 imooc css ch7 预处理器）的时候，如果没有 -g 全局下载，就直接在 `node_modules` 的包里去找 `bin` 里包含的执行文件。

### scripts

重点来了。

> 参考：[官方文档：scripts](https://docs.npmjs.com/cli/v9/using-npm/scripts)

先上个例子：

```json
{
	"scripts": {
    "dev": "rollup -w -c build/config.js --environment TARGET:web-full-dev"
    }
}
```

这段 json 代表什么呢？

当你在 shell 执行 `npm run dev` 时，关键词 `dev` 指向的就是这段 json。

其他待补充。 #fixme 

### dependencies

依赖包们。一般格式为：`[package_name]: "[version]"`

### devDependencies

当前包**在开发阶段**依赖的其他包（i.e. 在线上/生产环境和测试环境不需要）。

比如 `vue` 需要 `babel-core` 来编译代码；或者开发前端项目需要引入 UI 框架，比如 `element-ui`。指明这些外部依赖，就能在 `npm install` 时同时把依赖包全部下好，毕竟你不能指望别人下载你的包后，随便用个啥都报 `XXX Module not Found. `。 

### 其他描述性属性

- `name`：包名
- `version`：包版本
- `description`：包的描述，简介
- `repository`：包的代码仓库位置
- `keywords`：包的关键字描述，简介
- `author`：代码开发者
- `license`：授权证书。别人能用你的包干什么，能商用吗
- `bugs`：如果用户发现 bug，在哪里提交 issue（通常是 github 仓库的 issue，或者给一个开发者的邮箱地址）
- `homepage`：首页地址