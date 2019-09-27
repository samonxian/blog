# 如何写一个让面试官满意的 Promise？

**Promise 的实现没那么简单，也没想象中的那么难，200 行代码以内就可以实现一个可替代原生的 Promise。**

Promise 已经是前端不可缺少的 API，现在已经是无处不在。你确定已经很了解 Promise 吗？如果不是很了解，那么应该了解 Promise 的实现原理。如果你觉得你自己挺了解的，那么你自己实现过 Promise 吗？

无论如何，了解 Promise 的实现方式，对于提升我们的前端技能都有一定的帮助。

**下面的代码都是使用 ES6 语法实现的，不兼容 ES5，在最新的谷歌浏览器上运行没问题。**

如果你想先直接看效果，可以看文章最后的完整版，也可以看 [github](https://github.com/xianshannan/interview)，github 上包括了单元测试。

## 部分术语

- executor

  ```js
  new Promise( function(resolve, reject) {...} /* executor */  );
  ```

  executor 是带有 `resolve` 和 `reject` 两个参数的函数 。

- onFulfilled

  ```js
  p.then(onFulfilled, onRejected);
  ```

  当 Promise 变成接受状态（fulfillment）时，该参数作为 then 回调函数被调用。

- onRejected
  当Promise变成拒绝状态（rejection ）时，该参数作为回调函数被调用。

  ```js
  p.then(onFulfilled, onRejected);
  p.catch(onRejected);
  ```

## Promise 实现原理

这里是个人总结的代码实现的基本原理。

首先我们先了解下 Promise 的实现的基本原理，然后才能更好的实现。首先你可以看看 [Promises/A+规范](https://link.jianshu.com/?t=https%3A%2F%2Fpromisesaplus.com%2F)。

### Promise 的三种状态

- pending
- fulfilled
- rejected

**pending** 状态的 Promise 对象可能会变为 **fulfilled **和 **rejected**，但是不可以逆转状态。

then、catch 的回调方法只有在非 pending 状态才能执行。

### Promise 生命周期

为了更好理解，本人总结了 Promise 的生命周期，生命周期分为两种情况，而且生命周期是不可逆的。

pending -> fulfilld

pending -> rejected

executor、then、catch、finally 的执行都是有各自新的生命周期。

### 链式原理

如何保持 `.then` 、`.catch`、`.finally` 获取链式调用呢？

其实每个链式调用的方法返回一个新的 Promise 实例（其实这也是 Promises/A+ 规范之一）就可以解决这个问题，同时保证了每个链式方式的 Promise 的初始状态为 pending 状态，每个 then、catch、finally 都有自身的 Promise 生命周期。

```js
Promise.prototype.then = function(onFulfilled,onReject) {
  return new Promise(() => {
    // 这里处理后续的逻辑即可
  })
}
```

**但是需要考虑断链的情况，断链后继续使用链式的话，Promise 的状态是非 pending 状态。**

不断链的例子如下：

```js
new Promise(...).then(...)
// 这种情况不属于断链
```

断链的例子如下：

```js
const a = new Promise(...)
a.then(...)
// 这种情况属于断链
```

所以需要考虑这两种情况。

### 异步列队

这个需要了解**宏任务和微任务**，但是不是多有浏览器 JavaScript API 都提供微任务这一类的方法。

**所以这里先使用 setTimeout 代替**。

虽然**非**异步 resolve 或者 reject 的时候，不使用异步列队方式也可以实现，不过原生的 Promise 所有 then 等回调函数都是在异步列队中执行的。

**这里注意一下，后面逐步说明的例子中，前面的一些是没使用异步列队的方式的，后面涉及到异步 resolve 或者 reject 的场景才加上去的。**

## 第一步定义好结构

这里为了跟原生的 Promise 做区别，加了个前缀，改为 NPromise。

定义状态

```js
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected
```

定义 Promise 实例方法

```js
class NPromise {
  constructor(executor) {}
  then(onFulfilled, onRejected) {}
  catch(onRejected) {}
  finally(onFinally) {}
}
```

定义拓展方法

```js
NPromise.resolve = function(value){}
NPromise.reject = function(resaon){}
NPromise.all = function(values){}
NPromise.race = function(values){}
```

## 简单的 Promise

第一个简单 Promsie 不考虑异步 resolve 的等情况，**这一步只是用法像，then 的回调也不是异步的**。

不着急，一步一步来。

```js
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

/**
 * @param {Function} executor
 * executor是带有 resolve 和 reject 两个参数的函数
 * Promise 构造函数执行时立即调用 executor 函数
 */
class NPromise {
  constructor(executor) {
    // 这里有些属性变量是可以不定义的，不过提前定义一下，提高可读性

    // 初始化状态为 pending
    this._status = PENDING
    // 传递给 then 的 onFulfilled 参数
    this._nextValue = undefined
    // 错误原因
    this._error = undefined
    executor(this._onFulfilled, this._onRejected)
  }

  /**
   * 操作成功
   * 每次运行这个函数的时候需要重置 this._status = PENDING
   * @param {Any} value 操作成功传递的值
   */
  _onFulfilled = (value) => {
    if (this._status === PENDING) {
      this._status = FULFILLED
      this._nextValue = value
      this._error = undefined
    }
  }

  /**
   * 操作失败
   * 每次运行这个函数的时候需要重置 this._status = PENDING
   * @param {Any} reason 操作失败传递的值
   */
  _onRejected = (reason) => {
    if (this._status === PENDING) {
      this._status = REJECTED
      this._error = reason
      this._nextValue = undefined
    }
  }

  then(onFulfilled, onRejected) {
    return new NPromise((resolve, reject) => {
      if (this._status === FULFILLED) {
        if (onFulfilled) {
          const _value = onFulfilled
            ? onFulfilled(this._nextValue)
            : this._nextValue
          // 如果 onFulfilled 有定义则运行 onFulfilled 返回结果
          // 否则跳过，这里下一个 Promise 都是返回 fulfilled 状态
          resolve(_value)
        }
      }

      if (this._status === REJECTED) {
        if (onRejected) {
          resolve(onRejected(this._error))
        } else {
          // 没有直接跳过，下一个 Promise 继续返回 rejected 状态
          reject(this._error)
        }
      }
    })
  }

  catch(onRejected) {
    // catch 其实就是 then 的无 fulfilled 处理
    return this.then(null, onRejected)
  }

  finally(onFinally) {
    // 这个后面实现
  }
}
```

测试例子（setTimeout 只是为了区分执行先后）

```js
setTimeout(() => {
  new NPromise((resolve) => {
    console.log('resolved:')
    resolve(1)
  })
    .then((value) => {
      console.log(value)
      return 2
    })
    .then((value) => {
      console.log(value)
    })
})

setTimeout(() => {
  new NPromise((_, reject) => {
    console.log('rejected:')
    reject('err')
  })
    .then((value) => {
      // 这里不会运行
      console.log(value)
      return 2
    })
    .catch((err) => {
      console.log(err)
    })
})
// 输出
// resolved:
// 1
// 2
// rejected:
// err
```

## 考虑捕获错误

还是先不考虑异步 resolve 的等情况，**这一步也只是用法像，then 的回调也不是异步的**。

**相对于上一步的改动点：**

- executor 的运行需要 try catch

  then、catch、finally 都是要经过 executor 执行的，所以只需要 try catch executor  即可。

- 没有定义 promise.catch 则直接打印错误信息

  不是 throw，而是 console.error，原生的也是这样的，所以直接 try catch 不到 Promise 的错误。

- then 的回调函数不是函数类型则跳过

  这不属于错误捕获处理范围，这里顺带提一下。

**代码实现**

```js
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

function isFunction(fn) {
  return typeof fn === 'function'
}

/**
 * @param {Function} executor
 * executor是带有 resolve 和 reject 两个参数的函数
 * Promise 构造函数执行时立即调用 executor 函数
 */
class NPromise {
  constructor(executor) {
    // 这里有些属性变量是可以不定义的，不过提前定义一下，提高可读性

    try {
      // 初始化状态为 pending
      this._status = PENDING
      // 传递给 then 的 onFulfilled 参数
      this._nextValue = undefined
      // 错误原因
      this._error = undefined
      executor(this._onFulfilled, this._onRejected)
    } catch (err) {
      this._onRejected(err)
    }
  }

  /**
   * 如果没有 .catch 错误，则在最后抛出错误
   */
  _throwErrorIfNotCatch() {
    setTimeout(() => {
      // setTimeout 是必须的，等待执行完毕，最后检测 this._error 是否还定义
      if (this._error !== undefined) {
        // 发生错误后没用 catch 那么需要直接提示
        console.error('Uncaught (in promise)', this._error)
      }
    })
  }

  /**
   * 操作成功
   * 每次运行这个函数的时候需要重置 this._status = PENDING
   * @param {Any} value 操作成功传递的值
   */
  _onFulfilled = (value) => {
    if (this._status === PENDING) {
      this._status = FULFILLED
      this._nextValue = value
      this._error = undefined
      this._throwErrorIfNotCatch()
    }
  }

  /**
   * 操作失败
   * 每次运行这个函数的时候需要重置 this._status = PENDING
   * @param {Any} reason 操作失败传递的值
   */
  _onRejected = (reason) => {
    if (this._status === PENDING) {
      this._status = REJECTED
      this._error = reason
      this._nextValue = undefined
      this._throwErrorIfNotCatch()
    }
  }

  then(onFulfilled, onRejected) {
    return new NPromise((resolve, reject) => {
      const handle = (reason) => {
        function handleResolve(value) {
          const _value = isFunction(onFulfilled) ? onFulfilled(value) : value
          resolve(_value)
        }

        function handleReject(err) {
          if (isFunction(onRejected)) {
            resolve(onRejected(err))
          } else {
            reject(err)
          }
        }

        if (this._status === FULFILLED) {
          return handleResolve(this._nextValue)
        }

        if (this._status === REJECTED) {
          return handleReject(reason)
        }
      }
      handle(this._error)
      // error 已经传递到下一个 NPromise 了，需要重置，否则会打印多个相同错误
      // 配合 this._throwErrorIfNotCatch 一起使用，
      // 保证执行到最后才抛出错误，如果没有 catch
      this._error = undefined
    })
  }

  catch(onRejected) {
    // catch 其实就是 then 的无 fulfilled 处理
    return this.then(null, onRejected)
  }

  finally(onFinally) {
    // 这个后面实现
  }
}
```

测试例子（setTimeout 只是为了区分执行先后）

```js
setTimeout(() => {
  new NPromise((resolve) => {
    console.log('executor 报错:')
    const a = 2
    a = 3
    resolve(1)
  }).catch((value) => {
    console.log(value)
    return 2
  })
})

setTimeout(() => {
  new NPromise((resolve) => {
    resolve()
  })
    .then(() => {
      const b = 3
      b = 4
      return 2
    })
    .catch((err) => {
      console.log('then 回调函数报错：')
      console.log(err)
    })
})

setTimeout(() => {
  new NPromise((resolve) => {
    console.log('直接打印了错误信息，红色的：')
    resolve()
  }).then(() => {
    throw Error('test')
    return 2
  })
})


// executor 报错:
//  TypeError: Assignment to constant variable.
//     at <anonymous>:97:7
//     at new NPromise (<anonymous>:21:7)
//     at <anonymous>:94:3
//  then 回调函数报错： TypeError: Assignment to constant variable.
//     at <anonymous>:111:9
//     at <anonymous>:59:17
//     at new Promise (<anonymous>)
//     at NPromise.then (<anonymous>:54:12)
//     at <anonymous>:109:6
// 直接打印了错误信息，红色的：
// Uncaught (in promise) Error: test
//     at <anonymous>:148:11
//     at handleResolve (<anonymous>:76:52)
//     at handle (<anonymous>:89:18)
//     at <anonymous>:96:7
//     at new NPromise (<anonymous>:25:7)
//     at NPromise.then (<anonymous>:73:12)
//     at <anonymous>:147:6
```

## 考虑 resolved 的值是 Promise 类型

还是先不考虑异步 resolve 的等情况，**这一步也只是用法像，then 的回调也不是异步的**。

**相对于上一步的改动点：**

- 新增 isPromise 判断方法
- then 方法中处理 this._nextValue 是 Promise 的情况

**代码实现**

```js
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

function isFunction(fn) {
  return typeof fn === 'function'
}

function isPromise(value) {
  return (
    value &&
    isFunction(value.then) &&
    isFunction(value.catch) &&
    isFunction(value.finally)
    // 上面判断也可以用一句代码代替，不过原生的试了下，不是用下面的方式
    // value.then instanceof NPromise ||
  )
}

/**
 * @param {Function} executor
 * executor是带有 resolve 和 reject 两个参数的函数
 * Promise 构造函数执行时立即调用 executor 函数
 */
class NPromise {
  constructor(executor) {
    // 这里有些属性变量是可以不定义的，不过提前定义一下，提高可读性

    try {
      // 初始化状态为 pending
      this._status = PENDING
      // 传递给 then 的 onFulfilled 参数
      this._nextValue = undefined
      // 错误原因
      this._error = undefined
      executor(this._onFulfilled, this._onRejected)
    } catch (err) {
      this._onRejected(err)
    }
  }

  /**
   * 如果没有 .catch 错误，则在最后抛出错误
   */
  _throwErrorIfNotCatch() {
    setTimeout(() => {
      // setTimeout 是必须的，等待执行完毕，最后检测 this._error 是否还定义
      if (this._error !== undefined) {
        // 发生错误后没用 catch 那么需要直接提示
        console.error('Uncaught (in promise)', this._error)
      }
    })
  }

  /**
   * 操作成功
   * 每次运行这个函数的时候需要重置 this._status = PENDING
   * @param {Any} value 操作成功传递的值
   */
  _onFulfilled = (value) => {
    if (this._status === PENDING) {
      this._status = FULFILLED
      this._nextValue = value
      this._error = undefined
      this._throwErrorIfNotCatch()
    }
  }

  /**
   * 操作失败
   * 每次运行这个函数的时候需要重置 this._status = PENDING
   * @param {Any} reason 操作失败传递的值
   */
  _onRejected = (reason) => {
    if (this._status === PENDING) {
      this._status = REJECTED
      this._error = reason
      this._nextValue = undefined
      this._throwErrorIfNotCatch()
    }
  }

  then(onFulfilled, onRejected) {
    return new NPromise((resolve, reject) => {
      const handle = (reason) => {
        function handleResolve(value) {
          const _value = isFunction(onFulfilled) ? onFulfilled(value) : value
          resolve(_value)
        }

        function handleReject(err) {
          if (isFunction(onRejected)) {
            resolve(onRejected(err))
          } else {
            reject(err)
          }
        }

        if (this._status === FULFILLED) {
          if (isPromise(this._nextValue)) {
            return this._nextValue.then(handleResolve, handleReject)
          } else {
            return handleResolve(this._nextValue)
          }
        }

        if (this._status === REJECTED) {
          return handleReject(reason)
        }
      }
      handle(this._error)
      // error 已经传递到下一个 NPromise 了，需要重置，否则会打印多个相同错误
      // 配合 this._throwErrorIfNotCatch 一起使用，
      // 保证执行到最后才抛出错误，如果没有 catch
      this._error = undefined
    })
  }

  catch(onRejected) {
    // catch 其实就是 then 的无 fulfilled 处理
    return this.then(null, onRejected)
  }

  finally(onFinally) {
    // 这个后面实现
  }
}
```

测试代码

```js

setTimeout(() => {
  new NPromise((resolve) => {
    resolve(
      new NPromise((_resolve) => {
        _resolve(1)
      })
    )
  }).then((value) => {
    console.log(value)
  })
})

setTimeout(() => {
  new NPromise((resolve) => {
    resolve(
      new NPromise((_, _reject) => {
        _reject('err')
      })
    )
  }).catch((err) => {
    console.log(err)
  })
})
```

## 考虑异步的情况

异步 resolve 或者 reject，相对会复杂点，需要放进一个列队中，然后等待 pending 状态变为其他状态后才执行。

原生 JavaScript 自带异步列队，我们可以利用这一点，这里先使用 setTimeout 代替，所以这个 Promise 的优先级跟 setTimeout 是同一个等级的（原生的是 Promise 优先于 setTimeout 执行）。

**相对于上一步的改动点：**

- this._onFulfilled 加上了异步处理
- this._onRejected 加上了异步处理

- 新增 this._callbackQueue，初始值为空数组

- 新增 this.__runCallbackQueue 方法运行异步列队

  **同步**的时候 Promise 的状态会**立即**切换为非 pending 状态，异步列队为空。异步的改变 Promise 状态的之前，一值是 pending 状态，那么 then、catch、finally 的回调函数都会进入列队中等待执行。

  **无论是异步还是同步，this.__runCallbackQueue 都只会在当前 Promise 生命周期中执行一次。**

- then 方法中需要根据 Promise 状态进行区分处理

  如果非 pending 状态，那么立即执行回调函数（如果没回调函数，跳过）。

  如果是 pending 状态 ，那么加入异步列队，等待 Promise 状态为非 pending 状态才依次执行。

- 实现了 finally 方法

**代码实现**

```js
// Promises/A+ 规范 https://promisesaplus.com/
// 一个 Promise有以下几种状态:
// pending: 初始状态，既不是成功，也不是失败状态。
// fulfilled: 意味着操作成功完成。
// rejected: 意味着操作失败。

// pending 状态的 Promise 对象可能会变为 fulfilled
// 状态并传递一个值给相应的状态处理方法，也可能变为失败状态（rejected）并传递失败信息。
// 当其中任一种情况出现时，Promise 对象的 then 方法绑定的处理方法（handlers）就会被调用
// then方法包含两个参数：onfulfilled 和 onrejected，它们都是 Function 类型。
// 当 Promise 状态为 fulfilled 时，调用 then 的 onfulfilled 方法，
// 当 Promise 状态为 rejected 时，调用 then 的 onrejected 方法，
// 所以在异步操作的完成和绑定处理方法之间不存在竞争。

const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

function isFunction(fn) {
  return typeof fn === 'function'
}

function isPromise(value) {
  return (
    value &&
    isFunction(value.then) &&
    isFunction(value.catch) &&
    isFunction(value.finally)
    // 上面判断也可以用一句代码代替，不过原生的试了下，不是用下面的方式
    // value.then instanceof NPromise ||
  )
}

/**
 * @param {Function} executor
 * executor是带有 resolve 和 reject 两个参数的函数
 * Promise 构造函数执行时立即调用 executor 函数
 */
class NPromise {
  constructor(executor) {
    if (!isFunction(executor)) {
      throw new TypeError('Expected the executor to be a function.')
    }
    try {
      // 初始化状态为 PENDING
      this._status = PENDING
      // fullfilled 的值，也是 then 和 catch 的 return 值，都是当前执行的临时值
      this._nextValue = undefined
      // 当前捕捉的错误信息
      this._error = undefined
      // then、catch、finally 的异步回调列队，会依次执行
      this._callbacQueue = []
      executor(this._onFulfilled, this._onRejected)
    } catch (err) {
      this._onRejected(err)
    }
  }

  /**
   * 如果没有 .catch 错误，则在最后抛出错误
   */
  _throwErrorIfNotCatch() {
    setTimeout(() => {
      // setTimeout 是必须的，等待执行完毕，最后检测 this._error 是否还定义
      if (this._error !== undefined) {
        // 发生错误后没用 catch 那么需要直接提示
        console.error('Uncaught (in promise)', this._error)
      }
    })
  }

  _runCallbackQueue = () => {
    if (this._callbacQueue.length > 0) {
      // resolve 或者 reject 异步运行的时候，this._callbacQueue 的 length 才会大于 0
      this._callbacQueue.forEach(fn => {
        fn()
      })
      this._callbacQueue = []
    }
    this._throwErrorIfNotCatch()
  }

  /**
   * 操作成功
   * 每次运行这个函数的时候需要重置 this._status = PENDING
   * @param {Any} value 操作成功传递的值
   */
  _onFulfilled = value => {
    setTimeout(() => {
      if (this._status === PENDING) {
        this._status = FULFILLED
        this._nextValue = value
        this._error = undefined
        this._runCallbackQueue()
      }
    })
  }

  /**
   * 操作失败
   * 每次运行这个函数的时候需要重置 this._status = PENDING
   * @param {Any} reason 操作失败传递的值
   */
  _onRejected = reason => {
    setTimeout(() => {
      if (this._status === PENDING) {
        this._status = REJECTED
        this._error = reason
        this._nextValue = undefined
        this._runCallbackQueue()
      }
    })
  }

  then(onFulfilled, onRejected) {
    return new NPromise((resolve, reject) => {
      const handle = reason => {
        try {
          function handleResolve(value) {
            const _value = isFunction(onFulfilled) ? onFulfilled(value) : value
            resolve(_value)
          }

          function handleReject(err) {
            if (isFunction(onRejected)) {
              resolve(onRejected(err))
            } else {
              reject(err)
            }
          }

          if (this._status === FULFILLED) {
            if (isPromise(this._nextValue)) {
              return this._nextValue.then(handleResolve, handleReject)
            } else {
              return handleResolve(this._nextValue)
            }
          }

          if (this._status === REJECTED) {
            return handleReject(reason)
          }
        } catch (err) {
          reject(err)
        }
      }
      if (this._status === PENDING) {
        // 默认不断链的情况下，then 回电函数 Promise 的状态为 pending 状态
        // 如 new NPromise(...).then(...) 是没断链的
        // 但是 NPromise.resolve(...).then(...) 是断链的了，相当于
        // var a = NPromise(...); a.then(...)
        this._callbacQueue.push(() => {
          // 先使用 setTimeout 代替
          // 保证状态切换为非 PENDING 状态才会执行后续的 then、catch 或者 finally 的回调函数
          handle(this._error)
          // error 已经传递到下一个 NPromise 了，需要重置，否则会抛出多个相同错误
          // 配合 this._throwErrorIfNotCatch 一起使用，
          // 保证执行到最后才抛出错误，如果没有 catch
          this._error = undefined
        })
      } else {
        // 断链的情况下，then 回调函数 Promise 的状态为非 pending 状态
        // 如 var a = NPromise(...); a.then(...) 就是断链的场景
        handle(this._error)
        // error 已经传递到下一个 NPromise 了，需要重置，否则会打印多个相同错误
        // 配合 this._throwErrorIfNotCatch 一起使用，
        // 保证执行到最后才抛出错误，如果没有 catch
        this._error = undefined
      }
    })
  }

  catch(onRejected) {
    return this.then(null, onRejected)
  }

  finally(onFinally) {
    return this.then(
      () => {
        onFinally()
        return this._nextValue
      },
      () => {
        onFinally()
        // 错误需要抛出，下一个 Promise 才会捕获到
        throw this._error
      }
    )
  }
}
```

测试代码

```js
new NPromise((resolve) => {
  setTimeout(() => {
    resolve(1)
  }, 1000)
})
  .then((value) => {
    console.log(value)
    return new NPromise((resolve) => {
      setTimeout(() => {
        resolve(2)
      }, 1000)
    })
  })
  .then((value) => {
    console.log(value)
  })
```

## 拓展方法

拓展方法相对难的应该是 Promise.all，其他的都挺简单的。

```js
NPromise.resolve = function(value) {
  return new NPromise(resolve => {
    resolve(value)
  })
}

NPromise.reject = function(reason) {
  return new NPromise((_, reject) => {
    reject(reason)
  })
}

NPromise.all = function(values) {
  return new NPromise((resolve, reject) => {
    let ret = {}
    let isError = false
    values.forEach((p, index) => {
      if (isError) {
        return
      }
      NPromise.resolve(p)
        .then(value => {
          ret[index] = value
          const result = Object.values(ret)
          if (values.length === result.length) {
            resolve(result)
          }
        })
        .catch(err => {
          isError = true
          reject(err)
        })
    })
  })
}

NPromise.race = function(values) {
  return new NPromise(function(resolve, reject) {
    values.forEach(function(value) {
      NPromise.resolve(value).then(resolve, reject)
    })
  })
}
```

## 最终版

也可以看 [github](https://github.com/xianshannan/interview)，上面有单元测试。

```js
// Promises/A+ 规范 https://promisesaplus.com/
// 一个 Promise有以下几种状态:
// pending: 初始状态，既不是成功，也不是失败状态。
// fulfilled: 意味着操作成功完成。
// rejected: 意味着操作失败。

// pending 状态的 Promise 对象可能会变为 fulfilled
// 状态并传递一个值给相应的状态处理方法，也可能变为失败状态（rejected）并传递失败信息。
// 当其中任一种情况出现时，Promise 对象的 then 方法绑定的处理方法（handlers）就会被调用
// then方法包含两个参数：onfulfilled 和 onrejected，它们都是 Function 类型。
// 当 Promise 状态为 fulfilled 时，调用 then 的 onfulfilled 方法，
// 当 Promise 状态为 rejected 时，调用 then 的 onrejected 方法，
// 所以在异步操作的完成和绑定处理方法之间不存在竞争。

const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

function isFunction(fn) {
  return typeof fn === 'function'
}

function isPromise(value) {
  return (
    value &&
    isFunction(value.then) &&
    isFunction(value.catch) &&
    isFunction(value.finally)
    // 上面判断也可以用一句代码代替，不过原生的试了下，不是用下面的方式
    // value.then instanceof NPromise ||
  )
}

/**
 * @param {Function} executor
 * executor是带有 resolve 和 reject 两个参数的函数
 * Promise 构造函数执行时立即调用 executor 函数
 */
class NPromise {
  constructor(executor) {
    if (!isFunction(executor)) {
      throw new TypeError('Expected the executor to be a function.')
    }
    try {
      // 初始化状态为 PENDING
      this._status = PENDING
      // fullfilled 的值，也是 then 和 catch 的 return 值，都是当前执行的临时值
      this._nextValue = undefined
      // 当前捕捉的错误信息
      this._error = undefined
      // then、catch、finally 的异步回调列队，会依次执行
      this._callbacQueue = []
      executor(this._onFulfilled, this._onRejected)
    } catch (err) {
      this._onRejected(err)
    }
  }

  /**
   * 如果没有 .catch 错误，则在最后抛出错误
   */
  _throwErrorIfNotCatch() {
    setTimeout(() => {
      // setTimeout 是必须的，等待执行完毕，最后检测 this._error 是否还定义
      if (this._error !== undefined) {
        // 发生错误后没用 catch 那么需要直接提示
        console.error('Uncaught (in promise)', this._error)
      }
    })
  }

  _runCallbackQueue = () => {
    if (this._callbacQueue.length > 0) {
      // resolve 或者 reject 异步运行的时候，this._callbacQueue 的 length 才会大于 0
      this._callbacQueue.forEach(fn => {
        fn()
      })
      this._callbacQueue = []
    }
    this._throwErrorIfNotCatch()
  }

  /**
   * 操作成功
   * 每次运行这个函数的时候需要重置 this._status = PENDING
   * @param {Any} value 操作成功传递的值
   */
  _onFulfilled = value => {
    setTimeout(() => {
      if (this._status === PENDING) {
        this._status = FULFILLED
        this._nextValue = value
        this._error = undefined
        this._runCallbackQueue()
      }
    })
  }

  /**
   * 操作失败
   * 每次运行这个函数的时候需要重置 this._status = PENDING
   * @param {Any} reason 操作失败传递的值
   */
  _onRejected = reason => {
    setTimeout(() => {
      if (this._status === PENDING) {
        this._status = REJECTED
        this._error = reason
        this._nextValue = undefined
        this._runCallbackQueue()
      }
    })
  }

  then(onFulfilled, onRejected) {
    return new NPromise((resolve, reject) => {
      const handle = reason => {
        try {
          function handleResolve(value) {
            const _value = isFunction(onFulfilled) ? onFulfilled(value) : value
            resolve(_value)
          }

          function handleReject(err) {
            if (isFunction(onRejected)) {
              resolve(onRejected(err))
            } else {
              reject(err)
            }
          }

          if (this._status === FULFILLED) {
            if (isPromise(this._nextValue)) {
              return this._nextValue.then(handleResolve, handleReject)
            } else {
              return handleResolve(this._nextValue)
            }
          }

          if (this._status === REJECTED) {
            return handleReject(reason)
          }
        } catch (err) {
          reject(err)
        }
      }
      if (this._status === PENDING) {
        // 默认不断链的情况下，then 回电函数 Promise 的状态为 pending 状态
        // 如 new NPromise(...).then(...) 是没断链的
        // 但是 NPromise.resolve(...).then(...) 是断链的了，相当于
        // var a = NPromise(...); a.then(...)
        this._callbacQueue.push(() => {
          // 先使用 setTimeout 代替
          // 保证状态切换为非 PENDING 状态才会执行后续的 then、catch 或者 finally 的回调函数
          handle(this._error)
          // error 已经传递到下一个 NPromise 了，需要重置，否则会抛出多个相同错误
          // 配合 this._throwErrorIfNotCatch 一起使用，
          // 保证执行到最后才抛出错误，如果没有 catch
          this._error = undefined
        })
      } else {
        // 断链的情况下，then 回调函数 Promise 的状态为非 pending 状态
        // 如 var a = NPromise(...); a.then(...) 就是断链的场景
        handle(this._error)
        // error 已经传递到下一个 NPromise 了，需要重置，否则会打印多个相同错误
        // 配合 this._throwErrorIfNotCatch 一起使用，
        // 保证执行到最后才抛出错误，如果没有 catch
        this._error = undefined
      }
    })
  }

  catch(onRejected) {
    return this.then(null, onRejected)
  }

  finally(onFinally) {
    return this.then(
      () => {
        onFinally()
        return this._nextValue
      },
      () => {
        onFinally()
        // 错误需要抛出，下一个 Promise 才会捕获到
        throw this._error
      }
    )
  }
}

NPromise.resolve = function(value) {
  return new NPromise(resolve => {
    resolve(value)
  })
}

NPromise.reject = function(reason) {
  return new NPromise((_, reject) => {
    reject(reason)
  })
}

NPromise.all = function(values) {
  return new NPromise((resolve, reject) => {
    let ret = {}
    let isError = false
    values.forEach((p, index) => {
      if (isError) {
        return
      }
      NPromise.resolve(p)
        .then(value => {
          ret[index] = value
          const result = Object.values(ret)
          if (values.length === result.length) {
            resolve(result)
          }
        })
        .catch(err => {
          isError = true
          reject(err)
        })
    })
  })
}

NPromise.race = function(values) {
  return new NPromise(function(resolve, reject) {
    values.forEach(function(value) {
      NPromise.resolve(value).then(resolve, reject)
    })
  })
}
```

## 总结

经过一些测试，除了下面两点之外：

- 使用了 setTimeout 的宏任务列队外替代微任务
- 拓展方法 Promise.all 和 Promise.race 只考虑数组，不考虑迭代器。

Promise 单独用法和效果上基本 100% 跟原生的一致。

如果你不相信，你试试下面的代码，你会发现是有那么道理的：

注意 **N**Promise 和 **Promise** 的。

```js
new NPromise((resolve) => {
  resolve(Promise.resolve(2))
}).then((value) => {
  console.log(value)
})
```

或者

```js
new Promise((resolve) => {
  resolve(NPromise.resolve(2))
}).then((value) => {
  console.log(value)
})
```

上面的结果都是返回正常的。