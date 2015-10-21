title: React 小技巧与最佳实践
date: 2015-09-11
categories: [技术研究]
tags: [翻译, Javascript, React]
toc: true

---

在过去的一年里我在工作中大量的使用了 [React](http://facebook.github.io/react/)。这段时间内我编写、重构和重写了大量的组件，并且发现了一些最佳实践和反模式。

由于网上已经有大量的介绍 React 的文章，因此本文不准备详细介绍什么是 React 以及我们应该怎么使用 React。我们也假设本文的读者已经知道 React 的基本使用且已经自己写了一两个组件了。

<!--more-->

# 使用 PureRenderMixin

[PureRenderMixin](http://facebook.github.io/react/docs/pure-render-mixin.html) 是一个重写了 `shouldComponentUpdate` 方法的混合方法（Mixin），它只有当 `props` 或 `state` 真正改变的时候才会重新渲染组件。这是在 React 已经十分优异的表现上又进行的一个巨大优化。这也意味着我们可以频繁使用 `setState` 方法而不必担心虚拟重绘。我们也不必像下面一样在代码里写一堆检查搞乱我们的代码：

```javascript
if (this.state.someVal !== computedVal) {
    this.setState({someVal: computedVal})
}
```

这个混合方法要求我们的 `render()` 方法必须是“纯粹的”。

【上面这一段不是很理解】

在实践中使用 `PureRenderMixin` 通常不会有什么问题。我将它混入到好多个组件中之后依然发现其表现良好。如果你使用了它组件却没有工作，你就应该重构一下你的组件直到它工作为止——因为大部分情况是你自己的代码出了问题。

一个重要的一点是仅对 `nextProps` 和 `nextState` 进行『浅对比』。如果属性或者状态是对象或者数组，那么修改它的属性或者增加一个值并不会使其引用发生改变，因此也就不会触发组件的更新。因此我们需要在改变值的时候令 `props` 返回一个指向新的，已更新的对象的新引用。

有很多方法可以做到这一点。[Immutable.js](http://facebook.github.io/immutable-js/) 和 [mori](http://swannodette.github.io/mori/) 引入了一种新的不可变的数据类型。如果我们想使用 JavaScript 原生的数据类型，我们可以使用 React 的[不可变数据辅助工具](http://facebook.github.io/react/docs/update.html)或者我自己写的 [icepick](https://npmjs.org/package/icepick)，假定这种不变的 JavaScript 对象是持久性数据集合。

这些库给 `props` 对象的处理增加了额外的工作开销，但是由于其能够有效的检测出虚拟重绘，因此这些额外的开销也是值得的。也有一个争论的说法是尽量避免在 `props` 或 `state` 中使用对象，属性的减少就意味着触发重绘的机会也减少了。

如果你还没有使用过 `PureRenderMixin`，你就错过了 React 最好的特性之一，它简化了代码且带来了良好的性能提升。

# 使用属性类型

当我们的应用变得复杂且我们开始将组建组合成复杂结构的时候，寻找丢失的或者不恰当的 `props` 将会是极为痛苦的。幸运的是，React 提供了一种叫做 [PropTypes](http://facebook.github.io/react/docs/reusable-components.html#prop-validation) 的方式来指定一个组件中每一个属性所期望的数据类型且每个属性是否为必填。每一个 React 的类都可以定义一个 `propTypes` 集合以给每一个属性指定一个校验方法来确保每个属性只能接受其预先设定的数据类型。如果校验失败，控制台就会输出对应的警告信息。如果我们忘记给一个 `isRequired` 的属性指定值或者给一个期望传入 `object` 的属性传入了字符串，那控制台就会输出帮助信息。

`propTypes` 也是一个注释组件属性类型的便捷方法。

还有就是 `proType` 仅仅在生产环境之外才会进行检查，因此要确保设置 `NODE_ENV="production"` 以避免在我们最终上线的应用总产生不必要的检查以影响效率。Browserify 的 transform 组件 [envify](https://github.com/hughsk/envify) 结合 `uglify --compress` 也是个解决这个问题的便捷方式。

另外，所有不是 `isRequired` 的 `propType` 在 `getDefaultProps()` 中都应该有一个对应的值。（由于方法 `getDefaultProps()`在每个 `createClass` 之前仅执行一次，而不是在每个实例之前都执行，这就意味着对象属性在每个实例中都会被共享，因此该方法仅能在基本类型的情况下运作良好）。如果一个 `prop` 不是必填的且没有恰当的默认值，那还需要它干啥？

# 避免使用状态

『避免状态』对于所有编程语言来说都是一句魔咒，React 也不例外。即使 React 还是大发慈悲的给了我们确切操作状态的手段（例如 `setState` 方法等等），但是我们还是应该避免状态的操作。

状态不是一个好东西，它让组件变得冗杂费解，即使使用了 `PureRenderMixin` 也会导致虚拟重绘。也许我们能做的最坏的事情就是在 `componentDidMount()` 或 `componentDidUpdate()` 的渲染之后立刻调用 `setState()`。在这个时间它尝试去生成的 DOM 上读取内容，也就意味着大多数 `render()` 实际上会被调用两次。如果一个子组件做同样的事情，就会产生至少四次 `render()` 调用，两次来自父组件的渲染，两次来自其内部的 `setState()` 方法。如果你组合使用组件的话就会导致属性或布局的抖动，且当你的组件结构加深时重绘会以指数方式增长。

简单的说，还是有一些情况下我们真正需要使用状态，但是仅在状态能被组件完美的封装的前提下使用。当一个父组件需要关心太多子组件上的状态的时候就说明我们的组件架构出了问题——我们的抽象一定有漏洞。这就需要重构了。

# 中心化状态

如果我们的组件里必须存在状态，且该状态不能被完全封装、需要和上层父组件通信的时候怎么办呢？在这种情况下，我们需要将状态整个从组件的结构抽取出来并将其放到一个中央状态对象上。让状态仅仅作为属性流向组件——组件本身并不管理状态。这也正是 Flux 架构的中心思想——拥有一系列的 Stores 来管理组件的状态，以及事件驱动的 Actions 来修改 Stores 和 Om。我们整个应用的状态都被存储在一个原子程序中，任何改动都会导致整个应用的重绘。实现单向的数据流成为了我们的目标。

实际上这就意味着我们的组件必须主要依赖于 `props`。组件之间的通信不会调用 `setState`，而是通过中央数据存储的顺序，或者使用事件、流、通道、回调或者直接的方法调用。中央的数据存储包含了应用状态的分层表现，并在每一次改动时重绘整个应用。单向的数据流可能看上去比较低效，但 React 中有句名句：『原生 JavaScript 之快，远超你的想象』。要记住 React 是建立在对于 DOM 进行睿智而高效的更新的基础之上。如果我们拥有纯净的组件，那我们就可以在 React 已经对原生 DOM 进行优化的基础上，通过 PureRenderMixin 对虚拟 DOM 进行优化更新。

上面的说法有点违反直觉，且不是可以从文档中明显读出来的。但是在尝试了其他的方法之后，我意识到上述方法是创建复杂应用时最有效的办法。

中心化状态和单向数据流的好处在于使我们的应用变得可被推导。数组都只会来自一个地方，应用十分简单。

下面是一个介绍的简图：
















