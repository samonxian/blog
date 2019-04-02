# JavaScript 之错误异常处理

> 我的建议是不要隐藏错误，勇敢地抛出来。没有人会因为代码出现 bug 导致程序崩溃而羞耻，我们可以让程序中断，让用户重来。错误是无法避免的，如何去处理它才是最重要的。

JavaScript 提供一套异常处理机制，异常是干扰程序正常流程的非正常的事故。而没人可以保持程序没有 bug，那么上线后遇到特殊的 bug，如何更快的定位问题所在呢？这就是我们这个专题需要讨论的问题。

下面会从 JavaScript Error 基础知识、如何捕获异常、如何方便的在线上报错误等方面来叙述，本人也是根据网上的知识点进行了一些总结和分析（我只是互联网的搬运工，不是创造者），如果有什么错漏的情况，请在 issue 上狠狠的批评我。

**这个专题目前是针对浏览器的，还没考虑到 node.js**，不过都是 JavaScript Es6 语法，大同小异。

## 什么时候 JavaScript 会抛出错误呢？

一般分为两种情况：

- JavaScript 自带错误
- 开发者主动抛出的错误

### JavaScript 引擎自动抛出的错误

大多数场景下我们遇到的错误都是这个错误。这个很好理解，Javscript 语法错误、代码调用顺序问题、类型问题等，JavaScript 引擎就会自动触发此类错误。如下一些场景：

- 场景一

  ```js
  console.log(a.notExited)
  // 浏览器会抛出 Uncaught ReferenceError: a is not defined
  ```

- 场景二

  ```js
  const a;
  // 浏览器抛出 Uncaught SyntaxError: Missing initializer in const declaration
  ```

    语法错误一般直接浏览器第一时间就抛出错误，不会等到执行的时候才会报错。a

- 场景三

  ```js
  let data;
  data.forEach(v=>{})
  // Uncaught TypeError: Cannot read property 'forEach' of undefined
  ```

### 手动抛出的错误

一般都是类库开发的自定义异常（如参数等不合法的异常抛出）。或者重新修改错误 message 的情况进行上报，以方便理解。

- 场景一

  ```js
  function sum(a,b){
    if(typeof a !== 'number') {
      throw TypeError('Expected a to be a number.');
    }
    if(typeof b !== 'number') {
      throw TypeError('Expected b to be a number.');
    }
    return a + b;
  }
  sum(3,'d');
  // 浏览器抛出 uncaught TypeError: Expected b to be a number.
  ```

- 场景二

  当然我们不一定需要这样做。

  ```js
  let data;
  
  try {
    data.forEach(v => {});
  } catch (error) {
    error.message = 'data 没有定义，data 必须是数组。';
    error.name = 'DataTypeError';
    throw error;
  }
  ```

## 如何创建 Error 对象？

创建语法如下：

```js
new Error([message[,fileName,lineNumber]])
```

省略 `new ` 语法也一样。

其中`fileName` 和 `lineNumber` 不是所有浏览器都兼容的，谷歌也不支持，所以可以忽略。

`Error` 构造函数是通用错误类型，还有 `TypeError`、`RangeError` 等类型。

### Error 实例

这里列举的都是 Error 层的原型链，更深层的原型链的继承属性和方便不做说明。一些有兼容性的而且不常用的属性和方法不做说明。

```js
console.log(Error.prototype)
// 浏览器输出 {constructor: ƒ, name: "Error", message: "", toString: ƒ}
```

其他错误类型构造函数是继承 `Error`，实例是一致的。

#### 属性

- Error.prototype.message

  错误信息， `Error("msg").message === "msg"`。

- Error.prototype.name

  错误类型(名字)， `Error("msg").name === "Error”`。如果是 TypeError，那么 name 为 TypeError。

#### 方法

- Error.prototype.constructor

  这个不用说明了，必然有一个构造函数。

- Error.prototype.toString

  返回值跟 Error.prototype.name 一致。

## Error 类型

除了通用的 `Error` 构造函数外，JavaScript还有常见的 5 个其他类型的错误构造函数。

### TypeError

创建一个 Error 实例，表示错误的原因：**变量或参数不属于有效类型**。

```js
throw TypeError("类型错误");
// Uncaught TypeError: 类型错误
```

### RangeError

创建一个 `Error` 实例，表示错误的原因：**数值变量或参数超出其有效范围**。

```js
throw RangeError("数值超出有效范围");
// Uncaught RangeError: 数值超出有效范围
```

