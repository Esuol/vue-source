# 异步组件

在我们平时的开发工作中，为了减少首屏代码体积，往往会把一些非首屏的组件设计成异步组件，按需加载。Vue 也原生支持了异步组件的能力，如下：

```js
Vue.component('async-example', function (resolve, reject) {
   // 这个特殊的 require 语法告诉 webpack
   // 自动将编译后的代码分割成不同的块，
   // 这些块将通过 Ajax 请求自动下载。
   require(['./my-async-component'], resolve)
})
```

示例中可以看到，Vue 注册的组件不再是一个对象，而是一个工厂函数，函数有两个参数 resolve 和 reject，函数内部用 setTimout 模拟了异步，实际使用可能是通过动态请求异步组件的 JS 地址，最终通过执行 resolve 方法，它的参数就是我们的异步组件对象。

在了解了异步组件如何注册后，我们从源码的角度来分析一下它的实现。

上一节我们分析了组件的注册逻辑，由于组件的定义并不是一个普通对象，所以不会执行 Vue.extend 的逻辑把它变成一个组件的构造函数，但是它仍然可以执行到 createComponent 函数，我们再来对这个函数做回顾，它的定义在 src/core/vdom/create-component/js 中：

```js
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  if (isUndef(Ctor)) {
    return
  }

  const baseCtor = context.$options._base

  // plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }

  // ...

  // async component
  let asyncFactory
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor, context)
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }
}
```