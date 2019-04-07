# JavaScript 之函数式编程

是个程序员都知道**函数**，但是有些人不一定清楚**函数式编程**的概念。

应用的迭代使程序变得越来越复杂，那么程序员很有必要创造一个结构良好、可读性好、重用性高和可维护性高的代码。

**函数式编程**就是一个良好的代码方式，但是这不代表函数式编程是必须的。你的项目没用到函数式编程，不代表项目不好。

## 什么是函数式编程（FP）？

> 函数式编程关心数据的映射，命令式编程关心解决问题的步骤。

**函数式编程**的对立面就是**命令式编程**。

> **函数式编程**语言中的变量也不是**命令式编程**语言中的变量，即存储状态的单元，而是代数中的变量，即一个值的名称。 变量的值是不可变的（immutable），也就是说不允许像**命令式编程**语言中那样多次给一个变量赋值。

**函数式编程**只是一个概念（一致编码方式），并没有严格的定义。本人根据网上的知识点，简单的定义一下函数式编程（本人总结，或许有人会不同意这个观点）。

**函数式编程就是纯函数的应用，然后把不同的逻辑分离为许多独立功能的纯函数（模块化思想），然后再整合在一起，变成复杂的功能。**

## 什么是纯函数？

> 一个函数如果输入确定，那么输出结果是唯一确定的，并且没有副作用，那么它就是纯函数。

一般符合上面提到的两点就算纯函数：

- 相同的输入必定产生相同的输出
- 在计算的过程中，不会产生副作用

**那怎么理解副作用呢？**

简单的说就是变量的值不可变，包括函数外部变量和函数内部变量。

> 所谓**副作用**，指的是函数内部与外部互动（最典型的情况，就是修改全局变量的值），产生运算以外的其他结果。

**这里说明一下`不可变`，`不可变`指的是我们不能改变原来的变量值。或者原来变量值的改变，不能影响到返回结果。不是只变量值本来就是不可变。**

## 纯函数特性对比例子

上面的理论描述对于刚接触这个概念的程序员，或许不好理解。下面会通过纯函数的特点一一举例说明。

### 输入相同返回值相同

纯函数

```js
function test(pi) {
  // 只要 pi 确定，返回结果就一定确定。
  return pi + 2;
}
test(3);
```

非纯函数

```js
function test(pi) {
  // 随机数返回值不确定
  return pi + Math.random();
}

test(3);
```

### 返回值不受外部变量的影响

非纯函数，返回值会被其他变量影响（说明有副作用），返回值不确定。

```js
let a = 2;
function test(pi) {
  // a 的值可能中途被修改
  return pi + a;
}
a = 3;
test(3);
```

非纯函数，返回值受到对象 getter 的影响，返回结果不确定。

```js
const obj = Object.create(
  {},
  {
    bar: {
      get: function() {
        return Math.random();
      },
    },
  }
);

function test(obj) {
  // obj.a 的值是随机数
  return obj.a;
}
test(obj);
```

纯函数，参数唯一，返回值确定。

```js
function test(pi) {
  // 只要 pi 确定，返回结果就一定确定。
  return pi + 2;
}
test(3);
```

### 输入值是不可以被改变的

非纯函数，这个函数已经改变了外面 personInfo 的值了（产生了副作用）。

```js
const personInfo = { firstName: 'shannan', lastName: 'xian' };

function revereName(p) {
  p.lastName = p.lastName
    .split('')
    .reverse()
    .join('');
  p.firstName = p.firstName
    .split('')
    .reverse()
    .join('');
  return `${p.firstName} ${p.lastName}`;
}
revereName(personInfo);
console.log(personInfo);
// 输出 { firstName: 'nannahs',lastName: 'naix' }
// personInfo 被修改了
```

纯函数，这个函数不影响外部任意的变量。

