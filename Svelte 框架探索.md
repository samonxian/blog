# Svelte 框架探索

`Svelte` 的作者也是 `rollup` 的作者 **Rich Harris**，前端界的轮子哥。`Sevlte` 项目首次提交于 2016 年 11 月 16 日，目前版本是 `3.6.1`（2019-06-27），v3 版本进行了大改动，跟之前的版本有很大的差别（v1、v2 版本 API 用法跟 vue 很像，v3 完全属于自己的风格）。

看下 2016-12-02，**尤雨溪**大神对此框架的评价（当然已经过时了，但是核心的思想还是一致的）:

> 这个框架的 API 设计是从 Ractive 那边传承过来的（自然跟 Vue 也非常像），但这不是重点。Svelte 的核心思想在于『通过静态编译减少框架运行时的代码量』。举例来说，当前的框架无论是 React Angular 还是 Vue，不管你怎么编译，使用的时候必然需要『引入』框架本身，也就是所谓的运行时 (runtime)。但是用 Svelte 就不一样，一个 Svelte 组件编译了以后，所有需要的运行时代码都包含在里面了，除了引入这个组件本身，你不需要再额外引入一个所谓的框架运行时！

## 什么是 Svelte？

`Svelte` 跟 vue 和 react一样，是一个数据驱动组件框架。但是也有很大的不同，它是一个运行时框架，无需引入框架本身，同时也没用到虚拟 DOM（运行时框架特性决定了这个框架跟虚拟 DOM 无缘）。

>Svelte runs at *build time*, converting your components into highly efficient *imperative* code that surgically updates the DOM. As a result, you're able to write ambitious applications with excellent performance characteristics.

虽然没使用到虚拟 DOM，但一样可以达到出色的性能，而且对开发者编写代码是十分便捷。

## 与 React 和 Vue 有和不同？

那么我们先看下 `svelte` 的因为意思：苗条的。苗条的框架正是作者的初始目的，苗条包括代码编写量、打包大小等等。

总结一下这个框架的优势，即作者开发新框架的目的。

- 静态编译，无需引入框架自身

  一个 Svelte 组件是静态编译，所有需要的运行时代码都包含在里面了，除了引入这个组件本身，你感觉不到框架存在。

- 编写更少代码

  svelte 模板提供一些简便的用法，在维护和编写上都变得更简单，代码量更少（维护的代码），这些模板会编译为最终的js 代码。

- 只会打包使用到的代码

  即 tree shaking，这个概念本来也是作者首先提出来的，webpack 是参考了 rollup。

- 无需虚拟 DOM 也可进行响应式数据驱动

- 更便捷的响应式绑定

  既有响应式数据的优点，v3 版本也解决了 vue 数据绑定缺点，用起来十分方便。

## 简单用法对比

**react hook**

```jsx
import React, { useState } from 'react';
export default () => {
  const [a, setA] = useState(1);
  const [b, setB] = useState(2);
  function handleChangeA(event) {
    setA(+event.target.value);
  }
  function handleChangeB(event) {
    setB(+event.target.value);
  }
  return (
    <div>
      <input type="number" value={a} onChange={handleChangeA}/>
      <input type="number" value={b} onChange={handleChangeB}/>
      <p>{a} + {b} = {a + b}</p>
    </div>
  );
};
```

**vue**

```vue
<template>
  <div>
    <input type="number" v-model.number="a">
    <input type="number" v-model.number="b">
    <p>{{a}} + {{b}} = {{a + b}}</p>
  </div>
</template>

<script>
  export default {
    data: function() {
      return {
        a: 1,
        b: 2
      };
    }
  };
</script>
```

**svelte**

```html
<script>
  let a = 1;
  let b = 2;
</script>

<input type="number" bind:value={a} />
<input type="number" bind:value={b} />
<p>{a} + {b} = {a + b}</p>
```

