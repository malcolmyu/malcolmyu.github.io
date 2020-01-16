title: 光速理解 Vue 源码
date: 2020-01-07
categories: 源码分析
tags: [Vue, Javascript, MVVM]

---

熟悉 MVVM 框架原理的同学在阅读 Vue 源码时有一种翻抽屉的感觉，“擦，原来我之前那本书放在这里”。

按照 MobX 的世界观，Observable 框架主要就是两种要素的结合：Observable（可观察对象）、Derivation（衍生函数），再要多说一点就是还有一个以及集两者于一身的 Computed Value（计算属性）。这些概念在 Vue 里面都有很好的对应，Vue 所有跟 Observable 相关的代码都集中在 observer 文件夹内。

