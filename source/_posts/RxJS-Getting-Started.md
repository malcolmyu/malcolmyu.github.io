title: RxJS 光速入门
date: 2020-01-21
categories: 技术研究
tags: [RxJS, Observable, Reactive]
toc: true

---

{% asset_img title.gif title %}

众所周知，RxJS 的学习曲线十分陡峭。尽管网上已经存在诸多入门科普文章，但当年我作为一个新人，最直接的感受还是：一看文章天花乱坠，一写代码啥也不会。在经历了一个请求重试写了一整天的噩梦后，我决定把我的心路历程记录下来，给在 RxJS 门槛前望之兴叹的同学们一点帮助。

<!--more-->

## 看个例子

我们先看一个前端语境下非常常见的例子<del>（也可能是一个较为常见的面试题，反正我经常问）</del>：

假设页面上有几个垂类 Tab，每次切换都会发送请求更新下发的数据来源。那么如何才能保证用户在快速切换 Tab 时下发数据来源不错乱？

![Tab 切换](./tab.gif)

大家的思路一般分为两种：

1. 从确保顺序入手：请求携带标识，请求返回后根据标识判断是否渲染；
2. 从降低频率入手：切换行为使用 debounce 消抖，尽量确保请求的有序；

我们先不管这两种方法是否能完美工作，先尝试把它们用代码撸一遍：

```js
tabChange = (tabId) => {
  this.tabId = tabId;
  fetch(`/api?id=${tabId}`).then((data) => {
    if (this.tabId === data.tabId) {
      render();
    }
  });
}

onTabChange = debounce(this.tabChange, 300);
```

相信对有些追求极致的同学，看到这段代码本身可能就会眉头一皱：它有一些坏味道，但有说不清楚坏在哪里。

![事情并不简单](./not-easy.png)

如果细细的品味一下，会发现它坏味道体现在以下两点：

