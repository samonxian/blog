# JavaScript 的一些设计模式

> 设计模式的定义：在面向对象软件设计过程中针对特定问题的简洁而优雅的解决方案

设计模式是前人解决某个特定场景下对而总结出来的一些解决方案。可能刚开始接触编程还没有什么经验的时候，会感觉设计模式没那么好理解，这个也很正常。有些简单的设计模式我们有时候用到，不过没意识到也是存在的。

学习设计模式，可以让我们在处理问题的时候提供更多更快的解决思路。

当然设计模式的应用也不是一时半会就会上手，很多情况下我们编写的业务逻辑都没用到设计模式或者本来就不需要特定的设计模式。

## 适配器模式

这个使我们常使用的设计模式，也算最简单的设计模式之一，好处在于可以保持原有接口的数据结构不变动。

> 适配器模式（Adapter Pattern）是作为两个不兼容的接口之间的桥梁。

### 例子

适配器模式很好理解，假设我们和后端定义了一个接口数据结构为（可以理解为旧接口）：

```json
[
  {
    "label": "选择一",
    "value": 0
  },
  {
    "label": "选择二",
    "value": 1
  }
]
```

但是后端后面因为其他原因，需要定义返回的结构为（可以理解为新接口）：

```json
[
  {
    "label": "选择一",
    "text": 0
  },
  {
    "label": "选择二",
    "text": 1
  }
]
```

然后我们前端的使用到后端接口有好几处，那么我可以把新的接口字段结构适配为老接口的，就不需要各处去修改字段，只要把源头的数据适配好就可以了。

当然上面的是非常简单的场景，也是经常用到的场景。或许你会认为后端处理不更好了，的确是这样更好，但是这个不是我们讨论的范围。

## 单例模式

单例模式，从字面意思也很好理解，就是实例化多次都只会有一个实例。

有些场景实例化一次，可以达到缓存效果，可以减少内存占用。还有些场景就是必须只能实例化一次，否则实例化多次会覆盖之前的实例，导致出现 bug（这种场景比较少见）。

### 例子

实现弹框的一种做法是先创建好弹框, 然后使之隐藏, 这样子的话会浪费部分不必要的 DOM 开销, 我们可以在需要弹框的时候再进行创建, 同时结合单例模式实现只有一个实例, 从而节省部分 DOM 开销。下列为登入框部分代码:

```js
const createLoginLayer = function() {
  const div = document.createElement('div')
  div.innerHTML = '登入浮框'
  div.style.display = 'none'
  document.body.appendChild(div)
  return div
}
```

使单例模式和创建弹框代码解耦

```js
const getSingle = function(fn) {
  const result
  return function() {
    return result || result = fn.apply(this, arguments)
  }
}
const createSingleLoginLayer = getSingle(createLoginLayer)

document.getElementById('loginBtn').onclick = function() {
  createSingleLoginLayer()
}
```

## 代理模式

> 代理模式的定义：为一个对象提供一个代用品或占位符，以便控制对它的访问。

代理对象拥有本体对象的一切功能的同时，可以拥有而外的功能。而且代理对象和本体对象具有**一致的接口**，对使用者友好。

### 虚拟代理

下面这段代码运用代理模式来实现图片预加载,可以看到通过代理模式巧妙地将创建图片与预加载逻辑分离,，并且在未来如果不需要预加载，只要改成请求本体代替请求代理对象就行。

```js
const myImage = (function() {
  const imgNode = document.createElement('img')
  document.body.appendChild(imgNode)
  return {
    setSrc: function(src) {
      imgNode.src = src
    }
  }
})()

const proxyImage = (function() {
  const img = new Image()
  img.onload = function() { // http 图片加载完毕后才会执行
    myImage.setSrc(this.src)
  }
  return {
    setSrc: function(src) {
      myImage.setSrc('loading.jpg') // 本地 loading 图片
      img.src = src
    }
  }
})()

proxyImage.setSrc('http://loaded.jpg')
```

### 缓存代理

在原有的功能上加上结果缓存功能，就属于缓存代理。

原先有个功能是实现字符串反转（reverseString），那么在不改变 `reverseString` 的现有逻辑，我们可以使用缓存代理模式实现性能的优化，当然也可以在值改变的时候去处理下其他逻辑，如 Vue computed 的用法。

```js
function reverseString(str) {
  return str
    .split('')
    .reverse()
    .join('')
}
const reverseStringProxy = (function() {
  const cached = {}
  return function(str) {
    if (cached[str]) {
      return cached[str]
    }
    cached[str] = reverseString(str)
    return cached[str]
  }
})()
```

## 订阅发布模式

