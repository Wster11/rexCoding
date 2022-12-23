---
title: React自定义Hooks
date: 2020-08-01 17:48:44
tags: React
---

# 自定义Hooks

> 自定义 Hook 是 React Hooks 中最有趣的功能，或者说特色。简单来说，它用一种高度灵活的方式，能够让你在不同的函数组件之间共享某些特定的逻辑。

## 一个简单的自定义 Hooks

```javascript
import { useState, useEffect } from 'react';
// 自定义hook useMousePostion
const useMousePostion = () => {
    // 使用hookuseState初始化一个state
    const [postion, setPostion] = useState({ x: 0, y: 0 });
    function handleMove(e) {
        setPostion({ x: e.clientX, y: e.clientY });
    }
    // 使用useEffect处理class中生命周期可以做到的事情
    useEffect(() => {
        // 同时可以处理 componentDidMount 以及 componentDidUpdate 中的事情
        window.addEventListener('mousemove', handleMove);
        document.title = `(${postion.x},${postion.y})`;
        return () => {
            // return的function 可以相当于在组件被卸载的时候执行 类似于 componentWillUnmount
            window.removeEventListener('mousemove', handleMove);
        };
        // [] 是参数，代表deps，也就是说react触发这个hook的时机会和传入的deps有关，内部利用object.is实现
        // 默认不给参数会在每次render的时候调用，给空数组会导致每次比较是一样的，只执行一次，这里正确的应该是给postion
    }, [postion]);
    // postion 可以被直接return，这样达到了逻辑的复用~，哪里需要哪里调用就可以了。
    return postion;
};

export default useMousePostion
```
> 通过观察，我们可以发现自定义 Hook 具有以下特点：
 - 表面上：一个命名格式为 useXXX 的函数，但不是 React 函数式组件
 - 本质上：内部通过使用 React 自带的一些 Hook （例如 useState 和 useEffect ）来实现某些通用的逻辑

有很多地方可以去做自定义 Hook：DOM 副作用修改/监听、动画、请求、表单操作、数据存储等等。
> 推荐两个强大的 React Hooks 库：React Use 和 Umi Hook。它们都实现了很多生产级别的自定义 

## useCallback
> 依赖数组在判断元素是否发生改变时使用了 Object.is 进行比较，因此当 deps 中某一元素为非原始类型时（例如函数、对象等），每次渲染都会发生改变，从而每次都会触发 Effect，失去了 deps 本身的意义。

关于 Effect 无限循环

```javascript

function useEndlessCounter({ converter = data => data }) {
  const [counter, setCount] = useState(0);

  useEffect(() => {
    setCount(converter(counter));
  }, [converter]);

  return counter;
}

```
我们的组件陷入了：渲染 => 触发 Effect => 修改状态 => 触发重渲染的无限循环。

我们知道，在 JavaScript 中，原始类型和非原始类型在判断值是否相同的时候有巨大的差别：

```javascript
// 原始类型
true === true // true
1 === 1 // true
'a' === 'a' // true

// 非原始类型
{} === {} // false
[] === [] // false
() => {} === () => {} // false

```

同样，每次传入的 converter 函数虽然形式上一样，但仍然是不同的函数（引用不相等），从而导致每次都会执行 Effect 函数

### 关于记忆化缓存（Memoization）

Memoization，一般称为记忆化缓存（或者“记忆”），听上去是很高深的计算机专业术语，但是它背后的思想很简单：假如我们有一个计算量很大的纯函数（给定相同的输入，一定会得到相同的输出），那么我们在第一次遇到特定输入的时候，把它的输出结果“记”（缓存）下来，那么下次碰到同样的输出，只需要从缓存里面拿出来直接返回就可以了，省去了计算的过程！

实际上，除了节省不必要的计算、从而提高程序性能之外，Memoization 还有一个用途：用了保证返回值的引用相等。

我们先通过一段简单的求平方根的函数，熟悉一下 Memoization 的原理。首先是一个没有缓存的版本：

