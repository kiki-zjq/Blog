# Vue2 源码解析 —— 响应式原理

## 什么是响应式？

我们知道 Vue 的一个核心就是***响应式系统***。  
例如我们的页面中有这么一部分代码 `<div>{{ name }}</div>`，当我们的 name 的值改变的时候，视图会自动的进行更新。这就是所谓的响应式系统。

Vue 2 和 Vue 3 对响应式的实现略有不同，这篇文章将会从源代码的角度开始讨论在 Vue 2 中，我们是怎么实现响应式的。

## 响应式入口 `initState`

在 Vue 实例初始化的过程中，会执行一个 `initState` 函数，这个函数位于 `/src/core/instance/state.js` 中。  
这个函数是数据响应式的入口，依次调用 `initProps` `initMethods` `initData` `initComputed` `initWatch` 方法，完成 props methods data computed watch 属性的响应式处理

```JavaScript
/**
 * 两件事：
 *   数据响应式的入口：分别处理 props、methods、data、computed、watch
 *   优先级：props、methods、data、computed 对象中的属性不能出现重复，优先级和列出顺序一致
 *         其中 computed 中的 key 不能和 props、data 中的 key 重复，methods 不影响
 */
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options  
  if (opts.props) initProps(vm, opts.props) // 1. 对props配置做响应式处理 2. 代理props配置上的key到vue实例，支持this.propsKey的方式访问
  if (opts.methods) initMethods(vm, opts.methods) // 1. 判重处理 methods中属性和props中属性不可重复 2. 支持this.methodsKey的方式访问
  if (opts.data) {
    initData(vm) // 1. 判重处理 2. 代理 将data中属性代理到vue实例，支持this.key的方式访问 3. 响应式处理
  } else {
    // 如果不存在opts.data 那么直接调用observe函数去监测一个空对象
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed) 
  // 1. computed是通过watcher来实现的 对每一个computedKey实例化一个watcher，默认懒执行
  // 2. 将computedKey代理到vue实例上 支持通过this.computedKey的方式访问computedKey属性
  // 3. 注意理解computed缓存的实现原理

  // 核心：实例化一个watcher实例 并且返回一个unwatch
  if (opts.watch && opts.watch !== nativeWatch) { // 这里判断的时候要注意不能是浏览器原生的watch
    // 1. 处理watch对象
    // 2. 为每一个watch.key 创建 watcher实例，key和watcher实例可能是一对多的关系
    // 3. 如果设置了immediate 则立即执行回调函数
    initWatch(vm, opts.watch)
  }
}
```

接下来，我们就看看这些函数分别做了什么吧～

## initProps

```JavaScript
// 处理 props 对象，为 props 对象的每个属性设置响应式，并将其代理到 vm 实例上
// 这里触发响应式的方案是 defineReactive(props, key, value)
function initProps (vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  // 缓存props中的每一个key 来进行性能优化
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent

  if (!isRoot) {
    toggleObserving(false)
  }
  for (const key in propsOptions) {
    keys.push(key) // 缓存key
    const value = validateProp(key, propsOptions, propsData, vm)

    // 对props数据进行响应式处理
    defineReactive(props, key, value)
    
    // 处理 this.propsKey
    if (!(key in vm)) {
      // 代理key到vm对象上 --> 可允许用户通过 this.keyName 直接访问这个key
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}
```

在这里，我们去掉了一部分不重要的代码，整个 initProps 方法的核心其实就是两个函数，`defineReactive` 和 `proxy`，这两个函数我们会在后文讲解，在这里我们可以对其有一个大致的印象～
- `defineReactive`: 响应式设置的真正核心。`Object.defineProperty` 都在这里这个方法中定义。
- `proxy`: 将当前的 key 代理到 Vue 实例上，例如原本我们需要通过 `this._props.keyName` 访问的数据，现在可以直接通过 `this.keyName` 访问了

## initMethods

```JavaScript
/**
 * 做了以下三件事，其实最关键的就是第三件事情
 *   1、校验 methoss[key]，必须是一个函数
 *   2、判重
 *         methods 中的 key 不能和 props 中的 key 相同
 *         methos 中的 key 与 Vue 实例上已有的方法重叠，一般是一些内置方法，比如以 $ 和 _ 开头的方法
 *   3、将 methods[key] 放到 vm 实例上，得到 vm[key] = methods[key]，可以直接通过 this.methodName 访问这个 method
 */
function initMethods (vm: Component, methods: Object) {
  const props = vm.$options.props
  // 判重处理 methods中的key不能和props中的key重复。props中key的优先级更高
  for (const key in methods) {
    if (process.env.NODE_ENV !== 'production') {
      if (typeof methods[key] !== 'function') {
        warn(
          `Method "${key}" has type "${typeof methods[key]}" in the component definition. ` +
          `Did you reference the function correctly?`,
          vm
        )
      }
      if (props && hasOwn(props, key)) {
        warn(
          `Method "${key}" has already been defined as a prop.`,
          vm
        )
      }
      if ((key in vm) && isReserved(key)) {
        warn(
          `Method "${key}" conflicts with an existing Vue instance method. ` +
          `Avoid defining component methods that start with _ or $.`
        )
      }
    }
    // 将method中所有方法赋值到vue实例中，支持通过this.menthodKeys的方式访问定义的方法
    vm[key] = typeof methods[key] !== 'function' ? noop : bind(methods[key], vm)
  }
}
```

## initData

```JavaScript
/**
 * 做了三件事
 *   1、判重处理，data 对象上的属性不能和 props、methods 对象上的属性相同
 *   2、代理 data 对象上的属性到 vm 实例
 *   3、为 data 对象的上数据设置响应式 
 *      -> observe(data, true)
 */
function initData (vm: Component) {
  let data = vm.$options.data
  // 保证后续处理的data是一个对象
  // 其实 vm.$options.data 在mergeOptions的时候已经被处理成了一个函数了，但是这里为什么还要判断一次呢？
  // 因为在mergeOption和这一步initData之间还存在beforeCreated这一步，这一步有可能对vm.$options.data进行修改，因此这里还是要加一下类型判断的～
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  // 如果data还不是一个对象，就会返回一个空对象 然后给出警告
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }

  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    // 判重处理 data中的属性不能和props和methods中的key重复
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }

    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      // 如果与props 和 methods中的key都没有重复 并且这个key也不是保留字
      // 代理data的属性到vue实例 支持通过this.key来进行访问
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}
```

在这里我们又看到了第三个关键的函数 `observe`（前面两个是 `proxy` 和 `defineReactive`），这个函数也是为数据设置响应式的，那么它和前文的 `defineReactive` 又是什么关系呢？


## `proxy`

```JavaScript
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}

// 代理函数，将 key 代理到 vue 实例上
export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  // 拦截对 this.key 的访问。 sharedPropertyDefinition定义了getter和setter
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

proxy 其实就是一个非常直观的代理函数，通过使用 `Obejct.defineProperty` 来重写 `getter` 和 `setter`，让我们可以通过 `this[keyName]` 直接访问到我们需要的属性

## `defineReactive`