都不用多说，一眼就看出来，svelte 简单多了。

## 为什么不使用虚拟 DOM？

在 react 和 vue 盛行的时代，你会听说虚拟 DOM 速度快，而且还可能被灌输一个概念，虚拟 DOM 的速度比真实 DOM 的速度要快。

所以如果你有这个想法，那么你肯定疑惑 svelte 没用到虚拟 DOM，它的速度为什么会快？

其实虚拟 DOM 并不是什么时候都快，看下粗糙的对比例子。

```mermaid
graph LR
A[状态数据] -->B(虚拟 DOM) 
B--> C[真实 DOM]
```

### 对比例子

这里并没有直接统计渲染的时间，通过很多条数据我们就可以感受出来他们直接的性能。**特别是点击每条数据的时候，明显感觉出来（由于是在线上的例子，所以首次渲染速度不准确，主要看点击的响应速度）。**

当然这仅仅是在 50000 条数据下的测试，对比一下框架所谓的速度，实际的情况下我们是不会一次性展示这么多数据的。所以在性能还可行的情况下，更多的选择是框架所带来的的便利，包括上手难度、维护难度、社区大小等等条件。

**svelte**

https://svelte.dev/repl/367a28b1867e41948b7897131dc87757?version=3.6.1

```html
<script>
  import { onMount } from "svelte";

  const  = [];
  for (let i = 0; i < 50000; i++) {
    list.push(i);
  }
  const beginTime = +new Date();
  let name = "world";
  let data = list;
  function click(index) {
    return () => {
      data[index] = "test";
    };
  }
  onMount(() => {
    const endTime = +new Date();
    console.log((endTime - beginTime) / 1000, 1);
  });
</script>

{#each data as d, i}
  <h1 on:click={click(i)}>
    <span>
      <span>
        <span>
          <span>Hello {name} {i} {d}!</span>
        </span>
      </span>
    </span>
  </h1>
{/each}
```

**vue**

http://jsrun.net/kFyKp/edit

```html
<div id="component-demo" class="demo">
  <div v-for="(d, i) in list" @click="click(i)"> 
    <span>
      <span> 
        <span>
          <span>
            Hello {{name}} {{i}} {{d}}!
          </span>
        </span>
      </span>
    </span>
  </div>
</div>
```

```js
const list  = []
for(let i = 0; i < 50000; i++) {
  list.push(i);
}
const beginTime = +new Date();
new Vue({
  el: '#component-demo',
  data: {
    list: list,
    name: 'Hello'
  },
  methods:{
    click(index){
      const list = new Array(50000);
      list[index] = 'test'
      this.list = list
    }
  },
  mounted(){
    const endTime = +new Date();
	  console.log((endTime - beginTime) / 1000,1);
  }
})
```

**react**

http://jsrun.net/TFyKp/edit

```jsx
const list  = []
for(let i = 0; i < 50000; i++) {
  list.push(i);
}
class App extends React.Component {
  constructor(props){
    super(props);
    this.state = {
      list
    } 
  }
  click(i) {
    return ()=>{
      list[i] = 'test'
      this.setState({
        list,
      })
    }
  }
  render() {
    return (
      <div>
        {this.state.list.map((v,k)=>{
          return(
            <h1 onClick={this.click(k)}>
              <span>
                <span>
                  <span>
                    <span>
                      Hello wolrd {k} {v}!
                    </span>
                  </span>
                </span>
              </span>
            </h1>
          )
        })}
      </div>
    )
  }
}

function render() {
  ReactDOM.render(
    <App />,
    document.getElementById('root')
  );
}

render();
```

### 总结

首先虚拟 DOM 不是一个功能，它只是实现数据驱动的开发的手段，没有虚拟 DOM 我们也可以实现数据驱动的开发方式，svelte 正是做了这个事情。