```js
const personInfo = { firstName: 'shannan', lastName: 'xian' };

function reverseName(p) {
  const lastName = p.lastName
    .split('')
    .reverse()
    .join('');
  const firstName = p.firstName
    .split('')
    .reverse()
    .join('');
  return `${firstName} ${lastName}`;
}
revereName(personInfo);
console.log(personInfo);
// 输出 { firstName: 'shannan',lastName: 'xian' }
// personInfo 还是原值
```

那么你们是不是有疑问，personInfo 对象是引用类型，异步操作的时候，中途改变了 personInfo，那么输出结果那就可能不确定了。

如果函数存在异步操作，的确有存在这个问题，的确应该确保 personInfo 不能被外部再次改变（可以通过深度拷贝）。

但是，这个简单的函数里面并没有异步操作，reverseName 函数运行的那一刻 p 的值已经是确定的了，直到返回结果。

下面的异步操作才需要确保 personInfo 中途不会被改变：

```js
async function reverseName(p) {
  await new Promise(resolve => {
    setTimeout(() => {
      resolve();
    }, 1000);
  });
  const lastName = p.lastName
    .split('')
    .reverse()
    .join('');
  const firstName = p.firstName
    .split('')
    .reverse()
    .join('');
  return `${firstName} ${lastName}`;
}

const personInfo = { firstName: 'shannan', lastName: 'xian' };

async function run() {
  const newName = await reverseName(personInfo);
  console.log(newName);
}

run();
personInfo.firstName = 'test';
// 输出为 tset naix，因为异步操作的中途 firstName 被改变了
```

修改成下面的方式就可以确保 personInfo 中途的修改不影响异步操作：

```js
async function reverseName(p) {
  // 浅层拷贝，这个对象并不复杂
  const newP = { ...p };
  await new Promise(resolve => {
    setTimeout(() => {
      resolve();
    }, 1000);
  });
  const lastName = newP.lastName
    .split('')
    .reverse()
    .join('');
  const firstName = newP.firstName
    .split('')
    .reverse()
    .join('');
  return `${firstName} ${lastName}`;
}

const personInfo = { firstName: 'shannan', lastName: 'xian' };

async function run() {
  const newName = await reverseName(personInfo);
  console.log(newName);
}

// 当然小先运行 run，然后再去改 personInfo 对象。
run();
personInfo.firstName = 'test';
// 输出为 nannahs naix
```

这个还是有个缺点，就是外部 personInfo 对象还是会被改到，但不影响之前已经运行的 run 函数。如果再次运行 run 函数，输入都变了，输出当然也变了。

### 参数和返回值可以是任意类型

那么返回函数也是可以的。

```js
function addX(y) {
  return function(x) {
    return x + y;
  };
}
```

### 尽量只做一件事

当然这个要看实际应用场景，这里举个简单例子。

两件事一起做（不太好的做法）：

```js
function getFilteredTasks(tasks) {
  let filteredTasks = [];
  for (let i = 0; i < tasks.length; i++) {
    let task = tasks[i];
    if (task.type === 'RE' && !task.completed) {
      filteredTasks.push({ ...task, userName: task.user.name });
    }
  }
  return filteredTasks;
}
const filteredTasks = getFilteredTasks(tasks);
```

getFilteredTasks 也是纯函数，但是下面的纯函数更好。

两件事分开做（推荐的做法）：

```js
function isPriorityTask(task) {
  return task.type === 'RE' && !task.completed;
}
function toTaskView(task) {
  return { ...task, userName: task.user.name };
}
let filteredTasks = tasks.filter(isPriorityTask).map(toTaskView);
```

`isPriorityTask` 和 `toTaskView` 就是纯函数，而且都只做了一件事，也可以单独反复使用。

### 结果可缓存

根据纯函数的定义，只要输入确定，那么输出结果就一定确定。我们就可以针对纯函数返回结果进行缓存（缓存代理设计模式）。