1. **竞态危害**：第二行的 `tabId` 可能跟相隔两行的第 4 行的 `tabId` 的值并不相同，这一不确定性对于传统的同步代码一把梭的前端开发来说是非常令人头疼的。这个问题在多线程的语境下被称作“[竞态危害（Race Hazard）](https://zh.wikipedia.org/wiki/競爭危害)”，指一段代码的执行结果依赖两个异步逻辑的彼此的执行顺序。
2. 为解决上一问题所带来的**模式混用**：例如回调与 Promise 的混用、全局变量与局部变量的混用。代码如果几经流转，可能就会变成无尽 callback+Promise 地狱。

为什么不试试神奇的 RxJS 呢？代码如下：

```js
const tabSwitch$ = fromEvent(tab, 'click');

tabSwitch$.pipe(
    debounceTime(300),
    switchMap((curTab) => from(fetch(`/api/${curTab}`))),
    tap(render),
).subscribe();
```

可以看到，虽然一下子变成了天书，但至少它模式统一、行数减少，非常的 exciting！

![通宵就完事了](./tongxiao.jpeg)

 <figcaption>异步代码有害身心，我建议你直接 RxJS</figcaption>


## 怎么解决问题

先翻开这一篇天书，我们来看看 RxJS 是通过什么样的方式来解决模式混用和竞态危害两个问题的？

### Observable

这一点其实也是 RxJS 最难理解的一点：它建立了一层高 Level 的抽象，创造了 **Observable** 的概念。

有的文章在这里会引入“流（stream）”的概念，数据流跟 Observable 还是有一些区别，简单理解来说：**流仅仅是随着时间维度而增加值的数组，它是 Observable 生产的数据类型**。

![](./stream.gif)

<p><figcaption>每次鼠标点击位置 x 轴坐标产生的数据流</figcaption></p>

而 Observable 本身的概念要复杂的多。它的诞生在 Andre 的[这篇文章](https://staltz.com/javascript-getter-setter-pyramid.html)里有详尽的叙述，和 Promise 类似，是一个由**生产者决定消费者何时消费数据**的模型。

一个典型的 Observable 如下所示：

```js
const ob$ = Observable.create(observer => { // 生产者
  observer.next(1); // 触发权柄控制在数据生产者这里
  setTimeout(() => {
    observer.next(2);
    observer.complete();
  }, 1000);
});

const observer = {  // 消费者
  next: result => console.log('result: ', result),
  error: err => console.error('something wrong occurred: ' + err),
  complete: () => console.log('done'),
};

ob$.subscribe(observer); // 生产者调用消费者

// 输出结果：
// result: 1
// result: 2
// done
```

这里我们只需要记住 Observable 对象的两个特点：

1. 它跟 Promise 很类似，区别在于它可以提供多值传递，因此共有 `next`、`error`、`complete` 三种状态；
2. 它通过 `subscribe` 方法关联生产者与消费者，`subscribe` 像一条生产线的开关，只有它启动了生产者的生产才会开始；

由于各种异步行为本身就是一个在时间维度上生产数据的过程，因此他们都可以通过 RxJS 的创建类操作符转换为 Observable 对象。例如上述代码中的 `fromEvent`，就是将 tab 的点击事件转换为一个 Observable 对象：

```js
const tabSwitch$ = fromEvent(tab, 'click');
```

RxJS 有大量的创建类操作符，可以把你能想象的所有同步异步事件进行统一转换，例如请求、DOM 事件、websocket 消息、定时器等。

![创建类操作符](./create.png)

 <p><figcaption>创建类操作符</figcaption></p>

```js
/* 使用创建类操作符对异步事件输入的统一 */
import { from, fromEvent, of, interval, merge } from 'rxjs'; 
const ajax = () => fetch(url).then(v => v.json())

merge(
  from([1 ,2, 3]), // 从数组多值创建
  interval(3000),  // 定时器轮训创建
  fromEvent(document.getElementById('btn'), 'click'), // 按钮点击事件
  of({ hello: 'world' }), // 单值创建
  from(ajax()),    // 从接口请求创建
).subscribe()
```

有了 Observable 对象，等于说 RxJS 给异步代码提供了统一的范式，那么它是具体怎么解决竞态危害的呢？

### 操作符

其实上面已经提到了创建类操作符，RxJS 还提供种类繁多的大量操作符，专门用于解决竞态危害：

```js
const tabSwitch$ = fromEvent(tab, 'click');

tabSwitch$.pipe(
    debounceTime(300), // 300ms 没有变动触发一次
    switchMap((curTab) => from(fetch(`/api/${curTab}`))), // 转换为新的请求流
    tap(render),
).subscribe();
```

例如上面提到的 `switchMap`、`debounceTime` 操作符，就是专门处理高阶 Observable 的顺序问题。

![操作符一览](./operators.png)

<p><figcaption>操作符整体概览</figcaption></p>

我们在这里先不深入探究每一种操作符的用途，只需要知道操作符是专门对数据流进行处理的工具即可。有了操作符就可以确保这一点：尽管数据在获取时纷繁芜杂，但操作符可以保证我们在处理数据时有条不紊。

如果要用现实中的物体来做比喻，RxJS 就像个传送带：

![整体观感](./stream.png)

- 传送带的头部有一个转换器，它能够把输入的各种各样的原料都捏成一个叫做 Observable 的物体；
- 传送带的中段是一条生成线，生产线上有各种各样叫做操作符的机械臂，每个机械臂承担单一而确定的职责：有的机械臂会处理原料、有的会丢弃一些不需要的原料、有的会把原料聚合到一定个数才放到传送带上；
- 传送带的尾部有一个叫做 `subscribe` 的工具人，只有它开动了按钮整条传送带才会运转，杜绝浪费。

## TLDR

作为一篇光速入门文章，不能说的太多，我们在这里尝试总结一下：

1. RxJS 通过了Observable 和操作符，解决了异步编程的模式统一和时序问题，让你的异步代码规范而简洁；
2. 什么场景下应该使用 RxJS？根据上文所述，RxJS 是用于解决竞态危害问题的。如果你的业务场景中有大量的异步行为，而且它们的执行顺序错乱会导致输出的不正确，这时候就应该考虑引入 RxJS 来规范你的代码了。

## 参考

- [What does it mean to be Reactive?](https://www.youtube.com/watch?v=sTSQlYX5DU0)，核心思想。虽然口音很重，演讲风格也很浮夸😂
- [javascript-getter-setter-pyramid](https://staltz.com/javascript-getter-setter-pyramid.html)，为什么要有 Observable，编程方式的底层解构。这里是英文原版，可以看这一篇[中文解析](https://zhuanlan.zhihu.com/p/98745778)；
- [What are the differences between Promises, Observables, and Streams?](https://medium.com/javascript-in-plain-english/promise-vs-observable-vs-stream-165a310e886f)
- [RxJS - 封裝程式的藝術](https://www.bilibili.com/video/av60370503)，有些观点的中文视频讲解；
- [RXViz，RxJS 代码图形化工具](https://rxviz.com/)；