### ReferenceError

创建一个 `Error` 实例，表示错误的原因：**无效引用**。

```js
throw ReferenceError("无效引用");
// Uncaught ReferenceError: 无效引用
```

### SyntaxError

创建一个 `Error` 实例，表示错误的原因：**语法错误**。这种场景很少用，除非类库定义了新语法(如模板语法)。

```js
throw SyntaxError("语法错误");
// Uncaught SyntaxError: 语法错误
```

### URIError

创建一个 `Error` 实例，表示错误的原因：**涉及到 uri 相关的错误**。

```js
throw URIError("url 不合法");
// Uncaught RangeError: url 不合法
```

## 自定义 Error 类型

自定义新的 Error 类型需要继承 Error ，如下自定义 `CustomError`：

```js
function CustomError(...args){
  class InnerCustomError extends Error {
    name = "CustomError";
  }
  return new InnerCustomError(...args);
}
```

继承 Error 后，我们只需要对 `name` 做重写，然后封装成可直接调用的函数即可。

## 如何拦截 JavaScript 错误？

**拦截并非单纯的获取。**

既然没人能保证 web 应用不会出现 bug，那么出现异常报错时，如何拦截并进行一些操作呢？

### try…catch… 拦截

这是拦截 JavaScript 错误，拦截后，如果不手动抛出错误，这个错误将静默处理。平常写代码如果我们知道某段代码可能会出现报错问题，就可以使用这种方式。如下：

```js
const { data } = this.props;
try {
  data.forEach(d=>{});
  // 如果 data 不是数组就会报错
} catch(err){
  console.error(err);
  // 这里可以做上报处理等操作
}
```

#### 一些使用方式

##### 十分不友好的处理方式

`try...catch...` 使用需要注意，try…catch… 后，错误会被拦截，如果不主动抛出错误，那么无法知道报错位置。如下面的处理方式就是不好的。

```js
function badHandler(fn) {
  try {
    return fn();
  } catch (err) { /**noop，不做任何处理**/ }
  return null;
}
badHandler();
```

这样 fn 回调发送错误后，我们无法知道错误是在哪里发生的，因为已经被 try…catch 了，那么如何解决这个问题呢？

##### 相对友好但糟糕的处理方式

```js
function CustomError(...args){
  class InnerCustomError extends Error {
    name = "CustomError";
  }
  return new InnerCustomError(...args);
}
function uglyHandlerImproved(fn) {
  try {
    return fn();
  } catch (err) { 
    throw new CustomError(err.message);
  }
  return null;
}
badHandler();
```

现在，这个自定义的错误对象包含了原本错误的信息，因此变得更加有用。但是因为再度抛出来，依然是未处理的错误。

#### try…catch… 可以拦截异步错误吗？

这个也要分场景，也看个人的理解方向，首先理解下面这句话：

**try…catch 只会拦截当前执行环境的错误，try 块中的异步已经脱离了当前的执行环境，所以 try…catch… 无效。**

`setTimeout` 和 `Promise` 都无法通过 try…catch 捕获到错误，指的是 try 包含异步（非当前执行环境），不是异步包含 try（当前执行环境）。异步无效和有效 try…catch 如下：

- setTimeout

  这个无效：

  ```js
  try {
    setTimeout(() => {
      data.forEach(d => {});
    });
  } catch (err) {
    console.log('这里不会运行');
  }
  ```

  下面的 try…catch 才会有效：

  ```js
  setTimeout(() => {
    try {
      data.forEach(d => {});
    } catch (err) {
      console.log('这里会运行');
    }
  });
  ```

- Promise

  这个无效：

  ```js
  try {
    new Promise(resolve => {
      data.forEach(d => {});
      resolve();
    });
  } catch (err) {
    console.log('这里不会运行');
  }
  ```

  下面的 try…catch 才会有效：

  ```js
  new Promise(resolve => {
    try {
      data.forEach(d => {});
    } catch (err) {
      console.log('这里会运行');
    }
  });
  ```

#### 小结

不是所有场景都需要 try…catch… 的，如果所有需要的地方都 try…catch，那么代码将变得臃肿，可读性变差，开发效率变低。那么我需要统一获取错误信息呢？有没有更好的处理方式？当然有，后续会提到。

### Promise 错误拦截