```js
const personInfo = { firstName: 'shannan', lastName: 'xian' };

function reverseName(firstName, lastName) {
  const newLastName = lastName
    .split('')
    .reverse()
    .join('');
  const newFirstName = firstName
    .split('')
    .reverse()
    .join('');
  console.log('在 proxyReverseName 中，相同的输入，我只运行了一次');
  return `${newFirstName} ${newLastName}`;
}

const proxyReverseName = (function() {
  const cache = {};
  return (firstName, lastName) => {
    const name = firstName + lastName;
    if (!cache[name]) {
      cache[name] = reverseName(firstName, lastName);
    }
    return cache[name];
  };
})();
```

## 函数式编程有什么优点？

实施函数式编程的思想，我们应该尽量让我们的函数有以下的优点：

- 更容易理解
- 更容易重复使用
- 更容易测试
- 更容易维护
- 更容易重构
- 更容易优化
- 更容易推理

## 函数式编程有什么缺点？

- 性能可能相对来说较差

  函数式编程可能会牺牲**时间复杂度**来换取了可读性和维护性。但是呢，这个对用户来说这个性能十分微小，有些场景甚至可忽略不计。前端一般场景不存在非常大的数据量计算，所以你尽可放心的使用函数式编程。看下上面提到个的例子（数据量要稍微大一点才好对比）：

  首先我们先赋值 10 万条数据：

  ```js
  const tasks = [];
  for (let i = 0; i < 100000; i++) {
    tasks.push({
      user: {
        name: 'one',
      },
      type: 'RE',
    });
    tasks.push({
      user: {
        name: 'two',
      },
      type: '',
    });
  }
  ```

  **两件事一起做，代码可读性不够好，理论上时间复杂度为 o(n)，不考虑 push 的复杂度**。

  ```js
  (function() {
    function getFilteredTasks(tasks) {
      let filteredTasks = [];
      for (let i = 0; i < tasks.length; i++) {
        let task = tasks[i];
        if (task.type === 'RE' && !task.completed) {
          filteredTasks.push({ ...task, userName: task.user.name });
        }
      }
      return filteredTasks;
    }

    const timeConsumings = [];

    for (let k = 0; k < 100; k++) {
      const beginTime = +new Date();
      getFilteredTasks(tasks);
      const endTime = +new Date();

      timeConsumings.push(endTime - beginTime);
    }

    const averageTimeConsuming =
      timeConsumings.reduce((all, current) => {
        return all + current;
      }) / timeConsumings.length;

    console.log(`第一种风格平均耗时：${averageTimeConsuming} 毫秒`);
  })();
  ```

  **两件事分开做，代码可读性相对好，理论上时间复杂度接近 o(2n)**

  ```js
  (function() {
    function isPriorityTask(task) {
      return task.type === 'RE' && !task.completed;
    }
    function toTaskView(task) {
      return { ...task, userName: task.user.name };
    }

    const timeConsumings = [];

    for (let k = 0; k < 100; k++) {
      const beginTime = +new Date();
      tasks.filter(isPriorityTask).map(toTaskView);
      const endTime = +new Date();

      timeConsumings.push(endTime - beginTime);
    }

    const averageTimeConsuming =
      timeConsumings.reduce((all, current) => {
        return all + current;
      }) / timeConsumings.length;

    console.log(`第二种风格平均耗时：${averageTimeConsuming} 毫秒`);
  })();
  ```

  上面的例子多次运行得出耗时平均值，在数据较少和较多的情况下，发现两者平均值并没有多大差别。10 万条数据，运行 100 次取耗时平均值，第二种风格平均多耗时 15 毫秒左右，相当于 10 万条数据多耗时 1.5 秒，1 万条数多据耗时 150 毫秒（150 毫秒用户基本感知不到）。

  虽然理论上时间复杂度多了一倍，但是在数据不庞大的情况下（会有个临界线的），这个性能相差其实并不大，完全可以牺牲浏览器用户的这点性能换取可读和可维护性。

