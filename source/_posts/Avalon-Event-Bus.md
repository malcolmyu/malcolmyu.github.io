title: avalon 事件总线与依赖调度系统
date: 2015-04-19
categories: 源码分析
tags: [avalon, Javascript]

---
在工作中经常使用到[司徒](http://www.cnblogs.com/rubylouvre/)的 [avalon](https://github.com/RubyLouvre/avalon) ，由于坑点太多，有时需要经常查阅其源码实现。而 avalon 由于方兴未艾，网上对其进行源码解析的文章并不多，查了半天也就只有这篇 [MVVM 大比拼](http://www.cnblogs.com/sskyy/p/3679572.html)，以及这篇 [avalon 源码分析](http://www.cnblogs.com/aaronjs/archive/2013/06/16/3138631.html)。个人认为这两篇文章写得都并不算好，其一是成文较早，研究的源码还是 1.2.5 版本，而目前的新版本已经到了 1.4+，比之前不知道高到哪里去。其二是大比拼一文作者阅码无数，心中早已无码，写分析只观其大要，似乎在和原作者谈笑风生；而后者的分析仿佛又只是对源码的粗略通读，也没怎么经过实践，有些图样图森破。因此自己决定安下心来写点源码分析。

<!--more-->

本文源起自同事的一个疑问，“*可否使用事件中的 `$fire` 来改变一个普通双绑属性的值？*”这样问也是有它的理由：因为在视图模型中，可以用 `$watch` 来监听属性的变化；而在事件总线中，可以用 `$fire` 来触发 `$watch` 的回调，那似乎用 `$fire` 来改变双绑属性的值也变得可以接受。但实际测试中，发现这样操作是不起作用的。如：

```javascript
var demo = avalon.define({
    $id: 'demo',
    a: 1
});

demo.$fire('a', 2);
// nothing output
```

这是为什么呢？起初我认为事件总线与依赖调度实际上是绑在一起的，也就是说双绑属性上的事件——如值改变后通知视图——也是通过事件总线来实现的；后来想到 avalon 师承 knockout，而事件是 angular 中才有的概念，可能是后来单独实现的也说不定。带着这样的疑问，我翻开了源码：

```javascript
var EventBus = {
    $watch: function (type, callback) {
        if (typeof callback === "function") {
            var callbacks = this.$events[type]
            if (callbacks) {
                callbacks.push(callback)
            } else {
                this.$events[type] = [callback]
            }
        } else { //重新开始监听此VM的第一重简单属性的变动
            this.$events = this.$watch.backup
        }
        return this
    },
    // 省略了 unwatch 事件
    $fire: function (type) {
        // 这里对 $fire 处理 dom 层级关系的部分做了大量省略
        var callbacks = events[type] || []
        var all = events.$all || []
        for (i = 0; callback = callbacks[i++]; ) {
            if (isFunction(callback))
                callback.apply(this, args)
        }
        for (i = 0; callback = all[i++]; ) {
            if (isFunction(callback))
                callback.apply(this, arguments)
        }
    }
}
```

我们可以看到，在源码中出现最多的东西就是这个 `$events` 。上面两段代码的主要意思就是说，在 `$watch` 的时候，查看一下当前层级的 `$event` 中是否有对应名称的事件队列，在确保 `$watch` 参数是函数的前提下将回调添加到事件队列中；而 `$fire` 的时候需要遍历 `$event` 对应的事件队列，取出是函数的部分执行它。

这里就产生了一个疑问：我之前以为依赖调度的事件也是存放在 `$events` 队列中的，比如上文的视图模型 demo，加入页面上有写了双绑的 a，如 `<span ms-if="a"></span>` 之类，那么 a 的依赖调度就是储存在 `demo.$events` 中的。既然事件总线的回调也是储存在 `$events` 中，那二者的实现方式又有何区别？

继续翻到依赖调度系统的代码：

```javascript
function registerSubscriber(data) {
    Registry[expose] = data
    // 暴光此函数,方便collectSubscribers收集
    var fn = data.evaluator
    if (fn) { //如果是求值函数
        var c = ronduplex.test(data.type) ? data : fn.apply(0, data.args)
        data.handler(c, data.element, data)
    }
}

function collectSubscribers(list) {
    var data = Registry[expose]
    // 收集依赖于这个访问器的订阅者
    if (list && data && avalon.Array.ensure(list, data) && data.element) {
        //只有数组不存在此元素才push进去
        addSubscribers(data, list)
    }
}

function notifySubscribers(list) {
    // 通知依赖于这个访问器的订阅者更新自身
    if (list && list.length) {
        // 省略了定时检查移除订阅和监控数组的代码
        // ...
        for (var i = list.length, fn; fn = list[--i]; ) {
            var fun = fn.evaluator || noop
            fn.handler(fun.apply(0, fn.args || []), el, fn)
        }
    }
}
```

过一遍依赖调度的代码我们会发现它的三个主要的功能：

- `registerSubscriber` 用于给依赖调度系统曝光一个依赖。该方法是在智能代理 `parseExprProxy` 中调用的，而这一方法是承自 avalon 的两大系统之一——**扫描系统**——在扫描绑定的时候执行的，而参数 `data` 就是扫描绑定生成的对象。
- `collectSubscribers` 用于收集访问器的订阅者。这个方法是在**访问器（accessor）**中进行收集的，当一个属性被 get 的时候进行收集。
- `notifySubscribers` 通知依赖于这个访问器的订阅者更新自身。这个方法也是在访问器中进行收集的，对应了属性 set 的情况。

由于访问器是在模型工厂中被生成的，也就是说，**依赖调度系统是链接 avalon 两大系统——扫描系统（scan）和模型工厂（define）的重要模块**，是将扫描页面产生的绑定与处理 viewModel 生成的属性结合起来的重要手段。

这是依赖调度系统的功能，这一枝我们按下不表，有机会会开篇详述。我们关注的问题还是在于：为何依赖调度与事件总线都是通过 `$events` 来进行处理，二者却花开两朵呢？

对上面两个重要的方法进行再一次简化，得到如下的代码：

```javascript
function registerSubscriber(data) {
    Registry[expose] = data
    data.evaluator.apply(0, data.args)
}
function collectSubscribers(list) {
    var data = Registry[expose]
    avalon.Array.ensure(list, data)
}
```

在 `registerSubscriber` 中，执行了一步求值函数。这时候就会触发访问器的 get 方法，继而触发 `collectSubscribers`；而后者的参数就是对应的 `$events`，在这里将刚才曝光的 `data` 添加到 `$events` 队列中。这里的 `data` 不是回调函数，而是扫描绑定生成的数据，包含 DOM 节点、求值函数、vm等等。

看到这里我们就明白了——尽管依赖调度和事件总线的内容都寄存在 `$events` 中，但一个存的是对象，一个存的是回调函数，所以两者实际上走的是不同的处理路线。

观察一下访问器的代码，证实了我们这一观点：

```javascript
// 这里只探究简单属性，故而对其进行了简化
if (arguments.length) { // 有参数表示 set
    if (!isEqual(oldValue, newValue)) {
        $model[name] = newValue
        notifySubscribers($events[name]) // 同步视图
        safeFire($vmodel, name, newValue, oldValue) // 触发$watch回调
    }
} else { // 无参数表示 get
    collectSubscribers($events[name]) // 收集视图函数
    return accessor.svmodel || oldValue
}
```

在访问器中，需要分别进行同步视图与触发 `$watch` 回调的操作，分别处理了两个系统的事务。回到之前同事的问题：

```javascript
var demo = avalon.define({
    $id: 'demo',
    a: 1
});
demo.$watch('a', function(a) {
    console.log(a);
});
demo.$fire('a', 2); // 输出2
demo.a = 3;         // 输出3
```

由于依赖调度与事件总线是两个不同的系统，因此 `$fire` 只能触发事件总线中挂载在 a 上的回调，而无法触发访问器；而由于访问器中在 set 时对两个系统都进行了触发，因此可以即同步视图，又能触发事件回调，所以可以使用 `$watch` 来监听属性值的变化。

