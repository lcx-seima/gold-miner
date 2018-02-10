> * 原文地址：[How JavaScript works: The building blocks of Web Workers + 5 cases when you should use them](https://blog.sessionstack.com/how-javascript-works-the-building-blocks-of-web-workers-5-cases-when-you-should-use-them-a547c0757f6a)
> * 原文作者：[Alexander Zlatkov](https://blog.sessionstack.com/@zlatkov?source=post_header_lockup)
> * 译文出自：[掘金翻译计划](https://github.com/xitu/gold-miner)
> * 本文永久链接：[https://github.com/xitu/gold-miner/blob/master/TODO/how-javascript-works-the-building-blocks-of-web-workers-5-cases-when-you-should-use-them.md](https://github.com/xitu/gold-miner/blob/master/TODO/how-javascript-works-the-building-blocks-of-web-workers-5-cases-when-you-should-use-them.md)
> * 译者：[刘嘉一](https://github.com/lcx-seima)
> * 校对者：

# JavaScript 工作原理：Web Workers 的内部构造以及 5 种你应当使用它的场景

![](https://cdn-images-1.medium.com/max/800/0*b5WMJNTRt9QqN-Zy.jpg)

这是探索 JavaScript 及其内建组件系列文章的第 7 篇。在认识和描述这些核心元素的过程中，我们也会分享我们在构建 [SessionStack](https://www.sessionstack.com/?utm_source=medium&utm_medium=source&utm_content=javascript-series-web-workers-intro) 时所遵循的一些经验规则。SessionStack 是一个轻量级 JavaScript 应用，它协助用户们构造出实时性要求高的 Web 应用，因此其自身不仅需要足够健壮还要有不俗的性能表现。

如果你错过了前面的文章，你可以在下面找到它们：

* [对引擎、运行时和调用栈的概述](https://juejin.im/post/5a05b4576fb9a04519690d42)
* [深入 V8 引擎以及 5 个写出更优代码的技巧](https://juejin.im/post/5a102e656fb9a044fd1158c6)
* [内存管理以及四种常见的内存泄漏的解决方法](https://juejin.im/post/59ca19ca6fb9a00a42477f55)
* [事件循环和异步编程的崛起以及 5 个如何更好的使用 async/await 编码的技巧](https://juejin.im/post/5a221d35f265da43356291cc)
* [深入剖析 WebSockets 和拥有 SSE 技术 的 HTTP/2，以及如何在二者中做出正确的选择](https://blog.sessionstack.com/how-javascript-works-deep-dive-into-websockets-and-http-2-with-sse-how-to-pick-the-right-path-584e6b8e3bf7?source=collection_home---4------0----------------)
* [How JavaScript works: A comparison with WebAssembly + why in certain cases it’s better to use it over JavaScript](https://blog.sessionstack.com/how-javascript-works-a-comparison-with-webassembly-why-in-certain-cases-its-better-to-use-it-d80945172d79)

这一次我们将剖析 Web Workers：在简单概述 Web Workers 后，我们将分别讨论不同类型的 workers 以及它们内部组件的运作方法，同时也会以场景为例说明它们各自的优缺点。在文章的最后，我们将提供最适合使用 Web Workers 的 5 个用例场景。

我们在 [之前的文章](https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf) 中已经详尽地讨论了 JavaScript 的单线程运行机制，你应当对此已经了然于胸。即便如此，JavaScript 也允许开发者在单线程模型上书写异步代码。

#### 异步编程的 “天花板”

我们已经讨论过了 [异步编程](https://blog.sessionstack.com/how-javascript-works-event-loop-and-the-rise-of-async-programming-5-ways-to-better-coding-with-2f077c4438b5?source=---------2----------------) 的概念及其使用场景。

异步编程通过把部分代码 “放置” 到事件循环较后的时间点执行，保证了 UI 渲染始终处于较高的优先级，这样你的 UI 就不会出现卡顿无响应的情况。

AJAX 请求是异步编程的最佳实践之一。通常网络请求不会在短时间内得到响应，因此异步的网络请求能让客户端在等待响应结果的同时执行其他业务代码。

```
// 假设你使用的是 jQuery
jQuery.ajax({
    url: 'https://api.example.com/endpoint',
    success: function(response) {
        // 正确响应后需要执行的代码
    }
});
```

当然这里有个问题，上例能够进行异步请求是依靠了浏览器提供的 API，其他代码又该如何实现异步执行呢？例如，在上例 success 回调函数中存在着 CPU 密集型计算：

```

var result = performCPUIntensiveCalculation();
```

假如 `performCPUIntensiveCalculation` 不是一个 HTTP 请求，而是一段可以阻塞线程的代码（例：一段巨型 `for` 循环代码）。这样会使 event loop 不堪重负，浏览器 UI 也随之阻塞 —— 用户将面对卡顿无响应的网页。

这就说明了使用异步函数只能解决 JavaScript 单线程模型带来的一小部分问题。

在一些因大量计算引起的 UI 阻塞问题中，使用 `setTimeout` 来解决阻塞的效果还不错。例如，我们可以把一系列的复杂计算分批放到单独的 `setTimeout` 中执行，这样做等于是把连续的计算分散到了 event loop 中的不同位置，以此为 UI 的渲染和事件响应让出了时间。

让我们来看看一个简单的函数，它被用于计算数值型数组的均值：

```

function average(numbers) {
    var len = numbers.length,
        sum = 0,
        i;

    if (len === 0) {
        return 0;
    } 
    
    for (i = 0; i < len; i++) {
        sum += numbers[i];
    }
   
    return sum / len;
}
```

下面是对上方代码的一个重写，使其获得了异步性：

```
function averageAsync(numbers, callback) {
    var len = numbers.length,
        sum = 0;

    if (len === 0) {
        return 0;
    } 

    function calculateSumAsync(i) {
        if (i < len) {
            // 把下一次函数调用放入 event loop
            setTimeout(function() {
                sum += numbers[i];
                calculateSumAsync(i + 1);
            }, 0);
        } else {
            // 计算完数组中所有元素后，调用回调函数返回结果
            callback(sum / len);
        }
    }

    calculateSumAsync(0);
}
```

通过使用 `setTimeout` 可以把每一步计算都放置到 event loop 较后的时间点执行。在每两次的计算间隔，event loop 便会有足够的时间执行其他的计算，从而保证浏览器不会一 ”冻“ 不动。

#### 力挽狂澜的 Web Worker

[HTML5](https://www.w3schools.com/html/html5_intro.asp) 已经提供了不少开箱即用的好东西，包括：

* SSE （在 [上一篇文章](https://blog.sessionstack.com/how-javascript-works-deep-dive-into-websockets-and-http-2-with-sse-how-to-pick-the-right-path-584e6b8e3bf7) 中已经谈过它的特性并与 WebSocket 进行了对比)
* 地理信息
* 应用缓存
* LocalStorage
* 拖放手势
* **Web Worker**

Web Worker 是内建在浏览器中的轻量级 **线程**，使用它执行 JavaScript 代码不会阻塞 event loop。

非常神奇吧，本来 JavaScript 中的所有范例都是基于单线程模型实现的，但这里的 Web Worker 却（在一定程度上）打破了这一限制。

从此开发者可以远离 UI 阻塞的困扰，通过把一些执行时间长、计算密集型的任务交由 Web Worker 放到后台完成，使他们应用的响应变得更加迅速。更重要的是，我们再也不需要对 event loop 施加任何的 `setTimeout` 黑魔法。

这里有一个简单的数组排序 [demo](http://afshinm.github.io/50k/) ，其中对比了使用 Web Worker 和不使用 Web Worker 时的区别。

#### **Web Worker 概览**

Web Worker 允许你在执行大量计算密集型任务时，还不阻塞 UI 进程的执行。事实上，二者互不不阻塞的原因就是它们是并行执行的，可以看出 Web Worker 是货真价实的多线程。

你可能想说 — ”JavaScript 不是一个在单线程上执行的语言吗？“。

你可能会惊讶 JavaScript 作为一门编程语言，却没有定义任何的线程模型。因此 Web Worker 并不属于 JavaScript 语言的一部分，它仅仅是浏览器提供的一项特性，只是它使用了 JavaScript 作为访问中介罢了。过往的众多浏览器都是单线程程序（以前的理所当然，现在也有了些许变化），并且浏览器一直以来也是 JavaScript 主要的运行环境。对比在 Node.JS 中就没有 Web Worker 的相关实现 — 虽然 Web Worker 对应着 Node.JS 中的 “cluster” 或 “child_process” 概念，不过它们还是有所区别的。

值得注意的是，Web Worker 的 [定义](http://www.whatwg.org/specs/web-workers/current-work/) 中一共包含了 3 种类型的 Worker：

* [Dedicated Worker（专用 Worker）](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers)
* [Shared Worker（共享 Worker）](https://developer.mozilla.org/en-US/docs/Web/API/SharedWorker)
* [Service worker（服务 Worker）](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorker_API)

#### Dedicated Worker（专用 Worker）

Dedicated Worker 由主线程实例化且只能与它通信。

![](https://cdn-images-1.medium.com/max/800/1*ya4zMDfbNUflXhzKz9EBIw.png)

Dedicated Worker 浏览器兼容性一览

#### Shared Worker（共享 Worker）

Shared Worker 可以被同一域（浏览器中不同的 tab、iframe 或其他 Shared Worker）下的所有线程访问。

![](https://cdn-images-1.medium.com/max/800/1*lzOIevUBVy5eWyf2kHf--w.png)

Shared Worker 浏览器兼容一览

#### Service worker（服务 Worker）

Service Worker 是一个事件驱动型 Worker，它的初始化注册需要网页/站点的 origin 和路径信息。一个注册好的 Service Worker 可以控制相关网页/网站的导航、资源请求以及进行粒度化的资源缓存操作，因此你可以极好地控制应用在特定环境下的表现（如：无网络可用时）。

![](https://cdn-images-1.medium.com/max/800/1*6o2TRDmrJlS97vh1wEjLYw.png)

Service Worker 浏览器兼容一览

在本文中，我们主要讨论 Dedicated Worker，后文简称为 ”Web Worker“ 或 “Worker”。

#### Web Worker 工作原理

Web Worker 最终实现保存为一系列的 `.js` 文件，网页会通过异步 HTTP 请求来加载它们。当然 [Web Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) 已经包办了这一切，上述加载对使用者完全无感。

Worker 利用类似线程的消息机制保持了与主线程的平行，它是提升你应用 UI 体验的不二人选，使用 Worker 保证了 UI 渲染的实时性、高性能和快速响应。

Web Worker 是运行在浏览器内部的一条独立线程，因此需要使用 Web Worker 运行的代码块也必须存放在一个 **独立文件** 中。这一点需要牢记在心。

让我们看看，如何创建一个基础 Worker：

```
var worker = new Worker('task.js');
```

如果此处的 “task.js” 存在且能被访问，那么浏览器会创建一个新的线程去异步地下载源代码文件。一旦下载完成，代码将立刻执行，此时 Worker 也就开始了它的工作。
如果提供的代码文件不存在返回 404，那么 Worker 会静默失败并不抛出异常。

为了启动创建好的 Worker，你需要显式地调用 `postMessage` 方法：

```
worker.postMessage();
```

#### Web Worker 通信

为了使创建好的 Worker 和创建它的页面能够通信，你需要使用 `postMessage` 方法或 [Broadcast Channel（广播通道）](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel).

#### 使用 postMessage 方法

在较新的浏览器中，postMessage 方法支持 `JSON` 对象作为函数的第一个入参，但是在旧版本浏览器中它还是只支持 `string`。

下面的 demo 会展示 Worker 是如何与创建它的页面进行通信的，同时我们将使用 JSON 对象作为通信体，让这个 demo 看起来稍微 “复杂” 一点。若改为传递的是字符串，方法也不言而喻了。

让我们看看下面的 HTML 页面（或者准确地说是片段）：

```
<button onclick="startComputation()">Start computation</button>

<script>
  function startComputation() {
    worker.postMessage({'cmd': 'average', 'data': [1, 2, 3, 4]});
  }
  var worker = new Worker('doWork.js');
  worker.addEventListener('message', function(e) {
    console.log(e.data);
  }, false);
  
</script>
```

这部分则是 Worker 脚本中的内容：

```
self.addEventListener('message', function(e) {
  var data = e.data;
  switch (data.cmd) {
    case 'average':
      var result = calculateAverage(data); // 一个计算数值型数组元素均值的函数
      self.postMessage(result);
      break;
    default:
      self.postMessage('Unknown command');
  }
}, false);
```

当主页面中的 button 被按下，触发了 `postMessage` 的调用。`worker.postMessage` 这行代码会传递一个 `JSON` 对象给Worker，对象中包含了 `cmd` 和 `data` 两个键以及它们对应的值。相应的，Worker 会通过定义的 `message` 响应方法拿到和处理上面传递过来的消息内容。

当消息到达 Worker 后，实际的计算便开始运行，这样完全不会阻塞 event loop。在此过程中，Worker 只会检查传递来的事件 `e`，然后像往常执行 JavaScript 函数一样继续执行。当最终执行完成，执行结果会回传回主页面。

在 Worker 的执行上下文中，`self` 和 `this` 都指向 Worker 的全局作用域。

> 有两种停止 Worker 的方法：1、在主页面中显示地调用 `worker.terminate()` ；2、在脚本中调用 `self.close()` 让 Worker 做个 “自我了断”。

#### Broadcast Channel（广播通道）

[Broadcast Channel](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel) 是更纯粹地为通信而生的 API。它允许我们在同域下的所有的上下文中发送和接收消息，包括浏览器 tab、iframe 和 Worker：

```
// 创建一个连接到 Broadcast Channel
var bc = new BroadcastChannel('test_channel');

// 发送一段简单的消息
bc.postMessage('This is a test message.');

// 在 handler 中接收并打印消息到终端
bc.onmessage = function (e) { 
  console.log(e.data); 
}

// 断开与 Broadcast Channel 的连接
bc.close()
```

下图会帮助你理解 Broadcast Channel 的工作原理：

![](https://cdn-images-1.medium.com/max/800/1*NVT6WbNrH_mQL64--b-l1Q.png)

使用 Broadcast Channel 会有更严格的浏览器兼容限制：

![](https://cdn-images-1.medium.com/max/800/1*81mCsOzyJj-HfQ1lP_033w.png)

#### 消息的大小

一共有 2 种给 Web Worker 发送消息的方法：

* **拷贝消息：** 这种方法下消息会被序列化、拷贝然后再发送出去，接收方接收后则进行反序列化取得消息。因此上例中的页面和 Worker 不会共享同一个消息实例，它们之间每发送一次消息就会多创建一个消息的副本。大多数浏览器都采用这样的消息发送方法，并且会在发送和接收端自动进行 JSON 编码/解码。如你所预料的，这些数据处理会给消息传送带来不小的负担。传送的消息越大，时间开销就越大。
* **传递消息：** 使用这种方法意味着消息发送者一旦成功发送消息后，就再也无法使用发出的消息数据了。消息的传送几乎不耗费任何时间，美中不足的是只有 [ArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer) 支持以这种方式发送。

#### Web Worker 中支持的 JavaScript 特性

因为 Web Worker 的多线程天性使然，它只能访问 **一小撮** JavaScript 提供的特性，列表如下：

* `navigator` 对象
* `location` 对象（只读）
* `XMLHttpRequest`
* `setTimeout()/clearTimeout()` 与 `setInterval()/clearInterval()`
* [应用缓存](https://www.html5rocks.com/tutorials/appcache/beginner/)
* 使用 `importScripts()` 引入外部 script
* [创建其他的 Web Worker](https://www.html5rocks.com/en/tutorials/workers/basics/#toc-enviornment-subworkers)

#### Web Worker 的局限性

令人遗憾的是 Web Worker 无法访问一些非常重要的 JavaScript 特性：

* DOM 元素（访问不是线程安全的）
* `window` 对象
* `document` 对象
* `parent` 对象

这意味着 Web Worker 不能做任何的 DOM 操作（也就是 UI 层面的工作）。刚开始这会显得略微棘手，不过一旦你学会了如何正确使用 Web Worker，你就只会把它用作单独的 ”计算机器“ ，因此 UI 变更只会发生在页面代码中。你可以把所有的脏活累活交给 Web Worker 完成，再将它劳作的结果传到页面并在那里进行必要的 UI 操作。

#### 异常处理

像对待任何 JavaScript 代码一样，你希望处理 Web Worker 抛出的任何错误。当 Worker 在运行时发生错误，它会触发 `ErrorEvent` 事件。该接口包含 3 个有用的属性，它们能帮助你定位代码出错的原因：

* **filename** - 发生错误的 script 文件名
* **lineno** - 发生错误的代码行号
* **message** - 错误信息

这有一个例子：

```
function onError(e) {
  console.log('Line: ' + e.lineno);
  console.log('In: ' + e.filename);
  console.log('Message: ' + e.message);
}

var worker = new Worker('workerWithError.js');
worker.addEventListener('error', onError, false);
worker.postMessage(); // 不传递消息仅启动 Worker
```

```
self.addEventListener('message', function(e) {
  postMessage(x * 2); // 此行故意使用了未声明的变量 'x'
};
```

可以看到，我们在这儿创建了一个 Worker 并监听着它发出的 `error` 事件。

通过使用一个在作用域内未定义的变量 `x` 作乘法，我们在 Worker 内部（`workerWithError.js` 文件内）故意制造了一个异常。这个异常会被传递到最初创建 Worker 的 scrpit 中，同时调用 `onError` 函数。

#### Web Worker 的最佳实践

So far we’ve listed the strengths and limitations of Web Workers. Let’s see now what are the strongest use-cases for them:

* **Ray tracing**: ray tracing is a [rendering](https://en.wikipedia.org/wiki/Rendering_%28computer_graphics%29 "Rendering (computer graphics)") technique for generating an image by tracing the path of [light](https://en.wikipedia.org/wiki/Light "Light") as pixels. Ray tracing uses very CPU-intensive mathematical computations in order to simulate the path of light. The idea is to simulate some effects like reflection, refraction, materials, etc. All this computational logic can be added to a Web Worker to avoid blocking the UI thread. Even better — you can easily split the image rendering between several workers (and respectively between several CPUs). Here is a simple demo of ray tracing using Web Workers — [https://nerget.com/rayjs-mt/rayjs.html](https://nerget.com/rayjs-mt/rayjs.html).

* **Encryption:** end-to-end encryption is getting more and more popular due to the increasing rigorousness of regulations on personal and sensitive data. Encryption can be a something quite time-consuming, especially if there’s a lot of data that has to be frequently encrypted (before sending it to the server, for example). This is a very good scenario in which a Web Worker can be used since it doesn’t require any access to the DOM or anything fancy — it’s pure algorithms doing their job. Once in the worker, it is seamless to the end user and doesn’t impact thеir experience.

* **Prefetching data:** in order to optimize your website or web application and improve data loading time, you can leverage Web Workers to load and store some data in advance so that you can use it later when needed. Web Workers are amazing in this case because they won’t impact your app’s UI, unlike when this is done without workers.

* **Progressive Web Apps:** they have to load quickly even when the network connection is shaky. This means that data has to be stored locally in the browser. This is where [IndexDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) or similar APIs comes into play. Basically, a client-side storage is needed. In order to be used without blocking the UI thread, the work has to be done in Web Workers. Well, in the case of IndexDB, there is an asynchronous API that allows you to do this even without workers, but there was a synchronous API before (it might be introduced again) which should only be used inside workers.

* **Spell checking:** a basic spell checker works in the following way — the program reads a dictionary file with a list of correctly spelled words. The dictionary is being parsed as a search tree to make the actual text search-efficient. When a word is provided to the checker, the program checks whether it exists in the pre-built search tree. If the word is not found in the tree, the user can be provided with alternate spellings, by substituting alternate characters and test if it’s a valid word — if it’s the word that the user wanted to write. All this processing can easily be offloaded to a Web Worker so that the user can just type words and sentences without any blocking of the UI, while the worker performs all the searching and providing of suggestions.

Performance and reliability are very critical for us at [SessionStack](https://www.sessionstack.com/?utm_source=medium&utm_medium=source&utm_content=javascript-series-web-workers-outro). The reason why they’re so important is that once SessionStack is integrated into your web app, it starts recording everything from DOM changes and user interaction to network requests, unhandled exceptions and debug messages. All this data is transmitted to our servers in **real-time** which allows you to replay issues from your web apps as videos and see everything that happened to your users. This all takes place with minimum latency and no performance overhead for your app.

This is why we’re offloading (wherever it makes sense) logic from both our monitoring library and our player to Web Workers that are handling very CPU-intensive tasks like hashing to validate data integrity, rendering, etc.

Web technologies constantly change and develop so We go the extra mile to ensure SessionStack is very lightweight and has zero performance impact on our users’ apps.

There is a free plan if you’d like to [give SessionStack a try](https://www.sessionstack.com/?utm_source=medium&utm_medium=source&utm_content=javascript-series-web-workers-try-now).

![](https://cdn-images-1.medium.com/max/800/1*YKYHB1gwcVKDgZtAEnJjMg.png)

#### Resources

* [https://www.html5rocks.com/en/tutorials/workers/basics/](https://www.html5rocks.com/en/tutorials/workers/basics/)
* [https://hacks.mozilla.org/2015/07/how-fast-are-web-workers/](https://hacks.mozilla.org/2015/07/how-fast-are-web-workers/)
*   [https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API](https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API)


---

> [掘金翻译计划](https://github.com/xitu/gold-miner) 是一个翻译优质互联网技术文章的社区，文章来源为 [掘金](https://juejin.im) 上的英文分享文章。内容覆盖 [Android](https://github.com/xitu/gold-miner#android)、[iOS](https://github.com/xitu/gold-miner#ios)、[前端](https://github.com/xitu/gold-miner#前端)、[后端](https://github.com/xitu/gold-miner#后端)、[区块链](https://github.com/xitu/gold-miner#区块链)、[产品](https://github.com/xitu/gold-miner#产品)、[设计](https://github.com/xitu/gold-miner#设计)、[人工智能](https://github.com/xitu/gold-miner#人工智能)等领域，想要查看更多优质译文请持续关注 [掘金翻译计划](https://github.com/xitu/gold-miner)、[官方微博](http://weibo.com/juejinfanyi)、[知乎专栏](https://zhuanlan.zhihu.com/juejinfanyi)。
