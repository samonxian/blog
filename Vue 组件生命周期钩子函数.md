# Vue 组件生命周期钩子函数

> 例子是在 jsrun.net 平台编写，不支持移动端平台，所以本文建议在 PC 端进行阅读。

所谓**生命周期钩子函数**（简称**生命周期函数**），指的是组件的**创建**、**更新**、**销毁**三个阶段所触发执行的函数。根据每个阶段触发的钩子函数，我们可以相应的做一些操作，如获取后端接口数据、监听事件、执行事件、执行定时器、移除事件、清理定时器等等。

生命周期根据上面的三个阶段，本人总结为：

- 实例化期

  组件创建

- 存在期

  组件更新

- 销毁期

  组件销毁

后续会根据这三大周期，分别说明生命周期函数。

为了方便理解，本人在 [http://jsrun.net](http://jsrun.net/) 上编写了例子。（一个跟 jsfiddle 差不多的网站，国内 jsfiddle 被墙了）

## 生命周期示意图

首先看下官网的生命周期示意图，这里的生命周期函数都是针对浏览器端的，服务端目前只支持 `beforeCreate` 和 `created` 这两个生命周期函数。

其中官网并没有把 `render`、`renderError` 函数归纳为生命周期钩子函数。

![](https://cn.vuejs.org/images/lifecycle.png)

其中 `Has "el" option` 对比如下：

**有 el**

```js
// 有el属性的情况下
new Vue({
  el: "#app",
  beforeCreate: function() {
    console.log("调用了beforeCreate");
  },
  created: function() {
    console.log("调用了created");
  },
  beforeMount: function() {
    console.log("调用了beforeMount");
  },
  mounted: function() {
    console.log("调用了mounted");
  }
});

// 输出结果
// 调用了beforeCreate
// 调用了created
// 调用了beforeMount
// 调用了mounted
```

**无 el**

```js
// 有el属性的情况下
const vm new Vue({
  beforeCreate: function() {
    console.log("调用了beforeCreate");
  },
  created: function() {
    console.log("调用了created");
  },
  beforeMount: function() {
    console.log("调用了beforeMount");
  },
  mounted: function() {
    console.log("调用了mounted");
  }
});

// 输出结果
// 调用了beforeCreate
// 调用了created
```

无 el 时，如果需要挂载，可以这样处理：`vm.$mount('#app')`。效果一样了，本质上没区别，只是用法更灵活。

## 实例化期

实例化期会涉及到以下生命周期函数（执行顺序自上而下）：

- beforeCreate
- created
- beforeMount
- mouted

其中 `beforeCreate` 和 `created` 中间会触发 `render` 函数，如果有 template 会转换为 render 函数进行渲染。（当然如果组件的 定义了 render 函数，那么 render 函数优先级更高）

详细的例子请看 <http://jsrun.net/LZyKp/edit>

```js
// 输出请看 右下角 Console 命令行工具
new Vue({
  el: '#dynamic-component-demo',
  data: {
    num: 2,
  },
  beforeCreate(){
    console.log("beforeCreate",this.num,this.a);
    // 输出为 befoerCreate,,
    // this.num 数据还没监测，this.a 方法未绑定
  },
  created(){
    console.log("created",this.num,this.a,this.$el);
    // 输出为 created, 2, function () { [native code] }
  },
  beforeMount(){
    console.log(this.$el.innerText);
    // 输出 {{ num }}，还是原来的 DOM 内容
  },
  mounted(){
    console.log(this.$el.innerText);
    // 输出 2，已经是 vue 渲染的 DOM 内容
  },
  methods: {
    a(){}
  }
})
```

### beforeCreate

在实例初始化之后，数据观测 (data observer) 和 event/watcher 事件配置之前被调用。

### created

在实例创建完成后被立即调用。在这一步，实例已完成以下的配置：数据观测 (data observer)，属性和方法的运算，watch/event 事件回调。然而，挂载阶段还没开始，`$el` 属性目前不可见。

### beforeMount

在挂载开始之前被调用：相关的 `render` 函数首次被调用。`$el` 属性已经可见，但还是原来的 DOM，并非是新创建的。

### mounted

`el` 被新创建的 `vm.$el` 替换，并挂载到实例上去之后调用该钩子。如果 root 实例挂载了一个文档内元素，当 `mounted` 被调用时 `vm.$el` 也在文档内。

注意 `mounted` **不会**承诺所有的子组件也都一起被挂载。如果你希望等到整个视图都渲染完毕，可以用 [vm.$nextTick](https://cn.vuejs.org/v2/api/#vm-nextTick) 替换掉 `mounted`：

```js
mounted: function () {
  this.$nextTick(function () {
    // Code that will run only after the
    // entire view has been rendered
  })
}
```

## 存在期

存在期会涉及到以下生命周期函数：

- beforeUpdate
- updated

Vue 需要改变数据才会触发组件重新渲染，才会触发上面的存在期钩子函数。其中 `beforeUpdate` 和 `updated` 中间会触发 `render` 函数。

例子请看 <http://jsrun.net/8ZyKp/edit>。

```js
// 需要点击更新按钮
// 连续点击更新按钮，都会是 2 秒后不点击更新才会输出 “2 秒没更新了”
// 输出请看 右下角 Console 命令行工具
new Vue({
  el: '#dynamic-component-demo',
  data: {
    num: 2,
  },
  beforeUpdate(){
    clearTimeout(this.clearTimeout);
    this.clearTimeout = setTimeout(function(){
      console.log("2 秒没更新了");
    },2000);
    console.log("beforeUpdate",this.num,this.$el.innerText);
    // 第一次点击更新，输出为 beforeUpdate,3,点击更新 2
  },
  updated(){
    console.log("updated",this.num,this.$el.innerText);
    // 第一次点击更新，输出为 updated,3,点击更新 3
  },
  methods: {
    updateComponent(){
      this.num++;
    }
  }
})
```

### beforeUpdate

数据更新时，虚拟 DOM 变化之前调用，这里适合在更新之前访问现有的 DOM，比如手动移除已添加的事件监听器。

**请不要在此函数中更改状态，否则会触发死循环。**

### updated

数据更新和虚拟 DOM 变化之后调用。

当这个钩子被调用时，组件 DOM 已经更新，所以你现在可以执行依赖于 DOM 的操作。然而在大多数情况下，你应该避免在此期间更改状态。如果要相应状态改变，通常最好使用[计算属性](https://cn.vuejs.org/v2/api/#computed)或 [watcher](https://cn.vuejs.org/v2/api/#watch) 取而代之。

和 mounted 一样， `updated` **不会**承诺所有的子组件也都一起被重绘。如果你希望等到整个视图都重绘完毕，可以用 [vm.$nextTick](https://cn.vuejs.org/v2/api/#vm-nextTick) 替换掉 `updated`：

```js
updated: function () {
  this.$nextTick(function () {
    // Code that will run only after the
    // entire view has been re-rendered
  })
}
```

**请不要在此函数中更改状态，否则会触发死循环。**

## 销毁期

销毁期会涉及到以下生命周期函数：

- beforeDestroy
- destroyed

例子请看 <http://jsrun.net/QZyKp/edit>。

```js
// 切换 tab，看右下角 console 输出
Vue.component('tab-home', { 
	template: '<div>Home component</div>',
  beforeDestroy(){
    console.log("tab-home","beforeDestroy");
  },
  destroyed(){
    console.log("tab-home","destroyed");
  },
})
Vue.component('tab-posts', { 
	template: '<div>Posts component</div>',
  beforeDestroy(){
    console.log("tab-posts","beforeDestroy");
  },
  destroyed(){
    console.log("tab-posts","destroyed");
  },
})
Vue.component('tab-archive', { 
	template: '<div>Archive component</div>',
  beforeDestroy(){
    console.log("tab-archive","beforeDestroy");
  },
  destroyed(){
    console.log("tab-archive","destroyed");
  },
})

new Vue({
  el: '#dynamic-component-demo',
  data: {
    currentTab: 'Home',
    tabs: ['Home', 'Posts', 'Archive']
  },
  computed: {
    currentTabComponent: function () {
      return 'tab-' + this.currentTab.toLowerCase()
    }
  }
})
```

### beforeDestroy

实例销毁之前调用，在这一步，实例仍然完全可用。一般在这里移除事件监听器、定时器等，避免内存泄漏

### destroyed

Vue 实例销毁后调用。调用后，Vue 实例指示的所有东西都会解绑定，所有的事件监听器会被移除，所有的子实例也会被销毁。

所以如果需要用到 Vue 实例指示的所用绑定的东西，需要在 beforeDestroy 中使用。这么说，destroyed 函数能做的事，在 beforeDestroy 也能做，所以没必要在 destroyed 函数中处理。

### 其他不常用的生命周期函数

- activated

  当组件激活的时候调用，可以参考[构建组件 - keep-alive](<https://cn.vuejs.org/v2/api/#keep-alive>)

- deactivated

  当组件停用的时候调用，可以参考[构建组件 - keep-alive](<https://cn.vuejs.org/v2/api/#keep-alive>)

- errorCaptured

  这个生命钩子详细请看[官网](<https://cn.vuejs.org/v2/api/#errorCaptured>)，2.5.0 新增的，当捕获一个来自子孙组件的错误时被调用。

## 使用注意

生命周期函数请不要使用 ES6 箭头函数，否则 this 指向会有问题。

请看这个例子 <http://jsrun.net/cZyKp/edit>。

```js
// 输出请看 右下角 Console 命令行工具
new Vue({
  el: '#dynamic-component-demo',
  data: {
    num: 2,
  },
  created: ()=>{
    console.log("created",this);
    // 输出为 created,[object Window]
    // this 指向不是 Vue 实例而是父级 this
  }
})
```

## 发散思考

既然每个组件都有生命周期，那么父子、兄弟组件之间执行顺序有什么区别呢？

那么你首先需要了解到 `template` 什么时候转变成虚拟 DOM，而虚拟 DOM 又是在什么时候构建完成，然后又是什么时候挂载到真实的 DOM 上面。

这里不深入说明 vue 原理，大概了解下 template 到虚拟 DOM 的过程：

`template` -> `AST 树` -> `render 函数`

实际上我们关注点在 render 函数，可以了解下[运行时-编译器-vs-只包含运行时](https://cn.vuejs.org/v2/guide/installation.html#运行时-编译器-vs-只包含运行时)。虚拟 DOM 就是在 render 函数中形成。如果你使用过 JSX，那么可以简单的把 JSX 理解为一个个的虚拟 DOM。当然 render 函数也会执行挂载到真实 DOM 的逻辑。

**虚拟 DOM 就是一个树，vue 会把所有组件映射为虚拟 DOM 后渲染到真实 DOM 节点中，挂载无论如何一定是在最后执行的，所以所有组件 mounted 函数输出的日志必然是连在一起的，updated 也同理。**

如果你不理解这句话，可以看这个例子 <http://jsrun.net/7hyKp/edit>。

### 父子组件生命周期函数执行顺序

render 函数才是父子组件生命周期函数的分割线。

而 render 函数的顺序如下。

实例化期：`beforeMount` -> `render` -> `mounted`

存在期：`beforeUpdate` -> `render` -> `updated`

详细请看在线例子，<http://jsrun.net/vhyKp/edit>。

### 兄弟组件生命周期执行顺序

详细请看在线例子，<http://jsrun.net/NhyKp/edit>。

## 参考文章

- [实例生命周期钩子](https://cn.vuejs.org/v2/guide/instance.html#实例生命周期钩子)
- [如何解释vue的生命周期才能令面试官满意？](https://juejin.im/post/5ad10800f265da23826e681e#comment)
- [vue生命周期详解](https://juejin.im/post/5afd7eb16fb9a07ac5605bb3#heading-8)
- [Vue 生命周期深入](https://segmentfault.com/a/1190000014705819)
- [运行时-编译器-vs-只包含运行时](https://cn.vuejs.org/v2/guide/installation.html#运行时-编译器-vs-只包含运行时)

