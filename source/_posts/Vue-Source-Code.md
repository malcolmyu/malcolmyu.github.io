title: xxx 读懂 Vue 源码
date: 2019-07-25
categories: [技术研究]
tags: [Vue]
toc: true



一些思考

- 逐渐的你的工作由一个人怎么写好代码，变成了如何让一群人写好代码、少写代码甚至不写代码；
- 如何让一群人投入产出比增加是个值得长期思考的问题
  - 抽象业务模式、提供业务组件；
  - 数据流思考、管理；
  - 配置页面生成；



开源项目探究：

- 分支、版本、文档、合作者关系；
- 对源码理念的探索
- 对应用模式的探索



今天的目的是发现整理 vue 的全部功能，并对比找到 mpvue 差异点；

- 探究 mpvue 的 slot 到底怎么回事
- 探究 cover-view 到底怎么回事
- 看下 vuex 与 rxjs 怎么结合

## 基础阅读

### 基础

- data 都要求 function 了（源码理解

### 组件

vue 组件使用特别不友好，因为它有 mutable 的属性，导致组件的 properties 很尴尬，组件和页面模板是不同的，有点奇怪；

#### slot

1. slot 插槽设计有点铁憨憨，还是 children + props 更好理解一些

### 过渡动画

### 可复用性 & 组合

#### mixin

Vue 依然支持 mixin，混入作为最容易理解的 AOP 逻辑混用方法，一般都是首选策略。但我们不禁要问，为啥 React 已经从 HOC 到 Hook 了？（新的 Vue 应该也有 Hook）

mixin 的危害在于隐式依赖（无命名、无引用直接使用方法）致使使用者迷茫，引用一旦深入时难以溯源；命名冲突；改动时方案易滚雪球。

Vue3.0 显然准备引入 hooks（详见 [RFC](https://github.com/vuejs/rfcs/issues/63)，还有[这篇](https://github.com/vuejs/rfcs/pull/42/files?short_path=e560e5d#diff-e560e5df9f0620351fb72581fed9ba5e)）

#### jsx

vue 的 jsx 是真的铁憨憨，不支持双绑啊尼玛死，需要使用 babel 转义

## mpvue 基本原理

直接将小程序作为一个 platform 编译输出，编译完后给每个 Page 加一段 new Vue，每个 Page 一个实例，共用 _data 和 methods；

通过 $c 处理 child，\$k 处理 component，下标是留给组件和 child  的

- 对于 data 来说，通过 Proxy 绑到 Page.data.$root 上；
- 对于 methods 来说，所有 bind 统统绑在 handleProxy 上，一个统一的中介者，事件触发时走 $updateDataToMP 进行触发；
  - 找事件的时候直接拉出触发点上的 $rootVM，找到 VNode，然后一个一个找，找到对应的回调直接执行（这里感觉有点沙雕，为啥不直接在 vue 实例的 methods 里面捞）；
  - 执行时触发 set，然后走 patch、$updateDataToMP，全量 data format、diff、update；

鉴于 mpvue 这里的处理方式，它单页面公用一个实例的问题不太好处理（里面有组件、child、method 和 data、computed、watch 等）

