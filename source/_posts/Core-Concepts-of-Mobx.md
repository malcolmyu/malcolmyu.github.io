title: MobX 源码探究之一 —— 核心概念
date: 2018-09-09
categories: 源码分析
tags: [Javascript, MobX]
toc: true

---

> **Anything that can be derived from the application state, should be derived. Automatically.**
>
> 任何可以从应用状态中推导的东西，都应该被**自动地**推导出来。

上面两句话是 MobX 中文官网关于 MobX 的总结，初读起来觉得逼格慢慢，但毫无诚意——因为这样上升到哲学高度的话对于你对它工作原理的理解实际上是毫无帮助的，甚至于你用了框架若干年之后，依然无法理解这其中的含义。但是，哲学层次的总结总是需要在读完代码之后再细细品味的，就像老话说的，『当你千辛万苦读完代码爬上山顶的时候，框架作者装逼的哲学总结已经在此等候多时了』。

<!--more-->

## 核心概念

我们今天先抛开这段话，简单的看一下 MobX 基本的工作原理和几个核心的概念。MobX 是一个接地气的数据流框架，它给命令式编程和响应式编程搭建了一座桥梁，让前端小白们能够通过简单的赋值语句驱动 React 视图的渲染。这对于个把月也搞不明白 Redux 是个啥，或者即使搞明白也要天天撸模板代码的各位前端同僚们来说，节省了大把时间，下班回家的时间又可以提前几分钟<del>（不存在的）</del>。

我们来看一个最简单的例子：

```typescript
// 1. 封装可观测值
const store = observable({ a: 1 });

// 2. 依赖收集，打印 1
autorun(() => console.log(store.a);

// 3. 推导，触发 derivation 的重新执行，打印 2
store.a = 2;
```

这个例子就是一个最基础的 MobX 功能，它明显的分为三个步骤：

