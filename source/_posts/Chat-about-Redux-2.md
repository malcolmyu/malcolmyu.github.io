title: 闲聊 Redux（下）
date: 2017-02-02
categories: 技术研究
tags: [Javascript, Redux]
toc: true

---

![redux](https://ws1.sinaimg.cn/large/006cGJIjly1fizgyli3cwj30cc03czkg.jpg)

[上文](/2016/12/03/Chat-about-Redux-1/)里，我们剥茧抽丝，聊了聊 Redux 里面的各种烦人概念，以及这些概念是怎么来的；本文我们着重聊一聊，让 Redux 大行其道的『中间件』。

<!--more-->

## 什么是中间件

宏观意义上的中间件（middleware），定义十分模糊，只要能弥合底层操作系统之间差异，对不同顶层应用提供支持的软件，都算是中间件。按照笔者自己的理解，传统意义的中间件更像是一层封装，掩盖了底层的实现细节、提升了开发效率，例如游戏引擎、[JDBC](https://en.wikipedia.org/wiki/Java_Database_Connectivity)，甚至 jQuery 也算是广泛意义上的中间件。然而自动 TJ 大神搞了 Express 之后，中间件这个概念在前端的语境下也产生了一些变化，它逐渐的变成了一种**对框架的扩展方式**，成为一种开放接口，用户可以通过自行编写中间件，控制数据在框架内部的流动方式、数据的内容，或者是函数的应用方式。

拿 Express 来说，它的中间件实现的就是**数据与控制的集合**，它可以对 https 请求的请求（request）和相应（response）数据进行更改，并通过 `next` 方法进行对中间件的控制，让中间件的编写者有能力控制中间件的**进入与错误终止**。React-Router 的中间件技术也主要是一套异步控制手段，在 `onEnter` 生命周期中加入了对异步行为的支持，异步的中间件可以通过调用 `next` 方法，自行控制进入到下一个中间件的时间点。

![著名的中间件洋葱示意图](https://ww1.sinaimg.cn/large/7921624bjw1fccmekzhe7j207805e74j.jpg)

在 Redux 这里呢，中间件的含义又发生了变化，Redux 官方文档这样说道：

> Redux 中间件与 Express 和 Koa 的中间件解决的问题并不相同，但二者在概念上却高度一致。**从一个 action 被调度（dispatch），到这个 action 到达 reducer，中间件在这两个时刻中间提供了一个第三方扩展的植入点**。

Redux 的中间件确实与 Express 和 Koa 的中间件神似，它处理了 action 的流动，并使用 next 控制下层中间件的进入，并提供了中间件的中断功能。我们就来看一下它是怎么实现这些功能的吧。

## 中间件的使用探索

既然 Redux 中间件的作用就是处理 action 的流动，那我们就先来看下，不使用中间件的时候，action 是怎样流动的。

```js
// 业务代码中调度 action
store.dispatch(action)

// reducer 中处理 action，返回 state
const reducer = (state, action) => {
  switch (action.type) {
    // ...
  }
}
```

可以看到，Redux 内部是隐藏了 action 流动细节的，但是我们在之前的[精简代码](/2016/12/03/Chat-about-Redux-1/#dispatch：更新状态、触发回调)中有看到，在执行 `dispatch` 之后，就会通过 `currentReducer` 来获取最新的状态。也就是说，执行 dispatch，遍历 reducer 是一个**同步的过程**。这一点非常重要，不过目前仅仅是进行日志记录，我们还并不关系它是同步还是异步。

我们接着就上面的代码来看，如果要实现 action 的日志记录，应该怎么办呢？当然，最快速也最耦合的办法，就是在业务代码中写：

```js
console.log('dispatching', action)
store.dispatch(action)
// dispatch 是个同步的过程，dispatch 之后 state 就会发生改变
console.log('next state', store.getState())
```

这样的写法当然很不友好，总不能在每次书写业务代码的时候，都写上这样一坨内容吧。那我们来想一下，我们期望用户在业务里书写的方式是什么呢？实际上就只要写一个 `store.dispatch(action)` 就可以了，不希望还有多余的操作；而这个日志记录，应该是作为顶层中间件，注入到这个 store 当中。这也就等于说，用户在执行 `store.dispatch` 的时候，实际上应该依次执行我们顶层注入的中间件的逻辑。

换句话说，**Redux 中间件实际上是对 `store.dispatch` 方法的劫持**。

意识到这一点，我们的程序也好写了，可以先通过 hack 的方式劫持一下 `dispatch` 方法：

```js
function patchLog(store) {
  const dispatch = store.dispatch
  store.dispatch = function(action) {
    console.log('dispatching', action)
    const result = dispatch(action)
    console.log('next state', store.getState())
    return result
  }
}
```

上面的写法虽然可行，但显然扩展性不强，一个扩展的场景还勉强够用，多个中间件同时存在就没法写了。有的童鞋也许会说，其实还是可以写的啊，比如：

```js
patchLog(store)
patchSomething(store)
```

但是通过赋值修改 `dispatch` 的方式有个两个非常严重的问题：

1. 传给第后面中间件的 store，它的 `dispatch` 方法已经被劫持了，这也就意味着通过这种方式无法给后续的中间件传递原始的 `store.dispatch` 方法。显然，这一点我们是无法接受的，因为原始的 `store.dispatch` 方法的调用，意味着中间件执行的**结束**；后续的中间件无法调用它，就失去了中断中间件执行的能力。
2. 用户在中间件里调用 `store.dispatch`，直接导致栈溢出；

我们来看一下 Redux 官方都是怎样应用中间件的，且怎样解决了以上两个问题。

## 官方中间件实现方式

中间件的编写，官方是这样实现的：

```js
const patchLog = store => next => action => {
  console.log('dispatching', action)
  const result = next(action)
  console.log('next state', store.getState())
  return result
}
```

中间件的应用，官方是这样实现的：

```js
applyMiddleware(patchLog, patchSomething)(store);
```

我们先来看下 Redux `applyMiddleware` 方法的精简实现：

```js
function applyMiddleware(...middlewares) {
  return (store) => {
    // 使用 reduceRight 确保先应用的中间件先执行
    const dispatch = middlewares.reduceRight(
      (next, middleware) => middleware(store)(next),
      store.dispatch
    );

    return { ...store, dispatch };
  }
}
```

实际上，Redux 的 `applyMiddleware` 通过一个技巧很好的解决了上面两个问题：**返回全新的 store 对象**。这确保原有 store 的 `dipatch` 方法没有被覆盖，且很好的保留了原始 `dispatch` 对象（就是 `store.dispatch`）。用户在中间件中可以自行选择调用 `store.dispatch(action)` 或是 `next(action)`：前者表示终止后续中间件的执行，直接将 action 送给 reducer；后者表示将 action 送给后续的中间件并继续执行。

## 异步中间件

介绍完官方中间件的实现与使用方式，我们再来看一下业务中必然会用到的**异步中间件**。在上文中我们说过：执行 dispatch 之后遍历 reducer 是一个**同步的过程**，那怎么执行异步操作呢？这也好办，因为上文我们也说过，**Redux 中间件实际上是对 `store.dispatch` 方法的劫持**，所以只要在异步的时间点去执行 `dispatch` 就行了。

我们来看一下著名的 [redux-thunk](https://github.com/gaearon/redux-thunk) 中间件的源码与用法：

```js
// 源码
function createThunkMiddleware() {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState);
    }

    return next(action);
  };
}

// 使用
store.dispatch(dispatch => {
  setTimeout(() => {
    // 异步触发 dispatch 打到异步效果
    dispatch({ type: 'AN_ANCTION' });
  }, 1000);
})
```

这里我们看到，该中间件会检测 action 是否是函数，如果是函数则将原始 `dispatch` 方法当做函数的参数传入并调用函数（这意味后面的中间件会被当前中间件阻断）；否则将 action 丢给下一个中间件。

从这里我们可以看出，实际上所有的异步中间件实现，本质上都是因为 Redux 的中间件机制提供了一个切入点，让用户可以自行控制 action 流向 reducer 的时机和 action 的内容；控制的方式就是实现一个 wrapper 来接替原始的 `dispatch` 方法，并在 wrapper 中可以异步执行原始 `dispatch`。

至于异步中间件的选型，本文不会详细介绍，可以参考这篇文章：[Redux 异步方案选型](https://zhuanlan.zhihu.com/p/24337401)。

## 总结

尽管 Redux 框架与 15 年刚刚问世的盛极一时相比已经风光不再，但它精巧的设计、良好的解耦与扩展性，以及中间件的扩展方式，都值得我们在进行设计框架时效仿学习。