`Promise.prototype.catch` 可以达到 try…catch 一样的效果，只要是在 Promise 相关的处理中报错，都会被 catch 到。当然如果你在相关回调函数中 try…catch，然后做了静默提示，那么也是 catch  不到的。

如下会被 catch 到：

```js
new Promise(resolve => {
  data.forEach(v => {});
}).catch(err=>{/*这里会运行*/})
```

下面的不会被 catch 到：

```js
new Promise(resolve => {
  try {
  	data.forEach(v => {});
  }catch(err){}
}).catch(err=>{/*这里不会运行*/})
```

Promise 错误拦截，这里就不详细说了，如果你看懂了 try…catch，这个也很好理解。

### setTimeout 等其他异步错误拦截呢？

目前没有相关的方式直接拦截 setTimeout 等其他异步操作。

如果要拦截 setTimeout 等异步错误，我们需要在异步回调代码中处理，如：

```js
setTimeout(() => {
  try {
    data.forEach(d => {});
  } catch (err) {
    console.log('这里会运行');
  }
});
```

这样可以拦截到 setTimeout 回调发生的错误，但是如果是下面这样 try…catch 是无效的：

```js
try {
  setTimeout(() => {
    data.forEach(d => {});
  });
} catch (err) {
  console.log('这里不会运行');
}
```

## 如何获取 JavaScript 错误信息？

你可以使用上面**拦截错误信息**的方式获取到错误信息。但是呢，你要每个场景都要去拦截一遍吗？首先我们不确定什么地方会发生错误，然后我们也不可能每个地方都去拦截错误。

不用担心，JavaScript 也考虑到了这一点，提供了一些便捷的获取方式(不是拦截，错误还是会终止程序的运行，除非主动拦截了)。

### window.onerror 事件获取错误信息

onerror 事件无论是异步还是非异步错误（除了 Promise 错误），onerror 都能捕获到运行时错误。

需要注意一下几点：

- window.onerror 函数只有在返回 true 的时候，异常才不会向上抛出，否则即使是知道异常的发生控制台还是会显示 Uncaught Error: xxxxx。如果使用 `addEventListener`，`event.preventDefault()` 可以达到同样的效果。
- window.onerror 是无法捕获到网络异常的错误、`<img/>`或`<script>`资源加载失败等错误。不过如果你使用了 fetch 等支持 promise 的方式，错误可以通过 unhandledrejection 方式拿到错误信息。

语法：

- window.onerror

  ```js
  window.onerror = function(message, source, lineno, colno, error) { ... }
  ```

- window.addEventListener('error')

  ```js
  window.addEventListener('error', function(event) { ... })
  ```

详细看[这里](https://developer.mozilla.org/zh-CN/docs/Web/API/GlobalEventHandlers/onerror)。

### unhandledrejection 事件获取 Promise 错误

unhandledrejection 存在兼容性问题，IE、Edge、火狐等目前都不支持。

用法如下：

```js
window.addEventListener("unhandledrejection", event => {
  console.warn(`UNHANDLED PROMISE REJECTION: ${event.reason}`);
});
```

或者

```js
window.onunhandledrejection = event => {
  console.warn(`UNHANDLED PROMISE REJECTION: ${event.reason}`);
};
```

详细看[这里](https://developer.mozilla.org/en-US/docs/Web/API/WindowEventHandlers/unhandledrejection_event)。

## 如何处理错误信息？

错误对象 `Error` 包含如下的字段返回：

- name

  错误的类型，一般有 `Error`、`TypeError`、`ReferenceError`、`RangeError`、`URIError` 等，当然也可以是自定义的错误类型。

- message

  错误的详细信息，可以是 JavaScript 自动抛出的错误信息，也可以是手动抛出的自定义信息。

- stack

  这个不是规范，存在兼容性。经测试，谷歌、火狐、Edge、safar 都支持此特性(都是在最新的版本下测试 2019-04-2)，IE 不支持。

`name` 和 `message` 我们都不用做什么处理，主要是要针对 stack 做处理，一般我们需要把，这三个字段的信息提交到错误处理系统，针对性处理。

同时我们显示的代码是被压缩后的代码，需要使用 sourceMap 进行映射还原代码。

**后续会补充这个讨论**。

## 参考文章

- [Error(MDN)](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Error)

- [A Guide to Proper Error Handling in JavaScript](https://www.sitepoint.com/proper-error-handling-javascript/)
- [Anatomy of a JavaScript Error](https://blog.bugsnag.com/anatomy-of-a-javascript-error/)