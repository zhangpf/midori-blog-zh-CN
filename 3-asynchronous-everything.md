---
title: Midori博客系列翻译（3）——一切皆异步
date: 2018-11-25 18:03:00
tags: [操作系统, Midori, 翻译, 异步, 并发]
categories: 中文
---

<!-- 
Midori was built out of many ultra-lightweight, fine-grained processes, connected through strongly typed message passing
interfaces.  It was common to see programs that'd've classically been single, monolithic processes -- perhaps with some
internal multithreading -- expressed instead as dozens of small processes, resulting in natural, safe, and largely
automatic parallelism.  Synchronous blocking was flat-out disallowed.  This meant that literally everything was
asynchronous: all file and network IO, all message passing, and any "synchronization" activities like rendezvousing
with other asynchronous work.  The resulting system was highly concurrent, responsive to user input, and scaled like the
dickens.  But as you can imagine, it also came with some fascinating challenges. 
-->
Midori由大量通过强类型消息传递接口相互连接的，超轻量级细粒度进程构建而成。
我们传统上常见的程序是可能带有一些内部线程的单一宏进程。
而在Midori中，进程则由数十个小型进程所表示，
从而实现自然，安全和大部分的自动并行化。
同时，在Midori中，显然同步阻塞是不允许的，
这意味着包括所有文件和网络I/O，消息传递，
以及诸如与其他异步任务进行会合等任何“同步”活动，在字面上的一切都是异步的。
这样的设计使得所实现的系统可高度并发，且能快速响应用户的输入，以及灵活的扩展。
但也正如你想象的那样，这在设计上也带来了一些具有吸引力的挑战。

<!-- 
## Asynchronous Programming Model 
-->
## 异步编程模型

<!-- 
The asynchronous programming model looked a lot like C#'s async/await on the surface. 
-->
乍一眼看，异步编程模型很像C#的async/await。

