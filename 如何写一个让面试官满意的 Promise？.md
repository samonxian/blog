# 如何写一个让面试官满意的 Promise？

**Promise 的实现没那么简单，也没想象中的那么难，200 行代码就可以实现一个可替代原生的 Promise。**

Promise 已经是前端不可缺少的 API 方法，现在已经是无处不在。你确定已经很了解 Promise 吗？如果不是很了解，那么应该了解 Promise 的实现原理。如果你觉得你自己挺了解的，那么你自己实现过 Promise 吗？

无论如何，了解 Promise 的实现方式，对于提升我们的前端技能都有一定的帮助。

**下面的代码都是使用 ES6 语法实现的，不兼容 ES5，在最新的谷歌浏览器上运行没问题。**

如果你想先直接看效果，可以看文章最后的完整版，也可以看 [github](https://github.com/xianshannan/interview)。

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

首先我们先了解下 Promise 的基本原理了，然后才能更好的实现。首先你可以看看 [Promises/A+规范](https://link.jianshu.com/?t=https%3A%2F%2Fpromisesaplus.com%2F)。

然后这里总结了下代码实现的基本原理。

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

其实每个链式调用的方法返回一个新的 Promise 实例就可以解决这个问题，同时保证了每个链式方式的 Promise 的初始状态为 pending 状态，每个 then、catch、finally 都有自身的 Promise 生命周期。

```js
Promise.prototype.then = function(onFulfilled,onReject) {
  return new Promise(() => {
    // 这里处理后续的逻辑即可
  })
}
```

### 异步列队

这个需要了解**宏任务和微任务**，但是不是多有浏览器 JavaScript API 都提供微任务 这一类的方法。

**所以这里先使用 setTimeout 代替**。

Promise fulfilled 和 rejected 状态，可以在异步环境返回，那么 then、catch、finally 的回调就不能同步执行，需要等待 pending 状态转变为其他两个状态，才能继续执行。

**所以 Promise 需要考虑同步和异步两种情况**，只有 resolve 和 reject 方法在异步环境执行的时候才会有自定义的异步执行列队。

同步的链式挺好理解，那么主要的在于自定义异步列队的执行。异步列队其实也没那么难，其实就是数组的按顺序执行，但是里面的各种状态会复杂点。

## 第一步定义好结构

这里为了跟原生的 Promise 做区别，加了个前缀。

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

第一个简单 Promsie 不考虑异步 resolve 的等情况。

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

还是先不考虑异步 resolve 的等情况。

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
      this._callbacQueue.forEach((fn) => {
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
  _onFulfilled = (value) => {
    if (this._status === PENDING) {
      this._status = FULFILLED
      this._nextValue = value
      this._error = undefined
      this._runCallbackQueue()
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
      this._runCallbackQueue()
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
      if (this._status === PENDING) {
        // NPromise 的 resolve 或者 reject 在异步中执行才会执行到这一步
        this._callbacQueue.push(() => {
          setTimeout(() => {
            // 先使用 setTimeout 代替
            // 保证状态切换为非 PENDING 状态才会执行后续的 then、catch 或者 finally 的回调函数
            handle(this._error)
            // error 已经传递到下一个 NPromise 了，需要重置，否则会抛出多个相同错误
            // 配合 this._throwErrorIfNotCatch 一起使用，
            // 保证执行到最后才抛出错误，如果没有 catch
            this._error = undefined
          })
        })
      } else {
        // 这里是非异步的情况
        handle(this._error)
        // error 已经传递到下一个 NPromise 了，需要重置，否则会打印多个相同错误
        // 配合 this._throwErrorIfNotCatch 一起使用，
        // 保证执行到最后才抛出错误，如果没有 catch
        this._error = undefined
      }
    })
  }

  catch(onRejected) {
    // catch 其实就是 then 的无 fulfilled 处理
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

NPromise.all = function(iterator) {
  return new NPromise((resolve, reject) => {
    const ret = iterator.reduce((ret, element) => {
      return ret.then(allValue => {
        return NPromise.resolve(element)
          .then(itemValue => {
            if (!allValue) {
              return
            }
            return allValue.concat(itemValue)
          })
          .catch(error => {
            // 发生错误立马 reject
            reject(error)
          })
      })
    }, NPromise.resolve([]))
    resolve(ret)
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
    if (this._status === PENDING) {
      this._status = FULFILLED
      this._nextValue = value
      this._error = undefined
      this._runCallbackQueue()
    }
  }

  /**
   * 操作失败
   * 每次运行这个函数的时候需要重置 this._status = PENDING
   * @param {Any} reason 操作失败传递的值
   */
  _onRejected = reason => {
    if (this._status === PENDING) {
      this._status = REJECTED
      this._error = reason
      this._nextValue = undefined
      this._runCallbackQueue()
    }
  }

  then(onFulfilled, onRejected) {
    return new NPromise((resolve, reject) => {
      const handle = reason => {
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
      if (this._status === PENDING) {
        // NPromise 的 resolve 或者 reject 在异步中执行才会执行到这一步
        this._callbacQueue.push(() => {
          setTimeout(() => {
            // 先使用 setTimeout 代替
            // 保证状态切换为非 PENDING 状态才会执行后续的 then、catch 或者 finally 的回调函数
            handle(this._error)
            // error 已经传递到下一个 NPromise 了，需要重置，否则会抛出多个相同错误
            // 配合 this._throwErrorIfNotCatch 一起使用，
            // 保证执行到最后才抛出错误，如果没有 catch
            this._error = undefined
          })
        })
      } else {
        // 这里是非异步的情况
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

NPromise.all = function(iterator) {
  return new NPromise((resolve, reject) => {
    const ret = iterator.reduce((ret, element) => {
      return ret.then(allValue => {
        return NPromise.resolve(element)
          .then(itemValue => {
            if (!allValue) {
              return
            }
            return allValue.concat(itemValue)
          })
          .catch(error => {
            // 发生错误立马 reject
            reject(error)
          })
      })
    }, NPromise.resolve([]))
    resolve(ret)
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

经过一些测试，出了使用了 setTimeout 外，Promise 单独用法和效果上基本 99% 跟原生的一致。还有一点不一致的就是，**非异步** resolve 或者 reject 的时候，then 的回调是同步运行的，不是异步运行，而原生的是异步运行。

