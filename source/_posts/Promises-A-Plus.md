title: Promise A+ 规范
date: 2015-06-12
categories: [技术研究]
tags: [翻译, Javascript, Promise]
toc: true

---
英文原文：[Promise/A+](https://promisesaplus.com/)
图灵译文：[【翻译】Promises/A+规范](http://www.ituring.com.cn/article/66566)

**译者序：**一年前曾译过 Promise/A+ 规范，适时完全不懂 Promise 的思想，纯粹将翻译的过程当作学习，旧文译下来诘屈聱牙，读起来十分不顺畅。谁知这样一篇拙译，一年之间竟然点击数千，成为谷歌搜索的头条。今日在理解之后重译此规范，以飨读者。

<!--more-->

**一个开放、健全且通用的 JavaScript Promise 标准。由开发者制定，供开发者参考。**

---

# 术语参考

- **Promise**：promise 是一个拥有 `then` 方法的对象或函数，其行为符合本规范；
- **thenable**：是一个定义了 `then` 方法的对象或函数；
- **值（value）**：指任何 JavaScript 的合法值（包括 `undefined` , thenable 和 promise）；
- **异常（exception）**：是使用 `throw` 语句抛出的一个值。
- **据因（reason）**：表示一个 promise 的拒绝原因。

译文术语：

- **解决（fulfill）**：指一个 promise 成功时进行的一系列操作，如状态的改变、回调的执行。虽然规范中用 `fulfill` 来表示解决，但在后世的 promise 实现多以 `resolve` 来指代之。
- **拒绝（reject）**：指一个 promise 失败时进行的一系列操作。
- **终值（eventual value）**：所谓终值，指的是 promise 被**解决**时传递给解决回调的值，由于 promise 有**一次性**的特征，因此当这个值被传递时，标志着 promise 等待态的结束，故称之终值。
- **据因（reason）**：也就是拒绝原因，指在 promise 被**拒绝**时传递给拒绝回调的值。

Promise 表示一个异步操作的最终结果，与之进行交互的方式主要是 `then` 方法，该方法注册了两个回调函数，用于接收 promise 的终值或本 promise 不能执行的原因。

本规范详细列出了 `then` 方法的执行过程，所有遵循 Promises/A+ 规范实现的 promise 均可以本标准作为参照基础来实施 `then` 方法。因而本规范是十分稳定的。尽管 Promise/A+ 组织有时可能会修订本规范，但主要是为了处理一些特殊的边界情况，且这些改动都是微小且向下兼容的。如果我们要进行大规模不兼容的更新，我们一定会在事先进行谨慎地考虑、详尽的探讨和严格的测试。

从历史上说，本规范实际上是把之前 [Promise/A 规范](http://wiki.commonjs.org/wiki/Promises/A) 中的建议明确成为了行为标准：我们一方面扩展了原有规范约定俗成的行为，一方面删减了原规范的一些特例情况和有问题的部分。

最后，核心的 Promises/A+ 规范不设计如何创建、解决和拒绝 promise，而是专注于提供一个通用的 `then` 方法。上述对于 promises 的操作方法将来在其他规范中可能会提及。

# 术语
---

参见上文术语参考。

# 要求
---

## Promise 的状态

一个 Promise 的当前状态必须为以下三种状态中的一种：**等待态（Pending）**、**执行态（Fulfilled）**和**拒绝态（Rejected）**。

### 等待态（Pending）
处于等待态时，promise 需满足以下条件：

- 可以迁移至执行态或拒绝态

### 执行态（Fulfilled）
处于执行态时，promise 需满足以下条件：

- 不能迁移至其他任何状态
- 必须拥有一个**不可变**的值

### 拒绝态（Rejected）
处于拒绝态时，promise 需满足以下条件：

- 不能迁移至其他任何状态
- 必须拥有一个**不可变**的据因

这里的不可变指的是恒等（即可用 `===` 判断相等），而不是意味着更深层次的不可变。