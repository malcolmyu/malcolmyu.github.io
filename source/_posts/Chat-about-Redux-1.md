title: 闲聊 Redux（上）
date: 2016-12-03
categories: 技术研究
tags: [Javascript, Redux]
toc: true

---

![redux](https://ws1.sinaimg.cn/large/006cGJIjly1fizao8avctj308k04tmxq.jpg)

Redux 算是 React 全家桶里最饱受争议的一个框架，就算是天天变 API 的 [React-Router](https://github.com/ReactTraining/react-router)，也因为占据了意识形态的最高点 —— 便捷优雅的声明式路由、异步路由控制与加载，导致尽管骂声不断，却没有一个人说不用。反观 Redux，用不惯的说它夹杂了太多的私心，找遍业务线所有的代码，也找不到一个逻辑用到这么复杂的函数式；用的惯的也嫌它使用太复杂，凭空多出几个文件夹、丑陋的 `switch`、一遍一遍的常量声明，真是端起键盘开发，放下鼠标骂娘。与 React-Router 横行社区不同，Redux 从来不缺竞争者，前有 Reflux，后有 [Mobx](https://github.com/mobxjs/mobx)；但似乎也没见有一个竞争者真正撼动 redux 的地位：完美适配服务端渲染，完善的调试工具与社区支持，都让 Redux 成为 React 学习者首选的数据流框架。

今天就来简单聊聊这个可能是这两年最火的数据流框架：[Redux](https://github.com/reactjs/redux)。

<!--more-->

本文将分为上下两部分，在第一部分中我们会聊一聊 Redux 那些烦人的概念，在第二部分中我们会聊一聊『中间件』这一使 Redux 获得社区广泛支持的大杀器。

## 烦人的概念

没用过 Redux 的开发者打开它的文档，扑面而来就是难懂的概念，什么 `compose`、`applyMiddleware`、`reducer` 之类，引得大家都尴尬了起来，关了网页翻身又刷知乎去了。实际上我个人认为，把这些繁杂的概念丢到 Redux 的首页介绍上简直就是一个败笔，这些东西实际上都是在后来的开发过程中，高度抽象的业务逻辑，根本没有必要直接介绍给用户。

那 Redux 到底是个啥咧？我们抛去这些复杂的概念不看，先看看我们开发中最常用到的，也是最核心的概念：[store](https://cn.redux.js.org/docs/basics/Store.html)。这个 store 顾名思义，就是一个数据存储的结构，我们可以自己先自己脑补一下，实现一个数据存储需要写什么功能呢？大概不外乎就是『获取数据』、『存储数据』，但是 Redux 的 store 显然不是这么简单，因为在修改数据之余，我们还需要监听数据变化，以便进行一些其他的操作：比如跟着修改组件监听的 `props`；而这个数据监听的行为，实际上是由使用者定义的，因此还需要接入一套事件管理的机制：『订阅事件』、『取消订阅』、『事件发布』。

简单来说，Redux 的 store 就是**数据管理与事件订阅的组合**，实际上，这个基本的内核也并非 Redux 的原创，而是 Facebook 最早提出来的 [Flux](https://facebook.github.io/flux/) 的概念。翻一下它的[源码](https://github.com/reactjs/redux/blob/master/src/createStore.js#L247)，可以明显的看到它对外提供了三个方法：`dispatch`，`subscribe` 和 `getState`，翻译一下就是『调度』、『订阅』和『状态获取』。这个 `getState` 就不多说了，就是获取一份 store 的当前数据，我们重点聊一聊 `dispatch` 和 `subscribe`。

### dispatch：更新状态、触发回调

我们说 store 是数据管理与事件订阅的组合，那么 `dispatch` 方法就是 store 最核心的功能。看一下它代码的精简版：

```js
function dispatch(action) {
  // 1. 通过 Reducer 遍历 action，更新 state
  currentState = currentReducer(currentState, action)

  // 2. 触发通过 subscribe 订阅的回调
  var listeners = currentListeners = nextListeners
  for (var i = 0; i < listeners.length; i++) {
    var listener = listeners[i]
    listener()
  }

  // 3. 返回 action
  return action
}
```

是不是简洁明了？基本上就干了两件事：**更新当前状态**，**触发监听回调**。读者也许会问，这里的 `currentReducer` 方法是个啥，`listeners` 又是哪里注册的？我们按下不表，只简单的说一下：前者跟 `reducer` 这个概念有关，我们随后会讲到；后者就是 `subscribe` 方法注册的，等下也会细说。

注意到，`dispatch` 函数的参数是 action，这个东西的含义我们会在讲 reducer 的时候[简单说明](#核心的思想)。

其实笔者个人认为，最需要注意的一点是上面注释中的第三点：**`dispatch` 方法的返回值是 action**。也就是说，对于没有使用中间件，或中间件也遵守这一原则的情况下，是可以利用 `dispatch` 方法的返回值，来处理一些贴近业务的耦合代码，例如使用了 [redux-promise](https://github.com/acdlite/redux-promise) 中间件时，我们可以获取 `dispatch` 的返回值的 `payload` 属性（它是那个处理中的 promise），然后等这个 `payload` 处理完成后进行一些业务逻辑操作。

### subscriber：注册回调、做好清理

我们再来看下 `subscriber` 代码的精简版，同样是非常的好理解：

```js
function subscribe(listener) {
  // 1. 注册回调
  nextListeners.push(listener)

  // 2. 返回清理函数
  return function unsubscribe() {
    var index = nextListeners.indexOf(listener)
    nextListeners.splice(index, 1)
  }
}
```

总共又是干了两件事，第一是把回调注册到 `nextListeners` 中，供 `dispatch` 时调用；第二是返回一个清理注册函数的函数，便于用户进行注册函数的清理。这里要注意的，就是第二步清理函数。react 初心者应该都会遇到下面的这行错误提示：

![setState 错误](https://ww1.sinaimg.cn/large/7921624bgw1fb55pktexgj21kw01qdhl.jpg)

导致这个错误的原因，就是异步调用 `setState`，结果执行的时候，由于异步的关系，代码所在的组件都被销毁了。当你使用 redux 时，你会发现基本不会出现这样的提示，那就是因为 redux **在组件销毁的时候调用了清理函数**，因此所有能导致状态发生变化的操作都会在组件移除时被清理。

## 精神内核

说完 redux 跟 flux 相关的内核，再说下 redux 独有的特性：reducer。reducer 是传入给 `createStore` 的参数，是创建 store 的最基本依赖。先说下 reducer 是什么，这东西基本等于 flux 中 [dispatcher](https://facebook.github.io/flux/docs/dispatcher.html) 的升级版。dispatcher 翻译过来就是『调度器』，我们知道 `dispatch` 是『进行调度』，`subscribe` 是『监听调度』，那 reducer 就是处理『怎么调度』。

如要实现下面这个简单的功能：

![简单的例子](https://ww1.sinaimg.cn/large/7921624bgw1fb5blrrg9vg204101pwed.gif)

就需要编写这样一个 reducer：

```js
const reducer = (state = 0, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1
    case 'DECREMENT':
      return state - 1
    default:
      return state
  }
}
```

它的作用就是根据 action 的类型来处理对 `state` 的改变，既然多次提到了 action，我们先解释一下它。

### action：仅仅是个对象

它是一个**纯对象**，且必须带一个 `type` 属性。

为什么要这样设计呢？想象一下上面的例子：我们对数字进行修改，点左边减少、点右边增加，那么每一次点击就会触发一次 action —— 因为每点击一次状态肯定会发生改变；但是左边的点击和右边的点击触发的肯定是**不同的 action** —— 因为改变状态的方式不同。区分 action 的就是它的 `type` 属性（在这个例子里就是 `'INCREMENT'` 和 `'DECREMENT'`）；而不同 action 对应的不同操作就是由 reducer 来处理（如上面的 `state + 1` 和 `state - 1`）。

根据 action，Redux 还衍生出一个 `actionCreator` 的概念 —— 其实这个概念也很无聊，它指代一个 action 的创造函数，毕竟从用户行为转换到 action 对象还需要一些过程，比如下面这样：

```js
function incrementIfOdd(num) {
  return (dispatch) => {
    if (num % 2 === 0) {
      return;
    }

    dispatch({
      type: 'INCREMENT',
      payload: num
    });
  };
}
```

这个例子，对 action 进行了逻辑处理，只有奇数才会触发 `dispatch` 行为。这里你可能会注意到，它返回的并非一个纯对象，这是因为使用了 [redux-thunk](https://github.com/gaearon/redux-thunk) 中间件，action 的形态就从纯对象变为函数，这在下篇中会有所讲述。

### reducer：可以自由组合的调度器

从上面的例子中，我们可以看到 reducer 的主要功能是进行状态的改变的调度控制。但这并非 reducer 的特色，其实它最大的特色正如其名，是进行状态树的**缩减合并**。这是什么意思呢，我们先看一段 js 数组 reducer 的用法：

```js
const arr = [ { a: 1 }, { b: 2 }, { c: 3 } ];
arr.reduce((prev, next) => ({ ...next, child: prev }), {});

// 会变成这样的对象：
// { c: 3, child: { b: 2, child: { a: 1, child: {} } } }
```

那么，redux 的 reducer 也起到了类似的作用（尽管它的源码没有一行用到 `reduce` 函数😂），它可以将我们编写的调度器 reducer 集合起来，最终集合成一个调度器树。这样做的意义在于，在保证了**唯一状态树的前提**之下，我们在编码时还能只关注状态调度的某一细节。那么集合 reducer 的方法呢，就叫做 [`combineReducers`](https://redux.js.org/docs/api/combineReducers.html)。

![combineReducer](https://7xinjg.com1.z0.glb.clouddn.com/combined-redux.png)

这里我们就不对具体的 API 进行讲解了，综合来说，reducer 带给 redux 最大的特性就在于：**编码时关心单个 reducer，但最终生成统一调度器，调度生成单一的状态树**。

## compose：函数式编程的私货

Redux 有一个单独的概念，叫做 compose，这其实是函数式编程里面的一个套路，可以单拿出来说一说。

前一阵阮老师翻译过一本《黑客与画家》，曾经掀起过一阵 lisp 学习小高潮。lisp 是一门函数是一等公民的语言，万物皆函数，要处理一个顺序逻辑，基本上就得这么写：`a(b(c(d(e()))))`，所以打开你的 lisp 代码，最后几行基本是这样的：

```lisp
        ))))))))))))
      ))))))))
    )))))
  ))
)
```

这简直就是阻碍函数式编程普及的恶魔啊！不过没关系，这里要将的 `compose` 方法，就是起到代码美化作用的：它可以把上面的 `a(b(c(d(e(123)))))` 改成：`compose(a,b,c,d,e)(123)`。是不是瞬间有了活下去的动力？

我们来考虑考虑，在 ES6 里，这个函数应该怎么写：

1. 首先我们应该把传入的一坨函数改造成数组，这个 ES6 里面的 [rest 参数](https://es6.ruanyifeng.com/#docs/function#rest参数)已经帮我们很好的搞定了；
2. 其次我们需要**逆序**执行函数，这就可以用到 `reduceRight`。

我们来实现一下：

```js
function compose(...funcs) {
   let last = funcs.pop();
   let rest = funcs;
   
   return (...args) => rest.reduceRight((compose, f) => f(compose), last(...args));
}
```

有了这个函数，就可以进行函数的连接操作了。当然，在上篇介绍的基本概念里并没有用到它的地方，它真正的用处还是在于下篇的中间件。

## 总结

本文介绍了 Redux 一些基本理念，剥去 Redux 神秘的面纱，我们发现它就是一个 flux 内核加 reducer 缩减组合的套路，后面这个套路好像还是从 Elm 里面抄的。下篇我们会对 Redux 的扩展手段『中间件』进行详细的讲解，这才是令 Redux 大行其道的最重要原因。
