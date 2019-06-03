# 合并配置
通过之前章节的源码分析我们知道，new Vue 的过程通常有 2 种场景，一种是外部我们的代码主动调用 new Vue(options) 的方式实例化一个 Vue 对象；另一种是我们上一节分析的组件过程中内部通过 new Vue(options) 实例化子组件。

无论哪种场景，都会执行实例的 _init(options) 方法，它首先会执行一个 merge options 的逻辑，相关的代码在 src/core/instance/init.js 中：
```js
Vue.prototype._init = function (options?: Object) {
  // merge options
  if (options && options._isComponent) {
    // optimize internal component instantiation
    // since dynamic options merging is pretty slow, and none of the
    // internal component options needs special treatment.
    initInternalComponent(vm, options)
  } else {
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
  // ...
}
```

可以看到不同场景对于 options 的合并逻辑是不一样的，并且传入的 options 值也有非常大的不同，接下来我会分开介绍 2 种场景的 options 合并过程。

为了更直观，我们可以举个简单的示例：

```js
import Vue from 'vue'

let childComp = {
  template: '<div>{{msg}}</div>',
  created() {
    console.log('child created')
  },
  mounted() {
    console.log('child mounted')
  },
  data() {
    return {
      msg: 'Hello Vue'
    }
  }
}

Vue.mixin({
  created() {
    console.log('parent created')
  }
})

let app = new Vue({
  el: '#app',
  render: h => h(childComp)
})
```
## 外部调用场景

当执行 new Vue 的时候，在执行 this._init(options) 的时候，就会执行如下逻辑去合并 options：

```js
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

这里通过调用 mergeOptions 方法来合并，它实际上就是把 resolveConstructorOptions(vm.constructor) 的返回值和 options 做合并，resolveConstructorOptions 的实现先不考虑，在我们这个场景下，它还是简单返回 vm.constructor.options，相当于 Vue.options，那么这个值又是什么呢，其实在 initGlobalAPI(Vue) 的时候定义了这个值，代码在 src/core/global-api/index.js 中：

```js
export function initGlobalAPI (Vue: GlobalAPI) {
  // ...
  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)
  // ...
}
```

首先通过 Vue.options = Object.create(null) 创建一个空对象，然后遍历 ASSET_TYPES，ASSET_TYPES 的定义在 src/shared/constants.js 中：

```js
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
```

所以上面遍历 ASSET_TYPES 后的代码相当于：

```js
Vue.options.components = {}
Vue.options.directives = {}
Vue.options.filters = {}
```

接着执行了 Vue.options._base = Vue，它的作用在我们上节实例化子组件的时候介绍了。

最后通过 extend(Vue.options.components, builtInComponents) 把一些内置组件扩展到 Vue.options.components 上，Vue 的内置组件目前有 keep-alive、transition 和 transition-group 组件，这也就是为什么我们在其它组件中使用 keep-alive 组件不需要注册的原因，这块儿后续我们介绍 keep-alive 组件的时候会详细讲。

那么回到 mergeOptions 这个函数，它的定义在 src/core/util/options.js 中：

```js
/**
 * Merge two option objects into a new one.
 * Core utility used in both instantiation and inheritance.
 */
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  if (typeof child === 'function') {
    child = child.options
  }

  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)
  const extendsFrom = child.extends
  if (extendsFrom) {
    parent = mergeOptions(parent, extendsFrom, vm)
  }
  if (child.mixins) {
    for (let i = 0, l = child.mixins.length; i < l; i++) {
      parent = mergeOptions(parent, child.mixins[i], vm)
    }
  }
  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```

mergeOptions 主要功能就是把 parent 和 child 这两个对象根据一些合并策略，合并成一个新对象并返回。比较核心的几步，先递归把 extends 和 mixins 合并到 parent 上，然后遍历 parent，调用 mergeField，然后再遍历 child，如果 key 不在 parent 的自身属性上，则调用 mergeField。

这里有意思的是 mergeField 函数，它对不同的 key 有着不同的合并策略。举例来说，对于生命周期函数，它的合并策略是这样的：

```js
function mergeHook (
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  return childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
}

LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})

```

这其中的 LIFECYCLE_HOOKS 的定义在 src/shared/constants.js 中：

```js
export const LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured'
]
```

这里定义了 Vue.js 所有的钩子函数名称，所以对于钩子函数，他们的合并策略都是 mergeHook 函数。这个函数的实现也非常有意思，用了一个多层 3 元运算符，逻辑就是如果不存在 childVal ，就返回 parentVal；否则再判断是否存在 parentVal，如果存在就把 childVal 添加到 parentVal 后返回新数组；否则返回 childVal 的数组。所以回到 mergeOptions 函数，一旦 parent 和 child 都定义了相同的钩子函数，那么它们会把 2 个钩子函数合并成一个数组。

关于其它属性的合并策略的定义都可以在 src/core/util/options.js 文件中看到，这里不一一介绍了，感兴趣的同学可以自己看。

通过执行 mergeField 函数，把合并后的结果保存到 options 对象中，最终返回它。

因此，在我们当前这个 case 下，执行完如下合并后：