> 在[软件架构](https://zh.wikipedia.org/wiki/软件架构)中，**发布-订阅**是一种[消息](https://zh.wikipedia.org/wiki/消息)[范式](https://zh.wikipedia.org/wiki/范式)，消息的发送者（称为发布者）不会将消息直接发送给特定的接收者（称为订阅者）。而是将发布的消息分为不同的类别，无需了解哪些订阅者（如果有的话）可能存在。同样的，订阅者可以表达对一个或多个类别的兴趣，只接收感兴趣的消息，无需了解哪些发布者（如果有的话）存在。

或许你用过 `eventemitter`、node 的 `events`、Backbone  的 `events` 等等，这些都是前端早期，比较流行的数据流通信方式，即**订阅发布模式**。

从字面意思来看，我们需要首先订阅，发布者发布消息后才会收到发布的消息。不过我们还需要一个中间者来协调，从事件角度来说，这个中间者就是事件中心，协调发布者和订阅者直接的消息通信。

完成订阅发布整个流程需要三个角色：

- **发布者**

- **事件中心**

- **订阅者**

  订阅者是可以多个的。

以事件为例，简单流程如下：

**发布者->事件中心<=>订阅者**，订阅者需要向事件中心订阅指定的事件 -> 发布者向事件中心发布指定事件内容 -> 事件中心通知订阅者 -> 订阅者收到消息（可能是多个订阅者），到此完成了一次订阅发布的流程。

简单的代码实现如下：

```js
class Event {
  constructor() {
    // 所有 eventType 监听器回调函数（数组）
    this.listeners = {}
  }
  /**
   * 订阅事件
   * @param {String} eventType 事件类型
   * @param {Function} listener 订阅后发布动作触发的回调函数，参数为发布的数据
   */
  on(eventType, listener) {
    if (!this.listeners[eventType]) {
      this.listeners[eventType] = []
    }
    this.listeners[eventType].push(listener)
  }
  /**
   * 发布事件
   * @param {String} eventType 事件类型
   * @param {Any} data 发布的内容
   */
  emit(eventType, data) {
    const callbacks = this.listeners[eventType]
    if (callbacks) {
      callbacks.forEach((c) => {
        c(data)
      })
    }
  }
}

const event = new Event()
event.on('open', (data) => {
  console.log(data)
})
event.emit('open', { open: true })
```

Event 可以理解为事件中心，提供了订阅和发布功能。

**订阅者在订阅事件的时候，只关注事件本身，而不关心谁会发布这个事件；发布者在发布事件的时候，只关注事件本身，而不关心谁订阅了这个事件。**

## 观察者模式

> **观察者模式**定义了一种一对多的依赖关系，让多个**观察者**对象同时监听某一个目标对象，当这个目标对象的状态发生变化时，会通知所有**观察者**对象，使它们能够自动更新。

观察者模式我们可能比较熟悉的场景就是响应式数据，如 Vue 的响应式、Mbox 的响应式。

观察者模式有完成整个流程需要两个角色：

- 目标
- 观察者

简单流程如下：

**目标<=>观察者**，观察者观察目标（监听目标）-> 目标发生变化-> 目标主动通知观察者（可能是多个）。

简单的代码实现如下：

```js
/**
 * 观察监听一个对象成员的变化
 * @param {Object} obj 观察的对象
 * @param {String} targetVariable 观察的对象成员
 * @param {Function} callback 目标变化触发的回调
 */
function observer(obj, targetVariable, callback) {
  if (!obj.data) {
    obj.data = {}
  }
  Object.defineProperty(obj, targetVariable, {
    get() {
      return this.data[targetVariable]
    },
    set(val) {
      this.data[targetVariable] = val
      // 目标主动通知观察者
      callback && callback(val)
    },
  })
  if (obj.data[targetVariable]) {
    callback && callback(obj.data[targetVariable])
  }
}
```

可运行例子如下：

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta
      name="viewport"
      content="width=device-width,initial-scale=1,maximum-scale=1,viewport-fit=cover"
    />
    <title></title>
  </head>
  <body>
    <div id="app">
      <div id="dom-one"></div>
      <br />
      <div id="dom-two"></div>
      <br />
      <button id="btn">改变</button>
    </div>
    <script>
      /**
       * 观察监听一个对象成员的变化
       * @param {Object} obj 观察的对象
       * @param {String} targetVariable 观察的对象成员
       * @param {Function} callback 目标变化触发的回调
       */
      function observer(obj, targetVariable, callback) {
        if (!obj.data) {
          obj.data = {}
        }
        Object.defineProperty(obj, targetVariable, {
          get() {
            return this.data[targetVariable]
          },
          set(val) {
            this.data[targetVariable] = val
            // 目标主动通知观察者
            callback && callback(val)
          },
        })
        if (obj.data[targetVariable]) {
          callback && callback(obj.data[targetVariable])
        }
      }

      const obj = {
        data: { description: '原始值' },
      }

      observer(obj, 'description', value => {
        document.querySelector('#dom-one').innerHTML = value
        document.querySelector('#dom-two').innerHTML = value
      })

      btn.onclick = () => {
        obj.description = '改变了'
      }
    </script>
  </body>
</html>
```

## 装饰者模式

> 装饰器模式（Decorator Pattern）允许向一个现有的对象添加新的功能，同时又不改变其结构。

ES6/7 的`decorator` 语法提案，就是装饰者模式。

### 例子

```js
class A {
  getContent() {
    return '第一行内容'
  }
  render() {
    document.body.innerHTML = this.getContent()
  }
}

function decoratorOne(cla) {
  const prevGetContent = cla.prototype.getContent
  cla.prototype.getContent = function() {
    return `
      第一行之前的内容
      <br/>
      ${prevGetContent()}
    `
  }
  return cla
}

function decoratorTwo(cla) {
  const prevGetContent = cla.prototype.getContent
  cla.prototype.getContent = function() {
    return `
      ${prevGetContent()}
      <br/>
      第二行内容
    `
  }
  return cla
}

const B = decoratorOne(A)
const C = decoratorTwo(B)
new C().render()
```

## 策略模式

> 在策略模式（Strategy Pattern）中，一个行为或其算法可以在运行时更改。

假设我们的绩效分为 A、B、C、D 这四个等级，四个等级的奖励是不一样的，一般我们的代码是这样实现：

```js
/**
 * 获取年终奖
 * @param {String} performanceType 绩效类型，
 * @return {Object} 年终奖，包括奖金和奖品
 */
function getYearEndBonus(performanceType) {
  const yearEndBonus = {
    // 奖金
    bonus: '',
    // 奖品
    prize: '',
  }
  switch (performanceType) {
    case 'A': {
      yearEndBonus = {
        bonus: 50000,
        prize: 'mac pro',
      }
      break
    }
    case 'B': {
      yearEndBonus = {
        bonus: 40000,
        prize: 'mac air',
      }
      break
    }
    case 'C': {
      yearEndBonus = {
        bonus: 20000,
        prize: 'iphone xr',
      }
      break
    }
    case 'D': {
      yearEndBonus = {
        bonus: 5000,
        prize: 'ipad mini',
      }
      break
    }
  }
  return yearEndBonus
}
```

使用策略模式可以这样：

```js
/**
 * 获取年终奖
 * @param {String} strategyFn 绩效策略函数
 * @return {Object} 年终奖，包括奖金和奖品
 */
function getYearEndBonus(strategyFn) {
  if (!strategyFn) {
    return {}
  }
  return strategyFn()
}

const bonusStrategy = {
  A() {
    return {
      bonus: 50000,
      prize: 'mac pro',
    }
  },
  B() {
    return {
      bonus: 40000,
      prize: 'mac air',
    }
  },
  C() {
    return {
      bonus: 20000,
      prize: 'iphone xr',
    }
  },
  D() {
    return {
      bonus: 10000,
      prize: 'ipad mini',
    }
  },
}

const performanceLevel = 'A'
getYearEndBonus(bonusStrategy[performanceLevel])
```

这里每个函数就是一个策略，修改一个其中一个策略，并不会影响其他的策略，都可以单独使用。当然这只是个简单的范例，只为了说明。

策略模式比较明显的特性就是可以减少 if 语句或者 switch 语句。

## 职责链模式

> 顾名思义，责任链模式（Chain of Responsibility Pattern）为请求创建了一个接收者对象的链。这种模式给予请求的类型，对请求的发送者和接收者进行解耦。这种类型的设计模式属于行为型模式。

在这种模式中，通常每个接收者都包含对另一个接收者的引用。如果一个对象不能处理该请求，那么它会把相同的请求传给下一个接收者，依此类推。

### 例子

```js
function order(options) {
  return {
    next: (callback) => callback(options),
  }
}

function order500(options) {
  const { orderType, pay } = options
  if (orderType === 1 && pay === true) {
    console.log('500 元定金预购, 得到 100 元优惠券')
    return {
      next: () => {},
    }
  } else {
    return {
      next: (callback) => callback(options),
    }
  }
}

function order200(options) {
  const { orderType, pay } = options
  if (orderType === 2 && pay === true) {
    console.log('200 元定金预购, 得到 50 元优惠券')
    return {
      next: () => {},
    }
  } else {
    return {
      next: (callback) => callback(options),
    }
  }
}

function orderCommon(options) {
  const { orderType, stock } = options
  if (orderType === 3 && stock > 0) {
    console.log('普通购买, 无优惠券')
    return {}
  } else {
    console.log('库存不够, 无法购买')
  }
}

order({
  orderType: 3,
  pay: true,
  stock: 500,
})
  .next(order500)
  .next(order200)
  .next(orderCommon)
// 打印出 “普通购买, 无优惠券”
```

上面的代码，对 order 相关的进行了解耦，`order500`，`order200`、`orderCommon` 等都是可以单独调用的。

## 参考文章

- [订阅发布模式和观察者模式真的不一样](https://www.cnblogs.com/onepixel/p/10806891.html)

- [JavaScript 中常见设计模式整理](https://juejin.im/post/5afe6430518825428630bc4d)