```javascript

function sqrt(arg) {
  return { result: Math.sqrt(arg) };  //返回对象和后面缓存版本做对比
}

```

缓存版本

```javascript

function memoizedSqrt(arg) {
  // 如果 cache 不存在，则初始化一个空对象
  if (!memoizedSqrt.cache) {
    memoizedSqrt.cache = {};
  }

  // 如果 cache 没有命中，则先计算，再存入 cache，然后返回结果
  if (!memoizedSqrt.cache[arg]) {
    return memoizedSqrt.cache[arg] = { result: Math.sqrt(arg) };
  }

  // 直接返回 cache 内的结果，无需计算
  return memoizedSqrt.cache[arg];
}

```

结果对比:

```javascript

sqrt(9)                      // { result: 3 }
sqrt(9) === sqrt(9)          // false
Object.is(sqrt(9), sqrt(9))  // false

memoizedSqrt(9)                              // { result: 3 }
memoizedSqrt(9) === memoizedSqrt(9)          // true
Object.is(memoizedSqrt(9), memoizedSqrt(9))  // true

```

普通的 sqrt 每次返回的结果的引用都不相同（或者说是一个全新的对象），而 memoizedSqrt 则能返回完全相同的对象。因此在 React 中，通过 Memoization 可以确保多次渲染中的 Prop 或者状态的引用相等，从而能够避免不必要的重渲染或者副作用执行。

让我们来总结一下记忆化缓存（Memoization）的两个使用场景：

- 通过缓存计算结果，节省费时的计算
- 保证相同输入下返回值的引用相等

> 为了解决函数在多次渲染中的引用相等（Referential Equality）问题，React 引入了一个重要的 Hook—— useCallback

```javascript
const memoizedCallback = useCallback(callback, deps);
```

第一个参数 callback 就是需要记忆的函数，第二个参数就是大家熟悉的 deps 参数，同样也是一个依赖数组（有时候也被称为输入 inputs）。在 Memoization 的上下文中，这个 deps 的作用相当于缓存中的键（Key），如果键没有改变，那么就直接返回缓存中的函数，并且确保是引用相同的函数。

在大多数情况下，我们都是传入空数组 [] 作为 deps 参数，这样 useCallback 返回的就始终是同一个函数，永远不会更新。

> 在学习 useEffect 的时候发现：我们并不需要把 useState 返回的第二个 Setter 函数作为 Effect 的依赖。实际上，React 内部已经对 Setter 函数做了 Memoization 处理，因此每次渲染拿到的 Setter 函数都是完全一样的，deps 加不加都是没有影响的。

### useCallback 和 useMemo 的关系

useCallback 有个好基友叫 useMemo, useCallback 主要是为了解决函数的”引用相等“问题，而 useMemo 则是一个”全能型选手“，能够同时胜任引用相等和节约计算的任务

useMemo 的使用方法如下：

```javascript
const memoizedValue = useMemo(() => fn(a, b), [a, b]);

```

其中第一个参数是一个函数，这个函数返回值的返回值 将返回给 memoizedValue 。因此以下两个钩子的使用是完全等价的：

```javascript
useCallback(fn, deps);
useMemo(() => fn, deps);
```
然后我们用 useCallback 去解决上面计数器无限循环的问题并且去看一个 useMemo 性能优化案例

## useReducer 和 useContext

useState 一个未解决的问题

你很有可能在使用 useState 的时候遇到过一个问题：通过 Setter 修改状态的时候，怎么读取上一个状态值，并在此基础上修改呢？如果你看文档足够细致，应该会注意到 useState 有一个函数式更新（Functional Update）的用法，以下面这段计数器（代码来自 React 官网）为例：

```javascript

function Counter({initialCount}) {
  const [count, setCount] = useState(initialCount);
  return (
    <>
      Count: {count}
      <button onClick={() => setCount(initialCount)}>Reset</button>
      <button onClick={() => setCount(prevCount => prevCount - 1)}>-</button>
      <button onClick={() => setCount(prevCount => prevCount + 1)}>+</button>
    </>
  );
}

```