- 很可能被过度使用

  过度使用反而是项目维护性变差。有些人可能写着写着，就变成别人看不懂的代码，自己觉得挺高大上的，但是你确定别人能快速的看懂不? 适当的使用才是合理的。

## 应用场景

概念是概念，实际应用却是五花八门，没有实际应用，记住了也是死记硬背。这里总结一些常用的**函数式编程**应用场景。

### 简单使用

有时候很多人都用到了函数式的编程思想（最简单的用法），但是没有意识到而已。下面的列子就是最简单的应用，这个不用怎么说明，根据上面的纯函数特点，都应该看的明白。

```js
function sum(a, b) {
  return a + b;
}
```

### 立即执行的匿名函数

匿名函数经常用于隔离内外部变量（变量不可变）。

```js
const personInfo = { firstName: 'shannan', lastName: 'xian' };

function reverseName(firstName, lastName) {
  const newLastName = lastName
    .split('')
    .reverse()
    .join('');
  const newFirstName = firstName
    .split('')
    .reverse()
    .join('');
  console.log('在 proxyReverseName 中，相同的输入，我只运行了一次');
  return `${newFirstName} ${newLastName}`;
}

// 匿名函数
const proxyReverseName = (function() {
  const cache = {};
  return (firstName, lastName) => {
    const name = firstName + lastName;
    if (!cache[name]) {
      cache[name] = reverseName(firstName, lastName);
    }
    return cache[name];
  };
})();
```

### JavaScript 的一些 API

如数组的 forEach、map、reduce、filter 等函数的思想就是函数式编程思想（返回新数组），我们并不需要使用 for 来处理。

```js
const arr = [1, 2, '', false];
const newArr = arr.filter(Boolean);
// 相当于 const newArr = arr.filter(value => Boolean(value))
```

### 递归

递归也是一直常用的编程方式，可以代替 while 来处理一些逻辑，这样的可读性和上手度都比 while 简单。

如下二叉树所有节点求和例子：

```js
const tree = {
  value: 0,
  left: {
    value: 1,
    left: {
      value: 3,
    },
  },
  right: {
    value: 2,
    right: {
      value: 4,
    },
  },
};
```

while 的计算方式：

```js
function sum(tree) {
  let sumValue = 0;
  // 使用列队方式处理，使用栈也可以，处理顺序不一样
  const stack = [tree];

  while (stack.length !== 0) {
    const currentTree = stack.shift();
    sumValue += currentTree.value;

    if (currentTree.left) {
      stack.push(currentTree.left);
    }

    if (currentTree.right) {
      stack.push(currentTree.right);
    }
  }

  return sumValue;
}
```

递归的计算方式：

```js
function sum(tree) {
  let sumValue = 0;

  if (tree && tree.value !== undefined) {
    sumValue += tree.value;

    if (tree.left) {
      sumValue += sum(tree.left);
    }
    if (tree.right) {
      sumValue += sum(tree.right);
    }
  }

  return sumValue;
}
```

递归会比 while 代码量少，而且可读性更好，更容易理解。

### 链式编程

如果接触过 jquery，我们最熟悉的莫过于 jq 的链式便利了。现在 ES6 的数组操作也支持链式操作：

```js
const arr = [1, 2, '', false];
const newArr = arr.filter(Boolean).map(String);
// 输出 "1", "2"]
```

或者我们自定义链式，加减乘除的链式运算：

```js
function createOperation() {
  let theLastValue = 0;
  const plusTwoArguments = (a, b) => a + b;
  const multiplyTwoArguments = (a, b) => a * b;

  return {
    plus(...args) {
      theLastValue += args.reduce(plusTwoArguments);
      return this;
    },
    subtract(...args) {
      theLastValue -= args.reduce(plusTwoArguments);
      return this;
    },
    multiply(...args) {
      theLastValue *= args.reduce(multiplyTwoArguments);
      return this;
    },
    divide(...args) {
      theLastValue /= args.reduce(multiplyTwoArguments);
      return this;
    },
    valueOf() {
      const returnValue = theLastValue;
      // 获取值的时候需要重置
      theLastValue = 0;
      return returnValue;
    },
  };
}
const operaton = createOperation();
const result = operation
  .plus(1, 2, 3)
  .subtract(1, 3)
  .multiply(1, 2, 10)
  .divide(10, 5)
  .valueOf();
console.log(result);
```

