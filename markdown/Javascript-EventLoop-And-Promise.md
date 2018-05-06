# Javascript的事件模型和Promise实现
`@bill_shooting` `2018-04-20` `字数 2291` [`Follow me on`<i class="icon-github"></i>][1]
标签（空格分隔）： javascript

---
**1. Javascript的运行时模型——事件循环**
JS的运行时是个单线程的运行时，它不像其他编程语言，比如C++，Java，C#这些可以进行多线程操作的语言。当它执行一个函数时，它只会一条路走到黑，不会在当前函数结束之前去调用其他的函数（除非当前函数主动调用其他函数）。它也不用担心会有其他线程打扰它，因为它的运行时只有一个线程。如果你还记得一些计算机原理的话，这种运行时只有一个栈，设计起来相当的简单。

一条路走到黑的设计很棒，因为它足够简单，但是又是谁决定哪个函数从开始进入栈内执行呢？答案是JS的运行时还有一个**事件等待队列**与栈搭配，每当运行栈为空时（也就是当前函数运行结束），JS的运行时就从当前的事件队列中取出一个消息处理，执行与这个消息相关联的函数。这种行为可以用以下代码来说明：
```Javascript
while (eventQueue.waitForMessage()) {
    let event = eventQueue.pop();
    let handler = event.handler;
    handler();                      //执行事件关联的函数
    context.scheduler.schedule();   //让调度器处理一下其他事务
}
```
有了运行栈和事件队列之后，我们的Javascript运行时已经初具雏形。不过Javascript中的变量都是对象，它们的大小通常很大，可不是一个小小的栈能放下的，如果我们熟悉C++，就会知道一般在C++中我们只在栈中存储基本类型（int, bool等）和指针，而指针所指的位置是内存堆中的一个地址，这也是JS的对象的存储地点。下面这张图可以形象地解释一下JS运行时的模型。
![JS的事件循环模型][2]

**2. 事件循环模型的优点和缺点**
**先说优点**。除了实现上的简单，Javascript的最大优点就是**完全异步，永不阻塞**。这句话可能有点令人迷糊，一个单线程的运行时怎么完全异步，永不阻塞？实际上虽然JS运行时单线程，但是浏览器是个多进程多线程的环境，这一个点在后端也一样，虽然Node是个单线程JS运行时，但是后端还有其他进程和线程配合Node一起完成响应操作。

以浏览器打开IndexedDB为例，当你执行`indexedDB.open()`的之后，当前的Javascript运行栈就结束了，JS可以处理其他事件的关联函数，所以JS不会阻塞。那原来的open操作交给谁了呢？浏览器会调用其他线程接管这个打开数据库的过程，当返回时，浏览器会在JS运行时的事件队列中添加一个`打开成功`或者`打开失败`的事件，同时将你当时添加的回调函数关联到事件。

**再说缺点**。我们都知道JS的调用函数只会一条路走到黑，而且没有正常的方法能打断这一过程，如果这一路恰好比较长（比如进行了大量的数学运算），就会使JS进入一种**类阻塞**的状态，页面会无法响应。等等！刚说JS永不阻塞，这里怎么又冒出一个类阻塞呢？这时因为我们所说的阻塞一般都是指**IO阻塞**，也就是CPU等IO结束的过程，这种情况在JS中**可以**永远不会发生（注意这里是可以，不是一定，某些IO操作是有同步的API可以调用的）。所谓类阻塞状态呢，就是在执行CPU密集型任务，这是一种不可避免的过程。那为什么这种情况下页面会没有响应呢？这时因为浏览器虽然会把事件放入事件队列里，但是由于前一个函数还没执行完，页面响应事件关联的函数得不到执行，自然页面会表现出不响应的状态。

**3. Promise的实现**
Promise是JS处理回调的一种方式，也是利用JS的事件循环模型的一个编程范式包装，也是ES7中`await` `async`的基础，下面给个例子来说明它和JS事件模型的联系。
```JavaScript
let getUrlAsync = (url) => {
    let promise = new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest();
        xhr.open('GET', url);
        xhr.onload = () => resolve(xhr.responseText);
        xhr.onerror = () => reject(xhr.statusText);
    });
    return promise;
};

getUrlAsync('http://exaple.com/text/11111')
    .then(res => console.log(res))
    .catch(error => console.log(error));
```
当调用`getUrlAsync`时，JS运行时做以下事情：
1. 会创建一个`Promise`对象，此时还是在`getUrlAsync`的栈帧里；
2. 然后创建一个`XMLHttpRequest`对象，此时还是在`getUrlAsync`的栈帧里；
3. 调用`XMLHttpRequest`的`open`方法，此时浏览器其他线程接管open过程，JS无需等待open结束；
4. 给`xhr`的`onload`事件关联一个处理函数(委托），注意此时该事件并没有进入事件队列；
5. 给`xhr`的`onerror`事件关联一个处理函数(委托），同样此时该事件没有进入运行时的事件队列；
6. 传入`res => console.log(res)`来具体化第4步中的委托；
7. 传入`error => console.log(error)`来具体化第5部中的委托，**此时当前的运行栈就退出了，运行时将处理其他事件**。
在某一个时刻，浏览器控制的open方法返回，它会在JS运行时的事件队列中添加一个事件，比如`onload`
8. JS运行时循环到`onload`事件，并找到它的关联处理函数，并运行这个函数。


  [1]: https://github.com/billshooting
  [2]: https://developer.mozilla.org/files/4617/default.svg