<!-- 
That's not a coincidence.  I was the architect and lead developer on [.NET tasks](
https://en.wikipedia.org/wiki/Parallel_Extensions).  As the concurrency architect on Midori, coming off just shipping
the .NET release, I admit I had a bit of a bias.  Even I knew what we had wouldn't work as-is for Midori, however, so
we embarked upon a multi-year journey.  But as we went, we worked closely with the C# team to bring some of Midori's
approaches back to the shipping language, and had been using a variant of the async/await model for about a year when C#
began looking into it.  We didn't bring all the Midori goodness to .NET, but some of it certainly showed up, mostly in
the area of performance.  It still kills me that I can't go back in time and make .NET's task a struct. 
-->
而这并不是巧合，因为我也是
[.NET task](https://en.wikipedia.org/wiki/Parallel_Extensions)的架构师和开发主管。
作为Midori的并发设计者，对于即将发布的.NET新版本，我必须承认我对异步编程模型存有偏爱。 
因此，即使知道对于Midori，它可能不会像预期的那样发展，但我们依旧开始了数年的开发旅程。
而当离开时，我们与C#团队密切合作，将一些在Midori上形成的方法带回到C#中，
并在C#开始探索异步模型时，也使用了async/await模型的变体大约一年之久。
虽然我们未能将Midori的所有优点带到.NET中，但也确实实现了其中的一部分，而这部分主要是在性能方面。 
而对于无法回到过去将.NET的task变成结构体，这一点至今让我感到遗憾。

<!-- 
But I'm getting ahead of myself.  The journey to get to this point was a long one, and we should start at the beginning. 
-->
但我却逐步超越了我自己。到达这一步的旅程是漫长的，让我们从头讲起。

## Promises

<!-- 
At the core of our asynchronous model was a technology called [promises](
https://en.wikipedia.org/wiki/Futures_and_promises).  These days, the idea is ubiquitous.  The way we used promises,
however, was more interesting, as we'll start to see soon.  We were heavily influenced by the [E system](
https://en.wikipedia.org/wiki/E_(programming_language)).  Perhaps the biggest difference compared to popular
asynchronous frameworks these days is there was no cheating.  There wasn't a single synchronous API available. 
-->
我们使用的异步模型的核心是一项名为[Promise](https://en.wikipedia.org/wiki/Futures_and_promises)的技术。
虽然在如今，该想法已是无处不在，
但是，你很快将会看到，我们所使用Promise的方式将更加有趣。
受到[E语言系统](https://en.wikipedia.org/wiki/E_(programming_language))的强烈影响，
使得与流行的异步框架相比，我们最大的不同在于做到了完全的异步——例如，在我们的系统中，没有一个同步的API。

<!-- 
The first cut at the model used explicit callbacks.  This'll be familiar to anybody who's done Node.js programming.  The
idea is you get a `Promise<T>` for any operation that will eventually yield a `T` (or fail).  The operation producing
that may be running asynchronously within the process or even remotely somewhere else.  The consumer doesn't need to
know or care.  They just deal with the `Promise<T>` as a first class value and, when the `T` is sought, must rendezvous. 
-->
该模型的最重要的方式是使用了显式的回调，
而这对于任何使用过Node.js编程的人都非常熟悉。 
这里的想法是为任何最终会产生`T`（或失败）的操作产生一个`Promise<T>`。 
产生的操作可以在进程内异步运行，甚至可以远程执行。
结果的使用者无需知道具体运行的位置，他们只需要将`Promise<T>`作为一等类型值来处理，
也就是说当需要获取`T`值时，必须进行会合（rendezvous）操作。

<!-- 
The basic callback model started something like this: 
-->
其基本的回调模型如下：

<!-- 
    Promise<T> p = ... some operation ...;

    ... optionally do some things concurrent with that operation ...;

    Promise<U> u = Promise.When(
        p,
        (T t) => { ... the T is available ... },
        (Exception e) => { ... a failure occurred ... }
    ); 
-->
    Promise<T> p = ... 一些操作 ...;

    ... 可选地，与该操作并发地完成其他操作 ...;

    Promise<U> u = Promise.When(
        p,
        (T t) => { ... 当T变得可用时 ... },
        (Exception e) => { ... 产生失败时 ... }
    ); 

<!-- 
Eventually we switched over from static to instance methods: 
-->
最终，我们从静态方法切换到实例方法：
<!-- 
    Promise<U> u = p.WhenResolved(
        (T t) => { ... the T is available ... },
        (Exception e) => { ... a failure occurred ... }
    ); 
-->
    Promise<U> u = p.WhenResolved(
        (T t) => { ... T变得可用时 ... },
        (Exception e) => { ... 产生失败时 ... }
    ); 

<!-- 
Notice that the promises chain.  The operation's callbacks are expected to return a value of type `U` or throw an
exception, as appropriate.  Then the recipient of the `u` promise does the same, and so on, and so forth. 
-->
请注意这里的Promise链：
操作的回调返回类型为`U`的值或者根据需要抛出异常。
然后，值为`u`的Promise使用者也将如此操作，依此类推进行下去。

<!-- 
This is [concurrent](https://en.wikipedia.org/wiki/Concurrent_computing#Concurrent_programming_languages) [dataflow](
https://en.wikipedia.org/wiki/Dataflow_programming) programming.  It is nice because the true dependencies of operations
govern the scheduling of activity in the system.  A classical system often results in work stoppage not because of true
dependencies, but [false dependencies](https://en.wikipedia.org/wiki/Data_dependency), like the programmer just
happening to issue a synchronous IO call deep down in the callstack, unbeknownst to the caller. -->
这是一种[并发](https://en.wikipedia.org/wiki/Concurrent_computing#Concurrent_programming_languages)的[数据流](https://en.wikipedia.org/wiki/Dataflow_programming)编程方式，
其优点是操作的正确依赖关系控制着系统中活动的调度。 
经典系统经常停止工作，不是因为正确的依赖关系，
而是由于[错误的依赖关系](https://en.wikipedia.org/wiki/Data_dependency)所导致，
例如在调用堆栈的底层中发出同步I/O调用，而上层的调用者却对此一无所知。

<!-- 
In fact, this is one of the reasons your screen bleaches so often on Windows.  I'll never forget a paper a few years
back finding one of the leading causes of hangs in Outlook.  A commonly used API would occasionally enumerate Postscript
fonts by attempting to talk to the printer over the network.  It cached fonts so it only needed to go to the printer
once in a while, at unpredictable times.  As a result, the "good" behavior led developers to think it safe to call from
the UI thread.  Nothing bad happened during testing (where, presumably, the developers worked on expensive computers
with near-perfect networks).  Sadly, when the network flaked out, the result was 10 second hangs with spinning donuts
and bleachy white screens.  To this day, we still have this problem in every OS that I use. 
-->
事实上，这也是Windows系统上经常出现白屏的原因之一。 
对此，我依然记得几年前的一篇论文，它指出了Outlook中出现挂起的主要原因：
某个常用的API偶尔会尝试通过网络，与打印机进行通信来枚举Postscript字体。 
请求所花费的时间虽然不可预测，但由于系统缓存了字体，所以只需要偶尔真正向打印机发出请求。 
因此，这种表面“良好”的行为使开发人员误以为从UI线程进行调用是安全的。 
测试期间（开发人员在造价昂贵的计算机上使用近乎完美的网络时）没有任何不良的后果产生， 
但遗憾的是，当网络状况恶化时，其造成的后果是与旋转的“甜甜圈”（光标）和白屏相伴的10秒钟系统挂起。
而到目前为止，在我所使用的所有操作系统中，此问题依然存在。

<!-- 
The issue in this example is the possibility for high latency wasn't apparent to developers calling the API.  It was
even less apparent because the call was buried deep in a callstack, masked by virtual function calls, and so on.  In
Midori, where all asynchrony is expressed in the type system, this wouldn't happen because such an API would
necessarily return a promise.  It's true, a developer can still do something ridiculous (like an infinite loop on the
UI thread), but it's a lot harder to shoot yourself in the foot.  Especially when it came to IO. -->
上述举例中产生问题的原因是，调用API的开发人员不清楚可能会产生高延迟。
这种延迟甚至非常隐蔽，因为其深埋于调用栈中，被虚函数调用所掩盖。 
在Midori中，所有的异步都在类型系统中表示，上述的例子将不会发生，因为这样的API将返回一个Promise类型。 
没错，开发人员虽然仍可以做一些荒谬的事情（比如在UI线程上产生死循环），
但像这种搬起石头砸自己脚的事情将会变得困难得多，特别是当涉及到I/O操作时。

<!-- 
What if you didn't want to continue the dataflow chain?  No problem. 
-->
如果想停止链式数据流怎么办？这也没有问题。
<!-- 
    p.WhenResolved(
        ... as above ...
    ).Ignore(); 
-->
    p.WhenResolved(
        ... 如上 ...
    ).Ignore(); 

<!-- 
This turns out to be a bit of an anti-pattern.  It's usually a sign that you're mutating shared state. 
-->
结果证明这是一种反模式，它通常表明你正在改变共享状态。

<!-- 
The `Ignore` warrants a quick explanation.  Our language didn't let you ignore return values without being explicit
about doing so.  This specific `Ignore` method also addded some diagnostics to help debug situations where you
accidentally ignored something important (and lost, for example, an exception). 
-->
这里的`Ignore`需要一点解释是——除非你显式地这样做，编程语言不允许忽略其返回值。 
同时，这种特定的`Ignore`使用还添加了一些诊断功能，
以帮助调试你意外忽略的重要事项（和异常等失败情况）。

<!-- 
Eventually we added a bunch of helper overloads and APIs for common patterns: 
-->
最后，我们为常见的模式添加了一些作为辅助的重载和API：

<!-- 
    // Just respond to success, and propagate the error automatically:
    Promise<U> u = p.WhenResolved((T t) => { ... the T is available ... });

    // Use a finally-like construct:
    Promise<U> u = p.WhenResolved(
        (T t) => { ... the T is available ... },
        (Exception e) => { ... a failure occurred ... },
        () => { ... unconditionally executes ... }
    );

    // Perform various kinds of loops:
    Promise<U> u = Async.For(0, 10, (int i) => { ... the loop body ... });
    Promise<U> u = Async.While(() => ... predicate, () => { ... the loop body ... });

    // And so on. 
-->
    // 只响应成功情况，并自动抛出错误：
    Promise<U> u = p.WhenResolved((T t) => { ... T变得可用... });

    // 使用类似finally的结构：
    Promise<U> u = p.WhenResolved(
        (T t) => { ... T可用 ... },
        (Exception e) => { ... 产生失败时 ... },
        () => { ... 无条件执行... }
    );

    // 执行各种循环：
    Promise<U> u = Async.For(0, 10, (int i) => { ... 循环体 ... });
    Promise<U> u = Async.While(() => ... predicate, () => { ... 循环体 ... });

    // 等等。

<!-- 
This idea is most certainly not even close to new.  [Joule](https://en.wikipedia.org/wiki/Joule_(programming_language))
and [Alice](https://en.wikipedia.org/wiki/Alice_(programming_language)) even have nice built-in syntax to make the
otherwise clumsy callback passing shown above more tolerable. 
-->
基本上可以确定的是，这并不能算的上是新颖的想法。 
[Joule语言](https://en.wikipedia.org/wiki/Joule_(programming_language))和[Alice语言](https://en.wikipedia.org/wiki/Alice_(programming_language))甚至都已有了良好的内置语法支持，
使上述繁琐笨拙的回调传递方法变得更容易使用。

<!-- 
But it was not tolerable.  The model tossed out decades of familiar programming language constructs, like loops. 
-->
但无法容忍的是，该模型抛弃了数十年以来熟悉的编程语言结构，例如循环。

<!-- 
It got really bad.  Like really, really.  It led to callback soups, often nested many levels deep, and often in some
really important code to get right.  For example, imagine you're in the middle of a disk driver, and you see code like: 
-->
这一点真的真的很糟糕，因为它往往会导致代码遭遇回调困境，
通常出现在多层次的嵌套和一些非常重要故而必须正确的代码中。 
例如，假设你在磁盘驱动程序中看到如下代码：

    Promise<void> DoSomething(Promise<string> cmd) {
        return cmd.WhenResolved(
            s => {
                if (s == "...") {
                    return DoSomethingElse(...).WhenResolved(
                        v => {
                            return ...;
                        },
                        e => {
                            Log(e);
                            throw e;
                        }
                    );
                }
                else {
                    return ...;
                }
            },
            e => {
                Log(e);
                throw e;
            }
        );
    }

<!-- 
It's just impossible to follow what's going on here.  It's hard to tell where the various returns return to, what
throw is unhandled, and it's easy to duplicate code (such as the error cases), because classical block scoping isn't
available.  God forbid you need to do a loop.  And it's a disk driver -- this stuff needs to be reliable! 
-->
所以根本不可能在这样的代码中搞清楚所有的逻辑，
因为这很难判断所有return的返回位置和所有未处理异常，
并且很容易出现重复的代码（例如错误处理），因为经典的块作用域不再适用。
上帝禁止你使用循环，但这里是磁盘驱动程序，它需要可靠性的保证！

<!-- 
## Enter Async and Await
-->
## 进入到Async和Await的世界

<!-- 
[Almost](https://msdn.microsoft.com/en-us/library/hh156528.aspx) [every](http://tc39.github.io/ecmascript-asyncawait/)
[major](https://www.python.org/dev/peps/pep-0492/) [language](
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4134.pdf) now features async and/or await-like constructs.
We began wide-scale use sometime in 2009.  And when I say wide-scale, I mean it. 
-->
[几乎](https://msdn.microsoft.com/en-us/library/hh156528.aspx)
[所有](http://tc39.github.io/ecmascript-asyncawait/)
的[主要](https://www.python.org/dev/peps/pep-0492/)
[编程语言](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4134.pdf)
都已具有类似async和/或await的数据结构，
我们在2009年时，也已经开始对其大规模使用。
这里我说大规模使用时，我是认真的。

<!-- 
The async/await approach let us keep the non-blocking nature of the system and yet clean up some of the above usability
mess.  In hindsight, it's pretty obvious, but remember back then the most mainstream language with await used at scale
was F# with its [asynchronous workflows](
http://blogs.msdn.com/b/dsyme/archive/2007/10/11/introducing-f-asynchronous-workflows.aspx) (also see [this paper](
http://research.microsoft.com/apps/pubs/default.aspx?id=147194)).  And despite the boon to usability and productivity,
it was also enormously controversial on the team.  More on that later. 
-->
async/await的方法让我们的系统保持了非阻塞的性质，并消除了上述可用性方面的混乱， 
事后看来，这种优势是非常明显的，
但是请不要忘了，在当时，大规模使用的最主流语言还是F#及其[异步工作流](
http://blogs.msdn.com/b/dsyme/archive/2007/10/11/introducing-f-asynchronous-workflows.aspx) 
（另见[此文](http://research.microsoft.com/apps/pubs/default.aspx?id=147194)）。 
尽管async/await对可用性和生产力方面带来了提升，
但我们团队对其也存在巨大争议，对此，我将在稍后详细介绍。

<!-- 
What we had was a bit different from what you'll find in C# and .NET.  Let's walk through the progression from the
promises model above to this new async/await-based one.  As we go, I'll point out the differences. 
-->
我们所设计的async/await的与C#和.NET中的略有不同。 
所以让我们来看看从上面的Promise模型演进到新的基于async/await模型的过程，
并且在抽丝剥茧的过程中，逐渐指出其中的差异。

<!-- 
We first renamed `Promise<T>` to `AsyncResult<T>`, and made it a struct.  (This is similar to .NET's `Task<T>`, however
focuses more on the "data" than the "computation.")  A family of related types were born:
-->
我们首先将`Promise<T>`重命名为`AsyncResult<T>`，并将其作为结构体 
（这一点类似于.NET的`Task<T>`，但其更多地关注于“数据”而不是“计算”）。
据此产生了如下的一系列相关的类型：

<!-- 
* `T`: the result of a prompt, synchronous computation that cannot fail.
* `Async<T>`: the result of an asynchronous computation that cannot fail.
* `Result<T>`: the result of a a prompt, synchronous computation that might fail.
* `AsyncResult<T>`: the result of an asynchronous computation that might fail. 
-->
* `T`：即时同步计算的结果，并且不会导致失败
* `Async<T>`：异步计算的结果，并且不会导致失败
* `Result<T>`：可能导致失败的即时同步计算的结果值
* `AsyncResult<T>`：可能导致失败的异步计算的结果值

<!-- 
That last one was really just a shortcut for `Async<Result<T>>`. 
-->
最后一个实际上是`Async<Result<T>>`的别名。

<!-- 
The distinction between things that can fail and things that cannot fail is a topic for another day.  In summary,
however, our type system guaranteed these properties for us. 
-->
可能失败的值和不会导致失败的值之间的区别又将是另外一个主题。
但总之，我们的类型系统为我们保证了它们的属性。

<!--
Along with this, we added the `await` and `async` keywords.  A method could be marked `async`: 
-->
同时增加了`await`和`async`关键字，
如果一个方法被标记为`async`：

    async int Foo() { ... }

<!-- 
All this meant was that it was allowed to `await` inside of it: 
-->
那么意味着该方法的内部允许存在`await`关键字：

    async int Bar() {
        int x = await Foo();
        ...
        return x * x;
    }

<!-- 
Originally this was merely syntactic sugar for all the callback goop above, like it is in C#.  Eventually, however, we
went way beyond this, in the name of performance, and added lightweight coroutines and linked stacks.  More below. 
-->
正如它们在C#中的那样，`async`/`await`仅仅作为上述回调方法的语法糖而存在。 
但最终，我们从性能的角度出发，使得其存在意义更加广泛，并在其基础上添加了轻量级协程和链接栈。 
其有关的更多内容如下。

<!-- 
A caller invoking an `async` method was forced to choose: use `await` and wait for its result, or use `async` and
launch an asynchronous operation.  All asynchrony in the system was thus explicit: 
-->
调用`async`方法的调用者必须进行如下的二选一：使用`await`并等待其结果，或使用`async`并启动异步的操作。 
因此，系统中的所有异步操作都将显式进行：
<!-- 
    int x = await Bar();        // Invoke Bar, but wait for its result.
    Async<int> y = async Bar(); // Invoke Bar asynchronously; I'll wait later.
    int z = await y;            // ...like now.  This waits for Bar to finish. 
-->
    int x = await Bar();        // 调用Bar，并在等待其结果返回。
    Async<int> y = async Bar(); // 异步调用Bar，并在未来某个时刻处理。
    int z = await y;            // ... 正如即使计算，它将等待Bar操作完成。

<!-- 
This also gave us a very important, but subtle, property that we didn't realize until much later.  Because in Midori the
only way to "wait" for something was to use the asynchronous model, and there was no hidden blocking, our type system
told us the full set of things that could "wait."  More importantly, it told us the full set of things that could not
wait, which told us what was pure synchronous computation!  This could be used to guarantee no code ever blocked the
UI from painting and, as we'll see below, many other powerful capabilities. 
-->
直到很久以后我们才意识到，这样的作法给我们带来了非常重要但又微妙的优势。 
因为在Midori中，“等待”某个事件的唯一方法是使用异步模型，并且没有任何隐藏的阻塞代码是由类型系统提供的。
更重要的是，它告诉了我们所有无法用于等待的情况，并告知我们什么是纯粹的同步计算！
因此，这种方式可用于保证没有代码会阻塞UI绘制，
也正如将在如下看到的一样，它还包含许多其他强大的功能。

<!-- 
Because of the sheer magnitude of asynchronous code in the system, we embellished lots of patterns in the language that
C# still doesn't support.  For example, iterators, for loops, and LINQ queries: 
-->
由于系统中异步代码的绝对规模，我们在语言中增加了许多C#仍然不支持的模式，这包括迭代器（iterator），for循环和LINQ查询：

    IAsyncEnumerable<Movie> GetMovies(string url) {
        foreach (await var movie in http.Get(url)) {
            yield return movie;
        }
    }

<!-- 
Or, in LINQ style: 
-->
或使用LINQ风格：

    IAsyncEnumerable<Movie> GetMovies(string url) {
        return
            from await movie in http.Get(url)
            ... filters ...
            select ... movie ...;
    }

<!-- 
The entire LINQ infrastructure participated in streaming, including resource management and backpressure. 
-->
整个LINQ基础架构，包括资源管理和[背压（backpressure）](https://github.com/ReactiveX/RxJava/wiki/Backpressure)，也参与到流式计算中。

<!-- 
We converted millions of lines of code from the old callback style to the new async/await one.  We found plenty of bugs
along the way, due to the complex control flow of the explicit callback model.  Especially in loops and error handling
logic, which could now use the familiar programming language constructs, rather than clumsy API versions of them. 
-->
我们将数百万行的代码从旧的回调方式转移到新的async/await模式中。 
在显式回调模型的复杂控制流中，我们发现了大量的bug，
特别对于循环和错误处理逻辑而言，现在可使用熟悉的编程语言结构，
而不是笨拙的API方式加以实现。

<!-- 
I mentioned this was controversial.  Most of the team loved the usability improvements.  But it wasn't unanimous. 
-->
我已经提到过，对于这点是有争议的：
虽然团队中的大多数都乐见其在可用性上的改进，但也并非所有人都持此观点。

<!-- 
Maybe the biggest problem was that it encouraged a pull-style of concurrency.  Pull is where a caller awaits a callee
before proceeding with its own operations.  In this new model, you need to go out of your way to *not* do that.  It's
always possible, of course, thanks to the `async` keyword, but there's certainly a little more friction than the old
model. The old, familiar, blocking model of waiting for things is just an `await` keyword away. -->
也许最大问题是旧的回调模型采用了一种pull风格的并发，
在这种方式中，调用者在继续自己的操作之前需等待被调用者。 
而在这个新模型中，你*不*能再这样做，
当然，可能总是这归功于`async`关键字，但这肯定比旧模型带来更多的摩擦。
旧式熟悉的阻塞模型变成了一个简单的`await`关键字。

<!-- 
We offered bridges between pull and push, in the form of [reactive](https://rx.codeplex.com/)
`IObservable<T>`/`IObserver<T>` adapters.  I wouldn't claim they were very successful, however for side-effectful
actions that didn't employ dataflow, they were useful.  In fact, our entire UI framework was based on the concept of
[functional reactive programming](https://en.wikipedia.org/wiki/Functional_reactive_programming), which required a
slight divergence from the Reactive Framework in the name of performance.  But alas, this is a post on its own. 
-->
我们以[反应式](https://rx.codeplex.com/)的`IObservable<T>`/`IObserver<T>`适配器形式提供了pull和push之间的桥接。 
虽然不敢宣称它非常成功，但是对于未使用数据流的带有副作用的行为，它们是很有用的。 
实际上，我们整个UI框架都基于[函数式反应式编程](https://en.wikipedia.org/wiki/Functional_reactive_programming)的概念。以性能的名义，它与Reactive Framework略有不同， 
但对于此问题，需要一篇独立的文章进行描述，本文将不再展开。

<!-- 
An interesting consequence was a new difference between a method that awaits before returning a `T`, and one that
returns an `Async<T>` directly.  This difference didn't exist in the type system previously.  This, quite frankly,
annoyed the hell out of me and still does.  For example: 
-->
一个有趣的后果是在返回`T`之前的await和直接返回`Async<T>`之间产生了差异，而在先前在类型系统中并不存在。 
坦率地说，对此我感到非常烦恼直到现在。 
比如说：

    async int Bar()  { return await Foo(); }
    Async<int> Bar() { return async Foo(); }

<!-- 
We would like to claim the performance between these two is identical.  But alas, it isn't.  The former blocks and keeps
a stack frame alive, whereas the latter does not.  Some compiler cleverness can help address common patterns -- this is
really the moral equivalent to an asynchronous tail call -- however it's not always so cut and dry. 
-->
我们想表达这两者的表现形式是等价的，但是事实并非如此。 
前者阻塞并保持堆栈帧活跃，而后者却不能。 
一些编译器可以巧妙地解决这种常见的模式——这实际上是异步尾调用的精神上的等价物——但事情并不总是这么简单。

<!-- 
On its own, this wasn't a killer.  It caused some anti-patterns in important areas like streams, however.  Developers
were prone to awaiting in areas they used to just pass around `Async<T>`s, leading to an accumulation of paused stack
frames that really didn't need to be there.  We had good solutions to most patterns, but up to the end of the project
we struggled with this, especially in the networking stack that was chasing 10GB NIC saturation at wire speed.  We'll
discuss some of the techniques we employed below. 
-->
就其本身而言，这个问题不是非常严重。 
然而，它在流式处理等重要领域引起了一些反模式情况。 
开发人员倾向于在他们过去传递`Async<T>`的区域进行await，
从而导致不必存在的暂停堆栈帧的大量积累。 
我们对大多数的模式都有很好的解决方案，但直到项目结束时我们都在努力解决这个问题，
尤其是在追求10Gb网卡传输速度饱和度的网络堆栈中。 
我们将在下面讨论所采用的一些技术。

<!-- 
But at the end of this journey, this change was well worth it, both in the simplicity and usability of the model, and
also in some of the optimization doors it opened up for us. 
-->
但是在这次探索之旅结束时，这样的变化是非常值得的，
不仅是在模型的简单性和可用性方面有提升，而且是在为我们后续优化打开了大门。

<!-- 
## The Execution Model 
-->
## 执行模型

<!-- 
That brings me to the execution model.  We went through maybe five different models, but landed in a nice place. 
-->
这种变化让我首先经历的是执行模型。 
我们经历了五种不同的模型，都取得不错的结果。

<!-- A key to achieving asynchronous everything was ultra-lightweight processes.  This was possible thanks to [software
isolated processes (SIPs)](http://research.microsoft.com/apps/pubs/default.aspx?id=71996), building upon [the foundation
of safety described in an earlier post](http://joeduffyblog.com/2015/11/03/a-tale-of-three-safeties/). -->
实现“异步皆一切”的关键是超轻量级的进程。
这归功于在[一篇早期帖子中描述的安全基础](/2018/10/24/midori/1-a-tale-of-three-safeties/)所建立的[软件隔离过程（SIP）](http://research.microsoft.com/apps/pubs/default.aspx?id=71996)机制之上。

<!-- 
The absence of shared, mutable static state helped us keep processes small.  It's surprising how much address space is
burned in a typical program with tables and mutable static variables.  And how much startup time can be spent
initializing said state.  As I mentioned earlier, we froze most statics as constants that got shared across many
processes.  The execution model also resulted in cheaper stacks (more on that below) which was also a key factor.  The
final thing here that helped wasn't even technical, but cultural.  We measured process start time and process footprint
nightly in our lab and had a "ratcheting" process where every sprint we ensured we got better than last sprint.  A group
of us got in a room every week to look at the numbers and answer the question of why they went up, down, or stayed the
same. We had this culture for performance generally, but in this case, it kept the base of the system light and nimble. 
-->
无共享可变静态状态使得将进程保持尽量小。 
令人惊讶的是，在具有table和可变静态变量的典型程序中，有大量的地址空间被破坏，
同时会花费大量的启动时间来初始化这些共享区域。 
正如我之前提到的，我们将大多数的静态函数固化为在许多进程中共享的常量。
执行模型使得堆栈的开销变得更小（更多细节将在下文中提到），这也是一个关键因素。 
但是最后的贡献因素甚至却都不是技术，而是文化。 
我们在实验环境中每天测量进程的启动时间和资源的使用量，
并在优化的过程，确保我们每次冲刺都比上一次有所提高。 
我们中的一群人每周会进入一个房间，
对着各种数字并回答关于性能为什么上升/下降或保持不变的问题。 
对于这种关于性能的文化我们是普遍存在的，
并通过这种方式，保持了系统基础的轻量和灵活。

<!-- 
Code running inside processes could not block.  Inside the kernel, blocking was permitted in select areas, but remember
no user code ever ran in the kernel, so this was an implementation detail.  And when I say "no blocking," I really mean
it: Midori did not have [demand paging](https://en.wikipedia.org/wiki/Demand_paging) which, in a classical system, means
that touching a piece of memory may physically block to perform IO.  I have to say, the lack of page thrashing was such
a welcome that, to this day, the first thing I do on a new Windows sytem is disable paging.  I would much rather have
the OS kill programs when it is close to the limit, and continue running reliably, than to deal with a paging madness. 
-->
在进程内部运行的代码无法被阻塞， 
而在内核中，代码允许在指定区域中被阻塞，
但是请记住在内核中没有运行任何用户态的代码，
因此（内核代码在指定区域中被阻塞）仅是一个实现的细节。
这里当我说“没有阻塞”时，我是认真的：
Midori没有[按需换页](https://en.wikipedia.org/wiki/Demand_paging)的机制。
在传统的操作系统中，按需换页意味着对内存的访问可能会物理阻塞以执行I/O操作。
我不得不说的是，没有按需换页所带来的页面抖动是如此的受欢迎，
以至于直到了今天，我在新的Windows系统上所做的第一件事就是禁用分页。
与其愚蠢地进行换页操作，我更希望操作系统在内存不足时杀死进程，并继续可靠地运行。

<!-- 
C#'s implementation of async/await is entirely a front-end compiler trick.  If you've ever run ildasm on the resulting
assembly, you know: it lifts captured variables into fields on an object, rewrites the method's body into a state
machine, and uses Task's continuation passing machinery to keep the iterator-like object advancing through the states. 
-->
C#的async/await实现完全是一个前端的编译技巧。 
如果你曾在生成的汇编上运行过ildasm，那么你就知道：
它将捕获的变量提升到对象的字段中，将方法的主体重写为状态机形式，
并使用Task的继续传递机制来保持像迭代器这类的对象向前通过状态。

<!-- 
We began this way and shared some of our key optimizations with the C# and .NET teams.  Unfortunately, the result at
the scale of what we had in Midori simply didn't work. 
-->
我们以这种方式开始并与C#和.NET团队分享了一些关键的优化方法。 
但不幸的是，在Midori的规模上，这种方法根本无法工作。

<!-- 
First, remember, Midori was an entire OS written to use garbage collected memory.  We learned some key lessons that were
necessary for this to perform adequately.  But I'd say the prime directive was to avoid superfluous allocations like the
plague.  Even short-lived ones.  There is a mantra that permeated .NET in the early days: Gen0 collections are free.
Unfortunately, this shaped a lot of .NET's library code, and is utter hogwash.  Gen0 collections introduce pauses, dirty
the cache, and introduce [beat frequency issues](https://en.wikipedia.org/wiki/Beat_(acoustics)) in a highly concurrent
system.  I will point out, however, one of the tricks to garbage collection working at the scale of Midori was precisely
the fine-grained process model, where each process had a distinct heap that was independently collectible.  I'll have an
entire article devoted to how we got good behavior out of our garbage collector, but this was the most important
architectural characteristic. 
-->
首先，请记住，Midori是一个为使用内存垃圾回收而编写的完整操作系统，
对于充分发挥这一功能，我们收获一些必要的关键教训。 
但我要说的是，主要方向是即使是短寿命的，也需避免像瘟疫一样多余的内存分配。 
在早期有一个贯穿.NET的口头禅：Gen0代回收是免费的。 
不幸的是，这种理念已经塑造了很多.NET库代码，并且是彻头彻尾的贯穿。
Gen0的回收导致暂停和脏缓存，并在高度并发的系统中导致[拍频问题](https://en.wikipedia.org/wiki/Beat_(acoustics))。 
然而，我要指出的是，在Midori规模上进行垃圾收集工作的一个技巧
恰恰是细粒度的进程模型，因为其中每个进程都有一个独立的堆，可以独立地进行回收。 
我将有一篇专门介绍我们如何通过垃圾回收器获得良好表现的文章，但这是最重要的架构上的特征。

<!-- 
The first key optimization, therefore, is that an async method that doesn't await shouldn't allocate anything. 
-->
因此，首要的关键优化是：async方法不应该分配任何内存。

<!-- 
We were able to share this experience with .NET in time for C#'s await to ship.  Sadly, by then, .NET's Task had
already been made a class.  Since .NET requires async method return types to be Tasks, they cannot be zero-allocation
unless you go out of your way to use clumsy patterns like caching singleton Task objects. 
-->
我们及时与.NET分享了这一经验，以便于C#中await的发布。 
可惜的是，在那时，.NET的Task已经成为了一个类。 
由于.NET要求异步方法的返回类型为Task，使得它们无法零分配，
除非你不遗余力地使用类似缓存单例Task对象之类的笨拙模式。

<!-- 
The second key optimization was to ensure that async methods that awaited allocated as little as possible. 
-->
第二个关键优化是：确保await所分配的async方法尽可能少。

<!-- 
In Midori, it was very common for one async method to invoke another, which invoked another ... and so on.  If you think
about what happens in the state machine model, a leaf method that blocks triggers a cascade of O(K) allocations, where K
is the depth of the stack at the time of the await.  This is really unfortunate. 
-->
在Midori中，一个async方法被另一个async方法所调用是非常常见的，
同时它也可能调用了另一个async方法……依此类推下去。 
如果考虑状态机模型中所发生的情况，则阻塞的最末端方法会触发级联分配的复杂度为O(K)，
其中K是await发生时堆栈的深度，
而这将是非常糟糕的情况。

<!-- 
What we ended up with was a model that only allocated when the await happened, and that allocated only once for an
entire such chain of calls.  We called this chain an "activity."  The top-most `async` demarcated the boundary of an
activity.  As a result, `async` could cost something, but `await` was free. 
-->
我们最终采用的是一个只在await发生时分配的模型，
并且只为整个这样的调用链分配一次，我们将此链称为“activity”。
最顶层的`async`划分了activity的边界。 
因此，`async`可能会产生一些开销，但`await`却是零开销的。

<!-- 
Well, that required one additional step.  And this one was the biggie. 
-->
因此，这需要一个额外的步骤，而这一点却是最重要的。

<!-- 
The final key optimization was to ensure that async methods imposed as little penalty as possible.  This meant
eliminating a few sub-optimal aspects of the state machine rewrite model.  Actually, we abandoned it: 
-->
最终的关键优化是确保`async`方法尽可能地减小开销， 
这意味着它消除了状态机重写模型的一些不太理想的地方。 
但是在实际中，我们最终弃用了它，其原因在于：

<!-- 
1. It completely destroyed code quality.  It impeded simple optimizations like inlining, because few inliners consider
   a switch statement with multiple state variables, plus a heap-allocated display frame, with lots of local variable
   copying, to be a "simple method."  We were competing with OS's written in native code, so this matters a lot. 
-->
1. 它完全破坏了代码质量，
   阻碍了像内联这样的简单优化。
   因为很少有内联使用者认为，
   带有多个状态变量的switch语句，
   并加上堆分配的显示框架（display frame），
   再包含大量局部变量的复制的过程，是一个“简单的方法”。
   由于我们与使用原生代码编写的OS进行竞争，因此这一点很重要。
<!-- 
1. It required changes to the calling convention.  Namely, returns had to be `Async*<T>` objects, much like .NET's
   `Task<T>`.  This was a non-starter.  Even though ours were structs -- eliminating the allocation aspect -- they were
   multi-words, and required that code fetch out the values with state and type testing.  If my async method returns
   an int, I want the generated machine code to be a method that returns an int, goddamnit. 
-->
2. 它需对调用约定进行修改。
   也就是说，返回的必须是`Async*<T>`对象，就像.NET的`Task<T>`一样。 
   这并非易事，即使我们返回的是结构并消除了分配的问题，而它们返回的是多字且要求代码通过状态和类型测试来获取值。
   也就是说，如果我的`async`方法返回的是int值，那么我也希望生成的机器码也TMD返回int值。
   
<!-- 
1. Finally, it was common for too much heap state to get captured.  We wanted the total space consumed by an awaiting
   activity to be as small as possible.  It was common for some processes to end up with hundreds or thousands of them,
   in addition to some processes constantly switching between them.  For footprint and cache reasons, it was important
   that they remain as small as the most carefully hand-crafted state machine as possible. 
-->
3. 最后，一个普遍的问题是捕获的堆状态过多。
   而我们希望等待中的activity所消耗的总空间尽可能小，
   除了在它们之间不断切换的一些进程之外，
   一些进程通常最终会包含数百或数千个轻量级进程。 
   出于占用空间和缓存的原因，
   它们作为人工精心编写的状态机应保持越小越好，而这一点非常重要。

<!-- 
The model we built was one where asynchronous activities ran on [linked stacks](https://gcc.gnu.org/wiki/SplitStacks).
These links could start as small as 128 bytes and grow as needed.  After much experimentation, we landed on a model
where link sizes doubled each time; so, the first link would be 128b, then 256b, ..., on up to 8k chunks.  Implementing
this required deep compiler support.  As did getting it to perform well.  The compiler knew to hoist link checks,
especially out of loops, and probe for larger amounts when it could predict the size of stack frames (accounting for
inlining).  There is a common problem with linking where you can end up relinking frequently, especially at the edge of
a function call within a loop, however most of the above optimizations prevented this from showing up.  And, even if
they did, our linking code was hand-crafted assembly -- IIRC, it was three instructions to link -- and we kept a
lookaside of hot link segments we could reuse. 
-->

我们构建的模型是异步activity在[链接堆栈（linked stack）](https://gcc.gnu.org/wiki/SplitStacks)上运行。 
这些连接从小到128字节开始，并可根据需要进行增长。 
经过多次实验，我们采用了一个每次链接大小加倍的模型：
也就是说，首个链接大小是128字节，然后是256字节...直到最大为8K大小的块。 
对此功能的实现需要编译器的深度支持，它也确实表现良好。 
编译器知道何时进行链接检查，特别是在循环之外，并且
（考虑到内联）当它可以预测堆栈帧大小时会探测更大数量的链接。 
但是链接存在一个常见问题，即最终可能会进行频繁地重新链接，
尤其是在循环内函数调用的边缘处。
但是上述的大多数优化都阻止了这种情况的出现，
而且，即使这种情况发生了，我们的链接代码也是由人工编写的汇编代码——IIRC，用于链接的三个指令，
另外我们也一直关注于可以重用的活跃链接段。

<!-- 
There was another key innovation.  Remember, I hinted earlier, we knew statically in the type system whether a function
was asynchronous or not, simply by the presence of the `async` keyword?  That gave us the ability in the compiler to
execute all non-asynchronous code on classical stacks.  The net result was that all synchronous code remained
probe-free!  Another consequence is the OS kernel could schedule all synchronous code on a set of pooled stacks.  These
were always warm, and resembled a classical thread pool, more than an OS scheduler.  Because they never blocked, you
didn't have O(T) stacks, where T is the number of threads active in the entire system.  Instead, you ended up with O(P),
where P is the number of processors on the machine.  Remember, eliminating demand paging was also key to achieiving this
outcome.  So it was really a bunch of "big bets" that added up to something that was seriously game-changing. 
-->
除此之外，我们还获得了其他的重要创新。 
还记得我之前曾暗示过，仅通过`async`关键字可以判断出，
静态地知道函数在类型系统是否是异步的。
这使我们能够在编译器中执行经典堆栈上的所有非异步代码，
使得最终结果是所有同步代码都保持无探针状态！ 
而另一项创新是OS内核可以在一组堆栈池中调度所有同步代码， 
这组堆栈池总是处于活跃状态，这一点类似于经典的线程池，但不局限于OS调度程序。
因为线程从未被阻塞，所以不会导致堆栈成为O(T)，其中T是整个系统中活跃的线程数，
相反，你最终得到的堆栈大小是O(P)，其中P是机器上的处理器数量。 
回想一下，消除按需分页也是达到这一目标的关键，
所以这些真是一堆“大赌注”，所有这一切加起来足以形成革命性的成果。

<!-- 
## Message Passing 
-->
## 消息传递

<!-- 
A fundamental part of the system has been missing from the conversation: message passing. 
-->
到目前为止，系统中尚未提及的基础部分是：消息传递。

<!-- 
Not only were processes ultra-lightweight, they were single-threaded in nature.  Each one ran an [event loop](
https://en.wikipedia.org/wiki/Event-driven_programming) and that event loop couldn't be blocked, thanks to the
non-blocking nature of the system.  Its job was to execute a piece of non-blocking work until it finished or awaited,
and then to fetch the next piece of work, and so on.  An await that was previously waiting and became satisfied was
simply scheduled as another turn of the crank. 
-->
进程不仅是超轻量级的，而且在本质上是单线程的。 
由于系统的非阻塞特性，每个进程都运行了一个无法被阻塞的
[事件循环（event loop）](https://en.wikipedia.org/wiki/Event-driven_programming)。 
它的任务是执行一段非阻塞的程序，直到程序结束或通过await挂起等待，
然后再获取下一个任务，依此类推。
之前处于等待状态的await如果事件被满足，这将被调度为新的转动曲柄。

<!-- 
Each such turn of the crank was called, fittingly, a "turn." 
-->
曲柄的每次这样的转动被恰如其分地被称为“旋转”。

<!-- 
This meant that turns could happen between asynchronous activities and at await points, nowhere else.  As a result,
concurrent interleaving only occurred at well-defined points.  This was a giant boon to reasoning about state in the
face of concurrency, however it comes with some gotchas, as we explore later. 
-->
这意味着在旋转只可能发生在异步activity和await等待点之间，
使得并发的交织只可能发生在定义明确的点上。 
这对于面向并发的状态推理来讲是一个巨大的福音，
但它也带来了一些陷阱，对此我们将稍后进行探讨。

<!-- 
The nicest part of this, however, was that processes suffered no shared memory race conditions. -->
然而，这种方式最大的优势是进程间不存在共享内存的竞争条件。

<!-- 
We did have a task and data parallel framework.  It leveraged the concurrency safety features of the languge I've
mentioned previously -- [immutability, isolation, and readonly annotations](
http://research.microsoft.com/apps/pubs/default.aspx?id=170528) -- to ensure that this data race freedom was
not violated.  This was used for fine-grained computations that could use the extra compute power.  Most of the system,
however, gained its parallel execution through the decomposition into processes connected by message passing. 
-->
我们确实开发了一个任务和数据并行框架，
其利用了我之前提到的语言的并发安全功能——
[不变性，隔离和只读注解](http://research.microsoft.com/apps/pubs/default.aspx?id=170528)——
来确保不违反这种数据竞争的自由度，这被用来进行使用额外计算能力的细粒度计算。
然而，大多数系统通过分解成通过消息传递连接的进程来进行并行执行。

<!-- 
Each process could export an asynchronous interface.  It looked something like this: 
-->
每个进程都可导出异步的接口，它看起来类似如下：
<!--
    async interface ICalculator {
        async int Add(int x, int y);
        async int Multiply(int x, int y);
        // Etc...
    }
-->
    async interface ICalculator {
        async int Add(int x, int y);
        async int Multiply(int x, int y);
        // 等等...
    }
<!-- 
As with most asynchronous RPC systems, from this interface was generated a server stub and client-side proxy.  On the
server, we would implement the interface: 
-->
与大多数异步RPC系统一样，从该接口可以生成服务器stub和客户端代理的代码。 
于服务器端，我们需实现如下接口：
<!-- 
    class MyCalculator : ICalculator {
        async int Add(int x, int y) { return x + y; }
        async int Multiply(int x, int y) { return x * y; }
        // Etc...
    } 
-->
    class MyCalculator : ICalculator {
        async int Add(int x, int y) { return x + y; }
        async int Multiply(int x, int y) { return x * y; }
        // 等等...
    }

<!-- 
Each server-side object could also request [capabilities](https://en.wikipedia.org/wiki/Capability-based_security)
simply by exposing a constructor, much like the program's main entrypoint could, as I described in [the prior post](
http://joeduffyblog.com/2015/11/10/objects-as-secure-capabilities/).  Our application model took care of activating and
wiring up the server's programs and services. 
-->
正如我在[前一篇文章](/2018/11/18/midori/2-objects-as-secure-capabilities/)中所描述的那样，
每个服务器端对象也可以，像程序的主入口点那样，
仅地通过暴露构造函数的方式来请求获取[权能](https://en.wikipedia.org/wiki/Capability-based_security)。
而我们的应用程序模型则会处理好激活和连接服务端程序和服务。

<!-- 
A server could also return references to other objects, either in its own process, or a distant one.  The system
managed the object lifetime state in coordination with the garbage collector.  So, for example, a tree: 
-->
服务器还可以在其自己的进程或远程进程中返回其他对象的引用， 
而系统与垃圾回收器一起协调管理对象的生存周期状态。 
因此，例如有一个如下的MyTree对象：

    class MyTree : ITree {
        async ITree Left() { ... }
        async ITree Right() { ... }
    }

<!-- 
As you might guess, the client-side would then get its hands on a proxy object, connected to this server object
running in a process.  It's possible the server would be in the same process as the client, however typically the
object was distant, because this is how processes communicated with one another: 
-->
你可能已经猜到，客户端将获得代理对象的引用，
该对象连接到在进程中运行的此服务器对象。 
服务器可能与客户端处于相同的进程中，
但通常情况下对象可能物理位置相距很远，这是由进程间通信方式所决定的：

    class MyProgram {
        async void Main(IConsole console, ICalculator calc) {
            var result = await calc.Add(2, 2);
            await console.WriteLine(result);
        }
    }

<!-- 
Imagining for a moment that the calculator was a system service, this program would communicate with that system
service to add two numbers, and then print the result to the console (which itself also could be a different service). 
-->
想象一下如果Calculator是系统服务，
则该程序将与该系统服务通信来求两个数字之和，
而后将结果打印到控制台
（该过程本身也可能是另一个不同的服务）。

<!-- 
A few key aspects of the system made message passing very efficient.  First, all of the data structures necessary to
talk cross-process were in user-mode, so no kernel-mode transitions were needed.  In fact, they were mostly lock-free.
Second, the system used a technique called "[pipelining](
https://en.wikipedia.org/wiki/Futures_and_promises#Promise_pipelining)" to remove round-trips and synchronization
ping-ponging.  Batches of messages could be stuffed into channels before they filled up.  They were delivered in chunks
at-a-time.  Finally, a novel technique called "three-party handoff" was used to shorten the communication paths between
parties engaging in a message passing dialogue.  This cut out middle-men whose jobs in a normal system would have been
to simply bucket brigade the messages, adding no value, other than latency and wasted work. 
-->
系统的一些关键特征使得消息传递非常高效。 
首先，跨进程通信所需的所有数据结构都处于用户空间态，
因此不需要到内核模式进行转换，事实上，他们大多是无锁的。 
其次，该系统使用一种称为[“流水线”](https://en.wikipedia.org/wiki/Futures_and_promises#Promise_pipelining)的技术来消除消息往返和同步回传。 
在通道填充满之前，可以将批量消息填充到通道中，
使得消息每次一批一批地传递。 
最后，一种称为“三方切换”的新技术被用来缩短参与消息传递对话的各方之间的通信路径，
该方法省去了在通常系统中仅负责简单地中转消息得中间人，因为它除了增加延迟和开销之外不具有任何价值。

{% asset_img 2015-11-19-asynchronous-everything.pipeline.jpg  消息传递示意图 %}

<!-- 
The only types marshalable across message passing boundaries were: 
-->
跨消息传递边界的可编组类型（可消息传递的数据类型）包括：

<!-- 
* Primitive types (`int`, `string`, etc).
* Custom PODs that didn't contain pointers (explicitly labeled marshalable).
* References to streams (see below).
* References to other async objects (e.g., our `ICalculator` above).
* A special `SharedData` object, which requires a bit more explanation. 
-->
* 基本类型（`int`，`string`等）。
* 不包含指针（明确标记为可编组）的自定义POD。
* 指向流的引用（见下文）。
* 指向其他async对象的引用（例如，上文中的`ICalculator`）。
* 特殊的`SharedData`对象，对此需要更多的解释。

<!-- 
Most of these are obvious.  The `SharedData` thing is a little subtle, however.  Midori had a fundamental philosophy of
"zero-copy" woven throughout its fabric.  This will be the topic of a future post.  It's the secret sauce that let us
out-perform many classical systems on some key benchmarks.  The idea is, however, no byte should be copied if it can
be avoided.  So we don't want to marshal a `byte[]` by copy when sending a message between processes, for example.  The
`SharedData` was a automatic ref-counted pointer to some immutable data in a heap shared between processes.  The OS
kernel managed this heap memory and reclaimed it when all references dropped to zero.  Because the ref-counts were
automatic, programs couldn't get it wrong.  This leveraged some new features in our language, like destructors. 
-->
上述的大部分内容是显而易见的，
然而，`SharedData`则有点微妙。
Midori的整个构造过程都采用了“零拷贝”的基本理念，这是未来其中一篇文章的主题，
也是让我们在一些关键的基准测试中超越许多经典系统的秘诀。 
但其中主要的思路是，如果可以避免则不应该复制任何数据。 
因此，我们不希望在进程之间发送消息时以复制的方式编组`byte[]`。 
`SharedData`是一个自动重新计数的指针，指向进程之间共享堆中的不可变数据，
操作系统内核管理该堆内存并在所有引用计数为零时对其进行回收。
因为引用计数是自动完成的，所以保证了程序不会出错，
这充分利用了我们语言中的一些新功能，如析构函数等。

<!-- 
We also had the notion of "near objects," which went an extra step and let you marshal references to immutable data
within the same process heap.  This let you marshal rich objects by-reference.  For example: 
-->
同时，我们还有“近对象（near objects）”的概念，它采取了额外的步骤，
可在同一进程堆中对不可变数据的引用进行编组， 
使得可以通过引用来编组富对象。 
例如：

<!-- 
    // An asynchronous object in my heap:
    ISpellChecker checker = ...;

    // A complex immutable Document in my heap,
    // perhaps using piece tables:
    immutable Document doc = ...;

    // Check the document by sending messages within
    // my own process; no copies are necessary:
    var results = await checker.Check(doc); 
-->    
    // 堆中的异步对象：
    ISpellChecker checker = ...;

    // 堆中的复杂的不可变的Document对象，可能使用了其他的table：
    immutable Document doc = ...;

    // 通过在自身进程中发送消息来对Document进行check，而无需复制：
    var results = await checker.Check(doc); 

<!-- 
As you can guess, all of this was built upon a more fundamental notion of a "channel."  This is similar to what you'll
see in [Occam](https://en.wikipedia.org/wiki/Occam_(programming_language)), [Go](
https://en.wikipedia.org/wiki/Go_(programming_language)) and related [CSP](
https://en.wikipedia.org/wiki/Communicating_sequential_processes) languages.  I personally found the structure and
associated checking around how messages float around the system more comfortable than coding straight to the channels
themselves, but your mileage may vary.  The result felt similar to programming with [actors](
https://en.wikipedia.org/wiki/Actor_model), with some key differences around the relationship between process and
object identity. 
-->
你可以猜到，所有这些都建立在一个更为基本的“通道（channel）”概念之上，
这与你在[Occam](https://en.wikipedia.org/wiki/Occam_(programming_language))，
[Go](https://en.wikipedia.org/wiki/Go_(programming_language))和相关的CSP语言中看到的相类似。 
我个人发现消息是如何在系统周围浮动的结构和相关检查比直接编码到通道本身更舒适，
但你的自身体验可能会有所不同。
其结果与使用[Actor](https://en.wikipedia.org/wiki/Actor_model)的编程相类似，
只是在进程和对象身份之间的关系方面存在一些关键差异。

<!-- 
## Streams 
-->
## 流

<!-- 
Our framework had two fundamental stream types: `Stream` held a stream of bytes and `Sequence<T>` held `T`s.  They
were both forward-only (we had separate seekable classes) and 100% asynchronous. 
-->
我们的框架有两种基本的流类型：`Stream`持有一个字节流，
而`Sequence <T>`持有数据`T`。 
它们都是前向的（我们有其他的可seek类）和100%纯异步的。

<!-- 
Why two types, you wonder?  They began as entirely independent things, and eventually converged to be brother and
sister, sharing a lot of policy and implementation with one another.  The core reason they remained distinct, however,
is that it turns out when you know you're just schlepping raw byte-streams around, you can make a lot of interesting
performance improvements in the implementation, compared to a fully generic version. 
-->
你可能会问，为什么有两种类型？
它们从完全独立的类开始，最终收敛到一起，
彼此共享了许多策略及实现。
然而，它们保持完全不同的根本原因是，如果我们知道只是简单地处理原始字节流，
那么与完全泛型的版本相比，可以在实现中做出很多有趣的性能改进。

<!-- 
For purposes of this discussion, however, just imagine that `Stream` and `Sequence<byte>` are isomorphic. 
-->
但是，为了讨论方便，这里你需认为`Stream`和`Sequence<byte>`是相互同构的。

<!-- 
As hinted at earlier, we also had `IAsyncEnumerable<T>` and `IAsyncEnumerator<T>` types.  These were the most general
purpose interfaces you'd code against when wanting to consume something.  Developers could, of course, implement their
own stream types, especially since we had asynchronous iterators in the language.  A full set of asynchronous LINQ
operators worked over these interfaces, so LINQ worked nicely for consuming and composing streams and sequences. 
-->
如前所述，我们还有`IAsyncEnumerable<T>`和`IAsyncEnumerator<T>`类型。
这些是代码中想要对流进行消费时，需要进行编码的最通用接口。
当然，开发人员可以实现自己的流类型，特别是由于我们在编程语言中使用了异步的迭代器。 
一整套的异步LINQ运算符可以在这些接口上工作，因此LINQ可以很好地用于消费和组合流和序列。

<!-- 
In addition to the enumerable-based consumption techniques, all the standard peeking and batch-based APIs were
available.  It's important to point out, however, that the entire streams framework built atop the zero-copy
capabilities of the kernel, to avoid copying.  Every time I see an API in .NET that deals with streams in terms of
`byte[]`s makes me shed a tear.  The result is our streams were actually used in very fundamental areas of the system,
like the network stack itself, the filesystem the web servers, and more. 
-->
除了基于可枚举的流消费技术之外，还提供了所有标准*瞥见*和批处理的API。 
然而，需要重点指出的是，整个流框架是构建在内核的零拷贝基础之上，以避免不必要的复制。 
每当我在.NET中看到一个用`byte[]`来流处理的API时，我就会流下眼泪。 
最终的结果是我们的流实际用于系统非常基础的区域，
如网络堆栈本身，文件系统，Web服务器等。

<!-- 
As hinted at earlier, we supported both push and pull-style concurrency in the streaming APIs.  For example, we
supported generators, which could either style: 
-->
如前所述，我们支持流API中的推（push）式和拉（pull）式并发，
例如，我们支持采用以下方式的生成器：

<!-- 
    // Push:
    var s = new Stream(g => {
        var item = ... do some work ...;
        g.Push(item);
    });

    // Pull:
    var s = new Stream(g => {
        var item = await ... do some work ...;
        yield return item;
    }); 
-->
    // 推：
    var s = new Stream(g => {
        var item = ... 做一些事情 ...;
        g.Push(item);
    });

    // 拉：
    var s = new Stream(g => {
        var item = await ... 做一些事情 ；...;
        yield return item;
    }); 

<!-- 
The streaming implementation handled gory details of batching and generally ensuring streaming was as efficient as
possible.  A key technique was [flow control](https://en.wikipedia.org/wiki/Transmission_Control_Protocol), borrowed
from the world of TCP.  A stream producer and consumer collaborated, entirely under the hood of the abstraction, to
ensure that the pipeline didn't get too imbalanced.  This worked much like TCP flow control by maintaining a so-called
"window" and opening and closing it as availability came and went.  Overall this worked great.  For example, our
realtime multimedia stack had two asynchronous pipelines, one for processing audio and the other for processing video,
and merged them together, to implement [A/V sync](https://en.wikipedia.org/wiki/Audio_to_video_synchronization).  In
general, the built-in flow control mechanisms were able to keep them from dropping frames. 
-->
流的实现方法照本宣科地搞定了批处理的细节，
并大体上确保了流式传输尽可能的高效。 
一个关键技术是借鉴了TCP中的[流量控制](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)技术，
流的生产者和消费者完全在抽象的框架下进行合作，
以确保流水线不会过于的不平衡。
在这一点上很像TCP的流量控制，通过维护所谓的“窗口”，随着可用性的来去不断打开和关闭控制阀门。 
总的来说，这种方式很有效，
例如，我们的实时多媒体栈中有两个异步的流水线，
一个用于处理音频，另一个用于处理视频，
并通过将它们合并在一起，以实现[A/V同步](https://en.wikipedia.org/wiki/Audio_to_video_synchronization)。 
总的来说，内置的流量控制机制能够有效地防止丢帧的情况发生。

<!-- 
## "Grand" Challenges 
-->
## “重大”的挑战

<!-- 
The above was a whirlwind tour.  I've glossed over some key details, but hopefully you get the picture. 
-->
以上的内容像是一场匆忙的旅行， 
我故意忽略了其中一些关键细节，但希望你能获得整体的轮廓。

<!-- 
Along this journey we uncovered several "grand challenges."  I'll never forget them, as they formed the outline of my
entire yearly performance review for a good 3 years straight.  I was determined to conquer them.  I can't say that our
answers were perfect, but we made a gigantic dent in them. 
-->
在这次旅程中，我们发现了几个“重大的挑战”。 
我永远不会忘记它们，因为它们连续三年组成了我整个年度绩效评估的大体轮廓。 
我决心征服他们，虽然不能说我们的成果是完美的，
但我们确实在此做出了巨大的改进。

<!-- 
### Cancellation 
-->
### Cancellation（取消）
<!-- 
The need to have cancellable work isn't anything new.  I came up with the [`CancellationToken` abstraction in .NET](
http://blogs.msdn.com/b/pfxteam/archive/2009/05/22/9635790.aspx), largely in response to some of the challenges we had
around ambient authority with prior "implicitly scoped" attempts. 
-->
对可取消任务的需求已经不是什么新鲜事， 
对此我设计了[.NET中的CancellationToken抽象](http://blogs.msdn.com/b/pfxteam/archive/2009/05/22/9635790.aspx)，
其主要目的是解决我们之前围绕带有“隐式范围
（implicitly scoped）”的环境权限所面临一些挑战。

<!-- 
The difference in Midori was the scale.  Asynchronous work was everywhere.  It sprawled out across processes and,
sometimes, even machines.  It was incredibly difficult to chase down run-away work.  My simple use-case was how to
implement the browser's "cancel" button reliably.  Simply rendering a webpage involved a handful of the browser's own
processes, plus the various networking processes -- including the NIC's device driver -- along with the UI stack, and
more.  Having the ability to instantly and reliably cancel all of this work was not just appealing, it was required.
-->
Midori与.NET的不同之处在于其具有的规模，使得其异步的任务无处不在，
不仅跨越多个进程，甚至是还是跨主机之间的。 
追踪失控的任务是一件非常困难的是，
我的一个简单例子是，如何可靠地实现浏览器的“取消”按钮。 
由于简单地渲染网页便涉及数个浏览器自身的进程，
以及各种网络进程——这包括NIC的设备驱动程序，以及UI堆栈等等。 
能够立即可靠地对所有这些工作执行取消操作不仅具有吸引力，并且也是必需的。

<!-- The solution ended up building atop the foundation of `CancellationToken`. -->
其解决方案最终建立在`CancellationToken`的基础之上。

<!-- 
They key innovation was first to rebuild the idea of `CancellationToken` on top of our overall message passing model,
and then to weave it throughout in all the right places.  For example: 
-->
其关键创新在于首次在我们的整体消息传递模型上重建了`CancellationToken`的思想，
并在所有正确的位置使用它。 
例如：

<!-- 
* CancellationTokens could extend their reach across processes.
* Whole async objects could be wrapped in a CancellationToken, and used to trigger [revocation](
  http://c2.com/cgi/wiki?RevokableCapabilities).
* Whole async functions could be invoked with a CancellationToken, such that cancelling propagated downward.
* Areas like storage needed to manually check to ensure that state was kept consistent. 
-->
* CancellationToken可以扩展它们的范围实现跨进程；
* 整个async对象可包含在CancellationToken中，并用于触发[撤销操作](http://c2.com/cgi/wiki?RevokableCapabilities)；
* 可使用CancellationToken调用整个async函数，以便取消其向下传播；
* 存储像这样的区域需进行手动检查以确保状态保持一致。

<!-- 
In summary, we took a "whole system" approach to the way cancellation was plumbed throughout the system, including
extending the reach of cancellation across processes.  I was happy with where we landedon this one. 
-->
总之，我们采用了“整体系统”的方法来解决整个系统中的取消问题，
包括扩展跨进程取消的范围。 
我对我们采用的这种思路非常满意。

<!-- 
### State Management 
-->
### 状态管理

<!-- 
The ever-problematic "state management" problem can be illustrated with a simple example: 
-->
可以通过一个简单的例子来说明可能会出问题的“状态管理”：

    async void M(State s) {
        int x = s.x;
        await ... something ...;
        assert(x == s.x);
    }

<!-- 
The question here is, can the assertion fire? 
-->
那么问题是，断言会触发吗？

<!-- 
The answer is obviously yes.  Even without concurrency, reentrancy is a problem.  Depending on what I do in "...
something ...", the `State` object pointed to by `s` might change before returning back to us. -->
答案显然是肯定的。 
即使没有并发，重入也会是一个问题。 
根据我在“await ... something ...”语句中所做的事情，
`s`所指向的`State`对象可能会在返回之前发生变化。

<!-- 
But somewhat subtly, even if "... something ..." doesn't change the object, we may find that the assertion fires.
Consider a caller: 
-->
但有些微妙的是，即使“await ... something ...”没有改变对象，
断言可能也会触发，考虑如下的一个调用：

    State s = ...;
    Async<void> a = async M(s);
    s.x++;
    await a;

<!-- 
The caller retains an alias to the same object.  If M's awaiting operation must wait, control is resumed to the caller.
The caller here then increments `x` before awaiting M's completion.  Unfortunately, when M resumes, it will discover
that the value of `x` no longer matches `s.x`. 
-->
调用者保留同一对象的别名。 
如果等待中操作M需要等待，则控制权将重新交还给调用者， 
而后，调用者在等待M完成之前将x递增。 
不幸的是，当M恢复时，它会发现x的值不再与s.x的值相匹配。

<!-- 
This problem manifests in other more devious ways.  For example, imagine one of those server objects earlier: 
-->
这个问题以其他更加隐蔽的方式表现出来。 
例如，回想一下之前的那些服务对象：

<!-- 
    class StatefulActor : ISomething {
        int state;
        async void A() {
            // Use state
        }
        async void B() {
            // Use state
        }
    } 
-->
    class StatefulActor : ISomething {
        int state;
        async void A() {
            // 使用状态A
        }
        async void B() {
            // 使用状态B
        }
    } 

<!-- 
Imagining that both A and B contain awaits, they can now interleave with one another, in addition to interleaving with
multiple activations of themselves.  If you're thinking this smells like a race condition, you're right.  In fact,
saying that message passing systems don't have race conditions is an outright lie.  There have even been papers
discussing this [in the context of Erlang](https://www.it.uu.se/research/group/hipe/dialyzer/publications/races.pdf).
It's more correct to say our system didn't have *data race* conditions. 
-->
设想一下，A和B都包含await操作，除了多次自我激活的交错之外，它们还可相互交错。 
如果你认为它们看起来像是一对竞争条件，那你猜对了。 
事实上，如果说消息传递系统没有竞争条件是一个彻头彻尾的谎言，
甚至有一些论文在[Erlang的背景下](https://www.it.uu.se/research/group/hipe/dialyzer/publications/races.pdf)讨论该问题。 
因此，说我们的系统没有数据竞争条件更为正确。

<!-- 
Anyway, there be dragons here. 
-->
无论如何，这里是有竞争冒险存在的。

<!-- 
The solution is to steal a page from classical synchronization, and apply one of many techniques: 
-->
其解决方案也从经典的同步方法中得到启发，并应用以下多种技术之一：

<!-- 
* Isolation.
* Standard synchronization techniques (prevent write-write or read-write hazards).
* Transactions. 
-->
* 隔离
* 标准同步技术（防止写后写或读后写冒险）。
* 事务处理

<!-- 
By far, we preferred isolation.  It turns out web frameworks offer good lessons to learn from here.  Most of the time,
a server object is part of a "session" and should not be aliased across multiple concurrent clients.  It tended to be
easy to partition state into sub-objects, and have dialogues using those.  Our language annotations around mutability
helped to guide this process. 
-->
直至今日，我们更喜欢隔离。 
事实证明，Web框架提供了很好的，可供学习的教训： 
大多数情况下，服务器对象是“会话”的一部分，
不应该跨多个并发客户端使用。 
它往往很容易将状态划分为子对象，并使用它们进行会话。
我们围绕可变性的语言注释有助于指导这一过程。

<!-- 
A lesser regarded technique was to apply synchronization.  Thankfully in our language, we knew which operations read
versus wrote, and so we could use that to block dispatching messages intelligently, using standard reader-writer lock
techniques.  This was comfy and cozy and whatnot, but could lead to deadlocks if done incorrectly (which we did our
best to detect).  As you can see, once you start down this path, the world is less elegant, so we discouraged it. 
-->
三者中较少考虑的技术是同步。
值得庆幸的是，在我们的语言中，我们知道哪些操作存在读与写的竞争，
因此可以利用它使用标准读写器加锁技术智能地阻止调度消息。
这看起来是个很不错的方法，
但如果做得不正确可能导致死锁（对此我们将尽力检测）。 
正如你所看到的，一旦你采用了这个方法，世界就不那么优雅了，所以我们不鼓励使用它。

<!-- 
Finally, transactions.  We didn't go there.  [Distributed transactions are evil](
http://c2.com/cgi/wiki?DistributedTransactionsAreEvil). 
-->
最后，对于事务处理，我们根本没有对其进行实践过，
因为[分布式事务是邪恶的](http://c2.com/cgi/wiki?DistributedTransactionsAreEvil)。

<!-- 
In general, we tried to learn from the web, and apply architectures that worked for large-scale distributed systems.
Statelessness was by far the easiest pattern.  Isolation was a close second.  Everything else was just a little dirty. 
-->
总的来说，我们尝试从互联网上学习，
并应用适用于大规模分布式系统的架构。 
无状态的方法是迄今为止最简单的模式，
而隔离紧随其后，其他一切方式都显得有点脏。

<!-- 
P.S.  I will be sure to have an entire post dedicated to the language annotations. 
-->
备注：未来我一定会写一篇专门讨论语言注释的文章。

<!-- 
### Ordering 
-->
### 有序性

<!-- 
In a distributed system, things get unordered unless you go out of your way to preserve order.  And going out of your
way to preserve order removes concurrency from the system, adds book-keeping, and a ton of complexity.  My biggest
lesson learned here was: distributed systems are unordered.  It sucks.  Don't fight it.  You'll regret trying. 
-->
在分布式系统中，除非你不遗余力地保持有序状态，否则事物会是无序的。 
并且不遗余力地保持有序状态会消弱系统的并发性，增加状态的记录开销和大量复杂性。 
于此我学到的最大教训是：分布式系统是无序的。 
这糟糕透了，不过不要去试图打败它，否则你会后悔尝试的。

<!-- 
Leslie Lamport has a classic must-read paper on the topic: [Time, Clocks, and the Ordering of Events in a Distributed
System](http://amturing.acm.org/p558-lamport.pdf). 
-->
Leslie Lamport有一篇关于该主题的经典必读文章：
[Time, Clocks, and the Ordering of Events in a Distributed System](http://amturing.acm.org/p558-lamport.pdf)。

<!-- 
But unordered events surprise developers.  A good example is as follows: 
-->
但无序的事件却让开发人员感到意外。 
下面是一个很好的例子：

<!-- 
    // Three asynchronous objects:
    IA a = ...;
    IB b = ...;
    IC c = ...;

    // Tell b to talk to a:
    var req1 = async b.TalkTo(a);

    // Tell c to talk to b:
    var req2 = async c.TalkTo(a);

    await Async.Join(req1, req2); 
-->
    // 三个异步对象：
    IA a = ...;
    IB b = ...;
    IC c = ...;

    // 让b与a交互：
    var req1 = async b.TalkTo(a);

    // 让c与a交互：
    var req2 = async c.TalkTo(a);

    await Async.Join(req1, req2); 

<!-- 
If you expected that `b` is guaranteed to talk with `a` before `c` talks with `a`, you're in for a bad day. 
-->
如果你认为`b`在`c`与`a`交互之前理应与`a`交互，那么你将陷入非常糟糕的一天。

<!-- 
We offered facilities for controlling order.  For example, you could flush all the messages from a channel, and await
their delivery.  You could, of course, always await the individual operations, but this introduces some amount of
unnecessary latency due to round-tripping.  We also had a "flow" abstraction that let you guarantee a sequence of
asynchronous messages were delivered in order, but in the most efficient way possible. 
-->
我们提供控制顺序的机制。 
例如，你可以刷新通道中的所有消息，并等待消息回传。 
当然，你随时可以await某个操作，但由于往返操作会引入一些不必要的延迟。
我们还有一个关于“流”的抽象，采用最有效的方式，保证一系列异步消息按顺序传递。

<!-- 
As with state management, we found that an abundance of ordering problems was often indicative of a design problem. 
-->
与状态管理层一样，我们发现大量的有序性问题的存在往往表明设计上出了问题。

<!-- ### Debugging -->
### 调试

<!-- 
With so much work flying around in the system, debugging was a challenge in the early days. 
-->
在早期，由于在系统中存在大量的任务，调试也是一项挑战。

<!-- 
The solution, as with many such challenges, was tooling.  We taught our tools that activities were as first class as
threads.  We introduced causality IDs that flowed with messages across processes, so if you broke into a message
dispatch in one process, you could trace back to the origin in potentially some other distant process.  The default
behavior for a crash was to gather this cross-process stack trace, to help figure out how you go to where you were. 
-->
与许多此类挑战一样，解决方案是使用工具。 
在我们的工具中，activity和线程一样作为一等公民而存在。 
同时，我们引入了跨进程的消息传递的因果ID，
因此如果从一个进程中进入消息调度，则可能会在其他远程进程中追溯到消息源。 
进程崩溃的默认行为是收集此跨进程堆栈追踪（stack trace），以帮助开发者找到目标位置。

<!-- 
Another enormous benefit of our improved execution model was that stacks were back!  Yes, you actually got stack traces
for asynchronous activities awaiting multiple levels deep at no extra expense.  Many systems like .NET's have to go out
of their way to piece together a stack trace from disparate hunks of stack-like objects.  We had that challenge across
processes, but within a single process, all activities had normal stack traces with variables that were in a good state. 
-->
我们改进后的执行模型的另一个巨大好处是堆栈又回来了，没错，
你实际上有异步activity的堆栈跟踪等待多个级别，而无需额外开销。
像.NET这样的许多系统都不得不竭尽全力将来自不同类似堆栈对象的堆栈跟踪拼凑在一起。
虽然我们在跨进程中也遇到了该挑战，但在单个进程内部，
所有activity都有正常的堆栈跟踪并且变量处于良好的状态。

<!-- 
### Resource Management 
-->
### 资源管理

<!-- 
At some point, I had a key realization.  Blocking in a classical system acts as a natural throttle on the amount of work
that can be offered up to the system.  Your average program doesn't express all of its latent concurrency and
parallelism by default.  But ours did!  Although that sounds like a good thing -- and indeed it was -- it came with a
dark side.  How the heck do you manage resources and schedule all that work intelligently, in the face of so much of it? 
-->
在某种程度上，我收获了一个重要认识，
传统系统中阻塞方法可作为提供给系统所有任务的自然节流措施。
默认情况下，你的普通程序并不能表达其所有潜在的并发性和并行性，
但我们的系统的确做到了，虽然这听起来像是一件好事，事实上也确实如此，
但它带来了暗黑的一面。
在面对如此多的情况时，你到底如何管理资源并智能地调度所有任务？

<!-- 
This was a loooooooong, winding road.  I won't claim we solved it.  I won't claim we even came close.  I will claim we
tackled it enough that the problem was less disastrous to the stability of the system than it would have otherwise been. 
-->
这是一条蜿蜒曲折的道路，我不会声称我们已经彻底解决， 
甚至都不会声称靠近这个目标。 
我只会说，在这个问题对于系统的稳定性来说比其他情况所造成的灾难性都要小。

<!-- 
An analogous problem that I've faced in the past is with thread pools in both Windows and the .NET Framework.  Given
that work items might block in the thread pool, how do you decide the number of threads to keep active at once?  There
are always imperfect heuristics applied, and I would say we did no worse.  If anything, we erred on the side of using
more of the latent parallelism to saturate the available resources.  It was pretty common to be running the Midori
system at 100% CPU utilization, because it was doing useful stuff, which is pretty rare on PCs and traditional apps. 
-->
我过去在Windows和.NET Framework的线程池上遇到了类似的问题，
鉴于任务项可能会在线程池中阻塞，那么将如何一次性确定保持活跃状态的线程数？
总有不完美的启发式方法可以使用，那么我可以说我们的方法并没有变得更糟。 
如果真的有的话，那么错误地使用较多的潜在并行性，可用资源变得饱和可以算作一个。 
以100%的CPU利用率运行Midori系统是很常见的，
因为它正在运行有意义的任务，而这在PC和传统应用程序中非常罕见。

<!-- 
But the scale of our problem was much worse than anything I'd ever seen.  Everything was asynchronous.  Imagine an app
traversing the entire filesystem, and performing a series of asynchronous operations for each file on disk.  In Midori,
the app, filesystem, disk drivers, etc., are all different asynchronous processes.  It's easy to envision the resulting
[fork bomb](https://en.wikipedia.org/wiki/Fork_bomb)-like problem that results. 
-->
但我们问题的规模比我见过的任何系统都要复杂得多，
因为所有的一切都是异步的。 
设想一下，如果某个应用程序遍历整个文件系统，
并为磁盘上的每个文件执行一系列异步操作。 
在Midori中，应用程序，文件系统和磁盘驱动程序等都是不同的异步进程。 
那么很容易想到会产生类似[fork炸弹](https://en.wikipedia.org/wiki/Fork_bomb)的问题。

<!-- 
The solution here broke down into a two-pronged defense: 
-->
因此，这里的解决方案可分解为双管齐下的防御措施：

<!-- 
1. Self-control: async code knows that it could flood the system with work, and explicitly tries not to.
2. Automatic resource management: no matter what the user-written code does, the system can throttle automatically. 
-->
1. 自我控制：异步代码明白它可能会使系统充满任务，并明确地尝试避免这样做；
2. 自动资源管理：无论用户编写什么样的代码，系统都可以自动加以控制。

<!-- 
For obvious reasons, we preferred automatic resource management. 
-->
出于显而易见的原因，我们倾向于自动资源管理

<!-- 
This took the form of the OS scheduler making decisions about which processes to visit, which turns to let run, and, in
some cases, techniques like flow control as we saw above with streams.  This is the area we had the most "open ended"
and "unresolved" research.  We tried out many really cool ideas.  This included attempting to model the expected
resource usage of asynchronous activities (similar to [this paper on convex optimization](
https://www.usenix.org/legacy/event/hotpar11/tech/final_files/Bird.pdf)).  That turned out to be very difficult, but
certainly shows some interesting long turn promise if you can couple it with adaptive techniques.  Perhaps surprisingly,
our most promising results came from adapting [advertisement bidding algorithms](
http://research.microsoft.com/en-us/um/people/nikdev/pubs/rtb-perf.pdf) to resource allocation.  Coupled with an element
of [game theory](https://en.wikipedia.org/wiki/Game_theory), this approach gets very interesting.  If the system charges
a market value for all system resources, and all agents in the system have a finite amount of "purchasing power," we
can expect they will purchase those resources that benefit themselves the most based on the market prices available. 
-->
这采用了决定要访问哪些进程，决定哪些进程运行的OS调度程序的形式，
以及在某些情况下，像流控制这样的技术，就像我们在上面看到的流一样。
这是我们最“开放式”和“未解决”的研究领域。
我们尝试了许多非常酷的想法，这包括尝试对异步activity的预期资源使用进行建模
（类似于[关于凸优化的论文](https://www.usenix.org/legacy/event/hotpar11/tech/final_files/Bird.pdf)）。
事实证明这非常困难，但如果你能将它与自适应技术相结合，肯定会获得一些有趣的长期回报。
也许令人惊讶的是，我们最有希望的结果来自于使[广告竞价算法](http://research.microsoft.com/en-us/um/people/nikdev/pubs/rtb-perf.pdf)适用于资源分配，
再加上[博弈论](https://en.wikipedia.org/wiki/Game_theory)的要素，使得这种方法变得非常有趣。
如果系统收取所有系统资源的市场价值，
并且系统中的所有代理商都具有有限的“购买力”，
我们可以预期他们将根据可用的市场价格购买那些最有利于自己的资源。

<!-- 
But automatic management wasn't always perfect.  That's where self-control came in.  A programmer could also help us out
by capping the maximum number of outstanding activities, using simple techniques like "wide-loops."  A wide-loop was an
asynchronous loop where the developer specified the maximum outstanding iterations.  The system ensured it launched no
more than this count at once.  It always felt a little cheesy but, coupled with resource management, did the trick. 
-->
但自动管理并不总是完美的，这就是自我控制发挥作用的地方。
程序员也可以通过使用诸如“wide-loops”之类的简单技术，来限制未完成的activity的最大数量。
wide-loops作为一个异步循环，其中开发人员指定了最大的未完成迭代数量，系统确保它不会立刻超过此计数的启动值。 
虽然总觉得有点俗气，但加上资源管理，就足够了。

<!-- 
I would say we didn't die from this one.  We really thought we would die from this one.  I would also say it was solved
to my least satisfaction out of the bunch, however.  It remains fertile ground for innovative systems research. 
-->
我可以说我们并没有栽在这里，
虽然我们真的以为我们会栽于此。 
但我也不得不说这是所有问题的解决方案中我最不满意的一个，
不过这方面仍然是创新性系统研究的沃土。

<!-- 
## Winding Down 
-->
## 放松一下

<!-- 
That was a lot to fit into one post.  As you can see, we took "asynchronous everywhere" to quite the extreme. 
-->
确实有太多的内容适合放进这篇文章中。
正如你所看到的，我们将“无处不在的异步”发挥到了极致。

<!-- 
In the meantime, the world has come a long way, much closer to this model than when we began.  In Windows 8, a large
focus was the introduction of asynchronous APIs, and, like with adding await to C#, we gave them our own lessons learned
at the time.  A little bit of what we were doing rubbed off, but certainly nothing to the level of what's above. 
-->
与此同时，世界也经过漫长的发展，
比我们开始时更接近这个模型。 
在Windows 8的开发中，异步API的引入是其中一个重点，就像在C#中添加await一样，
我们在当时提供给了他们，我们自己的经验和教训。
我们正所做事情对其造成了一点影响，但肯定没有达到上面所描述的水平。

<!-- 
The resulting system was automatically parallel in a very different way than the standard meaning.  Tons of tiny
processes and lots of asynchronous messages ensured the system kept making forward progress, even in the face of
variable latency operations like networking.  My favorite demo we ever gave, to Steve Ballmer, was a mock
implementation of Skype on our own multimedia stack that wouldn't hang even if you tried your hardest to force it. 
-->
所产生的系统以与标准含义完全不同的方式自动地进行并行。 
大量的微小进程和大量异步消息确保了系统不断前进，
即使面对网络等可变延迟操作也是如此。 
我最喜欢的一个演示，是向史蒂夫·鲍尔默展示
我们在自己的多媒体栈上模拟Skype的实现，即使再怎么尝试它也不会出现挂起现象。

<!-- 
As much as I'd like to keep going on architecture and programming model topics, I think I need to take a step back.  Our
compiler keeps coming up and, in many ways, it was our secret sauce.  The techniques we used there enabled us to achieve
all of these larger goals.  Without that foundation, we'd never have had the safety or been on the same playing ground
as native code.  See you next time, when we nerd out a bit on compilers. 
-->
尽管我想继续研究架构和编程模型相关的主题，但我却认为需要退后一步， 
因为我们的编译器在前文中的很多方面不断出现，它也是我们的秘密武器。 
我们在编译器上所使用的技术使我们能够实现所有这些更宏大的目标， 
没有这个基础，我们就永远不会同时实现安全性和原生代码。
下次当我们深入地探讨编译器时再见！