当然上面的例子不完全都是函数式编程，因为 valueOf 的返回值就不确定。

### 高阶函数

高阶函数（Higher Order Function）,按照维基百科上面的定义，至少满足下列一个条件的函数

- 函数作为参数传入
- 返回值为一个函数

简单的例子：

```js
function add(a, b, fn) {
  return fn(a) + fn(b);
}
function fn(a) {
  return a * a;
}
add(2, 3, fn); // 13
```

还有一些我们平时常用高阶的方法，如 map、reduce、filter、sort，以及现在常用的 redux 中的 connect 等高阶组件也是高阶函数。

#### 柯里化（闭包）

> 柯里化（Currying），又称部分求值（Partial Evaluation），是把接受多个参数的函数变换成接受一个单一参数（最初函数的第一个参数）的函数，并且返回接受余下的参数而且返回结果的新函数的技术。

柯里化的作用以下优点:

- 参数复用
- 提前返回
- 延迟计算/运行
- 缓存计算值

柯里化实质就是闭包。其实上面的立即执行匿名函数的例子就用到了柯里化。

```js
// 柯里化之前
function add(x, y) {
  return x + y;
}

add(1, 2); // 3

// 柯里化之后
function addX(y) {
  return function(x) {
    return x + y;
  };
}

addX(2)(1); // 3
```

#### 高阶组件

这是组件化流行后的一个新概念，目前经常用到。ES6 语法中 class 只是个语法糖，实际上还是函数。

一个简单例子：

```jsx
class ComponentOne extends React.Component {
  render() {
    return <h1>title</h1>;
  }
}

function HocComponent(Component) {
  Component.shouldComponentUpdate = function(nextProps, nextState) {
    if (this.props.id === nextProps.id) {
      return false;
    }
    return true;
  };
  return Component;
}

export default HocComponent(ComponentOne);
```

深入理解高阶组件请看这里。

### 无参数风格（Point-free）

其实上面的一些例子已经使用了无参数风格。无参数风格不是只没参数，只是省略了参数的那一步。看下面的一些例子就很容易理解了。

范例一：

```js
const arr = [1, 2, '', false];
const newArr = arr.filter(Boolean).map(String);
// 有参数的用法如下：
// arr.filter(value => Boolean(value)).map(value => String(value));
```

范例二：

```js
const tasks = [];
for (let i = 0; i < 1000; i++) {
  tasks.push({
    user: {
      name: 'one',
    },
    type: 'RE',
  });
  tasks.push({
    user: {
      name: 'two',
    },
    type: '',
  });
}
function isPriorityTask(task) {
  return task.type === 'RE' && !task.completed;
}
function toTaskView(task) {
  return { ...task, userName: task.user.name };
}
tasks.filter(isPriorityTask).map(toTaskView);
```

#### 无参数风格有什么优点？

更容易理解、代码简小，同时分离的回调函数，是可以复用的。如果使用了原生 js 如数组，还可以利用 Boolean 等构造函数的便捷性进行一些过滤操作。

## 参考文章

- [Learn the fundamentals of functional programming — for free, in your inbox](https://medium.freecodecamp.org/learning-the-fundamentals-of-functional-programming-425c9fd901c6)

- [Make your code easier to read with Functional Programming](https://medium.freecodecamp.org/make-your-code-easier-to-read-with-functional-programming-94fb8cc69f9d)
- [从高阶函数--->高阶组件](https://juejin.im/post/5b019b6bf265da0b95276368)
