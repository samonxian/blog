虽然现在使用 **async 函数** 就可以替代 **Generator** 执行器了，不过了解下 Generator 执行器的原理还是挺有必要的。

如果你不了解 Generator，那么你需要看[这里](http://es6.ruanyifeng.com/#docs/generator)。

**例子都可以在 Console 中运行的（谷歌版本 76.0.3809.100），都是以最新浏览器支持的 JavaScript 特性来编写的，不考虑兼容性。**

## 术语区分

Generator 是 Generator function 运行后返回的对象。

## 执行器的原理

### 原理一

有 Generator next 函数的特性，next 函数运行后会返回如下结构：

```json
{ value: 'world', done: false }
```

或者

```json
{ value: undefined, done: true }
```

那么我们可以使用**递归**运行 next 函数，如果 done 为 true 则停止 next 函数的运行。

### 原理二

yield 表达式本身没有返回值，或者说总是返回 undefined。

next 方法可以带一个参数，该参数就会被当作上一个 yield 表达式的返回值。

由于 next 方法的参数表示上一个 yield 表达式的返回值，所以在第一次使用 next 方法时，传递参数是无效的。

```js
function* test() {
  const a = yield "a"
  console.log(a)
  return true
}
const gen = test()
// 第一次 next() 会卡在 yield a 中，next 返回 {value: "a", done: false}
// 第一个 next() 函数的参数是无效的，无需传递
const nextOne = gen.next() 
// 传递参数 nextOne.value
// test 函数中的 console.log 输出 'a'
// 第二次的 next() 返回值为 {value: true, done: true}
gen.next(nextOne.value)
```

### 原理三

`gen.throw(exception)` 可以抛出异常，并恢复生成器的执行，返回带有 done 及 value 两个属性的对象。而且该异常通常可以通过 `try...catch` 块进行捕获。那么我们可以抛出 Promise 错误，这样就可以使用 `try catch` 拦截 Promise 的错误了。

不过需要注意一点：

**`gen.throw()` 在同步的环境中会直接终止代码运行，无法 try catch，了解这点特别重要。不过 `gen.throw()` 在异步 Promise 环境中抛出才会有返回值， 同时只要 generator 函数中 try catch yield 语句就可以捕获 `gen.throw` 出来的错误。**

所以需要确保把所有 `gen.throw()` 都包装在一个 Promise 执行环境中才会生效。

## 简单的执行器

简单的执行器代码如下：

```js
/**
 * generator 执行器
 * @param {Function} func generator 函数
 * @return {Promise} 返回一个 Promise 对象
 */
function generatorExecuter(func) {
  const gen = func()

  function recursion(prevValue) {
    const next = gen.next(prevValue)
    const value = next.value
    const done = next.done
    if (done) {
      return Promise.resolve(value)
    } else {
      return recursion(value)
    }
  }
  return recursion()
}
```

代码并不复杂，最简单的执行器就出来了。如果你是一步一步的看文章过来的，都理解了原理，那么这些代码也很好理解。

## 考虑 yield 的类型是 Promise

上面的代码执行如下的 Generator 函数是不正确的：

```js
function* test() {
  const a = yield Promise.resolve('a')
  console.log(a)
  return true
}
generatorExecuter(test)
```

运行上面的代码后，test 函数 console.log 输出的不是 `a`，而是 `Promise {<resolved>: "a"}`。

那么我们代码需要这样处理：

```js
/**
 * generator 执行器
 * @param {GeneratorFunction} generatorFunc generator 函数
 * @return {Promise} 返回一个 Promise 对象
 */
function generatorExecuter(generatorFunc) {
  return new Promise((resolve) => {
    const generator = generatorFunc()
    // 触发第一次执行下面定义的 next 函数
    onFullfilled()
    
    /**
     * Promise 成功处理的回调函数
     * 回调函数会执行 generator.next()
     * @param {Any} value yield 表达式的返回值
     */
    function onFullfilled(value) {
      let result
      // next() 会一步一步的执行 generator 函数体的代码，
      result = generator.next(value)
      next(result)
    }
    
    function next(result) {
      const value = result.value
      const done = result.done

      if (done) {
        return resolve(value)
      } else {
        return Promise.resolve(value).then(onFullfilled)
      }
    }
  })
}
```

这样就再运行 generatorExecuter(test) 就没问题了。

## 考虑 try catch 捕获 yield 错误

这里用到了上面提到的**原理三**，如果不清楚可以回去看看。

如下代码，使用上面的执行器是无法 try catch 到 yield 错误的:

```js
function* test() {
  try {
    const a = yield Promise.reject('error')
  }catch (err) {
    console.log('发生错误了：', err)
  }
  return true
}
generatorExecuter(test)
```

运行上面代码后，会报错 `Uncaught (in promise) error`，而不是拦截后输出 `发生错误了： error`。

要改成这样才行（使用 gen.throw ）：

```js 
/**
 * generator 执行器
 * @param {GeneratorFunction} generatorFunc generator 函数
 * @return {Promise} 返回一个 Promise 对象
 */
function generatorExecuter(generatorFunc) {
  return new Promise((resolve, reject) => {
    const generator = generatorFunc()
    // 触发第一次执行下面定义的 next 函数
    onFullfilled()
    /**
     * Promise 成功处理的回调函数
     * 回调函数会执行 generator.next()
     * @param {Any} value yield 表达式的返回值
     */
    function onFullfilled(value) {
      let result
      // next() 会一步一步的执行 generator 函数体的代码，
      // 如果报错，我们需要拦截，然后 reject
      try {
        // yield 表达式本身没有返回值，或者说总是返回 undefined。
        // generator.next 方法可以带一个参数，该参数就会被当作上一个 yield 表达式的返回值。
        // 由于 generator.next 方法的参数表示上一个 yield 表达式的返回值，
        // 所以在第一次使用 generator.next 方法时，传递参数是无效的。
        result = generator.next(value)
      } catch (error) {
        return reject(error)
      }
      next(result)
    }
    /**
     * Promise 失败处理的回调函数
     * 回调函数会执行 generator.throw() ,这样可以 try catch 拦截 yield xxx 的错误
     * @param {Any} reason 失败原因
     */
    function onRejected(reason) {
      let result
      try {
        // gen.throw() 方法用来向生成器抛出异常，并恢复生成器的执行，
        // 返回带有 done 及 value 两个属性的对象。
        // gen.throw() 在同步的环境中会直接终止代码运行，无法 try catch，了解这点特别重要
        // 不过 gen.throw() 在异步 Promise 环境中抛出才会有返回值，
        // 同时只要 generator 函数中 try catch yield 语句就可以捕获 gen.throw 出来的错误
        result = generator.throw(reason)
      } catch (error) {
        return reject(error)
      }
      // gen.throw() 的错误被捕获后，可以继续执行下去
      next(result)
    }

    function next(result) {
      const value = result.value
      const done = result.done

      if (done) {
        return resolve(value)
      } else {
        return Promise.resolve(value).then(onFullfilled, onRejected)
      }
    }
  })
}
```

## 考虑 yield 的其他类型

考虑 yield 的其他类型，如 generator 函数，**可以把这些类型适配为 Promise 就很好处理了**。

上面的代码执行如下的 Generator 函数是不正确的：

```js
function* aFun() {
  return 'a'
}

function* test() {
  const a = yield aFun
  console.log(a)
  return true
}
generatorExecuter(test)
```

运行上面的代码后，test 函数 console.log 输出的不是 `a`，而是输出下面的字符串：

```
ƒ* aFun() {
  return 'a'
}
```

那么我们代码需要这样处理：

```js
/**
 * generator 执行器
 * @param {GeneratorFunction | Generator} generatorFunc Generator 函数或者 Generator
 * @return {Promise} 返回一个 Promise 对象
 */
function generatorExecuter(generatorFunc) {
  if (!isGernerator(generatorFunc) && !isGerneratorFunction(generatorFunc)) {
    throw new TypeError(
      'Expected the generatorFunc to be a GeneratorFunction or a Generator.'
    )
  }

  let generator = generatorFunc
  if (isGerneratorFunction(generatorFunc)) {
    generator = generatorFunc()
  }

  return new Promise((resolve, reject) => {
    // 触发第一次执行下面定义的 next 函数
    onFullfilled()
    /**
     * Promise 成功处理的回调函数
     * 回调函数会执行 generator.next()
     * @param {Any} value yield 表达式的返回值
     */
    function onFullfilled(value) {
      let result
      // next() 会一步一步的执行 generator 函数体的代码，
      // 如果报错，我们需要拦截，然后 reject
      try {
        // yield 表达式本身没有返回值，或者说总是返回 undefined。
        // generator.next 方法可以带一个参数，该参数就会被当作上一个 yield 表达式的返回值。
        // 由于 generator.next 方法的参数表示上一个 yield 表达式的返回值，
        // 所以在第一次使用 generator.next 方法时，传递参数是无效的。
        result = generator.next(value)
      } catch (error) {
        return reject(error)
      }
      next(result)
    }
    /**
     * Promise 失败处理的回调函数
     * 回调函数会执行 generator.throw() ,这样可以 try catch 拦截 yield xxx 的错误
     * @param {Any} reason 失败原因
     */
    function onRejected(reason) {
      let result
      try {
        // gen.throw() 方法用来向生成器抛出异常，并恢复生成器的执行，
        // 返回带有 done 及 value 两个属性的对象。
        // gen.throw() 在同步的环境中会直接终止代码运行，无法 try catch，了解这点特别重要
        // 不过 gen.throw() 在异步 Promise 环境中抛出才会有返回值，
        // 同时只要 generator 函数中 try catch yield 语句就可以捕获 gen.throw 出来的错误
        result = generator.throw(reason)
      } catch (error) {
        return reject(error)
      }
      // gen.throw() 的错误被捕获后，可以继续执行下去
      next(result)
    }

    function next(result) {
      const value = toPromise(result.value)
      const done = result.done

      if (done) {
        return resolve(value)
      } else {
        return value.then(onFullfilled, onRejected)
      }
    }
  })
}

/**
 * 考虑 yield 的其他类型，如 generator 函数
 * 可以把这些类型适配为 Promise 就很好处理了
 */
function toPromise(value) {
  if (isGerneratorFunction(value) || isGernerator(value)) {
    // generatorExecuter 返回 Promise
    return generatorExecuter(value)
  } else {
    // 字符串、对象、数组、Promise 等都转成 Promise
    return Promise.resolve(value)
  }
}

/**
 * 是否是 generator 函数
 */
function isGerneratorFunction(target) {
  if (
    Object.prototype.toString.apply(target) === '[object GeneratorFunction]'
  ) {
    return true
  } else {
    return false
  }
}

/**
 * 是否是 generator
 */
function isGernerator(target) {
  if (Object.prototype.toString.apply(target) === '[object Generator]') {
    return true
  } else {
    return false
  }
}
```

这样就再运行 generatorExecuter(test) 就没问题了。

## 最终版

Generator 执行器没想的那么难，花点时间就可以吃透了。

```js
/**
 * generator 执行器
 * @param {GeneratorFunction | Generator} generatorFunc Generator 函数或者 Generator
 * @return {Promise} 返回一个 Promise 对象
 */
function generatorExecuter(generatorFunc) {
  if (!isGernerator(generatorFunc) && !isGerneratorFunction(generatorFunc)) {
    throw new TypeError(
      'Expected the generatorFunc to be a GeneratorFunction or a Generator.'
    )
  }

  let generator = generatorFunc
  if (isGerneratorFunction(generatorFunc)) {
    generator = generatorFunc()
  }

  return new Promise((resolve, reject) => {
    // 触发第一次执行下面定义的 next 函数
    onFullfilled()
    /**
     * Promise 成功处理的回调函数
     * 回调函数会执行 generator.next()
     * @param {Any} value yield 表达式的返回值
     */
    function onFullfilled(value) {
      let result
      // next() 会一步一步的执行 generator 函数体的代码，
      // 如果报错，我们需要拦截，然后 reject
      try {
        // yield 表达式本身没有返回值，或者说总是返回 undefined。
        // generator.next 方法可以带一个参数，该参数就会被当作上一个 yield 表达式的返回值。
        // 由于 generator.next 方法的参数表示上一个 yield 表达式的返回值，
        // 所以在第一次使用 generator.next 方法时，传递参数是无效的。
        result = generator.next(value)
      } catch (error) {
        return reject(error)
      }
      next(result)
    }
    /**
     * Promise 失败处理的回调函数
     * 回调函数会执行 generator.throw() ,这样可以 try catch 拦截 yield xxx 的错误
     * @param {Any} reason 失败原因
     */
    function onRejected(reason) {
      let result
      try {
        // gen.throw() 方法用来向生成器抛出异常，并恢复生成器的执行，
        // 返回带有 done 及 value 两个属性的对象。
        // gen.throw() 在同步的环境中会直接终止代码运行，无法 try catch，了解这点特别重要
        // 不过 gen.throw() 在异步 Promise 环境中抛出才会有返回值，
        // 同时只要 generator 函数中 try catch yield 语句就可以捕获 gen.throw 出来的错误
        result = generator.throw(reason)
      } catch (error) {
        return reject(error)
      }
      // gen.throw() 的错误被捕获后，可以继续执行下去
      next(result)
    }

    function next(result) {
      const value = toPromise(result.value)
      const done = result.done

      if (done) {
        return resolve(value)
      } else {
        return value.then(onFullfilled, onRejected)
      }
    }
  })
}

/**
 * 考虑 yield 的其他类型，如 generator 函数
 * 可以把这些类型适配为 Promise 就很好处理了
 */
function toPromise(value) {
  if (isGerneratorFunction(value) || isGernerator(value)) {
    // generatorExecuter 返回 Promise
    return generatorExecuter(value)
  } else {
    // 字符串、对象、数组、Promise 等都转成 Promise
    return Promise.resolve(value)
  }
}

/**
 * 是否是 generator 函数
 */
function isGerneratorFunction(target) {
  if (
    Object.prototype.toString.apply(target) === '[object GeneratorFunction]'
  ) {
    return true
  } else {
    return false
  }
}

/**
 * 是否是 generator
 */
function isGernerator(target) {
  if (Object.prototype.toString.apply(target) === '[object Generator]') {
    return true
  } else {
    return false
  }
}

```

运行例子如下，直接在谷歌 console 运行即可：

```js
// 例子
function* one() {
  return 'one'
}

function* two() {
  return yield 'two'
}

function* three() {
  return Promise.resolve('three')
}

function* four() {
  return yield Promise.resolve('four')
}

function* five() {
  const a = yield new Promise(resolve => {
    setTimeout(() => {
      resolve('waiting five over')
    }, 1000)
  })
  console.log(a)
  return 'five'
}

function* err() {
  const a = 2
  a = 3
  return a
}

function* all() {
  const a = yield one()
  console.log(a)
  const b = yield two()
  console.log(b)
  const c = yield three()
  console.log(c)
  const d = yield four()
  console.log(d)
  const e = yield five()
  console.log(e)
  try {
    yield err()
  } catch (err) {
    console.log('发生了错误', err)
  }
  return 'over'
}

generatorExecuter(all).then(value => {
  console.log(value)
})

// 或者
// generatorExecuter(all()).then(value => {
//   console.log(value)
// })
```