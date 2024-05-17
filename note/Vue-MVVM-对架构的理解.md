[toc]

> 参考：[Vue 旧指南](https://012.vuejs.org/guide/)

![](./img/vue-basic-mvvm.png)

上图中的黑色箭头代表**数据传递方向**：被监听者 -> 监听者。

DOM Listeners 的箭头代表监听器监听到 View 的变化，新数据传递给 Model，触发 Model 数据更新；
Directives 的箭头代表 Model 的数据更新，新数据传递给 View，触发 View 更新。

DOM Listeners 是监听 DOM 的一系列监听器，实现了 `$data` 响应 DOM property 的变化。前者是监听者，后者是被监听对象；前者响应变化，更新自身数据。
Directives 是 Vue.js 提供的指令，比如 `v-bind`，实现了 DOM property 响应 `$data` 中的目标属性的变化。前者是监听者，后者是被监听对象；前者响应变化，更新自身数据。

Model 利用 DOM Listeners “偷听” View 的变化，并随着偷听到的情报同步更新自身数据；
View 则利用 Directive “偷听” Model 的变化，并随着偷听到的情报同步更新自身数据。

 #todo 继续写更详细的代码级别的理解，加上 `Watcher`，把参考文章的 Concepts overview 看完。或者找 vue2.x 的对应的 guide。 