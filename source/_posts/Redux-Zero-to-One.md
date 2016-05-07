title: Redux 从无到有
date: 2016-04-16
categories: 技术研究
tags: [Javascript, Redux]
toc: true

---

![redux 图示](http://7xinjg.com1.z0.glb.clouddn.com/redux-and-angularjs-redux-arch.jpg)

redux 自诞生之日就被冠以『学习曲线陡峭』之名，让学习者望而却步。然而由于工作需要，最近还是硬着头皮把 redux 的这一套研究了一番，发现传说中陡峭的学习曲线好像也没有那么夸张，我们甚至可以从零开始实现一个基础版本的 redux。

<!--more-->

# 基础版本

设想一下，我们每天沉浸在复杂项目繁琐的开发之中，每天都在处理页面上的各种数据流动。比如我们做一个成绩单项目，可以在页面中发送 ajax 请求，然后用请求之后的数据渲染页面上的分数；亦或我们的页面使用了双向绑定，在用户输入发生变化的时候，页面分数就会自动发生改变。由于数据的来源多种多样，我们觉得数据的管理有必要单独抽取出来进行统一处理。

于是我们果断手工撸了以下代码：

<p data-height="266" data-theme-id="0" data-slug-hash="ONZbeK" data-default-tab="result" data-user="malcolmyu" data-embed-version="2" class="codepen">See the Pen <a href="http://codepen.io/malcolmyu/pen/ONZbeK/">Base Redux</a> by Malcolm (<a href="http://codepen.io/malcolmyu">@malcolmyu</a>) on <a href="http://codepen.io">CodePen</a>.</p>
<script async src="//assets.codepen.io/assets/embed/ei.js"></script>

上面就是 redux 的基本实现，~~好了本次的讲解结束了，谢谢大家~~ 在这个例子中，我们进行了以下尝试：

1. **抽象数据层。**我们将数据部分抽取出来进行统一管理，在页面层面我们使用 action 发送前端存在且受用户影响的数据；数据怎么处理，交给调度器 dispacher 解决；最终数据流向 store 对象（这里就是组件的 `state`）进行存储，并最终展现在页面某模块上。
1. **固定数据流。**我们保证数据流向永远是：action → dispacher → store → view，明确且清晰。

事实上，上面两点就是 flux 体系带给我们的；而 redux 作为 flux 的一种实现，尽管略显不同，但他还是很好的体现了 flux 体系的本质。

# 构建 store

然而上面的例子还是依赖于 react，我们需要将手工实现的粗糙的 redux 模块独立出来，使用 flux 里的发布/订阅概念来完成这一行为。也就是说，我们要构建出一个独立的 store。它拥有保存状态、订阅事件、发布事件等一系列功能，脱离了 react 体系也能独立工作。

对于这个函数，需要传入两个参数，第一个是具体的调度方法 `dispatcher`，第二个是初始的状态 `initialState`。然后它使用闭包来保持状态，对外提供三个方法，分别是获取状态的方法 `getState()`、发布方法 `publish()` 以及订阅方法 `subscribe()`。

```js
function createStore(dispatcher, initialState) {
    var currentState = initialState;
    var listeners = [];
    
    function getState() {
        return currentState;
    }
    
    // 订阅方法，注册回调，当状态发生改变时可以及时响应
    function subscribe() {}
    
    // 发布方法，由用户调用，发布对状态进行的改变
    function publish() {}
}
```

我们分析一下订阅方法，实际上其功能有点类似于 `$.Callback`，就是注册一堆回调函数，然后在发布的时候依次执行；而发布方法无非就是执行用户传入的调度函数，然后出发一下之前由订阅方法注册的回调。我们为这两个方法加入代码：

```js
function subscribe(listener) {
    listeners.push(listener);
}

function publish(action) {
    currentState = dispatcher(currentState, action);
    listeners.forEach(listener => listener());
    return action;
}
```

然后我们对上面的例子使用 `createStore` 方法进行一个升级：

<p data-height="266" data-theme-id="0" data-slug-hash="JXvrKE" data-default-tab="result" data-user="malcolmyu" data-embed-version="2" class="codepen">See the Pen <a href="http://codepen.io/malcolmyu/pen/JXvrKE/">createStore</a> by Malcolm (<a href="http://codepen.io/malcolmyu">@malcolmyu</a>) on <a href="http://codepen.io">CodePen</a>.</p>
<script async src="//assets.codepen.io/assets/embed/ei.js"></script>

如此神奇，感觉离 redux 更近了一步呢！

# 复杂数据结构

然而上面的例子仅仅是一个简单的数据结构，不足以体现出 redux 的伟大之处。假如我们增加一个 action，然后给 state 增加一个叫 `students` 的数据表示学生人数，这时候应该怎么处理呢？

<p data-height="266" data-theme-id="0" data-slug-hash="EKLwvN" data-default-tab="result" data-user="malcolmyu" data-embed-version="2" class="codepen">See the Pen <a href="http://codepen.io/malcolmyu/pen/EKLwvN/">Many Actions</a> by Malcolm (<a href="http://codepen.io/malcolmyu">@malcolmyu</a>) on <a href="http://codepen.io">CodePen</a>.</p>
<script async src="//assets.codepen.io/assets/embed/ei.js"></script>

聪明如你可能很快就会想到，我们给 `action` 加一个 `type` 字段来处理嘛！肥肠好，但是设想一下，如果我们的 store 非常复杂，那用户的 `dispatcher` 和 `state` 岂不是成了一大坨？如果有两个 action 具体行为差不多，但是有细微的不同，那 `dispatcher` 怎么复用呢？这个时候我们就遇到了 flux 体系结构的第一个问题：**复杂的 store 难以管理，可复用性差**。

这时，天纵英才的丹爷提出了一个概念：**使用 reducer**！

直到这里我们才开始脱离 flux，进入 redux 的思维领域。我们知道 js 里有一个函数叫 `reducer`，这是一个神奇的函数，经常会有意想不到的妙用。如不用 `reverse` 如何反转一个数组？用 `reduce` 就可以轻松完成：

```js
arr.reduce((a, b) => [b].concat(a), []);
```

说白了，`reduce` 干的事情就是用一个初始值遍历一个数组，然后获取最终结果。这其实与我们现在的 `dispatcher` 很像：我们使用一个最初的状态遍历所有的 `dispatcher`，然后获取最终的状态。所以我们干脆直接将这种调度器称为 `reducer`。

你说这只是改了个名字嘛，好像也没什么了不起。然而并不是这样，改成 `reducer` 之后，调度器就有了**可嵌套（nested）**功能。我们之前的调度器用的仅仅是 `switch...case` 语句，而到了 `reducer` 这里，我们使用了一系列函数对象。可以说，如果我们用**状态树**来形容 store 中存储的结构的话，那么 reducer 就是树的叶子节点。

![combined-redux](http://7xinjg.com1.z0.glb.clouddn.com/combined-redux.png)

我们可以看到，一个普通的状态树实际上是由多个连接过的 reducer 构成，其中连接的过程也会给状态树添加新的属性。比如我们初始的状态是空对象 `{}`，然后我们拥有 3 个 reducer：`r1` `r2` `r3`，假如我们想让前两个 reducer 结合起来，和 `r3` 并列，我们可以使用某种手段将 `r1` `r2` 组合成 `c1`，然后将 `c1` 和 `r3` 组合成最终的状态。那么最终的状态就是如下所示：

```js
{
    c1: {
        r1: initialStateOfR1,
        r2: initialStateOfR2
    },
    r3: initialStateOfR3
}
```

我们可以认为最终中间件的结合手段是**递归**的：结合之后的 `combined reducer` 只是深度更深了一层，实际上与 `reducer` 效用一致。我们来手动实现一个结合函数 `combineReducer`：

```js
function combineReducer(reducers) {
    let reducerKeys = Object.keys(reducers);
    return function combine(state = {}, action) {
        let nextState = {};
        reducerKeys.forEach(key => {
            let reducer = reducers[key];
            let previous = state[key];
            let next = reducer(previous, action);
            
            nextState[key] = next;
        });
        return nextState;
    }
}
```

我们把这个函数添加到之前的例子上，然后增加一个 reducer，使得这个例子变得复杂：

<p data-height="266" data-theme-id="0" data-slug-hash="VadBdq" data-default-tab="result" data-user="malcolmyu" data-embed-version="2" class="codepen">See the Pen <a href="http://codepen.io/malcolmyu/pen/VadBdq/">combineReducer</a> by Malcolm (<a href="http://codepen.io/malcolmyu">@malcolmyu</a>) on <a href="http://codepen.io">CodePen</a>.</p>
<script async src="//assets.codepen.io/assets/embed/ei.js"></script>

碉堡了，我们的国产 `redux` 已经有递归 `reducer` 了！还有什么能难倒我们？这时产品走了过来：『旁友，我们这儿有个变更，请把文科、理科分数改成异步获取的。嗯，我不管怎么实现，今天能搞定吗？』

虽然很不情愿，但是我们还是哼哧哼哧的改了起来。js 的哪一种异步我没有写过？分分钟就来了个异步写法：

```js
showScienceScore() {
    // 手动搞成异步的
    let score = new Promise(resolve =>
        setTimeout(() => resolve([90, 86]), 100)
    );
    score.then(data => {
        let [math, physics] = data;
    	   this.store.publish(getScienceScore(math, physics))
    });
}
```

【详细写一下，改 publish 为 dispatch】

是的，这样确实可以搞定，可是岂不是意味着两个分数就得写两个 promise，100 个异步请求岂不是得写 100 个 promise？之前我们或许确实是这样搞的，那主要是因为我们后续的操作并不统一；但在 `redux` 里完全不同，我们进行状态更改的手段都是 `dispatch` 函数，进行数据处理的都是 `reducer`，所以我们可以直接对 `dispatch` 进行修改以达到异步目的！而这个修改手段就是所谓的**中间件**。

# 使用中间件

## 什么是中间件

中间件实际上就是之前一种 Monkey Patch 的写法，比如我们想给一个作为属性的函数加上打印日志的功能，又不想改变函数名称，可以这么搞：

```js
var obj = { 
    foo: function() {
        console.log('foo!')
    }
};

var originFoo = obj.foo;
obj.foo = function(...args) {
    console.log('begin');
    originFoo.apply(this, args);
    console.log('end');
}
obj.foo();
// begin
// foo!
// end
```

不过这么写有点毛，每次还得搞一个变量。有了上面 `combineReducer` 的函数式编程经验，我们把这种 Monkey Patch 直接搞成函数嵌套，用闭包来保存函数，就感觉~~有逼格~~优雅多了：

```js
function patchLog(obj) {
    return function(...args) {
        console.log('begin');
        obj.func.apply(obj, args);
        console.log('end');
    }
}

obj.foo = patchLog(obj);
```

但是，上面的例子考虑很多处理函数参数和作用域的问题，然而对于我们的 `dispatch` 函数来说，它的参数和作用域都是固定的，所以写起来会更简单一点。

```js
function patchLog(store) {
    // 使用闭包保护之前的 dispatch 函数
    let dispatch = store.dispatch;
    
    return function(action) {
        console.log('begin');
        dispatch(action);
        console.log('end');
    }
}

store.dispatch = patchLog(store);
```

## next 登场

然而上文的 `let dispatch = store.dispatch` 感觉还是不够友好，使用者每次还得记着把 `dispatch` 拿出来存一下，不然就变成无限循环调用了。既然都函数式编程了，果断再搞一层，用传参数的方式把 `dispatch` 传进来，尽然都中间件了，就叫它 `next` 吧：

```js
function patchLog(store) {
    return function(next) {
        return function(action) {
            console.log('begin');
            next(action);
            console.log('end');
        }
    }
}
// 用上 ES6 的箭头函数，逼格在我体内流动
const patchLog = store => next => action => {
    console.log('begin');
    next(action);
    console.log('end');
}

// 再写一个中间件
const patchTimer = store => next => action => {
    console.log('begin time');
    next(action);
    console.log('end time');
}

store.dispatch = patchLog(store)(store.dispatch);
store.dispatch = patchTimer(store)(store.dispatch);
```

这里我们可以看到，每一层的 `next` 都表示后续包裹的 `dispatch` 函数，调用了就表示继续执行下面的中间键。

啥，你说为什么还要传 `store`？好问题，有两个原因：

1. `store` 又不止 `dispatch` 一个属性，传了它以后还可以用 `getState` 看看当前存了什么状态；
2. 提供了一种**阻断中间件的方式**：这个很重要，现在不着急说，下文我会表态的。

## 问题

但是这么整解决不了两个问题：

1. 感觉有点 low 啊，每次都一遍一遍的写参数，我感觉很难受；
2. 用户在中间件里调用 `store.dispatch`，不是又死循环了么？

第一个好办，刚才那样写太恶心，那我们这样写，就好看多了：

```js
applyMiddleware(patchLog, patchTimer)(store);
```

这就意味着 `applyMiddleware` 要把整个 patch 串联起来，然后把最后套了无数层的函数赋值给 `store.dispatch`。来我们手动撸一下这个实现：

```js
function applyMiddleware(...middlewares) {
    return (store) => {
        // 注意中间件的执行顺序，是 reduceRight
        store.dispatch = middlewares.reduceRight((next, middleware) => {
            return middleware(store)(next);
        }, store.dispatch);
    }
}
```

我们试一下输出结果：

```js
// 先写一个测试 store
var store = { dispatch: (action) => console.log(action) }
applyMiddleware(patchLog, patchTimer)(store);
store.dispatch('hello');
// 输出如下：
// begin
// begin time
// hello
// end time
// end
```

吼啊，这个问题就这么解决了，可是很明显第二个坑还是没填上，这个循环调用怎么破解呢？

这时丹爷又发话了：不用改 `store`，我们返回一个**新的** `store`。

当你调用的是一个新的 `dispatch`，那么在中间件里怎么操作 `store.dispatch` 其实都没有关系了。这样一方面保护了原来的 `dispatch` 方法不被破坏，同时还提供了上文所说的**阻断功能**：这时如果在中间件里调用 `store.dispatch` 而非 `next`，就可以直接阻断中间件的运行。代码实现如下：

```js
function applyMiddleware(...middlewares) {
    return (store) => {
        let dispatch = middlewares.reduceRight((next, middleware) => {
            return middleware(store)(next);
        }, store.dispatch);
        return { ...store, dispatch };
    }
}
```

我们接着来实现一个异步中间件，并把两个分数改成异步的：

<p data-height="266" data-theme-id="0" data-slug-hash="pyQwLB" data-default-tab="result" data-user="malcolmyu" data-embed-version="2" class="codepen">See the Pen <a href="http://codepen.io/malcolmyu/pen/pyQwLB/">applyMiddleware</a> by Malcolm (<a href="http://codepen.io/malcolmyu">@malcolmyu</a>) on <a href="http://codepen.io">CodePen</a>.</p>
<script async src="//assets.codepen.io/assets/embed/ei.js"></script>

额妹子嘤！我们加入了中间件之后，完全没有改动除了 action 和创建 store 以外的任何代码，就完成了各种异步操作，而完全不用在业务里写各种蛋疼的逻辑了。把特殊逻辑交给可插拔式的中间件，而不必改动业务代码，这种是中间件的强大之处。

# 函数式编程的碎碎念

看下上面代码，有一个地方还是不太爽，那就是这一行：

```js
this.store = applyMiddleware(promiseMiddleware)(createStore(reducer, this.state));
```

这有点长啊，而且函数套函数，看着都晕了有木有。等等，我们之前似乎解决过这个问题：我们的中间件本身就是函数套函数嘛，可是我们还不是能横着写？这就要扯到函数式编程上来了。

我们知道《黑客与画家》的作者泡爷就是 lisp 来写他的创业项目 Viaweb 的，甚至这本书一出，掀起了一阵学习 lisp 的装逼小高潮。听说前一阵有 lisp 的爱好者有人打印了一下 Viaweb 的源码，我们把最后一页发出来供大家参考学习：

```lisp
        ))))))))))))
      ))))))))
    )))))
  ))
)
```

……当然啦，上面的都是开玩笑；不过如果你写过 lisp，一定曾经对着这一堆函数嵌套和括号头疼过。这简直就是阻碍函数式编程普及的恶魔啊！试想一下，要是在 js 里你随便写个什么逻辑都得这样：`a(b(c(d(e()))))`，那前端狗们还不得『打得好，我选择屎亡』？

不过没关系，函数式编程的大哥们早就趟这些坑，他们提出了一个叫做 `compose` 的方法，用了这个方法，可以把上面的 `a(b(c(d(e(123)))))` 改成：`compose(a,b,c,d,e)(123)`。是不是瞬间有了活下去的动力？

我们来考虑考虑，在 ES6 里，这个函数应该怎么写：

1. 首先我们应该吧传入的一坨函数改造成数组，这个 ES6 里面的 [rest 参数](http://es6.ruanyifeng.com/#docs/function#rest参数)已经帮我们很好的搞定了；
2. 其次我们需要**逆序**执行函数，这一点就用到了上文说过的 `reduceRight`。

我们来实现一下：

```js
function compose(...funcs) {
   let last = funcs.pop();
   let rest = funcs;
   
   return (...args) => rest.reduceRight((compose, f) => f(compose), last(...args));
}
```

有了这个利器，我们同时还能改造 `applyMiddleware` 里的代码！

```js
function applyMiddleware(...middlewares) {
    return (store) => {
        let dispatch = store.dispatch;
        // 这里对 store 进行了一次复制
        // 防止中间件直接操作 store 本身
        // 最大程度防止 store 被中间件破坏
        let miniStore = {
            getState: store.getState,
            dispatch: action => dispatch(action)
        }
        let chain = middlewares.map(m => m(miniStore));
        // 使用 compose 简化中间件代码
        dispatch = compose(...chain)(dispatch);
        return { ...store, dispatch };
    }
}
```

# 与 react 的连接

# reselect 与 immutable







