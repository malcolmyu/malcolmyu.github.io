title: JavaScript Promise 探微
date: 2014-08-30
categories: [技术研究]
tags: [翻译, Javascript, Promise]
toc: true

---
原文链接：[JavaScript Promises ... In Wicked Detail](http://www.mattgreer.org/articles/promises-in-wicked-detail/)

我在 JavaScript 中使用 Promise 已经有一段时间了，目前我已经能高效的使用这一开始让我晕头转向的东西。但真要细说起来，我发现还是不能完全理解它的实现原理，这也正是本文写作的目的所在。如果诸位读者也处在一知半解的状态，那请读完这篇文章，相信你也会像我一样对 Promise 有更好的理解。

<!--more-->

我们将会循序渐进的建立一个 Promise 的实现，最终这个实现会基本符合 [Promise/A+ 规范](http://promisesaplus.com/)，在此过程中你会逐步的了解到 Promise 是如何实现了异步编程的需求。本文假设你已经有了一定的 Promise 基础，如果你对此一无所知，请移步[官网](https://www.promisejs.org/)先了解学习一下。

# 为什么要写这篇文章？

有些童鞋会问：为啥我们要对 Promise 了解得这么细呢，会用不就好了么？其实理解了一个东西的实现机理，可以提升你使用它的能力和效率，同时在使用出错的时候能更有效地 debug —— 我之所以写这篇文章就是因为有一次和同事掉进一个关于 Promise 的奇怪的坑里去了。要是当年我就和现在这样了解得这么透彻，我就不会掉坑了~

# 最简单的例子

让我们从最简单的例子开始实现我们的 Promise 实例，我想将这样的写法：

```javascript
doSomething(function(value) {
  console.log('Got a value:' + value);
});
```

实现为这样的写法：

```javascript
doSomething().then(function(value) {
  console.log('Got a value:' + value);
});
```

为了实现这个效果，我们只需要将 `donSomething()` 函数从这样的形式：

{% codeblock lang:javascript %}
    function doSomething(callback) {
      var value = 42;
      callback(value);
    }
{% endcodeblock %}

改成这种 “Promise” 基础版：

{% codeblock lang:javascript %}
    function doSomething() {
      return {
        then: function(callback) {
          var value = 42;
          callback(value);
        }
      };
    }
{% endcodeblock %}

这种写法只是给我们的回调模式写了一个简单且毫无意义的语法糖。我们目前还没有触及 Promise 背后的核心概念，但这也是个小小的开始。

**Promise 可以捕获最终值（the eventual value）这一概念并将其置入对象**

这正是 Promise 的有趣之处（**译者注：**所谓的最终值，实际上是规范里面的一个概念，表示异步操作的最终获取值。实际上上面这句话的意思就是将我们要传给回调函数的终值也保存在 Promise 对象里）。在后面的探索中，我们就会发现：一旦最终值的概念可以被这样捕获到，我们就可以干一些非常给力的事情。

## 定义 Promise 类型

让我们进行下一步，定义一个实际的 `Promise` 类型来扩展上面的代码：

{% codeblock lang:javascript %}
    function Promise(fn) {
      var callback = null;
      this.then = function(cb) {
        callback = cb;
      };

      function resolve(value) {
        callback(value);
      }

      fn(resolve);
    }
{% endcodeblock %}

然后用 Promise 类型重写 `doSomething()` 函数：

{% codeblock lang:javascript %}
    function doSomething() {
      return new Promise(function(resolve) {
        var value = 42;
        resolve(value);
      });
    }
{% endcodeblock %}

这里就遇到一个问题：如果你逐行执行代码，就会发现 `resolve()` 函数在 `then()` 函数之前被调用，这就意味着 `resolve()` 被调用的时候，`callback` 还是 `null` 。让我们用一个 hack 来干掉这个问题，引入 `setTimeout`，代码如下所示：

{% codeblock lang:javascript %}
    function Promise(fn) {
      var callback = null;
      this.then = function(cb) {
        callback = cb;
      };

      function resolve(value) {
        // 将 callback 打出当前执行线程，使之可以被 then 函数设定
        setTimeout(function() {
          callback(value);
        }, 1);
      }

      fn(resolve);
    }
{% endcodeblock %}

用了这么个毛招，我们的代码终于可以运行啦=。=

## 这个代码太毛啦

我们写的图样图森破的 Promise 必须要加入异步操作才能工作，这很容易使之再次失效。只要异步地调用 `then()` 函数，我们的 callback 又会马上变成 `null` 了。有作死的读者可能会问：为啥我要写出这么一个破代码让我这么快就感受到失败的挫折？因为我想用上面这个简单易懂的例子将 Promise 的两大关键概念—— `then()` 和 `resolve()` 深深地烙印在你的脑海中。它们会阴魂不散地跟随着你哟~

# Promise 是有状态（state）的

我们糟糕易崩溃代码暴露了一个之前我们没有想到的问题—— Promise 是具有状态的。我们在运行之前需要知道其当前所处的状态，并确保我们可以正确地进行状态转换。采用这种方式可以让我的代码稳定性强一些。

 1. promise 可以处在等待被赋值的**等待态（pending）**，可以被给予一个值并转为**解决态（resolved）**。
 2. 一旦 promise 被一个值 resolve 掉，其就会一直保持这个值并不会再被 resolve。

*（一个 promise 对象也可以被**拒绝 rejected**，我们在稍后的错误处理中会提到）*

让我们在实例中加入状态的跟踪，以此摆脱之前的毛招：

{% codeblock lang:javascript %}
    function Promise(fn) {
      var state = 'pending';
      var value;
      var deferred;

      function resolve(newValue) {
        value = newValue;
        state = 'resolved';

        if(deferred) {
          handle(deferred);
        }
      }

      function handle(onResolved) {
        if(state === 'pending') {
          deferred = onResolved;
          return;
        }

        onResolved(value);
      }

      this.then = function(onResolved) {
        handle(onResolved);
      };

      fn(resolve);
    }
{% endcodeblock %}

我们的代码变得更加复杂，但是这样就使得 Promise 对象调用者可以随时激活 `then()` 函数，被调用的 Promise 对象也可以随时激活 `resolve()` 方法。这在异步和同步的代码中都是完全适用的。

这正是 `state` 标志的功劳。新方法 `handle()` 与之前的两个重要概念 `then()` 和 `resolve()` 互不干涉，其将会根据情况在以下两种操作中选择一种执行：

- `then()` 在 `resolve()` 之前先被调用，意味着还没有最终值传递给回调函数。这种状态下即为等待态，我们便在内存中保存回调函数以便后续使用。当 `resolve()` 被调用时，我们激活回调函数并将终值传入。
- `reslove` 在 `then()` 之前被调用，这种情况下我们将终值保存在内存中，一旦 `then()` 被调用，我们就将终值传入。

注意到 `setTimeout` 不见了么，这个只是暂时的，它还会回来哒~

言归正传：

**使用 promises 的时候，我们调用其方法的顺序并不重要，可以按照自己的意愿随时调用 `then()` 和 `resolve()`，这就是将终值捕获并置于对象之中保存的强大优势。**

尽管我们还有一些事情米有做，我们的 promises 已经非常给力了。这套实现允许我们执行多次 `then()` —— 其每次都会获取到同样的终值。

{% codeblock lang:javascript %}
    var promise = doSomething();

    promise.then(function(value) {
      console.log('Got a value:', value);
    });

    promise.then(function(value) {
      // 此处获取的值和上一处相同
      console.log('Got the same value again:', value);
    });
{% endcodeblock %}

*（其实吧……这个地方并不是完全正确的，如果我们反过来操作，在执行 `resolve()` 之前多次执行 `then()` ，结果只有最后一次的执行会成功。如果要修复这个问题需要在 Promise 对象中维护一个队列来记录回调函数。但由于这篇文章已经够长了，所以我决定不这么搞了=v=）*

# 通通连起来吧

既然 Promise 将异步操作捕获到了对象中，我们就可以对其进行链式操作、map 操作以及串行并行等等其他高效率的操作。下列代码就是一个非常常见的 Promise 用法：

{% codeblock lang:javascript %}
    getSomeData()  
      .then(filterTheData)
      .then(processTheData)
      .then(displayTheData);
{% endcodeblock %}

由于可以调用 `then()` 函数，这证明 `getSomeData()` 返回的是一个 promise 对象；但是第一个怎返回的结果页必须是一个 promise 对象，然后我们才能再次调用 `then()` 函数（然后再次调用再次调用再次调用~）。而实际的 Promise 实现就是这样的效果，假如我们能够让 `then()` 函数返回一个 promise 对象，一切就变得更有趣起来。

**`then()` 永远返回一个 promise 对象。**

以下就是给我们的 promise 假如链式调用的情况：

{% codeblock lang:javascript %}
    function Promise(fn) {
      var state = 'pending';
      var value;
      var deferred = null;

      function resolve(newValue) {
        value = newValue;
        state = 'resolved';

        if(deferred) {
          handle(deferred);
        }
      }

      function handle(handler) {
        if(state === 'pending') {
          deferred = handler;
          return;
        }

        if(!handler.onResolved) {
          handler.resolve(value);
          return;
        }

        var ret = handler.onResolved(value);
        handler.resolve(ret);
      }

      this.then = function(onResolved) {
        return new Promise(function(resolve) {
          handle({
            onResolved: onResolved,
            resolve: resolve
          });
        });
      };

      fn(resolve);
    }
{% endcodeblock %}

额……已经变得有点令人抓狂啦，你是不是在庆幸我们进展的比较缓慢呢~这里的关键之处就在于：`then()` 函数返回了一个**新的 Promise 对象**。

*（由于 `then()` 永远返回一个新的 promise 对象，导致每次都至少有一个 promise 对象被创建、解决然后被忽略，这就产生了一定程度了内存浪费。这是 Promise 被诟病的一个原因，因为传统的回调金字塔就不存在这样的问题。由此你可以理解为啥一些 JavaScript 社区已经抛弃了 promise ）*

那第二个 promise 要 resolve 的值是什么呢？答案是：**第一个 promise 的返回值**。 `handle()` 函数的最后两行体现了这一点， `handler` 对象保存了 `onResolved()` 回调函数和 `resolve()` 函数的引用。在链式调用中保存了多个 `resolve()` 函数的拷贝，每一个 promise 对象的内部都拥有一个自己的 `resolve()` 方法，并在闭包中运行。 这建立起了第一个 promise 与第二个 promise 之间联系的桥梁。我们在这一行代码 resolve 了第一个 promise：

{% codeblock lang:javascript %}
    var ret = handler.onResolved(value);
{% endcodeblock %}

在上文的例子中，程序里的 `handler.onResolved` 是这个函数：

{% codeblock lang:javascript %}
    function(value) {  
      console.log('Got a value:', value);
    }
{% endcodeblock %}

换句话说，这就是我们第一次调用 `then()` 时传入的处理函数，第一个处理函数的返回值将会用来传递给第二个 promise，链式调用就这么完成啦~

{% codeblock lang:javascript %}
    doSomething().then(function(result) {
      console.log('first result', result);
      return 88;
    }).then(function(secondResult) {
      console.log('second result', secondResult);
    });

    // 输出结果是：
    // 
    // 第一个结果：42
    // 第二个结果：88


    doSomething().then(function(result) {
      console.log('first result', result);
      // 没有显示的返回值（也就是 undefined）
    }).then(function(secondResult) {
      console.log('second result', secondResult);
    });

    // 输出结果是：

    // 第一个结果：42
    // 第二个结果：undefined
{% endcodeblock %}

既然 `then()` 方法永远返回一个新的 promise ，因此这个链式调用就可以越链越深：

{% codeblock lang:javascript %}
    doSomething().then(function(result) {
      console.log('first result', result);
      return 88;
    }).then(function(secondResult) {
      console.log('second result', secondResult);
      return 99;
    }).then(function(thirdResult) {
      console.log('third result', thirdResult);
      return 200;
    }).then(function(fourthResult) {
      // 链呀链...
    });
{% endcodeblock %}

我们如果想在上面的例子中获取每次处理函数调用返回的结果集，就必须在链式调用中人工构建一个存放结果集的数组：

{% codeblock lang:javascript %}
    doSomething().then(function(result) {
      var results = [result];
      results.push(88);
      return results;
    }).then(function(results) {
      results.push(99);
      return results;
    }).then(function(results) {
      console.log(results.join(', ');
    });

    // 输出结果：

    // 42, 88, 99
{% endcodeblock %}

**Promise 每次只会 resolve 一个值，如果你想传递多个值，就需要建立一种存储方式来进行传递（如数组、对象和字符串）**

更好的解决途径是使用 Promise 库中的 `all()` 方法或其他实用的方法来提升 promise 的使用效率，这就有待诸位读者自己挖掘啦。

## 可选的回调函数

`then()` 中的回调函数并不是严格要求必写的，加入你不写这个回调， promise 也会用上一个 promise 返回的终值来传递。

{% codeblock lang:javascript %}
    doSomething().then().then(function(result) {
      console.log('got a result', result);
    });

    // 输出结果是：
    //
    // got a result 42
{% endcodeblock %}

你可以在 `handle()` 函数内部观察到这个情况，如果当前的 `then()` 没有传递回调函数，该函数就会直接使用前一个 promise 返回的终值来解决下一个 promise：

{% codeblock lang:javascript %}
    if(!handler.onResolved) {
      handler.resolve(value);
      return;
    }
{% endcodeblock %}

## 链中返回 promise

我们实现链式的实例还是略显简单，其仅仅是将解决终值传递下去，但如果有个终值就是 promise 咋办？举个栗子：

{% codeblock lang:javascript %}
    doSomething().then(result) {
      // doSomethingElse 返回一个 promise
      return doSomethingElse(result)
    }.then(function(finalResult) {
      console.log("the final result is", finalResult);
    });
{% endcodeblock %}

目前来看，上面的结果不会是我们期望的那样。`finalResult` 不会是的第一个 result 的值，而会是一个 promise 对象。为了达到我们期望的结果（也就是依然让返回值传递下去），我们需要这样做：

{% codeblock lang:javascript %}
    doSomething().then(result) {
      // doSomethingElse 返回一个 promise
      return doSomethingElse(result)
    }.then(function(anotherPromise) {
      anotherPromise.then(function(finalResult) {
        console.log("the final result is", finalResult);
      });
    });
{% endcodeblock %}    

=。=但是你会让这一坨翔一样的代码出现在你的项目中么…让我们在 promise 实例中隐式的处理掉这个问题。这个处理方式还是比较简单的，只要在 `resolve()` 方法中加入一个对返回值是 promise 对象的特殊处理就行啦：

{% codeblock lang:javascript %}
    function resolve(newValue) {
      if(newValue && typeof newValue.then === 'function') {
        newValue.then(resolve);
        return;
      }
      state = 'resolved';
      value = newValue;

      if(deferred) {
        handle(deferred);
      }
    }
{% endcodeblock %}

这样我们就可以继续持续的调用 `resolve()` 直到我们获取到一个 promise。当其返回值不是 promise 对象时，调用链就会和之前一样正常执行。

*这样的话可能会造成无穷回路（**译者注：**也就是 `then()` 返回 promise 对象然后又调用 `then()`）。尽管 A+ 规范里建议 promise 的实现中对无穷回路进行判断，但这种判断是没什么必要的（**译者注：**在规范的最后写了说明，规范本身建议判断，但 promise 的实现并不建议判断）。*

*另外，我们这个实现实际上并不完全符合规范，这篇文章里说的东西也没有完全符合规范。假如你对规范本身感兴趣，请移步文章开始处的规范链接。*

有没有注意到我们对于 `newValue` 是否为 promise 对象的检测是多么的宽松么，我们只是判断它是否拥有 `then()` 方法。这个鸭子类型是我故意这么写的（**译者注：**这不是一个 bug，这是个 feature）！这使得不同的 promise 实现可以相互运作，实际上这也是不同的第三方 promise 库的比较常见的混用方式。

**不同的 promise 实现只要恰当的遵循规范，就可以相互混用。**

搞定了链式调用之后，我们的实现基本接近完成，除了最初被我们完全忽略掉的一个问题 —— **错误处理**。

# Promise 的拒绝（reject）

当一个 promise 运行发生错误，其需要被**拒绝（reject）**并传入一个**原因（reason）**。那么调用者怎么知道何时进行 reject 呢？这可以通过给 `then()` 函数的第二个参数传入回调来实现。

{% codeblock lang:javascript %}
    doSomething().then(function(value) {
      console.log('Success!', value);
    }, function(error) {
      console.log('Uh oh', error);
    });
{% endcodeblock %}

**正如之前所提到的那样，promise 对象可以从 pending 转换到 resolved 或者 rejected，但不能同时 resolved 和 rejected。换言之，`then` 的两个回调中仅有一个会被调用。**

Promise 可以通过 `resolve()` 方法的孪生兄弟 —— `reject()` 方法来实现拒绝。下面是给 `doSomething()` 加入错误处理的情况：

{% codeblock lang:javascript %}
    function doSomething() {
      return new Promise(function(resolve, reject) {
        var result = somehowGetTheValue();
        if(result.error) {
          reject(result.error);
        } else {
          resolve(result.value);
        }
      });
    }
{% endcodeblock %}

在我们 promise 的实现中，我们也必须考虑到 reject 。一旦一个 promise 被拒绝，其后面的调用链中的 promise 也必须被拒绝。

让我们再来一起看一下完成版的 promise 实例的实现，这其中加入了拒绝的处理。

{% codeblock lang:javascript %}
    function Promise(fn) {
      var state = 'pending';
      var value;
      var deferred = null;

      function resolve(newValue) {
        if(newValue && typeof newValue.then === 'function') {
          newValue.then(resolve, reject);
          return;
        }
        state = 'resolved';
        value = newValue;

        if(deferred) {
          handle(deferred);
        }
      }

      function reject(reason) {
        state = 'rejected';
        value = reason;

        if(deferred) {
          handle(deferred);
        }
      }

      function handle(handler) {
        if(state === 'pending') {
          deferred = handler;
          return;
        }

        var handlerCallback;

        if(state === 'resolved') {
          handlerCallback = handler.onResolved;
        } else {
          handlerCallback = handler.onRejected;
        }

        if(!handlerCallback) {
          if(state === 'resolved') {
            handler.resolve(value);
          } else {
            handler.reject(value);
          }

          return;
        }

        var ret = handlerCallback(value);
        handler.resolve(ret);
      }

      this.then = function(onResolved, onRejected) {
        return new Promise(function(resolve, reject) {
          handle({
            onResolved: onResolved,
            onRejected: onRejected,
            resolve: resolve,
            reject: reject
          });
        });
      };

      fn(resolve, reject);
    }
{% endcodeblock %}

除去额外加入的 `reject()` 函数，`handle()` 函数本身也能对拒绝进行应对。其根据 `state` 的值来决定进行 resolve 还是 reject，而后 `state` 的值会被推送到下一个 promise 中，作为决定下个 promise 进行解决还是拒绝的依据（**译者注：**在这个实现中，并没有体现出这一点。因为本实例使用的是 `then()` 链而不是 `done()`  `fail()` 链，每次传递的都是一个新的 promise 对象，因此上一个 promise 被拒绝了，也仅仅会把其拒绝回调函数的返回值传递给下一个链的 resolve 回调。言下之意，本实例中的只有第一个 promise 对象可以被拒绝，第二个起直到链尾的 promise 其拒绝回调都无法被调用 —— 除非发生下一章节的非预期异常，有兴趣的读者可以自己试一试）。

*当使用 promise 的时候，我们很容易把错误处理的回调省略掉，但这样会导致我们无法捕获到任何报错。你至少应该在链式 promise 的最后写一个错误处理回调。这里可以参加下一章的错误吞没。*

## 非预期的错误也应该被拒绝

目前我们处理的错误仅仅是已知的错误，但也可能突然蹦出来一个意料之外的错误然后把一切搞崩掉。因此 promise 实例对这些异常进行捕获并拒绝也是十分必要的。

这就意味着 `resolve()` 方法需要被包裹在 try/catch 语句块中：

{% codeblock lang:javascript %}
    function resolve(newValue) {
      try {
        // ... 这里和以前一样
      } catch(e) {
        reject(e);
      }
    }
{% endcodeblock %}

保证 `then()` 中传入的回调函数不会抛出一些无法处理的异常也很重要。由于这些回调在 `handle()` 中被调用，因此我们最终的实现结果是这样的：

{% codeblock lang:javascript %}
    function handle(deferred) {
      // ... 一切如前

      var ret;
      try {
        ret = handlerCallback(value);
      } catch(e) {
        handler.reject(e);
        return;
      }

      handler.resolve(ret);
    }
{% endcodeblock %}

## promises 可能吞没错误！

（**译者注：**非常怀疑作者在文章开头掉进的大坑就是这个“错误吞噬”。）

*对于 promises 的误解可能会导致报错信息的丢失。这是一个不少人都会掉进去的大坑。*

我们看下面这个例子：

{% codeblock lang:javascript %}
    function getSomeJson() {
      return new Promise(function(resolve, reject) {
        var badJson = "<div>这东西根本不是JSON呀！</div>";
        resolve(badJson);
      });
    }

    getSomeJson().then(function(json) {
      var obj = JSON.parse(json);
      console.log(obj);
    }, function(error) {
      console.log('uh oh', error);
    });
{% endcodeblock %}

这里会发生什么事情呢？我们在 `then()` 中传递的回调函数期望获得一个有效的 JSON 串，并用原生方法去解析它，因此导致了一个异常。但是我们有一个处理错误的回调函数（也就是 reject 回调），所以是不是米有问题呢？

**大错特错。** reject 回调根本不会被调用到！如果你执行上述例子，你不会得到任何的输出。万籁此俱寂，没有错误输出，啥都米有。

为什么会这样呢？因为未经处理的异常在我们 `then()` 函数传入的回调中发生了，这在我们的实例中被 `handle()` 捕获到。这导致 `handle()` 拒绝的 promise 是这个 `then()` 函数返回的那个 promise，而不是当前的我们准备进行错误处理的 promise ，而当前这个 promise 已经被 resolve 掉了。

如果你想捕获上述的异常，你需要再链一个 `then()`：

{% codeblock lang:javascript %}
    getSomeJson().then(function(json) {
      var obj = JSON.parse(json);
      console.log(obj);
    }).then(null, function(error) {
      console.log("an error occured: ", error);
    });
{% endcodeblock %}

现在我们能正确的打印错误啦。

*根据我这么多年使用 promise 的经验，错误吞噬这东西是 promise 最大的坑了（**译者注：**果然是作者掉的那个坑=。=），请阅读下一章节发现更好的解决方案—— `done()`。*

## 救世者 `done()`

大多数的 promise 库中都集成了 `done()` 方法。它与 `then()` 十分类似，但他避免了上述陷阱。

`done()` 函数可以和 `then()` 函数一样被调用，其差异之处在于它不会返回一个 promise 对象，且在 `done()` 中未经处理的异常不会被 promise 实例所捕获。换句话说，当整个 promise 链被完全解决时才会调用 `done()`。我们的 `getSomeJson()` 的例子可以使用 `done()` 来让之变得更加健壮。

{% codeblock lang:javascript %}
    getSomeJson().done(function(json) {
      // 当抛出异常时不会被吞没
      var obj = JSON.parse(json);
      console.log(obj);
    });
 {% endcodeblock %}   

`done()` 函数也和 `then()` 一样有一个错误回调， `done(callback, errback)`，当整个 promise 链被执行完成后，你可以保证任何抛出的异常都在错误回调中被捕获。

*`done()` 目前为止还没有加入 promise/A+ 规范，所以某些 promise 库可能并不包含此功能。*

（**译者注：**实际上这里作者所叙述的 `done()` 和我们熟悉的 jQuery 里面实现的 `done()` 并不相同。jQuery 用 Callback 对象实现的 `done()` 方法，只能传递一个成功回调函数，且其返回的不是一个新的 promise 对象，而是当前的 promise 对象。）

# Promise 解决程序需要异步调用

在文章的开始我们使用 `setTimeout` 搞了一个毛招，当我们用“状态”这一概念解决掉这个毛招之后，我们就再未曾再见到过 `setTimeout` 了呢。但实际上 Promise/A+ 规范要求 promise 的解决程序必须是异步的。为了符合这个小需求，我们只需简单的将 `handle()` 方法中的大部分实现包裹在 `setTimeout` 中即可。

{% codeblock lang:javascript %}
    function handle(handler) {
      if(state === 'pending') {
        deferred = handler;
        return;
      }
      setTimeout(function() {
        // ... 一切如前
      }, 1);
    }
{% endcodeblock %}

以上就是我们所需要做的事情。事实上真正的 promise 库不必非使用 `setTimeout`。如果 promise 库是基于 NodeJS 的，那可能会用到 `process.nextTick`；如果基于前端浏览器可能会用到最新的 `setImmediate` 或是 [setImmediate shim](https://github.com/YuzuJS/setImmediate) （因为迄今为止只有 IE 支持 `setImmediate`），或者可能是一个异步的函数库，如 Kris Kowal 的 [asap](https://github.com/kriskowal/asap)（此人还写了一个著名的 promise 库 —— [Q](https://github.com/kriskowal/q)）。

## 为何在规范里要求异步调用？

这是为了保证一致性和可靠的执行流程，例如下面这个例子：

{% codeblock lang:javascript %}
    var promise = doAnOperation();
    invokeSomething();
    promise.then(wrapItAllUp);
    invokeSomethingElse();
{% endcodeblock %}

这里的执行流程会是怎样的呢？根据函数命名我们猜测应该是这样`invokeSomething()` -> `invokeSomethingElse()` -> `wrapItAllUp`。但这其实完全取决于你当前实现的 promise 的解决方式是同步的还是异步的。如果 `doAnOperation()` 是异步的，那其执行顺序就和我们猜测的一样；如果它是同步执行的，实际的执行顺序就会是这样：`invokeSomething()` -> `wrapItAllUp` -> `invokeSomethingElse()`，这可能就会出现问题。

为了处理这种情况， 即使异步不是必须的，但 **promise 的解决程序也必须是异步的**。这减少了不必要的困扰，也让使用者在使用过程中不必考率代码里的异步实现。

# 总结

能读到这里你也是挺给力的……本文涵盖了规范中所要求的 promise 的核心实现，但大多数的 promise 库都提供了更多的功能，如 `all()` 、`spread()` 、`race()` 、`denodeify()` 等等。如果想了解 promise 的更多功能，我建议诸位看看 Bluebird 函数库的 [API](https://github.com/petkaantonov/bluebird/blob/master/API.md)。

在我了解了 promise 的运作方式和可能的坑之后，我爱上了 promise =v=。她让我们的代码变得非常整洁和优雅。当然这篇文章仅仅是个开始，对于 promise 而言，能够讨论的东西还有太多太多。

如果你喜欢这篇文章，你可以在我的 [twitter](http://twitter.com/cityfortyone) 上关注我一下，当我有别的更新的时候你也能及时发现~

# 推荐阅读

 - [promisejs.org](https://www.promisejs.org/) 文本中多次提到的 promises 教学
 - [Q  的基本设计原理](https://github.com/kriskowal/q/blob/v1/design/README.js) 形式上和本文差不多的一篇文章，但细节上更加深入。作者就是 Q 之父 Kris Kowal。
 - [有关 done() 大法好不好的争论](https://github.com/domenic/promises-unwrapping/issues/19)
 - [扁平化的链式 Promise](http://solutionoptimist.com/2013/12/27/javascript-promise-chains-2/) 作者是 Thomas Burleson，一篇给力的文章探讨 promise 的进阶用法。如果本文主要讲了什么是 promise，那这篇文章就更多的围绕为什么这样实现来展开。
