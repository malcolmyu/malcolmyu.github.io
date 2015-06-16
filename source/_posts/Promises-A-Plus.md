title: Promise A+ 规范
date: 2015-06-12
categories: [技术研究]
tags: [翻译, Javascript, Promise]

---
英文原文：[Promise/A+](https://promisesaplus.com/)
图灵译文：[【翻译】Promises/A+规范](http://www.ituring.com.cn/article/66566)

**译者序：**一年前曾译过 Promise/A+ 规范，适时完全不懂 Promise 的思想，虽说翻译，也是学习。拙译一年之间点击数千，竟成为谷歌搜索的头条。昨日得知，不禁赧然。如今再审，觉得旧文佶屈聱牙，错漏百出，遂重译之，以飨读者。

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

- **终值（eventual value）**：所谓终值，指的是 promise 被解决时传递的值，由于 promise 有**一次性**的特征，因此当这个值被传递时，标志着 promise 等待态的结束，故称之终值。
- **据因（reason）**：也就是拒绝原因，按照规则来说，链式拒绝仅仅拒绝的是异常，之后不论链前解决、拒绝与否，后者均被解决，除非手工链式传递 promise。

Promise 表示一个异步操作的最终结果，与之进行交互的方式主要是 `then` 方法，该方法注册了两个回调函数，用于接收 Promise 的终值或 Promise 的据因。