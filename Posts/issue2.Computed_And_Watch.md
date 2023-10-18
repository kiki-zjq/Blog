# 从源代码的角度来解析 Vue2 中 Computed 和 Watch 的区别

> **请你回答一下 Vue 中 computed 和 watch 的区别是什么？**

相信 Vue 的开发者在学习或者面试的时候都或多或少被问到过这个经典的问题。
比较常见的回答可能是：
1. computed 是计算属性，当依赖的属性值发生变化的时候，数据才会更新
2. watch 是监听，监听数据的变化然后执行对应的操作
3. computed 是懒加载 + 有缓存的
4. watch 中可以进行异步操作，而 computed 中不可以
5. ... ... 

那么，到底是什么原因造就了 computed 和 watch 这样的区别呢？
这篇文章将会根据 ***Vue2*** 源码来对二者进行分析。如果想知道 Vue3 中 computed 和 watch 的区别，请关注作者的下一篇文章 XD

## computed 和 watch 的基本用法

首先我们看一个简单的例子，来大致了解一下 `computed` 和 `watch` 的用法
```js
computed: {
  reversedMessage: function() {
    return this.message.split('').reverse().join('')
  }
},
watch: {
  firstName: function(newVal, oldVal) {
    this.fullName = newVal + ' ' + this.lastName
  }
}
```
在上面的例子中，computed 的用法是，收集 `this.message` 属性的依赖，然后当 `this.message` 变动的时候，重新计算出一个新的值，我们可以直接在模版中使用 `{{ reversedMessage }}` 来获取计算而得的值
watch 的用法是，收集了 `this.firstName` 属性作为依赖，并且侦听它的改变，当其变化的时候，执行声明的这个方法，根据最新的 `this.firstName` 来为 `this.fullName` 赋值

## computed 源码

在源代码中，`src/core/instance/state.js` 中的 **`initComputed`** 方法，是处理 computed 的起点

```js
function initComputed (vm: Component, computed: Object) {
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
    // 获取computed的key  
    const userDef = computed[key]
    // computed[key]要么是个函数 要么是个对象，如果是个函数，getter就是这个函数本身
    // 如果是个配置对象 getter就是配置对象的get属性
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    // 如果没有配置 getter，那么开发环境会报出一个警告
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }
    if (!isSSR) {
      // 如果不是SSR 实例化一个watcher 所以computed的原理就是通过 watcher 来实现的
      // getter 获得函数的实体
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop, // computed的watcher没有callback方法
        computedWatcherOptions // 配置项 {lazy: true} --> computed默认是懒执行
      )
    }

    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } 
  }
}
```

注意，这里面最关键的一句是 `watchers[key] = new Watcher(...)`，说明了 computed 本质其实也是一个 **Watcher 对象**

## watch 源码

在源代码中，`src/core/instance/state.js` 中的 **`initWatch`** 方法，是处理 watch 的起点
```js
function initWatch (vm: Component, watch: Object) {
  // 遍历watch配置项
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}

function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  // 如果handler是对象，从handler属性中获取函数
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  // 如果是字符串，表示的是一个 methods 方法，直接通过this.methodsKey的方式拿到这个函数
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}

Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    const vm: Component = this
    // 处理一下回调函数CB是个对象的情况
    // 前面已经处理了为什么这里还要再处理呢？
    // 因为用户可以直接在外面通过 this.$watch(...)去调用，因此这里要再处理一次，保证CB一定是个函数
    if (isPlainObject(cb)) {
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    // options.user 表示用户 watcher，还有渲染 watcher，即 updateComponent 方法中实例化的 watcher
    options.user = true

    // 实例化watcher
    const watcher = new Watcher(vm, expOrFn, cb, options)

    // 如果设置watcher的时候设置了immediate: true选项， 则立即执行一波回调函数
    if (options.immediate) {
      const info = `callback for immediate watcher "${watcher.expression}"`
      pushTarget()
      invokeWithErrorHandling(cb, vm, [watcher.value], vm, info)
      popTarget()
    }

    // 返回一个 unwatch - 调用这个方法可以取消watch监听
    return function unwatchFn () {
      watcher.teardown()
    }
  }
```
从上面的代码中，我们可以看出来 watch 的本质其实也是创建一个 **Watcher 对象**。因此，我们可以得到一个重要的结论，<u>***computed 和 watch 的本质并没有区别，都是创建 Watcher 对象***</u>。


## Watcher 对象

Watcher 对象是 Vue 响应式的核心对象之一，正是因为有了 Watcher，用户与页面的交互才会被监听，我们才可以做出相应的处理，例如重新渲染等...

基于以上的源码分析，我们意识到 <u>***computed 和 watch 的本质并没有区别，都是创建 Watcher 对象***</u>。这里简化一下它们创建 Watcher 对象时候的语句
```js
// computed
new Watcher(vm, getter, noop, {lazy: true})
// watch
new Watcher(vm, expOrFn, cb, options)
```
这里有一个很重要的不同，我们在配置 computed 的时候声明的函数是 getter，会作为第二个参数传入到 Watcher 的构造函数中。而配置 watch 时声明的函数是 cb，会作为第三个参数传入到 Watcher 的构造函数中。



最后，实际上 computed 和 watch 还有一个最大的、最直观的不同，但是往往被大家所忽视了。那就是二者的 ***语义本身就完全不同*** 。
`computed` 是计算属性，可以理解成从某种已有的属性 ***计算*** 得到新的属性。对于它而言，计算得到的结果是关键点。
`watch` 是侦听，***侦听*** 某件事情的发生并且做出相应的行为。对于它而言，能够侦听到数据的改变，然后进行相应的响应，这个行为是关键点。
