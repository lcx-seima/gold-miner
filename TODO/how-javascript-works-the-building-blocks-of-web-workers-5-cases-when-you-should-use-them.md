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

让我们来看看一个简单的函数，它用于计算数值型数组的均值：

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

Web Worker 最终实现为一系列的 `.js` 文件，网页会通过异步 HTTP 请求加载它们。当然 [Web Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) 包办了这一切，上述过程对使用者完全无感。

Workers utilize thread-like message passing to achieve parallelism. They’re perfect for keeping your UI up-to-date, performant, and responsive for users.

Web Workers run in an isolated thread in the browser. As a result, the code that they execute needs to be contained in a **separate file**. That’s very important to remember.

Let’s see how a basic worker is created:

```
var worker = new Worker('task.js');
```

If the “task.js” file exists and is accessible, the browser will spawn a new thread which downloads the file asynchronously. Right after the download is completed, it will be executed and the worker will begin.
In case the provided path to the file returns a 404, the worker will fail silently.

In order to start the created worker, you need to invoke the `postMessage` method:

```
worker.postMessage();
```

#### Web Worker communication

In order to communicate between a Web Worker and the page that created it, you need to use the `postMessage` method or a [Broadcast Channel](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel).

#### The postMessage method

Newer browsers support a `JSON` object as a first parameter to the method while older browsers support just a `string`.

Let’s see an example of how the page that creates a worker can communicate back and forth with it, by passing a JSON object as a more “complicated” example. Passing a string is quite the same.

Let’s take a look at the following HTML page (or part of it to be more precise):

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

And this is how our worker script will look like:

```
self.addEventListener('message', function(e) {
  var data = e.data;
  switch (data.cmd) {
    case 'average':
      var result = calculateAverage(data); // Some function that calculates the average from the numeric array.
      self.postMessage(result);
      break;
    default:
      self.postMessage('Unknown command');
  }
}, false);
```

When the button is clicked, `postMessage` will be called from the main page. The `worker.postMessage` line passes the `JSON` object to the worker, adding `cmd` and `data` keys with their respective values. The worker will handle that message through the defined `message` handler.

When the message arrives, the actual computing is being performed in the worker, without blocking the event loop. The worker is checking the passed event `e` and executes just like a standard JavaScript function. When it’s done, the result is passed back to the main page.

In the context of a worker, both the `self` and `this` reference the global scope for the worker.

> There are two ways to stop a worker: by calling `worker.terminate()` from the main page or by calling `self.close()` inside of the worker itself.

#### Broadcast Channel

The [Broadcast Channel](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel) is a more general API for communication. It lets us broadcast messages to all contexts sharing the same origin. All browser tabs, iframes, or workers served from the same origin can emit and receive messages:

```
// Connection to a broadcast channel
var bc = new BroadcastChannel('test_channel');

// Example of sending of a simple message
bc.postMessage('This is a test message.');

// Example of a simple event handler that only
// logs the message to the console
bc.onmessage = function (e) { 
  console.log(e.data); 
}

// Disconnect the channel
bc.close()
```

And visually, you can see what Broadcast Channels look like to make it more clear:

![](https://cdn-images-1.medium.com/max/800/1*NVT6WbNrH_mQL64--b-l1Q.png)

Broadcast Channel has more limited browser support though:

![](https://cdn-images-1.medium.com/max/800/1*81mCsOzyJj-HfQ1lP_033w.png)

#### The size of messages

There are 2 ways to send messages to Web Workers:

* **Copying the message:** the message is serialized, copied, sent over, and then de-serialized at the other end. The page and worker do not share the same instance, so the end result is that a duplicate is created on each pass. Most browsers implement this feature by automatically JSON encoding/decoding the value at either end. As expected, these data operations add significant overhead to the message transmission. The bigger the message, the longer it takes to be sent.
* **Transferring the message:** this means that the original sender can no longer use it once sent. Transferring data is almost instantaneous. The limitation is that only [ArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer) is transferable.

#### Features available to Web Workers

Web Workers have access **only to a subset** of JavaScript features due to their multi-threaded nature. Here’s the list of features:

* The `navigator` object
* The `location` object (read-only)
* `XMLHttpRequest`
* `setTimeout()/clearTimeout()` and `setInterval()/clearInterval()`
* The [Application Cache](https://www.html5rocks.com/tutorials/appcache/beginner/)
* Importing external scripts using `importScripts()`
* [Creating other web workers](https://www.html5rocks.com/en/tutorials/workers/basics/#toc-enviornment-subworkers)

#### Web Worker limitations

Sadly, Web Workers don’t have access to some very crucial JavaScript features:

* The DOM (it’s not thread-safe)
* The `window` object
* The `document` object
* The `parent` object

This means that a Web Worker can’t manipulate the DOM (and thus the UI). It can be tricky at times, but once you learn how to properly use Web Workers, you’ll start using them as separate “computing machines” while all the UI changes will take place in your page code. The Workers will do all the heavy lifting for you and once the jobs are done, you’ll pass the results to the page which makes the necessary changes to the UI.

#### Handling errors

As with any JavaScript code, you’ll want to handle any errors that are thrown in your Web Workers. If an error occurs while a worker is executing, the `ErrorEvent` is fired. The interface contains three useful properties for figuring out what went wrong:

* **filename** - the name of the worker script that caused the error
* **lineno** - the line number where the error occurred
* **message** - a description of the error

This is an example:

```
function onError(e) {
  console.log('Line: ' + e.lineno);
  console.log('In: ' + e.filename);
  console.log('Message: ' + e.message);
}

var worker = new Worker('workerWithError.js');
worker.addEventListener('error', onError, false);
worker.postMessage(); // Start worker without a message.
```

```
self.addEventListener('message', function(e) {
  postMessage(x * 2); // Intentional error. 'x' is not defined.
};
```

Here, you can see that we created a worker and started listening for the `error` event.

Inside the worker (in `workerWithError.js`) we create an intentional exception by multiplying `x` by 2 while `x` is not defined in that scope. The exception is propagated to the initial script and `onError` is being invoked with information about the error.

#### Good use cases for Web Workers

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
