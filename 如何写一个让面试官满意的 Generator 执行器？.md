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

`gen.throw(exception)` 可以抛出异常，而且该异常通常可以通过 `try...catch` 块进行捕获。那么我们可以抛出 Promise 错误，这样就可以使用 `try catch` 拦截 Promise 的错误了。

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
      return Promise.resolve(value).then(finalValue => {
        return recursion(finalValue)
      })
    }
  }
  return recursion()
}
```

这样就再运行 generatorExecuter(test) 就没问题了。

## 考虑 yield 的类型是 Generator 函数

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
 * 运行 generator
 * @param {GeneratorFunction} generator generator函数运行
 */
function generatorExecuter(generatorFunc) {
  const generator = generatorFunc()
  function recursionGeneratorNext(prevValue) {
    const next = generator.next(prevValue)
    let value = next.value
    const done = next.done

    if (isGerneratorFunction(value)) {
      value = generatorExecuter(value)
    }

    if (done) {
      return Promise.resolve(value)
    } else {
      // value 可能是 Promise 对象
      return Promise.resolve(value).then(finalValue => {
        return recursionGeneratorNext(finalValue)
      })
    }
  }
  return recursionGeneratorNext()
}
```

这样就再运行 generatorExecuter(test) 就没问题了。

## 考虑 yield 的类型是 Generator

上面的代码执行如下的 Generator 函数是不正确的：

```js
function* aFun(value) {
  return value
}

function* test() {
  const a = yield aFun('a')
  console.log(a)
  return true
}
generatorExecuter(test)
```

运行上面的代码后，test 函数 console.log 输出的不是 `a`，而是 `aFun {<suspended>}`。

那么我们代码需要这样处理：

```js
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
 * 运行 generator
 * @param {Generator} generator generator函数运行后返回的对象
 */
function runGernerator(generator) {
  function recursionGeneratorNext(prevValue) {
    const next = generator.next(prevValue)
    let value = next.value
    const done = next.done

    if (isGernerator(value)) {
      value = runGernerator(value)
    } else if (isGerneratorFunction(value)) {
      value = generatorExecuter(value)
    }

    if (done) {
      return Promise.resolve(value)
    } else {
      // value 可能是 Promise 对象
      return Promise.resolve(value).then(finalValue => {
        return recursionGeneratorNext(finalValue)
      })
    }
  }
  return recursionGeneratorNext()
}

/**
 * generator 执行器
 * @param {Generator || GeneratorFunction} generatorFunc generator 函数或者 generator
 * @return {Promise} 返回一个 Promise 对象
 */
function generatorExecuter(generatorFunc) {
  let generator
  if (isGerneratorFunction(generatorFunc)) {
    generator = generatorFunc()
  } else if (isGernerator(generatorFunc)) {
    generator = generatorFunc
  } else {
    throw new TypeError(
      'Expected the generatorFunc to be a GeneratorFuntion or Generator.'
    )
  }
  return runGernerator(generator)
}
```
这样就再运行 generatorExecuter(test) 就没问题了。

## 考虑 try catch 捕获 Promise 错误

如下代码，使用上面的执行器是无法 try catch 到 Promise 错误的:

```js
function* aFun() {
  return Promise.reject('error')
}

function* test() {
  try {
    const a = yield aFun()
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
 * 是否是 generator
 */
function isGernerator(target) {
  if (Object.prototype.toString.apply(target) === '[object Generator]') {
    return true
  } else {
    return false
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
 * 运行 generator
 * @param {Generator} generator generator函数运行后返回的对象
 */
function runGernerator(generator) {
  function recursionGeneratorNext(prevValue) {
    const next = generator.next(prevValue)
    let value = next.value
    const done = next.done

    if (isGernerator(value)) {
      value = runGernerator(value)
    } else if (isGerneratorFunction(value)) {
      value = generatorExecuter(value)
    }

    if (done) {
      return Promise.resolve(value).catch(err => {
        generator.throw(err)
      })
    } else {
      // value 可能是 Promise 对象
      return Promise.resolve(value).then(finalValue => {
        return recursionGeneratorNext(finalValue)
      }).catch(err => {
        generator.throw(err)
      })
    }
  }
  return recursionGeneratorNext()
}

/**
 * generator 执行器
 * @param {Generator || GeneratorFunction} generatorFunc generator 函数或者 generator
 * @return {Promise} 返回一个 Promise 对象
 */
function generatorExecuter(generatorFunc) {
  let generator
  if (isGerneratorFunction(generatorFunc)) {
    generator = generatorFunc()
  } else if (isGernerator(generatorFunc)) {
    generator = generatorFunc
  } else {
    throw new TypeError(
      'Expected the generatorFunc to be a GeneratorFuntion or Generator.'
    )
  }
  return runGernerator(generator)
}
```

