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
