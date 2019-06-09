# 深入响应式原理

前面 2 章介绍的都是 Vue 怎么实现数据渲染和组件化的，主要讲的是初始化的过程，把原始的数据最终映射到 DOM 中，但并没有涉及到数据变化到 DOM 变化的部分。而 Vue 的数据驱动除了数据渲染 DOM 之外，还有一个很重要的体现就是数据的变更会触发 DOM 的变化。

其实前端开发最重要的 2 个工作，一个是把数据渲染到页面，另一个是处理用户交互。Vue 把数据渲染到页面的能力我们已经通过源码分析出其中的原理了，但是由于一些用户交互或者是其它方面导致数据发生变化重新对页面渲染的原理我们还未分析。

考虑如下示例：

```javaScript
<div id="app" @click="changeMsg">
  {{ message }}
</div>

```

```js
var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  },
  methods: {
    changeMsg() {
      this.message = 'Hello World!'
    }
  }
})

```