## 考虑 try catch 捕获 Generator 函数体错误

如下代码，使用上面的执行器是无法 try catch 到 Promise 错误的:

```js
function* aFun() {
  const a = 2
  a = 3
  return a
}

function* test() {
  try {
    const a = yield aFun()
  } catch (err) {
    console.log('发生错误了：', err)
  }
  return true
}
generatorExecuter(test)
```

运行上面代码后，会直接报错，无法直接使用 try catch 拦截。

要改成这样才行（try catch next 函数的运行，然后使用 gen.throw ）：

```js
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
 * 运行 generator
 * @param {Generator} generator generator函数运行后返回的对象
 */
function runGernerator(generator) {
  function recursionGeneratorNext(prevValue) {
    // 在这里做 try catch
    let next
    try {
    	next = generator.next(prevValue)
    } catch(err) {
      // 不能直接 gen.throw，必须使用 promise reject 后处理
			return Promise.reject(err).catch(err => {
        generator.throw(err)
      })
    }
    
    let value = next.value
    const done = next.done

    if (isGernerator(value)) {
      value = runGernerator(value)
    } else if (isGerneratorFunction(value)) {
      value = generatorExecuter(value)
    }

    if (done) {
      return Promise.resolve(value).catch(err => {
        generator.throw(err)
      })
    } else {
      // value 可能是 Promise 对象
      return Promise.resolve(value).then(finalValue => {
        return recursionGeneratorNext(finalValue)
      }).catch(err => {
        generator.throw(err)
      })
    }
  }
  return recursionGeneratorNext()
}

/**
 * generator 执行器
 * @param {Generator || GeneratorFunction} generatorFunc generator 函数或者 generator
 * @return {Promise} 返回一个 Promise 对象
 */
function generatorExecuter(generatorFunc) {
  let generator
  if (isGerneratorFunction(generatorFunc)) {
    generator = generatorFunc()
  } else if (isGernerator(generatorFunc)) {
    generator = generatorFunc
  } else {
    throw new TypeError(
      'Expected the generatorFunc to be a GeneratorFuntion or Generator.'
    )
  }
  return runGernerator(generator)
}
```

## 最终版

Generator 执行器没想的那么难，花点时间就可以吃透了。

```js
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
 * 运行 generator
 * @param {Generator} generator generator函数运行后返回的对象
 */
function runGernerator(generator) {
  function recursionGeneratorNext(prevValue) {
    // 在这里做 try catch
    let next
    let value
    
    try {
    	next = generator.next(prevValue)
      value = next.value
    } catch(err) {
      // 不能直接 gen.throw，必须使用 promise reject 后处理
			return Promise.reject(err).catch(err => {
        generator.throw(err)
      }).then(a=>{
        console.log(a,'ddddd')
      })
    }
    
    const done = next.done

    if (isGernerator(value)) {
      value = runGernerator(value)
    } else if (isGerneratorFunction(value)) {
      value = generatorExecuter(value)
    }

    if (done) {
      return Promise.resolve(value).catch(err => {
        generator.throw(err)
      })
    } else {
      // value 可能是 Promise 对象
      return Promise.resolve(value).then(finalValue => {
        return recursionGeneratorNext(finalValue)
      }).catch(err => {
        generator.throw(err)
      })
    }
  }
  return recursionGeneratorNext()
}

/**
 * generator 执行器
 * @param {Generator || GeneratorFunction} generatorFunc generator 函数或者 generator
 * @return {Promise} 返回一个 Promise 对象
 */
function generatorExecuter(generatorFunc) {
  let generator
  if (isGerneratorFunction(generatorFunc)) {
    generator = generatorFunc()
  } else if (isGernerator(generatorFunc)) {
    generator = generatorFunc
  } else {
    throw new TypeError(
      'Expected the generatorFunc to be a GeneratorFuntion or Generator.'
    )
  }
  return runGernerator(generator)
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