> 可以看到，我们传入 setCount 的是一个函数，它的参数是之前的状态，返回的是新的状态,这其实就是一个 Reducer 函数

### Reducer 函数

我们这里将先回归最基础的概念，暂时忘掉框架相关的知识。在学习 JavaScript 基础时，你应该接触过数组的 reduce 方法，它可以用一种相当炫酷的方式实现数组求和：

```javascript

const nums = [1, 2, 3]
const value = nums.reduce((acc, next) => acc + next, 0)

```

其中 reduce 的第一个参数 (acc, next) => acc + next 就是一个 Reducer 函数。从表面上来看，这个函数接受一个状态的累积值 acc 和新的值 next，然后返回更新过后的累积值 acc + next

从更深层次来说，Reducer 函数有两个必要规则：

- 只返回一个值
- 不修改输入值，而是返回新的值

第一点很好判断，其中第二点则是很多新手踩过的坑，对比以下两个函数：

```javascript

// 不是 Reducer 函数！ push会改变原数组
function buy(cart, thing) {
  cart.push(thing);
  return cart;
}

// 正宗的 Reducer 函数
function buy(cart, thing) {
  return cart.concat(thing);
}

```

调用数组的 push 方法，会就地修改输入的 cart 参数（是否 return 都无所谓了），违反了 Reducer 第二条规则，而下面的函数通过数组的 concat 方法返回了一个全新的数组，避免了直接修改 cart

此时我们在看 useState 的函数式写法 是一个很标准的 reducer 函数

```javascript
setCount(prevCount => prevCount + 1);
```

我们在之前大量地使用了 useState，你可能就此认为 useState 应该是最底层的元素了。但实际上在 React 的源码中，useState 的实现使用了 useReducer

```javascript

function basicStateReducer(state, action) {
  return typeof action === 'function' ? action(state) : action;
}

```

### useReducer 使用浅析

官方介绍的 reducer 用法

```javascript
const [state, dispatch] = useReducer(reducer, initialArg, init);

```

首先我们来看下 useReducer 需要提供哪些参数：

第一个参数 reducer 显然是必须的，它的形式跟 Redux 中的 Reducer 函数完全一致，即 (state, action) => newState
第二个参数 initialArg 就是状态的初始值
第三个参数 init 是一个可选的用于懒初始化（Lazy Initialization）的函数，这个函数返回初始化后的状态
返回的 state（只读状态）和 dispatch（派发函数）则比较容易理解了。我们来结合一个简单的计数器例子讲解一下

```javascript
// Reducer 函数
function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    default:
      throw new Error();
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  return (
    <>
      Count: {state.count}
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
    </>
  );
}

```

我们首先关注一下 Reducer 函数，它的两个参数 state 和 action 分别是当前状态和 dispatch 派发的动作。这里的动作就是普通的 JavaScript 对象，用来表示修改状态的操作，注意 type 是必须要有的属性，代表动作的类型。然后我们根据 action 的类型返回相应修改后的新状态。

然后在 Counter 组件中，我们通过 useReducer 钩子获取到了状态和 dispatch 函数，然后把这个状态渲染出来。在按钮 button 的 onClick 回调函数中，我们通过 dispatch 一个类型为 increment 的 Action 去更新状态。

### 什么时候使用 useReducer
你也许发现，useReducer 和 useState 的使用目的几乎是一样的：定义状态和修改状态的逻辑。useReducer 使用起来较为繁琐，但如果你的状态管理出现了至少一个以下所列举的问题：

- 需要维护的状态本身比较复杂，多个状态之间相互依赖
- 修改状态的过程比较复杂

就强烈建议你使用 useReducer 了
假设我们要做一个支持撤销和重做的编辑器

