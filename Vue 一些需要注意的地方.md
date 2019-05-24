# Vue 一些需要注意的地方

> 例子是在 jsrun.net 平台编写，不支持移动端平台，所以本文建议在 PC 端进行阅读。

这里总结了一些我们可能没注意到的 Vue 用法、可能产生 bug 的 Vue 用法。

## template 格式注意

官网建议使用小写的形式，驼峰是也是是支持的。

浏览器端直接解析 template 和使用 webpack vue-loader template 转换后的表现是不一样的。

### template 和 render 函数的优先级

在运行时解析 template 时，render 函数优先级高于 template。但是在 `.vue` 文件中 template 会被编译为 render 函数，会覆盖已有的 render 函数，优先级更高。

可以看这篇文章，[运行时-编译器-vs-只包含运行时](https://cn.vuejs.org/v2/guide/installation.html#运行时-编译器-vs-只包含运行时)。

### 浏览器端 vue 直接解析 template 

例子 <http://jsrun.net/hXyKp/edit>。

html

```html
<script src="https://unpkg.com/vue"></script>

<div id="component-demo">
  <div> 
    <h2>驼峰是大写命名的组件</h2>
    <component-one>组件1</component-one>
    <Component-one>组件2</Component-one>
    <component-One>组件3</component-One>
    <Component-One>组件4</Component-One>
    <ComponenT-One>组件5</ComponenT-One>
    <!-- 
      ComponentOne 应该显示“默认内容”，但是没用，说明组件失效，
      但是使用 webpack vue-loader 构建的是没问题的，浏览器 js 解析 template 
      有点不一样
    -->
    <ComponentOne></ComponentOne>
    <component-one></component-one>
    <h2>小写中划线命名的组件</h2>
    <component-two>组件1</component-two>
    <Component-two>组件2</Component-two>
    <component-Two>组件3</component-Two>
    <Component-Two>组件4</Component-Two>
    <ComponenT-Two>组件5</ComponenT-Two>
    <component-two></component-two>
    <!--这里必须使用 <></> 这种闭合方式-->
    <component-three />
    dd，后续的内容都没显示，应为被识别为 component-three 的子组件
    但是 component-three 没有使用插槽，所以没显示内容。
  </div>
</div>
```

js

```js
Vue.component('ComponentOne',{
  template: `
		<div>
			<slot>component-one 默认内容</slot>
		</div>
	`,
})
Vue.component('component-two',{
  template: `
		<div>
			<slot>component-two 默认内容</slot>
		</div>
	`,
})
Vue.component('component-three',{
  template: `
		<div>
		</div>
	`,
})

new Vue({
  el: '#component-demo',
})
```

经过实验可以知道，组件的命名是不分大小写的，驼峰式可以加上中划线，中划线也可以省略。

### vue-loader 的处理方式

驼峰是命名，下面的组件都能显示

```vue
<template>
  <div class="home">
    <img alt="Vue logo" src="../assets/logo.png" />
    <HelloWorld msg2="22" />
    <hello-world msg2="31" />
    <Hello-world msg2="41" />
    <Hello-World msg2="51" />
    <hello-World msg2="61" />
    <HelloWorld></HelloWorld>
  </div>
</template>

<script>
import HelloWorld from "@/components/HelloWorld.vue";
  
export default {
  name: "home",
  components: {
    HelloWorld
  }
};
</script>
```

小写中划线命名，下面的只有 `<hello-world msg2="31">` 可以显示，其他的会报错，看 console 即可。

```vue
<template>
  <div class="home">
    <img alt="Vue logo" src="../assets/logo.png" />
    <HelloWorld msg2="22" />
    <hello-world msg2="31" />
    <Hello-world msg2="41" />
    <Hello-World msg2="51" />
    <hello-World msg2="61" />
    <HelloWorld></HelloWorld>
  </div>
</template>

<script>
import HelloWorld from "@/components/HelloWorld.vue";
  
export default {
  name: "home",
  components: {
    // 一般我们都不至于处理，直接 HelloWorld 即可
    "hello-world": HelloWorld
  }
};
</script>
```

### 小结

建议规范为 `xxx-xxx` 小写中划线的使用方式，闭合方式使用 `<xx><xx/>`，不使用 `<xx/>` 形式。

```html
<template>
	<component-a></component-a>
  <component-b></component-b>
</template>
```

## data

可先阅读官网对 [data 的说明](<https://cn.vuejs.org/v2/api/#data>)。

### data 的两种类型

通过 Vue 创建实例的 data 需要时 object 类型，而注册组件的需要使用 function 类型（因为需要使用到 [Vue.extentd](<https://cn.vuejs.org/v2/api/#Vue-extend>)）。

> 通过提供 `data` 函数，每次创建一个新实例后，我们能够调用 `data` 函数，从而返回初始数据的一个全新副本数据对象。

当然一般我们都只有一个 Vue 实例，而组件是存在复用的情况。data 为函数形式，组件复用才不会使用同一个 data。

看下对比例子就清楚了，<http://jsrun.net/pXyKp/edit>。

html 文件

```html
<div id="component-demo">
  <div> 
    {{ title }}
    <button @click="update">更新父组件</button>
    <component-one></component-one>
  </div>
</div>
```

js 文件

```js

// Vue.component('component-one',{}) 等价于 
// Vue.component('component-one',Vue.extend({}))
Vue.component('component-one',{
  template: `
		<div>
			{{ title }}
			<button @click="update">更新子组件</button>
		</div>
	`,
  data: function(){
    return {
  		title: "子组件"
    }
  },
  // 这个将失效，this.title 为 undefined
  // data: {
  //   title: "子组件"
  // },
  methods: {
		update(){
  		this.title = "子组件更新了";
		}
  }
})

new Vue({
  el: '#component-demo',
  data: {
    title: "父组件",
  },
  // 创建 Vue 实例的时候，data 使用 function 的形式也是生效的
  // data: function(){
  //   return {
  //     title: "父组件"
  // 	 }
  // },
  methods: {
		update(){
  		this.title = "父组件更新了";
		}
  }
})
```

由例子可知道，创建 Vue 实例对 data 的两种数据类型都支持，本人建议所有的 data 都使用 function 类型。

### 不被代理的 data 属性

> 以 `_` 或 `$` 开头的属性 **不会** 被 Vue 实例代理，因为它们可能和 Vue 内置的属性、API 方法冲突。你可以使用例如 `vm.$data._property` 的方式访问这些属性。

看这个例子 <http://jsrun.net/YXyKp/edit>。

html

```html
<div id="component-demo">
  <div> 
    {{ $title }}
    <!-- 下面这种方式才可以 -->
    {{ $data.$title }}
    <br/>
    {{ _msg }}
    <!-- 下面这种方式才可以 -->
    {{ $data._msg }}
    <br/>
    <button @click="update">更新父组件</button>
  </div>
</div>
```

js

```js
new Vue({
  el: '#component-demo',
  data: {
    $title: "父组件",
    _msg: "展示信息"
  },
  methods: {
		update(){
      // this.$title = "父组件更新了" 这样更新无效
  		this.$data.$title = "父组件更新了";
		}
  }
})
```

console 警告输出如下

```sh
vue.js:634 [Vue warn]: Property "$title" must be accessed with "$data.$title" because properties starting with "$" or "_" are not proxied in the Vue instance to prevent conflicts with Vue internalsSee: https://vuejs.org/v2/api/#data

(found in <Root>)

vue.js:634 [Vue warn]: Property "_msg" must be accessed with "$data._msg" because properties starting with "$" or "_" are not proxied in the Vue instance to prevent conflicts with Vue internalsSee: https://vuejs.org/v2/api/#data

(found in <Root>)
```

## 非原始值的 prop 默认值

prop 的默认值类似，你需要对非原始值（即数组、对象等引用类型）使用一个工厂方法（函数返回非原始值）。

例子可以看这个 <http://jsrun.net/MXyKp/edit>。

```js
Vue.component('component-one',{
  template: `
		<div>
      <div v-for="item in list">
        {{ item }}
      </div>
		</div>
	`,
  props: {
    // 直接使用数组或者对象赋值，是没效果的
    // list: [1,2,3,4],
    list: {
      default: () => [1,2,3,4]
    }
  },
});
```

