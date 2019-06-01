# pathc

通过前一章的分析我们知道，当我们通过 createComponent 创建了组件 VNode，接下来会走到 vm._update，执行 vm.__patch__ 去把 VNode 转换成真正的 DOM 节点。这个过程我们在前一章已经分析过了，但是针对一个普通的 VNode 节点，接下来我们来看看组件的 VNode 会有哪些不一样的地方。

patch 的过程会调用 createElm 创建元素节点，回顾一下 createElm 的实现，它的定义在 src/core/vdom/patch.js 中：

```js
function createElm (
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  // ...
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }
  // ...
}
```

## createComponent

我们删掉多余的代码，只保留关键的逻辑，这里会判断 createComponent(vnode, insertedVnodeQueue, parentElm, refElm) 的返回值，如果为 true 则直接结束，那么接下来看一下 createComponent 方法的实现：

```js
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
  let i = vnode.data
  if (isDef(i)) {
    const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
    if (isDef(i = i.hook) && isDef(i = i.init)) {
      i(vnode, false /* hydrating */)
    }
    // after calling the init hook, if the vnode is a child component
    // it should've created a child instance and mounted it. the child
    // component also has set the placeholder vnode's elm.
    // in that case we can just return the element and be done.
    if (isDef(vnode.componentInstance)) {
      initComponent(vnode, insertedVnodeQueue)
      insert(parentElm, vnode.elm, refElm)
      if (isTrue(isReactivated)) {
        reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
      }
      return true
    }
  }
}
```