```javascript

// 用于懒初始化的函数
function init(initialState) {
  return {
    past: [],
    present: initialState,
    future: [],
  };
}

// Reducer 函数
function reducer(state, action) {
  const { past, future, present } = state;
  switch (action.type) {
    case 'UNDO':
      return {
        past: past.slice(0, past.length - 1),
        present: past[past.length - 1],
        future: [present, ...future],
      };
    case 'REDO':
      return {
        past: [...past, present],
        present: future[0],
        future: future.slice(1),
      };
    default:
      return state;
  }
}

``` 
如果用useState来写是比较困难的

## useContext 使用浅析

在 Hooks 诞生之前，React 已经有了在组件树中共享数据的解决方案：Context在类组件中，我们可以通过 createContext 属性获取到最近的 Context Provider

从 API 名字就可以看出， createContext 能够创建一个 React 的 上下文（context），然后订阅了这个上下文的组件中，可以拿到上下文中提供的数据或者其他信息。

```javascript

const MyContext = React.createContext(defaultValue)

```

使用 useContext 获取上下文

```javascript
// 从上面代码可以发现，useContext 需要将 MyContext 这个 Context 实例传入，不是字符串，就是实例本身。
const {funcName} = useContext(MyContext);
```

这种用法会存在一个比较尴尬的地方，父子组件不在一个目录中，如何共享 MyContext 这个 Context 实例呢？

一般这种情况下，会通过 Context Manager 统一管理上下文的实例，然后通过 export 将实例导出，在子组件中在将实例 import 进来。

案例: createContext 和 useContext 结合使用实现方法共享

> 举个实际的例子：子组件中修改父组件的 state, 一般的做法是将父组件的方法比如 setXXX 通过 props 的方式传给子组件，而一旦子组件多层级的话，就要层层透传。
使用 Context 的方式则可以免去这种层层透传

1、context-manager.js
创建一个上下文管理的组件，用来统一导出 Context 实例

```javascript
import React from 'react';

export const MyContext = React.createContext(null);
```

2、下面代码中，父组件引入了实例，并且通过 MyContext.Provider 将父组件包装，并且通过 Provider.value 将方法提供出去。

下面的实例提供了三个 state 操作方法：

setStep
setCount
setNumber
以及一个副作用方法：

fetchData
使用 useReducer 减少 Context 的复杂程度

3、子组件 useContext 解析上下文

下面是子组件，相同的，也需要从 context-manager 中引入 MyContext 这个实例，然后才能通过 const { setStep, setNumber, setCount, fetchData } = useContext(MyContext); 解析出上下文中的方法，在子组件中则可以直接使用这些方法，修改父组件的 state。

4、使用 useReducer 减少 Context 的复杂程度

上面的示例虽然实现了多级组件方法共享，但是暴露出一个问题：所有的方法都放在了 Context.Provider.value 属性中传递，必然造成整个 Context Provider 提供的方法越来越多，也会臃肿。
而像 setStep、setCount、setNumber 这三个方法，是可以通过 useReducer 包装，并且通过 dispatch 触发的，因此修改一下父组件：

5、将 state 也通过 Context 传递给子组件

上面的所有示例中，子组件获取父组件的 state 还是通过 props ，多级子组件又会存在层层嵌套
如果将整个 state 通过 Context 传入就无需层层组件的 props 传递（如果不需要整个state，可以只将某几个 state 给 Provider）

> 直接使用父组件 state 带来的性能问题,在点击子组件的 【number + step】 按钮的时候，虽然 count 的值没有发生任何变化，但是一直触发 re-render，即使子组件是通过 React.memo 包装过的。
出现这个问题原因是 React.memo 只会对 props 进行浅比较，而通过 Context 我们直接将 state 注入到了组件内部，因此 state 的变化必然会触发 re-render，整个 state 变化是绕过了 memo。

6、 使用 useMemo() 解决 state Context 透传的性能问题

> 既然 React.memo() 无法拦截注入到 Context 的 state 的变化，那就需要我们在组件内部进行更细粒度的性能优化，这个时候可以使用 useMemo()


