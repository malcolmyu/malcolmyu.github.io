title: 协程与事件循环
date: 2015-11-18
categories: [技术研究]
tags: [协程, yield, event loop]
toc: true

---

最近研究了一下 es6 的生成器函数，以及传说中的 co。虽然网上关于协程、co 源码分析的文章数不胜数，但是将其与先前异步实现的事件队列结合起来说明的文章却很难寻觅。之前只知道协程是实现异步的一种方式，那其和之前的各种异步实现究竟有什么本质区别呢？本文将根据协程机制简要探讨一下**引入协程之后的新的事件循环模型**。由于笔者基础知识不够扎实，所以会先讲述一大堆协程产生的背景和原理，再进行模型变化的讲解。

<!--more-->

# 协程实现机制

## 进程与线程

说起协程就不得不提到面试宝典必备题目之一：啥是进程啥是线程他俩有啥区别？ 之前阮一峰老师写过一篇[博文](http://www.ruanyifeng.com/blog/2013/04/processes_and_threads.html)进行科普，但是文中将电力比作 CPU 以及用厕所比喻锁都不合适。反而是评论里有位叫 viho_he 的哥们说的不错，在这里简单的把他的思路整理一下：

1. 单核 CPU 无法被平行使用。为了创造『共享』CPU 的假象，搞出了一个叫做时间片的概念，给任务分配时间片进行调度的调度器后来成为操作系统的核心组件；
2. 在调度的时候，如果不对内存进行管理，那么切换时间片的时候会造成程序上下文的互相污染。但是手工管理物理地址实在是太蛋疼了，因此引入了『虚拟地址』的概念，共包含三个部分：
	- CPU 增加了内存管理单元模块，来进行虚拟地址和物理地址的转换；
	- 操作系统加入了内存管理模块，负责管理物理内存和虚拟内存；
	- 发明了一个概念叫做**进程**。进程的虚拟地址一样，经过操作系统和 MMU 映射到不同的物理地址上。
3. 深入的谈一谈进程。进程是由一大堆元素组成的一个实体，其中最基本的两个元素就是**代码**和能够被代码控制的**资源**（包括内存、I/O、文件等）；一个进程从产生到消亡，可以被操作系统**调度**。掌控资源和能够被调度是进程的两大基本特点。但是进程作为一个基本的调度单位有点不人性：假如我想一边循环输出 hello world，一边接收用户输入计算加减法，就得起俩进程，那随便写个代码都像 chrome 一样变成内存杀手了。
4. 因此诞生了**线程**的概念，线程在进程内部，处理并发的逻辑，拥有**独立的栈**，却共享线程的资源。使用线程作为 CPU 的基本调度单位显得更加有效率，但也引发各种抢占资源的问题~~，使得笔试又多了一个考题~~。

最后总结一下就是：进程掌握着独立资源，线程享受着基本调度。一个进程里跑多个线程处理并发，6 的飞起。但纯粹的内核态线程有一个问题就是性能消耗：线程切换的时候，进程需要为了管理而切换到内核态，状态转换的消耗有点严重。为此又产生了一个概念，唤做**用户态线程**。用户态线程吼啊，程序自己控制状态切换，进程不用陷入内核态，会玩儿的开发者可以按照程序的特性来选择更适合的调度算法，代码的效率飞了起来。

## 协程、子例程与生成器

那协程是干啥的咧？实际上，协程的概念产生的非常早，[Melvin Conway](https://en.wikipedia.org/wiki/Melvin_Conway) 早在 1963 年就针对编辑器的设计提出一种将『语法分析』和『词法分析』分离的方案，把 token 作为货物，将其转换为经典的[『生产者-消费者问题』](https://zh.wikipedia.org/wiki/%E7%94%9F%E4%BA%A7%E8%80%85%E6%B6%88%E8%B4%B9%E8%80%85%E9%97%AE%E9%A2%98)。编译器的控制流在词法和语法解析之间来回切换：当词法模块读入足够多的 token 时，控制流交给语法分析；当语法分析消化完所有 token 后，控制流交给词法分析。从这一概念提出的环境我们可以看出，协程的核心思想在于：**控制流的主动让出和恢复**。

这一点和上文说的用户态线程有几分相似，但是用户态线程多在语言层面实现，对于使用者还是不够开放，无法提供显示的调度方式。但是协程做到了这一点，用户可以在编码阶段通过类似 `yieldto` 原语对控制流进行调度。

说到这里可能有读者会产生疑问：『我大协程这么吊，为何提出了这么多年一直不火啊？』这就要说到当年命令式编程与函数式编程的『剑气之争』，当年命令式编程围绕着自顶向下的开发理念，将子例程调用作为唯一的控制结构。实际上，子例程就是没用使用 `yield` 的协程，大宗师 [Donald E. Knuth](https://en.wikipedia.org/wiki/Donald_Knuth) 也曾经曰过：

> 子例程是协程的一种特例。

但不进行让步和恢复的协程，终究失掉了协程的灵魂内核，不能称之为协程。直到后来出现了一个叫做**迭代器（Iterator）**的神奇的东西。迭代器的出现主要还是因为数据结构日趋复杂，以前用 `for` 循环就可以遍历的结构需要抽象出一个独立的迭代器来支持遍历，用 js 为例。迭代器的遍历会搞成下面这个样子：

```js
for (let key of Object.keys(obj)) {
    console.log(key + ": " + obj[key]);
}
```

实际上，要实现这种迭代器的语法糖，就必须引入协程的思想：主执行栈在进入循环后先让步给迭代器，迭代器取出下一个迭代元素之后再恢复主执行栈的控制流。这种迭代器的实现就是因为内置了~~葛炮~~**生成器（generator）**。生成器也是一种特殊的协程，它拥有 `yield` 原语，但是却不能指定让步的协程，**只能让步给生成器的调用者或恢复者**。由于不能多个协程跳来跳去，生成器相对主执行线程来说只是一个可暂停的玩具，它甚至都**不需要**另开新的执行栈，只需要在让步的时候保存一下上下文就好。因此我们认为生成器与主控制流的关系是不对等的，也称之为**非对称协程（semi-coroutine）**。

由此我们也知道了，为啥 es6 一下引起了这么一大坨特性啊，因为引入迭代器，就必须引入生成器，这俩就是这种不可分割的基友关系。

# 现有协程的实现

自 es6 尝试引入生成器以来，大量的协程实现尝试开始兴起，协程一时间成为风靡前端界的新名词。但这些实现中有的仅仅是实现了一个看上去很像协程的语法糖，有的却 hack 了底层代码，实现了真正的协程。这里以 TJ 大神的 [co](https://github.com/tj/co) 和 [node-fibers](https://github.com/laverdet/node-fibers) 为例，浅析这两种协程实现方式上的差异。

## CO

co 实际上是一个语法糖，它可以包裹一个生成器，然后生成器里可以使用同步的方式来编写异步代码，效果如下：

```js
var fs = require('fs');

var readFile = function (fileName){
    return new Promise(function (resolve, reject){
        fs.readFile(fileName, function(error, data){
            if (error) reject(error);
            resolve(data);
        });
    });
};

co(function* (){
    let f1 = yield readFile('/etc/fstab');
    let f2 = yield readFile('/etc/shells');
    let sum = f1.toString().length + f2.toString().length;
    console.log(sum);
});
```

在 es7 中已经推出了一个更甜的语法糖： `async/await`，实现效果如下：

```js
async function (){
    let f1 = await readFile('/etc/fstab');
    let f2 = await readFile('/etc/shells');
    let sum = f1.toString().length + f2.toString().length;
    console.log(sum);
};
```

是不是碉堡了！这段代码仿佛是在说明：我们把 `readFile` 丢到另一个协程里去了！等他搞定之后就又回到主线程上！代码可读性 6666 啊！但事实真的是这样的么？我们来看一下 co 的不考虑异常处理的精简版本实现：

```js
function co(gen){
    let def = Promise.defer();
    let iter = gen();
    
    function resolve(data) {
        // 恢复迭代器并带入promise的终值
        step(iter.next(data));
    }
    
    function step(it) {
        it.done ?
            // 迭代结束则解决co返回的promise
            def.resolve(it.value) :
            // 否则继续用解决程序解决下一个让步出来的promise
            it.value.then(resolve);
    }

    resolve();
    return def.promise;
}
```

从 co 的代码实现可以看出，实际上 co 只是进行了对生成器让步、恢复的控制，把让步出来的 promise 对象求取终值，之后恢复给生成器——这都没有多个执行栈，并没有什么协程么！但是有观众会指出：这不是用了生成器么，生成器就是非对称协程，所以它就是协程！好的，我们再来捋一捋：

1. 协程在诞生之时，只有一个 Ghost，叫做**主动让步和恢复控制流**，协程因之而生；
2. 后来在实现上，发现可以采用可控用户态线程的方式来实现，因此这种线程成为了协程的一个 shell。
3. 后来又发现，生成器也可以实现一部分主动让步和恢复的功能，但是弱了一点，我们也称生成器为协程的一个弱弱的 shell。
4. 所以我们说起协程，实际上说的是它的 Ghost，只要能主动让步和恢复，就可以叫做协程；但协程的实现方式有多种，有的有独立栈，有的没有独立栈，他们都只是协程的壳，不要在意这些细节，嗯。

好的，因此 TJ 大神叫它 co，也还是有一定道理的，尽管可能产生一些奇怪的误导……我们来看一下，引入了 co 之后，js 原有的事件循环产生了什么改变呢？

【回头补个图】

好吧，实际上并没有什么改变。因为 promise 本身的实现机制还是回调，所以在 `then` 的时候就把回调扔给 webAPI 了，等到合适的时机又扔回给事件队列。事件队列中的代码需要等到主栈清空的时候再运行，这时候执行了 `iter.next` 来恢复生成器——而生成器是没有独立栈的，只有一份保存的上下文；因此只是把生产器的上下文再次加载到栈顶，然后沿着恢复的点继续执行而已。引入生成器之后，事件循环的一切都木有改变！

## Node-fibers

看完了生成器的实现，我们再来看下真·协程的效果。这里以 hack 了 node.js 线程的 `node-fibers` 为例，看一下真·协程与生产器的区别在何处。

首先，`node-fibers` 本身仅仅是实现了创造协程的功能以及一些原语，本身并没有类似 co 的异步转同步的语法糖，我们采用相似的方式来包裹一个，为了区别，就叫它 ceo 吧（什么鬼）：

```js
let Fiber = require('fibers');

function ceo(cb){
    let def = Promise.defer();
    // 注意这里传入的是回调函数
    let fiber = new Fiber(cb);
    
    function resolve(data) {
        // 恢复迭代器并带入promise的终值
        step(fiber.run(data));
    }
    
    function step(it) {
        !fiber.started ?
            // 迭代结束则解决co返回的promise
            def.resolve(it.value) :
            // 否则继续用解决程序解决下一个让步出来的promise
            it.then(resolve);
    }

    resolve();
    return def.promise;
}

ceo(() => {
    let f1 = Fiber.yield(readFile('/etc/fstab'));
    let f2 = Fiber.yield(readFile('/etc/shells'));
    let sum = f1.toString().length + f2.toString().length;
    console.log(sum);
});
```

观众老爷会说了，这不是差不多么！好像最大的区别就是生成器变成了回调函数，只是少了一个 `*` 嘛。错！这就是区别！这里最关键的一点在于：没有了生成器，我们可以在任意一层函数里进行让步，这里使用 `ceo` 包裹的这个回调，是一个真正**独立的执行栈**。在真·协程里，我们可以搞出这样的代码：

```js
ceo(() => {
    let foo1 = a => {
        console.log('read from file1');
        let ret = Fiber.yield(a);
        return ret;
    };
    let foo2 = b => {
        console.log('read from file2');
        let ret = Fiber.yield(b);
        return ret;
    };
    
    let getSum = () => {
        let f1 = foo1(readFile('/etc/fstab'));
        let f2 = foo2(readFile('/etc/shells'));
        return f1.toString().length + f2.toString().length;
    };
    
    let sum = getSum();
    
    console.log(sum);
});
```

通过这个代码可以发现，在第一次让步被恢复的时候，恢复的是一坨执行栈！从栈顶到栈底依次为：`foo1` => `getSum` => `ceo` 里的匿名函数；而使用生成器，我们就无法写出这样的程序，因为 `yield` 原语只能在生产器内部使用——无论什么时候被恢复，都是简单的恢复在生成器内部，所以说生成器是不用开新栈滴。

那么问题就来了，使用了真·协程之后，原先的事件循环模型是否会发生改变呢？是不是主执行栈调用协程的时候，协程就会在自己的栈里跑，而主栈就排空了可以执行异步代码呢？我们来看下面这个例子：

```js
"use strict";
var Fiber = require('fibers');

let syncTask = () => {
    var now = +new Date;
    while (new Date - now < 1000) {}
    console.log('SyncTask Loaded!');
};

let asyncTask = () => {
    setTimeout(() => {
        console.log('AsyncTask Loaded!');
    });
};

var fiber = Fiber(() => {
    syncTask();
    Fiber.yield();
});

function mainThread() {
    asyncTask();
    fiber.run();
}

mainThread();

// 输出：
// SyncTask Loaded!
// AsyncTask Loaded!
```

我们在主线程执行的时候抛出了一个异步方法，之后在协程里用冗长的同步代码阻塞它，这里我们可以清楚的看到：阻塞任何一个执行中的协程都会阻塞掉主线程！也就是说，即使加入了协程，js 还是可以被认为是单线程的，因为**同一时刻势必只有一个协程在运行**，在协程运行和恢复期间，js 会将当前栈保存，然后用对应协程的栈来填充主的执行栈。**只有所有协程都被挂起或运行结束，才能继续运行异步代码**。

因此，真·协程的引入对事件循环还是造成了一定的影响，可怜的异步代码要等的更久了。