单纯从上面的对比例子来看，svelte 的速度比虚拟 DOM 更快（不同框架虚拟 DOM 实现会有差别）。虽然没有进行更深层次的对比，但是如果认为虚拟 DOM 速度快的观点是不完全对的，应该说虚拟 DOM 可以构建大部分速度还可以的 Web 应用。

## Svelte 有哪些好用的特性？

- 完全兼容原生 html 用法

  编写代码是那么的自然，如下面就是一个组件。

  ```html
  <script>
    const content = 'test';
  </script>
  <div>
    { test }
  </div>
  ```

- 响应式也是那么的自然

  ```html
  <script>
  	let count = 0;
  	function handleClick () {
  		// calling this function will trigger an
  		// update if the markup references `count`
  		count = count + 1;
  	}
  </script>
  <button on:click="handleClick">+1</button>
  <div>{ count }</div>
  ```

- 表达式也可以是响应式的

  这个就牛逼了，更加的自然，这种特性只有静态编译才能做到，这个就是 svelte 目前独有的优势。

  ```html
  <script>
  	let numbers = [1, 2, 3, 4];
  	function addNumber() {
  		numbers.push(numbers.length + 1);
  	}
  	$: sum = numbers.reduce((t, n) => t + n, 0);
  </script>
  <p>{numbers.join(' + ')} = {sum}</p>
  <button on:click={addNumber}>Add a number</button>
  ```

- 自动订阅的 svelte store

  这个其实就是订阅发布模式，不过 svelte 提供了自身特有的便捷的绑定方式（自动订阅），用起来是那么的自然，那么的爽。

  这种特性只有静态编译才能做到，这个就是 svelte 目前独有的优势。

  **stores.js**

  ```js
  import { writable } from 'svelte/store';
  export const count = writable(0);
  ```

  **A.svelte**

  ```html
  <script>
  	import { count } from './stores.js';
  </script>
  <h1>The count is {$count}</h1>
  ```

  **B.svelte**

  ```html
  <script>
  	import { count } from './stores.js';
    function increment() {
  		$count += 1;
  	}
  </script>
  <button on:click={increment}>增加</button>
  ```

- 所有组件都可以单独使用

  可以直接在 react、vue、angular 等框架中使用。

  ```js
  // SvelteComponent.js 是已经编译后的组件
  import SvelteComponent from './SvelteComponent';
  
  const app = new SvelteComponent({
  	target: document.body,
  	props: {
  		answer: 42
  	}
  });
  ```

## Svelte 有什么缺点？

svelte 是一个刚起步不久的前端框架，无论在维护人员还是社区上都是大大不如三大框架，这里列举一下本人认为的 svelte 存在的缺点。

- props 是可变的

  当然这也是这个框架故意这样设计的，这样 props 也是可以响应式的。

  ```html
  <script>
    export let title;
    title = '前缀' + title
  </script>
  <h1>
    { title }
  </h1>
  ```

- props 目前无法验证类型

  ```html
  <script>
    export let propOne;
    export let propTwo = 'defaultValue';
  </script>
  ```

- 无法通过自定义组件本身直接访问原生 DOM

  需要利用 props 的双向绑定特性，这就可能导致深层次组件的需要层层传递 DOM 对象（是子父传递，不是父子传递）。

  App.svelte

  ```html
  <script>
    export let customDOM;
  </script>
  <A bind:dom={customDOM} />
  <!--bind:this 是无用的，只有再原生才有用-->
  ```

  A.svelte

  ```html
  <script>
    export let dom;
  </script>
  
  <div bind:this={dom}>
    test
  </div>
  ```

- 只有组件才支持 svelte 的静态模板特性

  js 文件是不支持 sevelte 静态模板特性的，像下面这样是会报错的。

  ```js
  import { count } from './stores.js';
  function increment() {
    $count += 1;
  }
  ```

- 组件不支持 ts 用法

  找了一下，没找到可以支持 ts 的解决方案，如果有解决方案可以评论下。











































