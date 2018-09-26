---
layout: post
title: Vue 响应式原理白话版
date: 2018-09-26
categories: 前端
cover: https://raw.githubusercontent.com/njleonzhang/image-bed/master/assets/893f05c7-7ee5-e7a8-1cd9-f626ae0425d4.png
tags: vue
---

作为一个前端的 MvvM 框架，Vue 的基本思路和 angular、React 并无二致，其核心就在于: 当数据变化时，自动去刷新页面 Dom，这使得我们能从繁琐的 Dom 操作中解放出来，从而专心地去处理业务逻辑。回想一下 jQuery 时代的痛点，现在的前端人真是赶上了好时代。😂 那么 Vue 是怎么做到这种自动更新的呢？

# 官方的简明解释
这个问题即为 Vue 的响应式原理，官方文档给出了一个简洁的[解释](https://vuejs.org/v2/guide/reactivity.html#ad):

![](https://raw.githubusercontent.com/njleonzhang/image-bed/master/assets/abe782d3-2489-c331-582e-3bae2f3b543b.png)

总结一下:

1. 任何一个 Vue Component 都有一个与之对应的 Watcher 实例。
2. Vue 的 `data` 上的属性会被添加 getter 和 setter 属性。
3. 当 Vue Component `render` 函数被执行的时候, `data` 上会被 `触碰(touch)`, 即被`读`, `getter` 方法会被调用, 此时 Vue 会去记录此 Vue component 所依赖的所有 `data`。(这一过程被称为依赖收集)
4. `data` 被改动时（主要是用户操作）, 即被`写`, `setter` 方法会被调用, 此时 Vue 会去通知所有依赖于此 `data` 的组件去调用他们的 render 函数进行更新。

图中的说法自然是正确的，但是没有涉及太多的细节。另外还有大量的文章从源码的角度去深入的探讨了整个 Vue 响应式的实现。比如: [Vue.js 技术揭秘](https://ustbhuangyi.github.io/vue-analysis/reactive/)、[深入解析Vue依赖收集原理
](https://www.ruphi.cn/archives/336/). 这些文章很是具体，但这个实现天生比较复杂，不是很容易理解。本篇会尝试去除大部分细枝末节, 只介绍一下实现方案的关键。


二话不说，先盗用 [Vue.js 技术揭秘](https://ustbhuangyi.github.io/vue-analysis/reactive/) 里的图:

![](https://ustbhuangyi.github.io/vue-analysis/assets/new-vue.png)

这个图展示了 Vue.js 是如何将一个 Vue 对象最终渲染成页面上的 html DOM的。

## data 的 reactive 化
 在 `init` 阶段(defineReactive), Vue 对象属性 **data 的属性**会被 reactive 化，即会被设置 `getter` 和 `setter` 函数。

  ```
  function defineReactive(obj: Object, key: string, ...) {
    const dep = new Dep()

    Object.defineProperty(obj, key, {
      enumerable: true,
      configurable: true,
      get: function reactiveGetter () {
        ....
        dep.depend()
        return value
        ....
      },
      set: function reactiveSetter (newVal) {
        ...
        val = newVal
        dep.notify()
        ...
      }
    })
  }
  ```

这个函数里 new 了一个 Dep 对象，这个对象的用法很特别。Vue 在对 `data` 上的任何一个属性进行 reactive 化的时候，都会调用 `defineReactive` 函数，也就是说**对于所有被 Vue reactive 化的属性来说都有一个 Dep 对象与之对应**。那么这个 Dep 是做什么的呢？先挖个坑，我们下面再说。

此时 getter 和 setter 函数并不会执行, 他们只是被绑定在了 `data` 的属性上，所以我们先不看 getter 和 setting 函数里的内容。

> 注意我们得到的这个关键的结论: **所有被 Vue reactive 化的属性都有一个 Dep 对象与之对应**

## Watcher 的创建

Vue 对象 init 后会进入 mount 阶段，这个阶段的关键一个函数就是 `mountComponent`:

  ```
  mountComponent(vm: Component, el: ?Element, ...) {
    vm.$el = el

    ...

    updateComponent = () => {
      vm._update(vm._render(), ...)
    }

    new Watcher(vm, updateComponent, ...)
    ...
  }
  ```

这里的 `Watcher` 的用法和上面的 `Dep` 类似，也是没头没脑的被 new 了出来😂 。对于每个 Vue 组件实例来说，只会经历一次 mount，所以对于每一个 Vue 实例来说都有一个 Watcher 与之对应（[官方的简明解释](#官方的简明解释)里的第 1 点），当然就是 mountComponent 里 new 出来的这个。但是这个 Vue 组件的对象上并没有存这个 Watcher 实例。所以 Vue 组件的实例并不知道这个 watcher 的存在。

那么反过来呢？Watcher 是知道自己是和哪个 Vue 组件实例绑定的，这点我们从其构造函数可以看出来: 这个构造接受的第一个参数就是 Vue component 的实例。第二个参数呢？这里是一个函数。这个函数里调用了 `vm._render` 和 `vm._update `。

* `vm._render` 的作用，可以简单的理解为把 Vue 对象渲染成虚拟 Dom 的过程。
* `vm._update` 方法，可以简单的理解为根据虚拟 Dom 去创建或更新真是 Dom 的过程。

所以 `Watcher` 构造函数的第二个参数，就是一个 Vue 实例刷新页面 Dom 的函数，我们称之为组件的 `更新函数` 吧。我们来看看 `Watcher` 的具体结构:

![](https://raw.githubusercontent.com/njleonzhang/image-bed/master/assets/b40280c0-131d-7806-1da3-429a9e77f4d3.png)

* `vm` 属性就是组件的实例
* `getter` 属性就是组件的 `更新函数`
* `update` 函数可以理解成是调用 `更新函数` 的一个封装。

> 当然 Watcher 有多种，我们只讨论是渲染 Watcher, 也只列举了我们涉及的几个属性

看到这些属性，再加上名字 `Watcher`, 我们也基本能猜到这个 `Watcher` 对象的作用了吧。它与 Vue 组件对象一一对应，组件需要更新的时候，`Watcher` 的 `run` 方法就会被调用，进而更新页面。**`Watcher`, 看守人, 哨兵，它确认是像哨兵一样在等待着更新页面的信号**。那么这个信号是什么呢？再挖个坑，我们下面再说。

## 依赖收集
Vue 的奇妙旅程开始了。起点就是上面提到的这个 Watcher 的构造函数调用。一般来说构造函数里不太会去做出了初始化以外的一些事情，但是 Vue 的源码里, 似乎处处都是**奇奇怪怪**的做法😂，此处也是如此:

```
class Watcher {
  getter: Function;

  // 代码经过简化
  constructor(vm: Component, expOrFn: string | Function, ...) {
    ...
    this.getter = expOrFn
    Dep.target = this                      // 暂且不管
    this.value = this.getter.call(vm, vm)  // 调用组件的更新函数
    ...
  }
}
```

我们可以发现，上面的 `new Watcher(vm, updateComponent, ...)` 的调用，在 Watcher 里，不仅仅创建了 Watcher 的实例，还 `偷偷摸摸` 的做了一件事，即: 调用了 vue 组件的`更新函数`, `updateComponent`。这会导致组件的渲染函数 `vm._render` 被调用。我们知道 [渲染函数](https://cn.vuejs.org/v2/guide/render-function.html) 是 Vue template 转换来的 (当然也可以直接写)。如果不记得怎么转的了，可以看一下官网上的这个例子：

{% raw %}

```
<template>
  <h1>{{ blogTitle }}</h1>
</template>

<script>
  export default {
    data() {
      return {
        blogTitle: "leon's blog",
        author: 'leon'
      }
    }
  }
</script>

-->

render: function (createElement) {
  return createElement('h1', this.blogTitle)
}
```
{% endraw %}

不难发现渲染函数的调用会导致**模板里涉及的属性**的被访问（上例中的 `blogTitle`）。这就和[官方的简明解释](#官方的简明解释)里的第 3 点对上了。这个访问它会导致的这个属性的 `getter` 函数（reactive 化时被设置的）被调用. 我们看一下 getter 函数的定义：

```
function defineReactive(obj: Object, key: string, ...) {
  const dep = new Dep()

  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      ....
      dep.depend()
      return value
      ....
    },
    ...
  })
}
```

这里的关键之处就在于调用了 `dep.depend()`. 前面的分析我们已经知道，**所有被 Vue reactive 化的属性都有一个 Dep 对象与之对应**。这里我们不得不仔细看一下 `Dep` 的结构了:

```
class Dep {
  static target: ?Watcher;
  subs: Array<Watcher>;

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
```

我看可以看到 Dep 是观察者模式里的一个 Observable 有一个 Watcher 的队列，而观察者是 Watcher. 我们知道 `Dep` 与 **Vue reactive 化的属性** 对应, 而 Watcher 与 Vue 组件梳理对应。那么我们可以这样理解 Dep, Dep 是一个 **账单**, 他记录了所有依赖于这个属性的 Vue 组件，一旦发生某个时间之后，则会通知 **账单** 上的所有 Vue 组件去刷新（因为调用的是 Watcher 的 update 方法）。

说到这了，前面的逻辑有点长了，我们总结一下:

1. Vue init 阶段，对所有的属性做了 reactive 化，为每一个属性绑定了 getter 函数, setter 函数以及一个 Dep 对象。
2. Vue 组件 mount 阶段里调用了 mountComponent方法，此方法中为 Vue 组件创建了一个 Watcher 对象。
3. Watcher 对象创建的时候，顺带执行了 Vue 的更新函数，这触发了 **Vue reactive 化的属性** 的 get 方法, 并调用了 `dep.depend()`。

好的，那么 depend 这个函数到底做了什么？从定义上看我们知道 `Dep.target` 是一个 Watcher, 在 `dep.depend()` 被调用, 然后 `Dep.target.addDep(this)` 被执行的时候, 此时 `Dep.target` 是什么呢？就是当前创建的这个 Watcher, 即当前 mount 的 Vue 组件对象对应的 Watcher。为什么？注意看上面 Watcher 构造函数里的代码, 在执行 Vue 组件的更新函数前, 有这么一句我们没有解释, 即 `Dep.target = this`. 这就是了, 当前的 Watcher 被预先赋给了 `Dep.target`。

搞清楚了这个问题后，我们知道了 `dep.depend()` 就等价于执行了当前 Watcher 对应的 addDep 方法, 参数为这个 reactive 属性对应的 Dep 实例. Watcher 的 `addDep` 方法的代码简化后，大致如下:

```
class Watcher {
  addDep (dep: Dep) {
    ...
    this.newDeps.push(dep)
    dep.addSub(this)
    ...
  }
}

class Dep {
  addSub (sub: Watcher) {
    this.subs.push(sub)
  }
}
```

`Watcher` 的 `addDep` 方法中, `Watcher` 把这个 `dep` 对象存了下来(这个自然是别有用处，我们就不讨论了)，然后反手又调用了, `dep` 的 `addSub` 方法，一个普通的观察者 subscribe 方法。

经过了这些步骤，`dep.depend()` 终于这个 `Watcher` 记到了自己的账本中了。

这就是 Vue 依赖收集的大致步骤了。

## 派发更新
我们知道当我们改变一个 **Vue reactive 化的属性** 方法的时候, 页面会得到刷新, 即和这个属性相关的 Vue 组件的更新函数会被调用。有了上面的依赖收集，这一步其实就非常简单了，修改 **Vue reactive 化的属性** 就会调用其 setter 方法, 我再看一眼它的定义:

```
function defineReactive(obj: Object, key: string, ...) {
  const dep = new Dep()

  Object.defineProperty(obj, key, {
    ...
    set: function reactiveSetter (newVal) {
      ...
      val = newVal
      dep.notify()
      ...
    }
  })
}
```

直接调用 dep 的 `notify`, 这个方法的代码我们上面贴过了，就是依次去调用所有依赖这个属性的所有的 Vue 组件的更新函数。

![](https://raw.githubusercontent.com/njleonzhang/image-bed/master/assets/893f05c7-7ee5-e7a8-1cd9-f626ae0425d4.png)

这个更新派发的过程大致可以表示成上图的样子，从图中我们也可以看出，(render) Watcher 和 Vue 组件实例是一一对应的，reactive 化的属性和 Dep 实例是一一对应的。

我们再借一张[xingbofeng](https://github.com/xingbofeng/xingbofeng.github.io/issues/15)的图，来描述整个过程，大家看看是否能理解:

![](https://raw.githubusercontent.com/njleonzhang/image-bed/master/assets/7d769e58-03f7-a633-121e-0133c47487fa.png)

关于更详细的实现细节，我这里也引用[RuphiLau](https://www.ruphi.cn/archives/336/) 的一张图, 有兴趣的同学可以参看后文贴出的参考文档。

![](https://raw.githubusercontent.com/njleonzhang/image-bed/master/assets/1cc9c79c-86af-8a2d-dddd-82bd309eaf44.png)

## 现实的问题
了解了的 Vue 响应式原理之后，我们可能就能够更清楚的理解一些现实的问题了, 比如:

```
var vm = new Vue({
  data:{
    a:1
  }
})

// `vm.a` 是响应的

vm.b = 2
// `vm.b` 是非响应的
```
我们知道属性 `b` 是非响应的, 官网也说了原因: **由于 Vue 会在初始化实例时对属性执行 getter/setter 转化过程**, 这个转化就是在上文介绍的 `defineReactive` 函数里实施的。

```
var vm = new Vue({
  data: {
    items: ['a', 'b', 'c']
  }
})

vm.items[1] = 'x' // 不是响应性的
vm.items.length = 2 // 不是响应性的
```

items 项是数组，数组的索引并不是属性，也很难用 Dep 去绑定，所以 Vue 没有处理; 数组的长度虽然是属性，也能够通过 defineProperty 处理，但尤大可能觉得它很特别，所以也没有处理。这就导致了现在的情况。

当然，尤大给我们提供了变通的方案:

```
vm.items.splice(indexOfItem, 1, newValue)
vm.items.splice(newLength)
```

参考文章:
* https://ustbhuangyi.github.io/vue-analysis/reactive/
* https://www.ruphi.cn/archives/336/
* https://github.com/xingbofeng/xingbofeng.github.io/issues/15
* https://github.com/muwenzi/Program-Blog/issues/83