1. **封装可观测值**，对其他 MVVM 框架有了解的同学们应该能理解这个过程，这一步的操作的目的是：把一个单纯对象，封装成一个巨复杂的对象。（如上例中的 `{ a: 1 }` 就变成了下图的样子）

   ![面目全非的 Plain Object](https://ww1.sinaimg.cn/large/7921624bly1fv3l54n33tj21bs06cabh.jpg)

   这个巨复杂的对象它的学名叫做**可观测值（Observable）**，是 MobX 里第一个重要的概念。

2. **依赖收集**，第二步使用 autorun 包裹一个回调的操作，学名叫做依赖收集。这里这个打印 a 的回调，MobX 叫它**衍生（Derivation）**，这是 MobX 里第二个重要的概念。但实际上，这里在运行时还暗含了一个步骤，就是**封装衍生**，衍生本身也是一个巨复杂的对象，在运行 `autorun` 的时候，会构造出这个对象。

3. **推导**，第三步给可观测值赋值的行为会触发推导，这是展示 MobX 膜法实力的一步，前面几部的铺垫实际上也是为了这一步而做准备。这里仅仅使用了一行赋值语句，便可以驱动第二步中的衍生函数自动执行，这就是文档里提到的桥接『命令式编程』与『响应式编程』。

上面的三个步骤还引出了 MobX 的两个核心概念：**可观测值**和**衍生**。MobX 的其他概念和行为要么是基于这两个概念实现的，要么是围绕着这两个概念展开的。

## 推导

说了这么多是不是懵逼了？别急，这几段如同开头的哲学语录一样，主要是切个题，回过头我们再来看这些概念。现在先看下 MobX 是怎么做到自动执行代码的？

大家应该都对前端里经常用到的一个设计模式『观察者模式』有一定的了解，DOM 的事件绑定本身就有这种模式，而且很多组件库里大多都实现了这种 `EventEimitter` 的效果，例如：

```js
const em = new EventEmitter();

// 注册回调，对应上面的第二步『依赖收集』
em.on('event-a', () => console.log('aaa'));

// 触发回调，对应上面的第三部『推导』
em.emit('event-a'); // 执行之前注册的回调，输出 aaa
```

其实细细看来，这个自动触发的效果，跟上面的 `autorun` 也有几分类似嘛！`autorun` 其实就是这里 `on` 方法的效果；对 `store.a` 的赋值，实际上就是 `emit` 的效果。实际上，除却 `computed` 计算属性，普通的自动执行或者叫**自动推导的原理，实际上跟 EventEmitter 是非常相似的**。

他们的区别仅仅在于：**回调注册和触发的手段，是隐式而非显式的**。为了这一 magic 的效果，各路 MVVM 框架做了好多工作，才让代码写起来如此的令人愉悦。

我们先看一下 MobX 里『触发回调』的实现手段：仅仅是一行简单的赋值语句：`store.a = 2`。这行我们天天都会写到的赋值，背后实际上干了许许多多的工作。我们思考一下，能够让一行赋值语句，能够执行其他回调的方式有什么办法呢？

是的，就是 `Object.definePorperty`（或者 Proxy），具体参考[这里](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)。因此我们可以简单的理解 MobX 在运行过程中，进行了这样的操作：

```js
const em = new EventEmitter();

Object.defineProperty(store, 'a', {
  set: function() {
    em.emit('event');
  },
});

store.a = 2; // 执行 set 方法，触发 event-a
```

好了，推导的手段有了，那么如何确定在 emit 的时候应该触发哪个方法呢？或者说 MobX 的回调是什么时候注册的呢？其实核心就在这个 `autorun` 方法之中。

## 依赖收集

我们看一下 `autorun` 的功能，它实际上是要把回调中运行的所有可观测值收集起来，等到其中一个值发生变动时，重新运行该回调。有心的同学可能会发现，上面的例子中在第二步有输出结果，也就是**在运行 `autorun` 方法的时候，内部的回调是会执行的**。如图：

![autorun 的自动执行](https://ww1.sinaimg.cn/large/7921624bly1fvjlw2yjqfj215y09ejt3.jpg)

这里的执行实际上就是关键：我们上文中提到，MobX 在赋值时，通过 `set` 成功的做了触发回调的行为，那么与 `set` 相对应的就是 `get` 方法。在代码执行的时候，涉及到取值操作，就会导致  `get` 方法的触发，MobX 就是在这里进行了依赖收集，或者叫回调注册的的操作。

我们把上面的代码再扩展一下：

```jsx
const em = new EventEmitter();

Object.defineProperty(store, 'a', {
  get: function() {
    em.on('event', /* 这里的回调怎么写呢？ */);
  },
  set: function() {
    em.emit('event');
  },
});

store.a = 2; // 执行 set 方法，触发 event-a
```

那么问题来了，我们知道 MobX 是在 get 中进行的依赖收集，那它是如何知道对应的回调是哪一个呢？这里其实就是一个问题，如何在函数运行时获得当前正在运行的函数？

有人可能会立刻提到一个手段：`arguments.callee`。首先，这个方法并不推荐使用，而且如果遇到下面的情况会引发问题：

```js
autorun(() => {
  // other code...
  const funcInAutorun = () => {
    console.log(store.a);
  }
  
  funcInAutorun();
  // other code...
});

// 这样会仅执行 funcInAutorun，而不会执行 autorun 的其他代码
store.a = 2;
```

那么 MobX 究竟是怎么操作的呢？其实很简单，MobX 用了一个全局变量。

MobX 在 `autorun` 的回调执行前，开了一个全局变量叫做 `pendingDerivation`，并把当前的回调赋值给它，当这个方法执行完成后再把这个变量置空（或者置回上层推导函数）。由于 `get` 方法也是同步执行，因此在执行时可以找到当前的 `pendingDerivation`，把它当做要收集的回调。

我们修改代码如下：

```js
const em = new EventEmitter();
// 简单起见就叫 pending 吧
let pending = null;

const autorun = (fn) => {
  pending = fn;
  fn();
  pending = null;
};

Object.defineProperty(store, 'a', {
  get: function() {
    // 直接注册当前执行的推导函数
    em.on('event', pending);
    return 1;
  },
  set: function() {
    em.emit('event');
  },
});

// 触发 get 方法，进行依赖收集
autorun(() => console.log(store.a));

store.a = 2; // 执行 set 方法，触发 event
```

其实在同步情况下，使用全局变量标记运行情况可以解决很多问题。

## 封装可观测值

实际上，做完这一步，就把自动推导的核心工作搞定了，不过我们上面的代码还有个问题没有考虑：没有对 a 本身的值进行管理，现在赋值和取值都只能触发副作用，而不能得到正确的值，这有点不太行。

好吧，因此我们需要处理一下上面简陋的 `defineProperty` 工作，也就是 MobX 里面的 `observable` 方法。

根据上面的探索，我们得出 `observable` 方法需要进行两步的工作：

1. 对本身的值进行存储；
2. 进行依赖收集和触发的封装；

我们定义一个 `_data` 私有属性，来存放它原本的值，代码如下：

```js
const EM = require('events').EventEmitter;
const em = new EM();

// -----框架代码-----
let pending = null;

const autorun = (fn) => {
  pending = fn;
  fn();
  pending = null;
};

const observable = (o) => {
  // 这个 _data 要是不可枚举的，放在被再次封装上 get、set 方法
  // 它仅用于进行值的存储
  Object.defineProperty(o, '_data', {
    enumerable: false,
    value: Object.assign({}, o),
  });
  
  // 封装 get/set
  Object.keys(o).forEach(key => {
    Object.defineProperty(o, key, {
      get: function() {
        if (pending) em.on('event', pending);
        return this._data[key];
      },
      set: function(v) {
        // 一个简单的处理，值不变时不触发
        if (this._data[key] !== v) {
          this._data[key] = v;
          em.emit('event');
        }
      }
    });
  });

  return o;
}

// -----业务代码-----
const store = observable({ a: 1 });

// 触发 get 方法，进行依赖收集
autorun(() => console.log(store.a)); // 成功工作执行 1

store.a = 2; // 成功工作执行 2
```

经过这样一通处理，我们用不到 50 行代码实现了一个简单的，只有一层深度可观测值，且推导函数不能嵌套的微型 MobX<del>（缺点好多啊好像没什么卵用）</del>。不过这次的代码编写只是想跟大家分享一下 MobX 的最最基本的原理，这时候我们再去回顾一下开篇的几个概念：

1. **封装可观测值**（box observable values），把一个单纯对象，封装成一个巨复杂的对象。在我们代码里，这里处理了事件收集、触发和值本身，是最复杂的逻辑；
2. **依赖收集**（tracking dependencies），依赖收集实际上是 `get` 里面做的工作，要注意的是 MobX 通过一个全局变量，完成了对当前正在运行的推导函数的收集；
3. **推导 **(derive)，经过上面两步的操作，推导反而成了最容易的一步，只要触发回调就好，水到渠成。

本文我们主要聊了聊 MobX 中或者说是 MVVM 框架中的几个核心概念，如果想阅读 MobX 源代码，希望参考我文末的文章，特别推荐 JSON 简时空的[几篇博文](https://segmentfault.com/a/1190000013682735)，起到了把代码翻译成人话的作用，对有志阅读代码的童鞋简直就是一盏指明灯。

下篇文章我会着重说下 MobX 作者最引以为豪的 `computed` 的实现，大多数自动推导的框架都没有在 `computed` 上进行像 MobX 这样的性能优化（同时回味一下作者那句哲学总结）。

---

*参考文章*

- [MobX 中文文档](https://cn.mobx.js.org/) 至少先搞明白咋用吧
- [MobX 原理](https://github.com/sorrycc/blog/issues/3) dva 作者，云谦老哥写的，简单易懂
- [用故事解读 MobX 源码系列](https://segmentfault.com/a/1190000013682735) 这个系列非常好，跟读源码差不多，作者对很多细节有图文并茂的解释 