---
title: Midori博客系列翻译（5）——错误模型
date: 2019-3-9 21:07:00
tags: [操作系统, Midori, 翻译, 错误模型, 编译, 语言设计]
categories: 中文
---

<!-- 
[Midori](http://joeduffyblog.com/2015/11/03/blogging-about-midori/) was written in [an ahead-of-time compiled, type-safe
language based on C#](http://joeduffyblog.com/2015/12/19/safe-native-code/).  Aside from our microkernel, the whole
system was written in it, including drivers, the domain kernel, and all user code.  I've hinted at a few things along
the way and now it's time to address them head-on.  The entire language is a huge space to cover and will take a series
of posts.  First up?  The Error Model.  The way errors are communicated and dealt with is fundamental to any language,
especially one used to write a reliable operating system.  Like many other things we did in Midori, a "whole system"
approach was necessary to getting it right, taking several iterations over several years.  I regularly hear from old
teammates, however, that this is the thing they miss most about programming in Midori.  It's right up there for me too.
So, without further ado, let's start.
-->
[Midori](/2018/10/20/midori/0-blogging-about-midori/)是由[基于C#，通过AOT编译的且类型安全的语言](/2019/02/17/midori/4-safe-native-code/)编写而成的操作系统。
除了其微内核部分之外，整个系统都由该语言而成，包括驱动程序、域内核和所有的用户态代码。
我在前面的文章中已经提及该语言设计的一些方面，
现在是时候正面介绍的时候了。
整个语言需要巨大的空间来覆盖，也需要一系列的文章来分析。
那从什么地方开始呢？错误模型。
传递和处理错误的方式是任何一门语言的基础，尤其是用于编写可靠操作系统的语言。
像我们在Midori上所做的许多其他事情一样，需要几年的几次迭代的“整个系统（wholesystem）”方法，
在错误模型上也是必要的。
然而，我经常听到以前团队成员们说，这是他们关于Midori最怀念的地方，而对我来说也是如此。
因此，不用多说了，让我们现在开始吧。

<!-- 
# Introduction 
-->
# 介绍

<!-- 
The basic question an Error Model seeks to answer is: how do "errors" get communicated to programmers and users of the
system?  Pretty simple, no?  So it seems. 
-->
错误模型试图回答的基本问题是：“错误”是如何传递给程序员和系统用户的。
很简单，不是吗？至少它看起来是这样的。

<!-- 
One of the biggest challenges in answering this question turns out to be defining what an error actually *is*.  Most
languages lump bugs and recoverable errors into the same category, and use the same facilities to deal with them.  A
`null` dereference or out-of-bounds array access is treated the same way as a network connectivity problem or parsing
error.  This consistency may seem nice at first glance, but it has deep-rooted issues.  In particular, it is misleading
and frequently leads to unreliable code. 
-->
回答这个问题的最大挑战之一是，定义错误*究竟是什么*。
大多数语言将程序bug（缺陷）和可恢复的错误归为同一类，并使用相同的工具来处理它们，
使得`null`指针的解引用或数组越界访问，与网络连接问题或解析错误的处理方式相同。
这样的一致性乍一眼看起来似乎很不错，但它有根深蒂固的问题，
特别是这种方式具有误导性并经常导致代码的不可靠。

<!-- 
Our overall solution was to offer a two-pronged error model.  On one hand, you had fail-fast -- we called it
*abandonment* -- for programming bugs.  And on the other hand, you had statically checked exceptions for recoverable
errors.  The two were very different, both in programming model and the mechanics behind them.  Abandonment
unapologetically tore down the entire process in an instant, refusing to run any user code while doing so.  (Remember,
a typical Midori program had many small, lightweight processes.)  Exceptions, of course, facilitated recovery, but had
deep type system support to aid checking and verification. 
-->
我们的整体解决方案是提供双管齐下的错误模型。
一方面，对于编程bug来讲，语言有快速失败机制，我们称之为*Abandonment（放弃）*，
另一方面，对于可恢复的错误，语言有静态检查性异常机制。
这两者在编程模型和它们背后的机制方面都大相径庭，
Abandonment意味着无条件地在瞬间终止整个进程，因此不再运行任何用户代码
（还记得么，一个典型的Midori程序由许多小型轻量级的进程组成），
而异常当然有助于程序的恢复，同时它也有深层次的类型支持以帮助检查和验证。

<!-- 
This journey was long and winding.  To tell the tale, I've broken this post into six major areas: 
-->
这段旅程漫长而曲折，
为了便于讲述，我将这篇文章分为六个主要部分：

<!-- 
* [Ambitions and Learnings](#ambitions-and-learnings)
* [Bugs Aren't Recoverable Errors!](#bugs-arent-recoverable-errors)
* [Reliability, Fault-Tolerance, and Isolation](#reliability-fault-tolerance-and-isolation)
* [Bugs: Abandonment, Assertions, and Contracts](#bugs-abandonment-assertions-and-contracts)
* [Recoverable Errors: Type-Directed Exceptions](#recoverable-errors-type-directed-exceptions)
* [Retrospective and Conclusions](#retrospective-and-conclusions) 
-->
* [野心和学习](#雄心和学习)
* [Bug不是可恢复的错误！](#Bug不是可恢复的错误！)
* [可靠性，容错和隔离](#可靠性，容错和隔离)
* [Bug：Abandonment，断言和合约](#Bug：Abandonment，断言和合约)
* [可恢复错误：类型导向的异常](#可恢复错误：类型导向的异常)
* [回顾与总结](#回顾与总结)

<!-- 
In hindsight, certain outcomes seem obvious.  Especially given modern systems languages like Go and Rust.  But some
outcomes surprised us.  I'll cut to the chase wherever I can but I'll give ample back-story along the way.  We tried out
plenty of things that didn't work, and I suspect that's even more interesting than where we ended up when the dust
settled. 
-->
事后看来，某些结果似乎很明显，特别是对于现代系统语言，
如Go和Rust来说，但另外一些结果也出乎我们的意料。
无论如何我都会尽力讲述这一切，并且会在此过程中提供充足的背景故事。
我们尝试了许多了不起作用的东西，因此我觉得这些尝试比最终已知结果时更加有趣。

<!-- 
# Ambitions and Learnings 
-->
# 雄心和学习

<!-- 
Let's start by examining our architectural principles, requirements, and learnings from existing systems. 
-->
让我们首先从检视我们的架构原则和要求，以及从现有的系统中学习开始。

<!-- 
## Principles 
-->
## 原则 

<!-- 
As we set out on this journey, we called out several requirements of a good Error Model: 
-->
在开始这段旅程时，我们提出了，对于一个错误模型来讲的其优异体现的几个要求：

<!-- 
* **Usable.**  It must be easy for developers to do the "right" thing in the face of error, almost as if by accident.  A
  friend and colleague famously called this falling into the [The Pit of Success](
  http://blogs.msdn.com/b/brada/archive/2003/10/02/50420.aspx).  The model should not impose excessive ceremony in order
  to write idiomatic code.  Ideally it is cognitively familiar to our target audience.

* **Reliable.**  The Error Model is the foundation of the entire system's reliability.  We were building an operating
  system, after all, so reliability was paramount.  You might even have accused us as obsessively pursuing extreme
  levels of it.  Our mantra guiding much of the programming model development was "*correct by construction*."

* **Performant.**  The common case needs to be extremely fast.  That means as close to zero overhead as possible for
  success paths.  Any added costs for failure paths must be entirely "pay-for-play."  And unlike many modern systems
  that are willing to overly penalize error paths, we had several performance-critical components for which this wasn't
  acceptable, so errors had to be reasonably fast too.

* **Concurrent.**  Our entire system was distributed and highly concurrent.  This raises concerns that are usually
  afterthoughts in other Error Models.  They needed to be front-and-center in ours.

* **Diagnosable.**  Debugging failures, either interactively or after-the-fact, needs to be productive and easy.

* **Composable.**  At the core, the Error Model is a programming language feature, sitting at the center of a
  developer's expression of code.  As such, it had to provide familiar orthogonality and composability with other
  features of the system.  Integrating separately authored components had to be natural, reliable, and predictable. 
-->

* **可用性**：
  开发者面对错误时必须很容易做出“正确”的事情，几乎就像下意识一样。
  一位朋友和同事很准确地将其称为“[成功之坑（The Pit of Success）](http://blogs.msdn.com/b/brada/archive/2003/10/02/50420.aspx)”。
  为了写出符合习惯的代码，模型不应该施加过多的限制，
  并且在理想情况下，对于目标受众来说，它应该在认知上是熟悉的。

* **可靠性**：
  错误模型是整个系统可靠性的基础，
  毕竟，我们正在构建的是一个操作系统，因此可靠性至关重要，
  你甚至可能会指责我们痴迷于在此方面追求到极致。
  我们对编程模型开发的大量指导原则就是“通过构造来纠正错误”。

* **高性能**：
  在正常情况，系统需要高效运行，这意味着成功路径的开销要尽可能地接近于零，
  并且失败路径的任何增加的开销必须完全是“按使用付费（pay-for-play）”。
  与许多愿意为失败路径接受过度开销的现代系统不同，
  我们有几个性能关键的组件，错误的过度开销对于它们来说是不可接受的，
  因此错误处理也必须要足够高效。

* **并发**：
  我们的整个系统是分布式且高度并发，
  所以这会引起在其他错误模型中通常也会遇到的问题，
  因此这部分需成为我们工作的最前沿和重心。

* **可诊断**：
  无论是交互式的还是事后进行的调试，都需要高效且简单。

* **可组合的**：
  错误模型是编程语言的核心功能，位于开发者代码描述的中心，
  因此，错误模型必须提供常见的正交性和与系统其他功能的可组合性，
  与单独编写的组件的集成也必须是自然、可靠和可预测的。

<!-- 
It's a bold claim, however I do think what we ended up with succeeded across all dimensions. 
-->
这是一个勇敢的要求，但我确实认为我们最终在所有方面上取得了成功。

<!-- 
## Learnings 
-->
## 学习

<!-- 
Existing Error Models didn't meet the above requirements for us.  At least not fully.  If one did well on a dimension,
it'd do poorly at another.  For instance, error codes can have good reliability, but many programmers find them error
prone to use; further, it's easy to do the wrong thing -- like forget to check one -- which clearly violates the
"pit of success" requirement. 
-->
现有的错误模型并不能满足我们的上述要求，至少不完全满足，
某些系统在一个维度上做得很好，但在另一个维度上表现不佳。
例如，错误码返回的方式具有良好的可靠性，但许多程序员发现容易错误地使用它们，
进而很容易出错，比如说忘记检查某些返回码，这显然违反了“成功之坑”的要求。

<!-- 
Given the extreme level of reliability we sought, it's little surprise we were dissatisfied with most models.
-->
鉴于对可靠性极高的追求，我们对大多数模型并不满意这一点上并不令人惊讶。

<!-- 
If you're optimizing for ease-of-use over reliability, as you might in a scripting language, your conclusions will
differ significantly.  Languages like Java and C# struggle because they are right at the crossroads of scenarios --
sometimes being used for systems, sometimes being used for applications -- but overall their Error Models were very
unsuitable for our needs. 
-->
如果你像使用脚本语言那样，对易用性的优化胜过可靠性，那么得到的结论将会非常不同。
而像Java和C#这样的语言很难再优化，是因为它们处于不同使用场景的十字路口上：
有时用于系统，有时又用于应用程序，但总体而言，它们的错误模型非常不适合我们的要求。

<!-- 
Finally, also recall that this story began in the mid-2000s timeframe, before Go, Rust, and Swift were available for our
consideration.  These three languages have done some great things with Error Models since then. -->
最后要提醒的是，本篇文章在时间轴上始于2000年代中期，
在那之后，Go、Rust和Swift语言在错误模型上做了一些很不错的工作，
但在当时，却没有这样的语言可供我们选择。


<!-- 
### Error Codes 
-->
### 错误码

<!-- 
Error codes are arguably the simplest Error Model possible.  The idea is very basic and doesn't even require language
or runtime support.  A function just returns a value, usually an integer, to indicate success or failure: 
-->
错误码可以说是最简单的错误模型。
该想法非常基础，甚至不需要语言或运行时的支持，只需要函数返回一个值，
这个值通常是一个整数，表示执行成功或失败：

<!-- 
    int foo() {
        // <try something here>
        if (failed) {
            return 1;
        }
        return 0;
    } 
-->

    int foo() {
        // <在这里尝试完成某项任务>
        if (failed) {
            return 1;
        }
        return 0;
    } 

<!-- 
This is the typical pattern, where a return of `0` means success and non-zero means failure.  A caller must check it: 
-->
以上是典型的模式，返回“0”表示成功，非零则表示失败，而调用者必须对其进行检查：

<!-- 
    int err = foo();
    if (err) {
        // Error!  Deal with it.
    } 
-->

    int err = foo();
    if (err) {
        // 发生错误！需对其进行处理 
    } 

<!-- 
Most systems offer constants representing the set of error codes rather than magic numbers.  There may or may not be
functions you can use to get extra information about the most recent error (like `errno` in standard C and
`GetLastError` in Win32).  A return code really isn't anything special in the language -- it's just a return value. 
-->
大多数系统提供表示错误码的常量集合，而不仅仅是幻数（magic number），
你可以使用或不使用相关函数来获取最近一次错误发生的额外信息
（例如标准C语言中的`errno`和Win32中的`GetLastError`）。
返回码在编程语言中并不特别，它从本质上讲仅仅只是一个返回值而已。

<!-- 
C has long used error codes.  As a result, most C-based ecosystems do.  More low-level systems code has been written
using the return code discipline than any other.  [Linux does](http://linux.die.net/man/3/errno), as do countless
mission-critical and realtime systems.  So it's fair to say they have an impressive track record going for them! 
-->
C语言长期使用错误码的方式，因此，大多数基于C的生态系统也都如此。
使用返回码规则编写的低层系统代码比任何其他类型的都要多，
[Linux是这么做的](http://linux.die.net/man/3/errno)，无数任务关键和实时系统也是如此。
因此可以说，它有令人印象深刻的历史记录！

<!-- 
On Windows, `HRESULT`s are equivalent.  An `HRESULT` is just an integer "handle" and there are a bunch of constants and
macros in `winerror.h` like `S_OK`, `E_FAULT`, and `SUCCEEDED()`, that are used to create and check values.  The most
important code in Windows is written using a return code discipline.  No exceptions are to be found in the kernel.  At
least not intentionally. 
-->
在Windows中，`HRESULT`与错误码是等效的。
`HRESULT`仅仅是一个整数的“句柄”，在`winerror.h`中有一堆常量和宏，
如`S_OK`，`E_FAULT`和`SUCCEEDED()`等，用于创建和检查这些返回码。
Windows中最重要的代码是使用返回码规则编写，
例如在内核中是没有异常的，至少不是故意这么做的。

<!-- 
In environments with manual memory management, deallocating memory on error is uniquely difficult.  Return codes can
make this (more) tolerable.  C++ has more automatic ways of doing this using [RAII](
https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization), but unless you buy into the C++ model whole hog
-- which a fair number of systems programmers don't -- then there's no good way to incrementally use RAII in your C
programs. 
-->
在手动管理内存的环境中，出错时释放内存是非常困难的一件事，
而返回码可以使这种情况（更容易）被容忍。
C++通过使用[RAII](https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)的方式来自动管理内存，
但除非你是C++模型（不是C语言那部分）的彻底贯彻者（事实上相当多的系统程序员并非如此），
否则并没有好的方法在你的C程序中增量式地使用RAII。

<!-- 
More recently, Go has chosen error codes.  Although Go's approach is similar to C's, it has been modernized with much
nicer syntax and libraries. 
-->
再后来，Go选择了错误码方式，虽然Go的方法与C类似，但它更现代化，并且具有更好的语法和库支持。

<!-- 
Many functional languages use return codes disguised in monads and named things like `Option<T>`, `Maybe<T>`, or
`Error<T>`, which, when coupled with a dataflow-style of programming and pattern matching, feel far more natural.  This
approach removes several major drawbacks to return codes that we're about to discuss, especially compared to C.  Rust
has largely adopted this model but has dome some exciting things with it for systems programmers. 
-->
许多函数式语言使用monad来封装返回码，并命名为`Option<T>`、`Maybe<T>`或`Error<T>`，
使得它们与数据流编程和模式匹配相结合时感觉更自然，
特别是与C相比，这种方法消除了我们即将讨论的几个返回码的主要缺点。
Rust已经在很大程度上采用了这个模型，它为系统程序员提供了一些令人兴奋的特性。

<!-- 
Despite their simplicity, return codes do come with some baggage; in summary: 
-->
虽然它很简单，返回码也确实存在一些缺陷，总的说来包括以下几点：

<!-- 
* Performance can suffer.
* Programming model usability can be poor.
* The biggie: You can accidentally forget to check for errors. 
-->
* 性能可能会受到影响；
* 编程模型的可用性会变差；
* 最重要的一点是，可能会意外地忘记对错误进行检查。

<!-- 
Let's discuss each one, in order, with examples from the languages cited above. 
-->
让我们依次用上面提及的语言中的例子来讨论每一个缺点。

<!-- 
#### Performance 
-->
#### 性能

<!-- 
Error codes fail the test of "zero overhead for common cases; pay for play for uncommon cases": -->
错误码未满足所谓“正常情况下零开销，非正常情况下才付费（zero overhead for common cases， pay for play for uncommon cases）”的要求：

<!-- 
1. There is calling convention impact.  You now have *two* values to return (for non-`void` returning functions): the
   actual return value and the possible error.  This burns more registers and/or stack space, making calls less
   efficient.  Inlining can of course help to recover this for the subset of calls that can be inlined. 
-->
1. 对调用约定会会造成影响。函数现在需要返回*两个*值（对于非void函数来说）：实际的返回值和可能的错误码，
   从而消耗掉更多寄存器和/或栈空间，导致调用效率降低。
   当然，内联化调用子集可以弥补这种开销。

<!-- 
1. There are branches injected into callsites anywhere a callee can fail.  I call costs like this "peanut butter,"
   because the checks are smeared across the code, making it difficult to measure the impact directly.  In Midori we
   were able to experiment and measure, and confirm that yes, indeed, the cost here is nontrivial.  There is also a
   secondary effect which is, because functions contain more branches, there is more risk of confusing the optimizer. 
-->
2. 在被调用点可能出现失败的任何地方需要注入分支。
   我将其称之为“花生酱（peanut butter）”开销，因为检查像花生酱一样被“涂抹”在代码中的各个地方，使得难以直接衡量其影响。
   在Midori中，我们能够通过试验和测量并确认这种影响——是的，这里的开销是不可忽略的。
   还有一个次要的影响是，因为函数包含更多的分支，所以更有可能混淆优化器。

<!-- 
This might be surprising to some people, since undoubtedly everyone has heard that "exceptions are slow."  It turns out
that they don't have to be.  And, when done right, they get error handling code and data off hot paths which increases
I-cache and TLB performance, compared to the overheads above, which obviously decreases them. 
-->
这样的事实对某些人来说可能会吃惊，因为每个人毫无疑问都听说过“异常处理很耗时”的说法。
事实证明，异常处理并不低效，当正确地执行时，代码会从热点路径上剔除错误处理的代码和数据，
与上面提到的返回码开销相比，这种情况增加了I-cache和TLB的性能。

<!-- 
Many high performance systems have been built using return codes, so you might think I'm nitpicking.  As with many
things we did, an easy criticism is that we took too extreme an approach.  But the baggage gets worse. 
-->
许多高性能系统都是使用返回码构建的，所以你可能会认为我在挑剔，
正如我们所做的许多其他事情一样，一种简单的批评方式是我们采取了极端的方法，
但事实是性能变得更糟糕。

<!-- 
#### Forgetting to Check Them 
-->
#### 忘记对它们进行检查

<!-- 
It's easy to forget to check a return code.  For example, consider a function: 
-->
忘记检查返回码的情况是很容易发生的。比如说，考虑如下函数：

    int foo() { ... } 


<!-- 
Now at the callsite, what if we silently ignore the returned value entirely, and just keep going?-->
在调用点上，如果现在我们悄悄地忽略掉返回码，然后继续执行，会出现什么情况呢？

<!-- 
    foo();
    // Keep going -- but foo might have failed! 
-->

    foo();
    // 继续执行，但foo可能执行失败了！ 

<!-- 
At this point, you've masked a potentially critical error in your program.  This is easily the most vexing and
damaging problem with error codes.  As I will show later, option types help to address this for functional languages.
But in C-based languages, and even Go with its modern syntax, this is a real issue. 
-->
此时，你已经掩盖了程序中可能存在的严重错误，这很容易成为错误码最棘手和最具破坏性的问题。
正如我稍后将要介绍的那样，选项类型（option）有助于在函数式语言中解决这个问题，
但是在基于C的语言，甚至是Go的现代语法中，这是一个真实存在的问题。

<!-- 
This problem isn't theoretical.  I've encountered numerous bugs caused by ignoring return codes and I'm sure you have
too.  Indeed, in the development of this very Error Model, my team encountered some fascinating ones.  For example, when
we ported Microsoft's Speech Server to Midori, we found that 80% of Taiwan Chinese (`zh-tw`) requests were failing.  Not
failing in a way the developers immediately saw, however; instead, clients would get a gibberish response.  At first, we
thought it was our fault.  But then we discovered a silently swallowed `HRESULT` in the original code.  Once we got it
over to Midori, the bug was thrown into our faces, found, and fixed immediately after porting.  This experience
certainly informed our opinion about error codes. 
-->
这个问题不仅仅是在理论上存在，
实际上我遇到了无数忽略返回码导致的错误，我相信你也遇到过。
正是在这个错误模型的开发中，我的团队遇到了一些令人大开眼界的问题。
例如，当我们将微软的语音服务器移植到Midori时，我们发现80%的繁体中文（`zh-tw`）请求都失败了，但开发者并不会立即看到失败并对其进行处理，相反地，客户端会得到莫名其妙的回应。
起初，我们认为这是Miodri的问题，但后来我们在原始代码中发现了一个悄悄忽略掉的`HRESULT`。
一旦我们把它移植到Midori上，这个bug就立刻出现在我们的眼帘，被找到并在移植后立即得到了修复。
毫无疑问地，这样的经验告诉了我们对错误码的上述看法。

<!-- 
It's surprising to me that Go made unused `import`s an error, and yet missed this far more critical one.  So close! 
-->
令我惊讶的是，Go将未使用的`import`当成错误对待，但却错过了这个更为关键性的错误，而它们仅仅一步之遥！

<!-- 
It's true you can add a static analysis checker, or maybe an "unused return value" warning as most commercial C++
compilers do.  But once you've missed the opportunity to add it to the core of the language, as a requirement, none of
those techniques will reach critical mass due to complaints about noisy analysis. 
-->
没错，你可以增加静态分析检查器，或者像大多数商业C++编译器那样增加“未使用的返回值”等警告方式。 
但是，一旦你错过了将其添加到语言核心并作为一项要求的机会，
开发者处于对烦人的分析和警告的抱怨，这些技术都不会起到关键的作用。

<!-- 
For what it's worth, forgetting to use return values in our language was a compile time error.  You had to explicitly
ignore them; early on we used an API for this, but eventually devised language syntax -- the equivalent of `>/dev/null`: 
--> 
不管值不值得，在我们的语言中忘记使用返回值会造成编译时错误，因此你必须显式地忽略它们。
在早先时候，我们用API实现这个功能，但最终将其设计成语言的语法部分，其功能相当于`>/dev/null`：

    ignore foo(); 

<!-- 
We didn't use error codes, however the inability to accidentally ignore a return value was important for the overall
reliability of the system.  How many times have you debugged a problem only to find that the root cause was a return
value you forgot to use?  There have even been security exploits where this was the root cause.  Letting developers say
`ignore` wasn't bulletproof, of course, as they could still do the wrong thing.  But it was at least explicit and
auditable. 
-->
虽然我们未选择使用错误码进行错误处理，但无法意外地忽略返回值对于系统的整体可靠性来讲非常重要。
试想你多少次在调试一个问题时，才发现其根本原因是忘记使用返回值，
更有甚者因为此问题造成了安全漏洞。当然，因为程序员依然可以做坏事，
所以让他们使用`ignore`并不是完全防弹的方法，但它至少是显式的和可审计的。

<!-- 
#### Programming Model Usability 
-->
#### 编程模型的可用性

<!-- 
In C-based languages with error codes, you end up writing lots of hand-crafted `if` checks everywhere after function
calls.  This can be especially tedious if many of your functions fail which, in C programs where allocation failures are
also communicated with return codes, is frequently the case.  It's also clumsy to return multiple values. 
-->
在具有错误码方式的基于C的语言中，如果在任何的函数调用位置进行检查，
那么会导致大量手动的`if`语句的检查。
尤其是如果大量函数调用失败是很常见的情况时，这可能会特别繁琐，
因为在C语言的程序中，分配失败也需要通过返回码来交互。
另外，多值返回也是一件很笨拙的事情。

<!-- 
A warning: this complaint is subjective.  In many ways, the usability of return codes is actually elegant.  You reuse
very simple primitives -- integers, returns, and `if` branches -- that are used in myriad other situations.  In my
humble opinion, errors are an important enough aspect of programming that the language should be helping you out. 
-->
警告：这样的抱怨其实是主观的，在许多方面，返回码的可用性实际上是优雅的。
你可以重用非常简单的原语——整型，return和`if`分支，并在无数其他情况下使用。
在我看来，错误也是编程的足够重要的一个方面，语言应该需要在这方面对你有所帮助。

<!-- 
Go has a nice syntactic shortcut to make the standard return code checking *slightly* more pleasant: 
-->
Go有很好的语法快捷方式，能够*稍微*更愉快地对标准的返回码进行检查：

<!-- 
    if err := foo(); err != nil {
        // Deal with the error.
    } 
-->

    if err := foo(); err != nil {
        // 处理错误 
    } 
    
<!-- 
Notice that we've invoked `foo` and checked whether the error is non-`nil` in one line.  Pretty neat. 
-->
注意，我们在一行中调用了`foo`并在检查错误是否为非`nil`，这种方式相当简约。

<!-- 
The usability problems don't stop there, however. 
-->
然而，可用性的问题并不止于此。

<!-- 
It's common that many errors in a given function should share some recovery or remediation logic.  Many C programmers
use labels and `goto`s to structure such code.  For example: 
-->
通常，对于给定函数中的许多错误应该共享一些恢复或补救的逻辑代码，
许多C程序员使用标签和`goto`语句来构建这样的代码，比如说：

<!-- 
    int error;

    // ...

    error = step1();
    if (error) {
        goto Error;
    }

    // ...

    error = step2();
    if (error) {
        goto Error;
    }

    // ...

    // Normal function exit.
    return 0;

    // ...
    Error:
    // Do something about `error`.
    return error; 
-->

    int error;

    // ...

    error = step1();
    if (error) {
        goto Error;
    }

    // ...

    error = step2();
    if (error) {
        goto Error;
    }

    // ...

    // 函数正常退出 
    return 0;

    // ...
    Error:
    // 错误相关的处理 
    return error; 

<!-- 
Needless to say, this is the kind of code only a mother could love. 
-->
不用说，这是只有妈妈才会爱的代码。

<!-- 
In languages like D, C#, and Java, you have `finally` blocks to encode this "before scope exits" pattern more directly.
Similarly, Microsoft's proprietary extensions to C++ offer `__finally`, even if you're not fully buying into RAII and
exceptions.  And D provides `scope` and Go offers `defer`.  All of these help to eradicate the `goto Error` pattern. 
-->
在D、C#和Java等语言中，可使用`finally`块对这种“在作用域退出之前”的模式进行更直接地编码，
同样地，即使没有完全采用RAII和异常的方式，微软也对C++提供了专有扩展`__finally`。
D提供`scope`，Go提供了`defer`，所有这些都有助于彻底消除`goto Error`的模式。

<!-- 
Next, imagine my function wants to return a real value *and* the possibility of an error?  We've burned the return slot
already so there are two obvious possibilities: 
-->
接下来，想象一下函数是如何返回真实的值*和*可能的错误码的？
由于我们已经破坏了返回槽，所以有两种明显的可能做法：

<!-- 
1. We can use the return slot for one of the two values (commonly the error), and another slot -- like a pointer
   parameter -- for the other of the two (commonly the real value).  This is the common approach in C.

2. We can return a data structure that carries the possibility of both in its very structure.  As we will see, this is
   common in functional languages.  But in a language like C, or Go even, that lacks parametric polymorphism, you lose
   typing information about the returned value, so this is less common to see.  C++ of course adds templates, so in
   principle it could do this, however because it adds exceptions, the ecosystem around return codes is lacking. 
-->
1. 可以将返回槽用于两个值中的一个（通常是错误码），
   而另一个槽，比如指针参数的形式，用于两者中的另一个（通常是实际值），
   这也是C中的常用做法；
2. 可以返回一个同时在结构中具有两者的数据结构。
   在后文中我们将会看到，这种做法在函数式语言中相当常见，但是对于C或Go语言，
   由于缺少参数多态，会丢失返回值的类型信息，所以这种方式不太常看到。
   当然由于C++添加模板，因此原则上它也可以做到，
   但另一方面C++也同时增加了异常机制，所以返回码在生态系统上是匮乏的。

<!-- 
In support of the performance claims above, imagine what both of these do to your program's resulting assembly code. 
-->
为了支持上面的性能要求，想象一下这两种方法对程序生成的汇编代码有何影响。

<!-- 
##### Returning Values "On The Side" 
-->
##### 从“侧面”返回值

<!-- 
An example of the first approach in C looks like this: 
-->
第一种方法在C中的例子如下所示：

<!-- 
    int foo(int* out) {
        // <try something here>
        if (failed) {
            return 1;
        }
        *out = 42;
        return 0;
    } 
-->
    
    int foo(int* out) {
        // <在这里尝试操作>
        if (failed) {
            return 1;
        }
        *out = 42;
        return 0;
    } 

<!-- 
The real value has to be returned "on the side," making callsites clumsy: 
-->
真正的返回值必须“在侧面”（在本例中通过指针参数）返回，使得调用变得十分笨拙：

<!-- 
    int value;
    int ret = foo(&value);
    if (ret) {
        // Error!  Deal with it.
    }
    else {
        // Use value...
    } 
-->

    int value;
    int ret = foo(&value);
    if (ret) {
        // 出现错误，对其进行处理 
    }
    else {
        // 使用返回值...
    } 

<!-- 
In addition to being clumsy, this pattern perturbs your compiler's [definite assignment analysis](
https://en.wikipedia.org/wiki/Definite_assignment_analysis) which impairs your ability to get good warnings about
things like using uninitialized values.
-->
除了变得笨拙之外，这种模式还会扰乱编译器的[定义赋值分析](https://en.wikipedia.org/wiki/Definite_assignment_analysis)，
削弱编译器对使用未初始化值等情况发出良好警告的能力。

<!-- 
Go also takes aim at this problem with nicer syntax, thanks to multi-valued returns: 
-->
归功于多值返回，Go还可以通过更好的语法来解决这个问题：

    func foo() (int, error) {
        if failed {
            return 0, errors.New("Bad things happened")
        }
        return 42, nil
    } 


<!-- 
And callsites are much cleaner as a result.  Combined with the earlier feature of single-line `if` checking for errors
-- a subtle twist, since at first glance the `value` return wouldn't be in scope, but it is -- this gets a touch nicer: 
-->
这使得调用点变得更加整洁，
结合先前提到的单行`if`检查错误（一个微小的改变，
因为乍一看返回的`value`不在作用域内）的特性，它确实成为更好的处理方式：

<!-- 
    if value, err := foo(); err != nil {
        // Error!  Deal with it.
    } else {
        // Use value ...
    } 
-->

    if value, err := foo(); err != nil {
        // 发生错误并处理它 
    } else {
        // 使用值 ...
    } 

<!-- 
Notice that this also helps to remind you to check the error.  It's not bulletproof, however, because functions can
return an error and nothing else, at which point forgetting to check it is just as easy as it is in C. 
-->
注意这也有助于提醒你对错误进行检查。
然而，它也不是完全没有问题，因为函数可以仅仅返回错误码，
此时忘记对其进行检查就像在C中一样的容易。

<!-- 
As I mentioned above, some would argue against me on the usability point.  Especially Go's designers, I suspect.  A big
appeal to Go using error codes is as a rebellion against the overly complex languages in today's landscape.  We have
lost a lot of what makes C so elegant -- that you can usually look at any line of code and guess what machine code it
translates into.  I won't argue against these points.  In fact, I vastly prefer Go's model over both unchecked
exceptions and Java's incarnation of checked exceptions.  Even as I write this post, having written lots of Go lately,
I look at Go's simplicity and wonder, did we go too far with all the `try` and `requires` and so on that you'll see
shortly?  I'm not sure.  Go's error model tends to be one of the most divisive aspect of the language; it's probably
largely because you can't be sloppy with errors as in most languages, however programmers really did enjoy writing code
in Midori's.  In the end, it's hard to compare them.  I'm convinced both can be used to write reliable code. 
-->
正如我上面提到的，有些人会在可用性上反驳我，我想对于Go的设计师来说，更是如此。
因为Go使用错误码的一个巨大吸引力是，这是当今对于过于复杂的语言的一种反叛，
我们已经失去了很多C语言的优雅特性——通常情况下，你可以查看任何一行C语言代码并猜测它编译后的机器代码。
我不反对这种观点，事实上，我喜欢Go的模型胜过非检查性异常或Java中的检查性异常的化身，
并且即使是在我最近写这篇文章的时候，也写了很多Go代码。
我看着Go的简单性并问自己，我们是否在所有的`try`和`requires`等你将很快会看到的内容上的走得太远？我不确定。
Go的错误模型往往是该语言最具分裂性的方面之一，
这很大程度上是因为你不能像大多数语言那样草率地对待错误，但程序员确实喜欢在Midori中编写代码。
最后，很难对它们进行比较，但我确信两者都可以用来编写可靠的代码。

<!-- 
##### Return Values in Data Structures 
-->
##### 从数据结构中返回

<!-- 
Functional languages address many of the usability challenges by packaging up the possibility of *either* a value
*or* an error, into a single data structure.  Because you're forced to pick apart the error from the value if you want
to do anything useful with the value at the callsite -- which, thanks to a dataflow style of programming, you probably
will -- it's easy to avoid the killer problem of forgetting to check for errors. 
-->
函数语言通过将值或错误的可能性打包到单个数据结构的方式解决许多可用性上挑战，
因为如果你想对调用点上的返回值做任何有意义的事情，就不得不从返回值中挑出错误。
这要归功于数据流风格的编程，使得很容易避免遗忘检查错误这种杀手级的问题。

<!-- 
For an example of a modern take on this, check out [Scala's `Option` type](
http://danielwestheide.com/blog/2012/12/19/the-neophytes-guide-to-scala-part-5-the-option-type.html).  The unfortunate
news is that some languages, like those in the ML family and even Scala (thanks to its JVM heritage), mix this elegant
model with the world of unchecked exceptions.  This taints the elegance of the monadic data structure approach. 
-->
有关此方法的现代语言示例，请查看[Scala种的`Option`类型](http://danielwestheide.com/blog/2012/12/19/the-neophytes-guide-to-scala-part-5-the-option-type.html)。
但不幸消息是，有些语言，例如ML系列中的语言，甚至是Scala（由于其JVM的血统），
将这种优雅的模型与非检查性异常方法混合在一起，从而玷污了monad数据结构方法的优雅性。

<!-- 
Haskell does something even cooler and [gives the illusion of exception handling while still using error values and
local control flow](https://wiki.haskell.org/Exception): 
-->
Haskell做的事情更酷，
[在仍然使用错误值和局部控制流的前提下，提供了类似异常处理的错觉](https://wiki.haskell.org/Exception)：

<!-- 
> There is an old dispute between C++ programmers on whether exceptions or error return codes are the right way.  Niklas
> Wirth considered exceptions to be the reincarnation of GOTO and thus omitted them in his languages.  Haskell solves
> this problem a diplomatic way: Functions return error codes, but the handling of error codes does not uglify the code. 
-->

> C++程序员之间就异常还是错误返回码是正确的方式上存在长期的争议。
> Niklas Wirth认为异常是goto方式的转世，因此在他的语言中消除了这种方式，
> Haskell以更圆滑的方式解决了这个问题：函数返回错误码，但错误码的处理不会让代码变丑。

<!-- 
The trick here is to support all the familiar `throw` and `catch` patterns, but using monads rather than control flow. 
-->
这里的技巧是使用monad而不是控制流，以支持所有熟悉的`throw`和`catch`模式。

<!-- 
Although [Rust also uses error codes](https://doc.rust-lang.org/book/error-handling.html) it is also in the style of
the functional error types.  For example, imagine we are writing a function named `bar` in Go: we'd like to call `foo`,
and then simply propagate the error to our caller if it fails: 
-->

虽然[Rust也使用错误码方式](https://doc.rust-lang.org/book/error-handling.html)，
但它也采用函数式错误类型的风格。
例如，假设我们在Go中编写了一个名为`bar`的函数，
它调用`foo`，并只是在调用出错时才将错误传播给调用者：

    func bar() error {
        if value, err := foo(); err != nil {
            return err
        } else {
            // 使用值...
        }
    } 


<!-- 
The longhand version in Rust isn't any more concise.  It might, however, send C programmers reeling with its foreign
pattern matching syntax (a real concern but not a dealbreaker).  Anyone comfortable with functional programming,
however, probably won't even blink, and this approach certainly serves as a reminder to deal with your errors: 
-->
而Rust中的速记版本就不那么简洁。
虽然这种外来的模式匹配语法（一个关注点而不是问题的死结）可能会让C程序员发昏，
然而，任何对函数编程感到满意的人可能甚至都不会眨眼，
并且这种方法毫无疑问地会对错误处理进行提醒：

<!-- 
    fn bar() -> Result<(), Error> {
        match foo() {
            Ok(value) => /* Use value ... */,
            Err(err) => return Err(err)
        }
    } 
-->

    fn bar() -> Result<(), Error> {
        match foo() {
            Ok(value) => /* 使用值... */,
            Err(err) => return Err(err)
        }
    } 

<!-- 
But it gets better.  Rust has a [`try!` macro](https://doc.rust-lang.org/std/macro.try!.html) that reduces boilerplate
like the most recent example to a single expression: 
-->
但这种方式还可以变得更好，Rust具有[`try!`宏](https://doc.rust-lang.org/std/macro.try!.html)，
可以将上述写法减少到单行的表达式：

<!-- 
    fn bar() -> Result<(), Error> {
        let value = try!(foo);
        // Use value ...
    } 
-->

    fn bar() -> Result<(), Error> {
        let value = try!(foo);
        // 使用值...
    } 

<!-- 
This leads us to a beautiful sweet spot.  It does suffer from the performance problems I mentioned earlier, but does
very well on all other dimensions.  It alone is an incomplete picture -- for that, we need to cover fail-fast (a.k.a.
abandonment) -- but as we will see, it's far better than any other exception-based model in widespread use today. 
-->
这种方式提供了在平衡各种因素下的最佳位置。
虽然它确实会遭遇到我之前提到的性能问题，但在所有其他方面都已经做得很好，
但仅性能这一点，它便不是完整的解决方案。
所以为此我们需要包含对快速失败（也就是Abandonment）的处理，
正如我们将会看到的，它远远优于任何当今广泛使用的其他基于异常的模型。

<!-- 
### Exceptions 
-->
### 异常 

<!-- 
The history of exceptions is fascinating.  During this journey I spent countless hours retracing the industry's steps.
That includes reading some of the original papers -- like Goodenough's 1975 classic, [Exception Handling: Issues
and a Proposed Notation](https://www.cs.virginia.edu/~weimer/2006-615/reading/goodenough-exceptions.pdf) -- in addition
to looking at the approaches of several languages: Ada, Eiffel, Modula-2 and 3, ML, and, [most inspirationally, CLU](
http://csg.csail.mit.edu/pubs/memos/Memo-155/Memo-155-3.pdf).  Many papers do a better job than I can summarizing the
long and arduous journey, so I won't do that here.  Instead, I'll focus on what works and what doesn't work for building
reliable systems. 
-->

异常的历史是令人着迷的。
在错误模型的研发过程中，我花了无数时间来回顾行业在这个领域的发展，
其中包括阅读的一些原始论文，如Goodenough在1975年经典文章，
[Exception Handling: Issues and a Proposed Notation](https://www.cs.virginia.edu/~weimer/2006-615/reading/goodenough-exceptions.pdf)。
除此之外，还研究了几种语言的方法：Ada，Eiffel，Modula-2和3，ML以及[给人以灵感的CLU](http://csg.csail.mit.edu/pubs/memos/Memo-155/Memo-155-3.pdf)。
许多论文比我能总结的漫长而艰辛之旅做得更好，所以我不将不在此赘述，
相反，我会专注于对于构建可靠系统来讲，哪些工作是有效以及哪些是无效的。

<!-- 
Reliability is the most important of our requirements above when developing the Error Model.  If you can't react
appropriately to failures, your system, by definition, won't be very reliable.  Operating systems generally speaking need
to be reliable.  Sadly, the most commonplace model -- unchecked exceptions -- is the worst you can do in this dimension. 
-->
在开发错误模型时，可靠性在我们上述的要求中是最重要的，
如果无法对故障做出适当的反应，那么根据定义，系统将不会非常可靠。
而通常来说，操作系统需要就是可靠性，
但可悲的是，最常见的模型——非检查性异常（unchecked exception），在这个维度上是做的最差的。

<!-- 
For these reasons, most reliable systems use return codes instead of exceptions.  They make it possible to locally
reason about and decide how best to react to error conditions.  But I'm getting ahead of myself.  Let's dig in. 
-->
出于以上原因，大多数可靠系统使用返回码而不是异常模型，
因为错误码使得，在局部进行分析并决定如何最好地对错误条件进行处理成为可能。
但是我说过头了，让我们继续深入下去。

<!-- 
### Unchecked Exceptions 
-->
### 非检查性异常

<!-- 
A quick recap.  In an unchecked exceptions model, you `throw` and `catch` exceptions, without it being part of the type
system or a function's signature.  For example: 
-->
让我们来快速回顾一下，
在非检查性异常模型中，代码`throw`并`catch`异常，
但异常不是类型系统或函数签名的一部分。
例如：

<!-- 
    // Foo throws an unhandled exception:
    void Foo() {
        throw new Exception(...);
    }

    // Bar calls Foo, and handles that exception:
    void Bar() {
        try {
            Foo();
        }
        catch (Exception e) {
            // Handle the error.
        }
    }

    // Baz also calls Foo, but does not handle that exception:
    void Baz() {
        Foo(); // Let the error escape to our callers.
    } 
-->

    // Foo抛出一个未处理的异常： 
    void Foo() {
        throw new Exception(...);
    }

    // Bar调用Foo，并处理该异常： 
    void Bar() {
        try {
            Foo();
        }
        catch (Exception e) {
            // 处理错误 
        }
    }

    // Baz也调用Foo，但不处理该异常： 
    void Baz() {
        Foo(); // 让错误逃逸到Baz的调用者 
    } 

<!-- 
In this model, any function call -- and sometimes any *statement* -- can throw an exception, transferring control
non-locally somewhere else.  Where?  Who knows.  There are no annotations or type system artifacts to guide your
analysis.  As a result, it's difficult for anyone to reason about a program's state at the time of the throw, the state
changes that occur while that exception is propagated up the call stack -- and possibly across threads in a concurrent
program -- and the resulting state by the time it gets caught or goes unhandled. 
-->
在这个模型中，任何函数调用（有时是任何*语句*）都可以抛出异常，
并将控制权转移到其他非局部的位置。那么到底转移到何处？没有人知道。
因为没有注解或类型系统的制品能辅助你的分析，因此任何人都很难推断程序在抛出时的状态，
和异常被捕获或保持未处理时的最终状态。
因为当异常在调用栈中向上传播，甚至在并发程序中以跨线程的方式传播时，程序的状态都可能发生更改。

<!-- 
It's of course possible to try.  Doing so requires reading API documentation, doing manual audits of the code, leaning
heavily on code reviews, and a healthy dose of luck.  The language isn't helping you out one bit here.  Because failures
are rare, this tends not to be as utterly disastrous as it sounds.  My conclusion is that's why many people in the industry
think unchecked exceptions are "good enough."  They stay out of your way for the common success paths and, because most
people don't write robust error handling code in non-systems programs, throwing an exception *usually* gets you out of a
pickle fast.  Catching and then proceeding often works too.  No harm, no foul.  Statistically speaking, programs "work." 
-->
当然这也是可以尝试的，为此需要阅读API文档，
对代码进行手动审计，并严重依赖于代码审查以及良好的运气，
但语言对此没有提供任何帮助。 
因为程序中的故障总是少数，所以这并不像听起来那样完全是灾难性的。
我的结论是，这就是为什么业内很多人认为非检查异常已经“足够好”，
他们会为了通常的成功路径而避免引入其它的干扰，
因为大多数人都不会在非系统程序中编写强大的错误处理代码，
而抛出异常*通常*会让你快速逃离错误引起的困境。
捕捉错误然后继续运行通常也是有效的，
没有造成伤害，没有不合规定。
因此从统计上讲，程序“能够工作”。

<!-- 
Maybe statistical correctness is okay for scripting languages, but for the lowest levels of an operating system, or any
mission critical application or service, this isn't an appropriate solution.  I hope this isn't controversial. 
-->
也许这种统计的正确性对于脚本语言是可行的，
但对于操作系统的最低层次或任何任务关键的应用程序和服务而言，
这不是一个合适的解决方案。
我希望对此是没有争议的。

<!-- 
.NET makes a bad situation even worse due to *asynchronous exceptions*.  C++ has so-called "asynchronous exceptions"
too: these are failures that are triggered by hardware faults, like access violations.  It gets really nasty in .NET,
however.  An arbitrary thread can inject a failure at nearly any point in your code.  Even between the RHS and LHS of an
assignment!  As a result, things that look atomic in source code aren't.  [I wrote about this 10 years ago](
http://joeduffyblog.com/2005/03/18/atomicity-and-asynchronous-exception-failures/) and the challenges still exist,
although the risk has lessened as .NET generally learned that thread aborts are problematic.  The new CoreCLR even
lacks AppDomains, and the new ASP.NET Core 1.0 stack certainly doesn't use thread aborts like it used to.  But the [APIs are
still there](https://github.com/dotnet/coreclr/blob/99e7f7c741a847454ab0ace1febd911378dcb464/src/mscorlib/src/System/Threading/Thread.cs#L518). 
-->
在.NET中，由于*异步异常*的存在，使得情况更加糟糕。
C++也有所谓的“异步异常”：由硬件故障触发的故障，例如访问违例。
然而，它在.NET中，它却显得非常令人讨厌，因为任意线程几乎可以在代码中的任何一处注入失败，
即使在赋值语句的RHS和LHS之间也是如此！ 
因此，源代码中看起来像原子的操作，在实际中却并不如此。
我在[10年前写过关于此的文章](http://joeduffyblog.com/2005/03/18/atomicity-and-asynchronous-exception-failures/)，
尽管现在已普遍意识到.NET的线程中止是有问题的并使得这种风险减弱，但挑战实际上仍然是存在的。
新的CoreCLR甚至缺少AppDomains，并且在ASP.NET Core 1.0中，栈肯定不会像过去那样使用线程中止，但[它们的API仍然存在](https://github.com/dotnet/coreclr/blob/99e7f7c741a847454ab0ace1febd911378dcb464/src/mscorlib/src/System/Threading/Thread.cs#L518)。

<!-- 
There's a famous interview with Anders Hejlsberg, C#'s chief designer, called [The Trouble with Checked Exceptions](
http://www.artima.com/intv/handcuffs.html).  From a systems programmer's perspective, much of it leaves you scratching
your head.  No statement affirms that the target customer for C# was the rapid application developer more than this: 
-->
有一个关于C#的首席设计师Anders Hejlsberg的著名采访，名为[The Trouble with Checked Exceptions](http://www.artima.com/intv/handcuffs.html)。
从系统程序员的角度来看，大部分的内容都会让你感到困惑，
但有一点却是可以理解的，没有声明能够肯定C#的目标客户是超过如下内容的快速应用程序开发者：

<!-- 
> *Bill Venners*: But aren't you breaking their code in that case anyway, even in a language without checked exceptions?
> If the new version of foo is going to throw a new exception that clients should think about handling, isn't their
> code broken just by the fact that they didn't expect that exception when they wrote the code?
> 
> *Anders Hejlsberg* : No, because in a lot of cases, people don't care. They're not going to handle any of these
> exceptions. There's a bottom level exception handler around their message loop. That handler is just going to bring
> up a dialog that says what went wrong and continue. The programmers protect their code by writing try finally's
> everywhere, so they'll back out correctly if an exception occurs, but they're not actually interested in handling
> the exceptions. 
-->
> *Bill Venners*：
> 但是你不是在这种情况下破坏他们的代码，即使是在缺乏检查性异常的语言中也是如此么？ 
> 如果foo的新版本将抛出一个应该处理的新异常类型，
> 那么他们的程序是不是仅仅因为在编写代码时没有预料该异常而导致崩溃呢？

> *Anders Hejlsberg*：
> 不会，因为在很多情况下，人们并不关心，
> 他们不会处理任何这些异常情况。
> 在他们的消息循环中有一个底层的异常处理程序，
> 该处理程序只是打开一个对话框，说明出了什么问题然后继续执行。
> 程序员通过在任何地方编写`try`和`finally`来保护他们的代码，
> 因此如果发生异常他们将正确退出，但他们实际上对异常的处理并不感兴趣。

<!-- 
This reminds me of `On Error Resume Next` in Visual Basic, and the way Windows Forms automatically caught and swallowed
errors thrown by the application, and attempted to proceed.  I'm not blaming Anders for his viewpoint here; heck, for
C#'s wild popularity, I'm convinced it was the right call given the climate at the time.   But this sure isn't the way
to write operating system code. 
-->
这让我想起了Visual Basic中的`On Error Resume Next`，以及Windows Forms自动捕获并透明处理应用程序抛出的错误，然后尝试继续执行的方式。 
我在这里并不责怪Anders的观点，由于C#的受欢迎程度，
我确信这是当时大背景下的正确选择，但这肯定不是编写操作系统代码的正确方法。

<!-- 
C++ at least *tried* to offer something better than unchecked exceptions with its [`throw` exception specifications](
http://en.cppreference.com/w/cpp/language/except_spec).  Unfortunately, the feature relied on dynamic enforcement which
sounded its death knell instantaneously. 
-->
C++至少*尝试过*使用它的[`throw`异常规范](http://en.cppreference.com/w/cpp/language/except_spec)
提供比非检查性异常更好的方式。 
但不幸的是，这个功能依赖于动态增强（dynamic enhancement），凭这一点便宣告了它的失败。

<!-- 
If I write a function `void f() throw(SomeError)`, the body of `f` is still free to invoke functions that throw things
other than `SomeError`.  Similarly, if I state that `f` throws no exceptions, using `void f() throw()`, it's still
possible to invoke things that throw.  To implement the stated contract, therefore, the compiler and runtime must ensure
that, should this happen, `std::unexpected` is called to rip the process down in response.
-->
如果我写一个函数`void f() throw(SomeError)`，
`f`的函数体仍然可以自由地调用抛出除SomeError之外的异常的函数；
类似地，如果我声明`f`不会抛出异常，
但使用`void f() throw()`仍然可以调用抛出异常的函数。
因此，为了实现所述的合约，编译器和运行时必须确保，
如果发生这两种情况，则需调用`std::unexpected`杀死进程以作为响应。

<!-- 
I'm not the only person to recognize this design was a mistake.  Indeed, `throw` is now deprecated.  A detailed WG21
paper, [Deprecating Exception Specifications](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2010/n3051.html),
describes how C++ ended up here, and has this to offer in its opening statement: 
-->
我不是唯一认识到这种设计是错误的人，事实上，`throw`已经遭到弃用。
一篇详细的WG21论文，[Deprecating Exception Specifications](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2010/n3051.html)，阐述了C++是如何沦落于此的，
在这里我提供了此文的开场白：

<!-- 
> Exception specifications have proven close to worthless in practice, while adding a measurable overhead to programs. 
-->
> 事实证明，异常规范在实践中几乎毫无价值，反而还为程序增加了可见的开销。

<!-- 
The authors list three reasons for deprecating `throw`.  Two of the three reasons were a result of the dynamic choice:
runtime checking (and its associated opaque failure mode) and runtime performance overheads.  The third reason, lack
of composition in generic code, could have been dealt with using a proper type system (admittedly at an expense). 
-->
作者列举了弃用`throw`的三个原因。
这三个原因中的两个是动态选择的结果：运行时检查（及其关联的opaque故障模式）和运行时性能开销。
第三个原因是，虽然泛型代码中缺乏合成，但可以使用适当的类型系统来解决（当然这也是有开销的）。

<!-- 
But the worst part is that the cure relies on yet another dynamically enforced construct -- the [`noexcept` specifier](
http://en.cppreference.com/w/cpp/language/noexcept_spec) -- which, in my opinion, is just as bad as the disease. 
-->
但最糟糕的是，这种方法依赖于另一个动态强制构造——
[`noexcept`分类符](http://en.cppreference.com/w/cpp/language/noexcept_spec)，
在我看来，这种解决方案与问题本身一样糟糕。

<!-- 
["Exception safety"](https://en.wikipedia.org/wiki/Exception_safety) is a commonly discussed practice in the C++
community.  This approach neatly classifies how functions are intended to behave from a caller's perspective with
respect to failure, state transitions, and memory management.  A function falls into one of four kinds: *no-throw* means
forward progress is guaranteed and no exceptions will emerge; *strong safety* means that state transitions happen
atomically and a failure will not leave behind partially committed state or broken invariants; *basic safety* means
that, though a function might partially commit state changes, invariants will not be broken and leaks are prevented; and
finally, *no safety* means anything's possible.  This taxonomy is quite helpful and I encourage anyone to be intentional
and rigorous about error behavior, either using this approach or something similar.  Even if you're using error codes.
The problem is, it's essentially impossible to follow these guidelines in a system using unchecked exceptions, except
for leaf node data structures that call a small and easily auditable set of other functions.  Just think about it: to
guarantee strong safety everywhere, you would need to consider the possibility of *all function calls throwing*, and
safeguard the surrounding code accordingly.  That either means programming defensively, trusting another function's
documented English prose (that isn't being checked by a computer), getting lucky and only calling `noexcept` functions,
or just hoping for the best.  Thanks to RAII, the leak-freedom aspect of basic safety is easier to attain -- and pretty
common these days thanks to smart pointers -- but even broken invariants are tricky to prevent.  The article
[Exception Handling: A False Sense of Security](
http://ptgmedia.pearsoncmg.com/images/020163371x/supplements/Exception_Handling_Article.html) sums this up well. 
-->
[“异常安全性（Exception safety）”](https://en.wikipedia.org/wiki/Exception_safety)是C++社区中经常讨论的一种实践方法。
这种方法巧妙地从调用者关于故障，状态转换和内存管理的视角对函数的意图进行分类。
函数被分类为四种类型之一：
*“no-throw”*意味着向前执行的程度得到保证且不会出现异常；
*“strong safety”*意味着状态转换以原子的方式发生，而故障不会导致部分提交的状态更改或不变量被破坏；
*“basic safety”*意味着，虽然函数可能发生部分提交的状态更改，但不会破坏不变量并能防止内存泄漏；
最后，*“no safety”*意味着一切皆有可能，无任何保障。
这种分类法非常有用，我鼓励任何人对错误行为采取有意和严谨的态度，
无论是使用这种方法还是其它类似的方法，即使是你使用的是错误码方式。

但问题是，除了叶节点数据结构调用一组小且易于审计的其他函数之外，
在使用非检查性异常的系统中遵循这些指南基本上是不可能的。
试想一下：为了保证各处的安全性，你需要考虑所有函数调用的可能性，并相应地对周围的代码进行保护。
这意味者要么进行防御性编程（Defensive programming），
要么信任所调用函数的（无法被计算机检查的）文档，
要么只调用`noexcept`函数而走运，要么只是希望最好的情况出现。
多亏有了RAII，免于泄露的基本安全性变得更容易实现
（由于智能指针的出现，现在这些安全也变得很常见），但破坏不变量这一点却也很难避免。
[“Exception Handling: A False Sense of Security”](http://ptgmedia.pearsoncmg.com/images/020163371x/supplements/Exception_Handling_Article.html)一文对此进行了很好地总结。

<!-- 
For C++, the real solution is easy to predict, and rather straightforward: for robust systems programs, don't use
exceptions.  That's the approach [Embedded C++](https://en.wikipedia.org/wiki/Embedded_C%2B%2B) takes, in addition to
numerous realtime and mission critical guidelines for C++, including NASA's Jet Propulsion Laboratory's.
[C++ on Mars sure ain't using exceptions anytime soon](https://www.youtube.com/watch?v=3SdSKZFoUa8). 
-->
对于C++而言，真正的解决方案很容易猜到，而且非常简单，那便是对于健壮的系统程序，不要使用异常处理。
这是[嵌入式C++](https://en.wikipedia.org/wiki/Embedded_C%2B%2B)的常用做法，
除此之外还有包括NASA喷气推进器实验室在内的C++的大量实时和任务关键指南也建议这么做。
因此，[火星上的C++代码肯定不会很快地使用上异常](https://www.youtube.com/watch?v=3SdSKZFoUa8)。

<!-- 
So if you can safely avoid exceptions and stick to C-like return codes in C++, what's the beef? -->
所以如果你可以安全地避免使用异常并坚持使用C++中的类C的返回码方式，那么问题又出在何处呢？

<!-- 
The entire C++ ecosystem uses exceptions.  To obey the above guidance, you must avoid significant parts of the language
and, it turns out, significant chunks of the library ecosystem.  Want to use the Standard Template Library?  Too bad, it
uses exceptions.  Want to use Boost?  Too bad, it uses exceptions.  Your allocator likely throws `bad_alloc`.  And so
on.  This even causes insanity like people creating forks of existing libraries that eradicates exceptions.  The Windows
kernel, for instance, has its own fork of the STL that doesn't use exceptions.  This bifurcation of the ecosystem is
neither pleasant nor practical to sustain. 
-->
问题出在，整个C++生态系统都在使用异常。
为了遵守上述指导原则，你必须避免使用该语言的相当一部分特性。
但事实证明，这些重要特性是库生态系统的重要组成部分。
想使用标准模板库？太糟糕了，它使用了异常，
想使用Boost？太糟糕了，它也使用异常。
你的内存分配器也可能会抛出`bad_alloc`异常等等。
甚至会导致创建现有库无异常处理分支一样的神经错乱做法，
例如，Windows内核有自己的STL分支，但它不使用异常处理，
但这种生态系统的分叉既不愉快也不实用。

<!-- 
This mess puts us in a bad spot.  Especially because many languages use unchecked exceptions.  It's clear that they are
ill-suited for writing low-level, reliable systems code.  (I'm sure I will make a few C++ enemies by saying this so
bluntly.)  After writing code in Midori for years, it brings me tears to go back and write code that uses unchecked
exceptions; even simply code reviewing is torture.  But "thankfully" we have checked exceptions from Java to learn and
borrow from ... Right? 
-->
这种混乱让我们陷入了困境，特别是因为许多语言使用非检查性异常。
很明显，它们不适合编写低层次的可靠的系统代码
（我如此会直截了说出来，肯定会招惹几个来自C++的反对者）。
在为Midori编写多年代码后，让我回去编写使用非检查性异常的代码，
会让我欲哭无泪，即使仅仅对代码进行审查也是一种折磨。
但“幸运的是”，我们已经有了来自Java的检查性异常用于学习和借鉴……不是吗？

<!-- 
### Checked Exceptions 
-->
### 检查性异常

<!-- 
Ah, checked exceptions.  The rag doll that nearly every Java programmer, and every person who's observed Java from an
arm's length distance, likes to beat on.  Unfairly so, in my opinion, when you compare it to the unchecked exceptions
mess. 
-->
检查性异常就像，几乎每个Java程序员以及近距离观察过Java的人都喜欢拍打的布娃娃。
在我看来，将它与非检查性异常的混乱进行比较是不公平的。

<!-- 
In Java, you know *mostly* what a method might throw, because a method must say so: 
-->
在Java中，因为方法必须进行如下申明，所以你知道*大多数*方法可能会抛出什么类型异常：

    void foo() throws FooException, BarException {
        ...
    } 


<!-- 
Now a caller knows that invoking `foo` could result in either `FooException` or `BarException` being thrown.  At
callsites a programmer must now decide: 1) propagate thrown exceptions as-is, 2) catch and deal with them, or 3) somehow
transform the type of exception being thrown (possibly even "forgetting" the type altogether).  For instance: 
-->
那么，现在调用者便知道调用`foo`可能导致抛出`FooException`或`BarException`类型的异常。
因此在调用点中，程序员必须如下决定：1）按原样传播抛出的异常；2）捕获并处理它们；
或者3）以某种方式转换抛出的异常类型（甚至可能是“完全遗忘掉”异常类型）。
例如：

<!-- 
    // 1) Propagate exceptions as-is:
    void bar() throws FooException, BarException {
        foo();
    }

    // 2) Catch and deal with them:
    void bar() {
        try {
            foo();
        }
        catch (FooException e) {
            // Deal with the FooException error conditions.
        }
        catch (BarException e) {
            // Deal with the BarException error conditions.
        }
    }

    // 3) Transform the exception types being thrown:
    void bar() throws Exception {
        foo();
    } 
-->

    // 1) 按原样传播异常： 
    void bar() throws FooException, BarException {
        foo();
    }

    // 2) 捕捉并处理它们： 
    void bar() {
        try {
            foo();
        }
        catch (FooException e) {
            // 处理FooException的错误情况 
        }
        catch (BarException e) {
            // 处理BarException的错误情况 
        }
    }

    // 3) 转换抛出的异常类型： 
    void bar() throws Exception {
        foo();
    } 

<!-- 
This is getting much closer to something we can use.  But it fails on a few accounts: 
-->
这与我们可以使用的东西越来越接近，但它在某些情况下也会失效：

<!-- 
1. Exceptions are used to communicate unrecoverable bugs, like null dereferences, divide-by-zero, etc.

2. You don't actually know *everything* that might be thrown, thanks to our little friend `RuntimeException`.  Because
   Java uses exceptions for all error conditions -- even bugs, per above -- the designers realized people would go mad
   with all those exception specifications.  And so they introduced a kind of exception that is unchecked.  That is, a
   method can throw it without declaring it, and so callers can invoke it seamlessly.

3. Although signatures declare exception types, there is no indication at callsites what calls might throw.

4. People hate them. 
-->

1. 异常用于传递不可恢复的错误，如null解引用，除零等；
2. 由于`RuntimeException`的存在，实际上并不知道可能抛出的*所有*内容，
   因为Java对所有错误条件使用异常，甚至对程序中的bug也是如此。
   设计师也意识到人们会对所有这些异常规范感到厌烦，
   因此他们引入了一种非检查性异常。
   也就是说，一个方法可以在不声明的情况下抛出这种异常，使得调用者可以无缝地调用它。
3. 虽然签名声明了异常类型，但在调用点没有迹象表明调用可能会抛出什么类型的异常；
4. 人们讨厌这种方式。

<!-- 
That last one is interesting, and I shall return to it later when describing the approach Midori took.  In summary,
peoples' distaste for checked exceptions in Java is largely derived from, or at least significantly reinforced by, the
other three bullets above.  The resulting model seems to be the worst of both worlds.  It doesn't help you to write
bulletproof code and it's hard to use.  You end up writing down a lot of gibberish in your code for little perceived
benefit.  And versioning your interfaces is a pain in the ass.  As we'll see later, we can do better. 
-->
最后一项是很有趣的，稍后我将在描述Midori所采用的方法时再回头来看看。
总而言之，人们对Java中检查性异常的厌恶主要源于上面其他三项，或者至少在其他三项上得到了增强。
由此产生的模型似乎是两种方式之间最糟糕的：它无法帮助你编写完全免于错误的代码，同时也难以实用。
最终，你在代码中写下了很多莫名其妙并且几乎没有什么好处的语句。
另外，对接口进行版本控制也是一件很痛苦的事。
正如我们稍后将会看到的，我们本来可以做得更好。

<!-- 
That versioning point is worth a ponder.  If you stick to a single kind of `throw`, then the versioning problem is no
worse than error codes.  Either a function fails or it doesn't.  It's true that if you design version 1 of your API to
have no failure mode, and then want to add failures in version 2, you're screwed.  As you should be, in my opinion.  An
API's failure mode is a critical part of its design and contract with callers.  Just as you wouldn't change the return
type of an API silently without callers needing to know, you shouldn't change its failure mode in a semantically
meaningful way.  More on this controversial point later on. 
-->
该版本控制点是值得深思的，如果你坚持使用单一类型的`throw`，
那么其版本控制问题并不比错误码更糟糕，函数要么失败或成功。
确实，如果你将API的第一版设计为无故障的模式，
然后想要在第二版中添加故障的抛出代码，那么事情就被你搞砸了，
在我看来，这种情况可能就会发生。
API的故障模式是其设计和与调用者之间的合约的关键部分，
正如你不会在调用者未知的情况下静默更改API的返回类型一样，
你也不应该以语义上有意义的方式更改其故障模式。
稍后将对这个有争议的问题进行讨论。

<!-- 
CLU has an interesting approach, as described in this crooked and wobbly scan of a 1979 paper by Barbara Liskov,
[Exception Handling in CLU](http://csg.csail.mit.edu/pubs/memos/Memo-155/Memo-155-3.pdf).  Notice that they focus a lot
on "linguistics"; in other words, they wanted a language that people would love.  The need to check and repropagate all
errors at callsites felt a lot more like return values, yet the programming model had that richer and slightly
declarative feel of what we now know as exceptions.  And most importantly, `signal`s (their name for `throw`) were
checked.  There were also convenient ways to terminate the program should an unexpected `signal` occur. 
-->
在CLU中有一种有趣的方法，正如Barbara Liskov在1979年的
[“Exception Handling in CLU”](http://csg.csail.mit.edu/pubs/memos/Memo-155/Memo-155-3.pdf)
一文中所描述的那样。
我注意到他们非常关注于“语言学”，换句话说，他们想要一种人们喜欢的编程语言：
在callites上检查和重新传播所有错误的感觉更像是返回值，
但编程模型对我们现在所知的异常有更丰富和略微声明式的感觉。
最重要的是，`signal`（它们现在的名字式`throw`）是检查性的，
并且如果发生意外`signal`，还有方便的方法来终止整个程序。

<!-- 
### Universal Problems with Exceptions 
-->
### 异常的中普遍存在的问题

<!-- 
Most exception systems get a few major things wrong, regardless of whether they are checked or unchecked. 
-->
大多数的异常系统，无论是检查性的还是非检查性的，都会出现一些普遍的问题。

<!-- 
First, throwing an exception is usually ridiculously expensive.  This is almost always due to the gathering of a stack
trace.  In managed systems, gathering a stack trace also requires groveling metadata, to create strings of function
symbol names.  If the error is caught and handled, however, you don't even need that information at runtime!
Diagnostics are better implemented in the logging and diagnostics infrastructure, not the exception system itself.  The
concerns are orthogonal.  Although, to really nail the diagnostics requirement above, *something* needs to be able to
recover stack traces; never underestimate the power of `printf` debugging and how important stack traces are to it. 
-->
首先，抛出异常开销通常非常大，
这基本上总是由收集堆栈跟踪（stack trace）所引起的。
在托管系统中，收集堆栈跟踪还需要对元数据进行搜索，以创建函数符号名称的字符串，
但是，如果错误得以捕获并处理，你甚至不需要在运行时获取这些信息；
诊断能够在日志记录和诊断基础设施中更好地被实现，而不是在异常系统本身中，
而上述这些关心点又是正交的。
但为了真正取得上述的诊断要求，*某些系统*需要能够恢复堆栈跟踪；
永远不要低估`printf`调试的强大能力以及堆栈跟踪对此的重要性。

<!-- 
Next, exceptions can significantly impair code quality.  I touched on this topic [in my last post](
http://joeduffyblog.com/2015/12/19/safe-native-code/), and there are [good papers on the topic in the context of C++](
http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.116.8337&rep=rep1&type=pdf).  Not having static type system
information makes it hard to model control flow in the compiler, which leads to overly conservative optimizers. 
-->
其次，异常会严重影响代码质量。
我在[上一篇文章](/2019/02/17/midori/4-safe-native-code/)中谈到了这个主题，
并且有[C++环境中关于该主题很好的论文](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.116.8337&rep=rep1&type=pdf)。
静态类型系统信息的缺乏使得很难在编译器中对控制流进行建模，从而导致优化器变得过于保守。

<!-- 
Another thing most exception systems get wrong is encouraging too coarse a granularity of handling errors.  Proponents
of return codes love that error handling is localized to a specific function call.  (I do too!)  In exception handling
systems, it's all too easy to slap a coarse-grained `try`/`catch` block around some huge hunk of code, without carefully
reacting to individual failures.  That produces brittle code that's almost certainly wrong; if not today, then after the
inevitable refactoring that will occur down the road.  A lot of this has to do with having the right syntaxes. 
-->
大多数异常系统出问题的另一个方面是鼓励对错误进行粗粒度的处理。
返回码的支持者喜欢将错误处理本地化为特定的函数调用（，而我也是这样做的）。
但在异常处理系统中，很容易在一些巨大的代码块周围加上粗粒度的`try`/`catch`块，而不会仔细地对单个故障做出反应。
这种方式产生的脆弱代码几乎肯定是会出错的。
如果不是今天，那么沿着这条路走，不可避免的重构必将会发生，
因此，这与拥有正确的语法有很大程度的关系。

<!-- 
Finally, control flow for `throw`s is usually invisible.  Even with Java, where you annotate method signatures, it's not
possible to audit a body of code and see precisely where exceptions come from.  Silent control flow is just as bad as
`goto`, or `setjmp`/`longjmp`, and makes writing reliable code very difficult. 
-->
最后，`throw`的控制流通常是不可见的，即使是使用Java的注解方法签名的方式，也无法对代码体进行审计并准确查看异常的来源。
静默的控制流与`goto`或`setjmp`/`longjmp`一样糟糕，并且使编写可靠代码变得非常困难。

<!-- 
## Where Are We? 
-->
## 我们身在何处？

<!-- 
Before moving on, let's recap where we are: 
-->
在继续前行之前，让我们回顾一下我们所处的位置：

<!-- ![Good-Bad-Ugly](/assets/img/2016-02-07-the-error-model-1.png) -->

{% asset_img 2016-02-07-the-error-model-1.png  “好家伙，坏家伙，丑家伙” %}

<!-- 
Wouldn't it be great if we could take all of The Goods and leave out The Bads and The Uglies? 
-->
如果我们可以带走所有的“Goods（好家伙）”，
并舍弃“Bads（坏家伙）”和“Uglies（丑家伙）”，那不是一件很好的事吗？

<!-- 
This alone would be a great step forward.  But it's insufficient.  This leads me to our first big "ah-hah" moment that
shaped everything to come.  For a significant class of error, *none* of these approaches are appropriate! 
-->
仅次一点就是向前迈出的一大步，但这还远远不够，
这让我想起了我们首个影响未来一切的“欢呼”时刻。
对于一大类的错误，表中的所有这些方法都*不*合适！

<!-- 
# Bugs Aren't Recoverable Errors! 
-->
# Bug不是可恢复的错误！

<!-- 
A critical distinction we made early on is the difference between recoverable errors and bugs: -->
我们早期做出的一个重要区分，是可恢复错误和bug之间的区别：

<!-- 
* A *recoverable error* is usually the result of programmatic data validation.  Some code has examined the state of the
  world and deemed the situation unacceptable for progress.  Maybe it's some markup text being parsed, user input from a
  website, or a transient network connection failure.  In these cases, programs are expected to recover.  The developer
  who wrote this code must think about what to do in the event of failure because it will happen in well-constructed
  programs no matter what you do.  The response might be to communicate the situation to an end-user, retry, or abandon
  the operation entirely, however it is a *predictable* and, frequently, *planned* situation, despite being called an
  "error." 
-->
* *可恢复的错误*通常是程序化的数据验证的结果，
  一些代码对上下文的状态进行了检查，并认为这种状态对于继续执行是不可接受的。
  也许是正在解析的标记文本、来自网站的用户输入或易失网络的连接故障等，
  在这些情况下，程序期望能够正常恢复。
  编写此代码的开发者必须考虑在发生故障时该怎么做，
  因为无论你怎么编写代码，这些情况都会在构造良好的程序中发生。
  响应的方式可能是将情况传达给终端用户、重试或完全放弃的操作，
  但这是一种*可预测的*，经常是*计划内的*情况，尽管它被称为“错误”。


<!-- 
* A *bug* is a kind of error the programmer didn't expect.  Inputs weren't validated correctly, logic was written wrong,
  or any host of problems have arisen.  Such problems often aren't even detected promptly; it takes a while until
  "secondary effects" are observed indirectly, at which point significant damage to the program's state might have
  occurred.  Because the developer didn't expect this to happen, all bets are off.  All data structures reachable by
  this code are now suspect.  And because these problems aren't necessarily detected promptly, in fact, a whole lot
  more is suspect.  Depending on the isolation guarantees of your language, perhaps the entire process is tainted. 
-->
* *bug*是程序员没有意料到的一种错误，这包括输入未正确验证、逻辑的编码错误或出现任何问题。
  这些问题通常甚至不能被及时发现，
  它的“次要影响”需要一段时间才能间接地观察到，
  而此时可能会对程序的状态造成重大破坏。
  因为开发者没料到会发生这种情况，所以一切都结束了。
  此代码可访问的所有数据结构都是被怀疑的对象，
  而且因为这些问题不一定能及时被发现，事实上，更多的部分将是不可信的。
  根据语言的隔离保证，可能使得整个进程都受到了污染。

<!-- 
This distinction is paramount.  Surprisingly, most systems don't make one, at least not in a principled way!  As we saw
above, Java, C#, and dynamic languages just use exceptions for everything; and C and Go use return codes.  C++ uses a
mixture depending on the audience, but the usual story is a project picks a single one and uses it everywhere.  You
usually don't hear of languages suggesting *two* different techniques for error handling, however. 
-->
这种区分至关重要。但令人惊讶的是，大多数系统都没有做到这一点，至少不是以一种原则性的方式进行区分！
如上所述，Java，C#和动态语言仅使用异常，而C和Go仅使用返回码。 
C++根据受众的不用使用两者混合的方式，但通常的情况是项目选择其中之一，并在代码的任何地方都这样使用。
但你通常不会听到某种语言建议使用*这两种*不同的错误处理技术。

<!-- 
Given that bugs are inherently not recoverable, we made no attempt to try.  All bugs detected at runtime caused
something called *abandonment*, which was Midori's term for something otherwise known as ["fail-fast"](
https://en.wikipedia.org/wiki/Fail-fast). 
-->
鉴于bug本质上是不可恢复的，我们没有试图对其进行`try`捕捉。
在运行时检测到的所有bug所导致的后果，在Midori的术语中被称为*Abandonment（放弃）*，
也就是所谓的[“快速失败”](https://en.wikipedia.org/wiki/Fail-fast)。

<!-- 
Each of the above systems offers abandonment-like mechanisms.  C# has `Environment.FailFast`; C++ has `std::terminate`;
Go has `panic`; Rust has `panic!`; and so on.  Each rips down the surrounding context abruptly and promptly.  The scope
of this context depends on the system -- for example, C# and C++ terminate the process, Go the current Goroutine, and
Rust the current thread, optionally with a panic handler attached to salvage the process.
-->
上述每种语言都提供类似Abandonment的机制：
C#有`Environment.FailFast`，C++有`std::terminate`，Go有`panic`，以及Rust有`panic!`等等。
每一方式都突然且迅速地销毁掉上下文环境，而此上下文的范围则取决于系统，
例如，对于C#和C++来说是终止进程，对于Go来说是Goroutine，而对于Rust来说则是当前线程，
并可选择地附加一个panic处理程序来对进程进行抢救。

<!--
Although we did use abandonment in a more disciplined and ubiquitous way than is common, we certainly weren't the first
to recognize this pattern.  This [Haskell essay](https://wiki.haskell.org/Error_vs._Exception), articulates this
distinction quite well: 
-->
虽然我们确实以比一般的更有纪律和无处不在的方式使用Abandonment，
但我们当然不是第一个认识到这种模式的团队。
这篇[Haskell文章](https://wiki.haskell.org/Error_vs._Exception)非常清楚地表达了这种区别：

<!-- 
> I was involved in the development of a library that was written in C++.  One of the developers told me that the
> developers are divided into the ones who like exceptions and the other ones who prefer return codes. As it seem to me,
> the friends of return codes won.  However, I got the impression that **they debated the wrong point: Exceptions and
> return codes are equally expressive**, they should however not be used to describe errors. Actually the return codes
> contained definitions like `ARRAY_INDEX_OUT_OF_RANGE`.  But I wondered: How shall my function react, when it gets this
> return code from a subroutine?  Shall it send a mail to its programmer?  It could return this code to its caller in
> turn, but it will also not know how to cope with it.  Even worse, since I cannot make assumptions about the
> implementation of a function, I have to expect an `ARRAY_INDEX_OUT_OF_RANGE` from every subroutine. My conclusion is
> that `ARRAY_INDEX_OUT_OF_RANGE` is a (programming) error.  **It cannot be handled or fixed at runtime, it can only be
> fixed by its developer. Thus there should be no according return code, but instead there should be asserts.** 
-->
> 我参与了用C++编写的库的开发。其中一位开发者告诉我，开发者可以分为喜欢异常和喜欢返回码两种类型。
> 在我看来，使用返回码的朋友赢了。
> 但是，我得到的印象是**他们论点是错误的：异常和返回代同有同样的表达能力**，但这并不适用于描述bug。
> 实际上，返回码包含`ARRAY_INDEX_OUT_OF_RANGE`等定义，
> 但我想知道：当程序从子程序获得该返回码时，它将如何对其作出反应？
> 它应该向程序员发送邮件告知吗？
> 它可以依次将此错误码返回给其调用者，但它们其实也都不知道如何对其进行处理。
> 更糟糕的是，由于我不能对函数的实现做出假设，
> 所以不得不认为每个子程序都有可能返回`ARRAY_INDEX_OUT_OF_RANGE`。
> 我的结论是`ARRAY_INDEX_OUT_OF_RANGE`是一个（编程性）的错误，
> **无法在运行时对其进行处理或修复，只能由其开发者来修复。因此，这不应该有这样的返回码存在，而是应该采用断言的方式**。

<!-- 
Abandoning fine grained mutable shared memory scopes is suspect -- like Goroutines or threads or whatever -- unless your
system somehow makes guarantees about the scope of the potential damage done.  However, it's great that these mechanisms
are there for us to use!  It means using an abandonment discipline in these languages is indeed possible. 
-->
放弃细粒度的可变共享内存作用域是不可信的，比如Goroutine、线程或其他方式，
除非你的系统以某种方式保证潜在的内存破坏范围。
但是，很不错的是这些机制是存在的并可以被我们所利用！
这也就意味着在这些语言中使用Abandonment机制确实是有可能的。

<!-- 
There are architectural elements necessary for this approach to succeed at scale, however.  I'm sure you're thinking
"If I tossed the entire process each time I had a null dereference in my C# program, I'd have some pretty pissed off
customers"; and, similarly, "That wouldn't be reliable at all!"  Reliability, it turns out, might not be what you think. 
-->
然而，它还必须具有大规模成功所必需的架构元素。
我相信你在想“如果我每次在我的C#程序中进行null解引用时都会抛出整个进程，
那么我的一些客户会非常的生气”，和“这根本不可靠！”等相似的想法。
事实证明，可靠性可能与你的想法有所不同。

<!-- 
# Reliability, Fault-Tolerance, and Isolation 
-->
# 可靠性，容错和隔离

<!-- 
Before we get any further, we need to state a central belief: <s>Shi</s> Failure Happens. 
-->
在我们进一步讨论之前，我们需要先表达中心信念：故障会发生。

<!-- 
## To Build a Reliable System 
-->
## 构建可靠的系统

<!-- 
Common wisdom is that you build a reliable system by systematically guaranteeing that failure can never happen.
Intuitively, that makes a lot of sense.  There's one problem: in the limit, it's impossible.  If you can spend millions
of dollars on this property alone -- like many mission critical, realtime systems do -- then you can make a significant
dent.  And perhaps use a language like [SPARK](https://en.wikipedia.org/wiki/SPARK_(programming_language)) (a set of
contract-based extensions to Ada) to formally prove the correctness of each line written.  However, [experience shows](
https://en.wikipedia.org/wiki/List_of_software_bugs) that even this approach is not foolproof. -->
这里的常识是通过系统地保证故障永不发生的方式，来构建一个可靠的系统。
直观上来说，这是很有道理的，但有一个问题：在极限的情况下，这是不可能的。
如果你可以单独花费数百万美元来获得这样的可靠性，就像许多任务关键的实时系统一样，那么会给你留下深刻的印象。
也许使用像[SPARK](http://t.cn/EMGjc0l)这样的语言（一组基于合约的Ada扩展）来形式化证明每行代码的正确性。
然而，[经验表明](https://en.wikipedia.org/wiki/List_of_software_bugs)即使这种方法也不是万无一失的。

<!-- 
Rather than fighting this fact of life, we embraced it.  Obviously you try to eliminate failures where possible.  The
error model must make them transparent and easy to deal with.  But more importantly, you architect your system so that
the whole remains functional even when individual pieces fail, and then teach your system to recover those failing
pieces gracefully.  This is well known in distributed systems.  So why is it novel? 
-->
我们接受而不是反对这一事实。
但如果显然试图在所有可能的情况下消除失败，那么错误模型必须使它们透明且易于处理。
更重要的是，系统被构建为即使单个部件出现故障，整个系统仍然可以正常运行，
然后再指导系统优雅地恢复那些故障部分。
这种原则在分布式系统中是众所周知的，那为什么它又是新颖的呢？

<!-- 
At the center of it all, an operating system is just a distributed network of cooperating processes, much like a
distributed cluster of microservices or the Internet itself.  The main differences include things like latency; what
levels of trust you can establish and how easily; and various assumptions about locations, identity, etc.  But failure
in [highly asynchronous, distributed, and I/O intensive systems](
http://joeduffyblog.com/2015/11/19/asynchronous-everything/) is just bound to happen.  My impression is that, largely
because of the continued success of monolithic kernels, the world at large hasn't yet made the leap to "operating system
as a distributed system" insight.  Once you do, however, a lot of design principles become apparent. 
-->
处于这一切的中心的操作系统，只是协作进程们的分布式网络，就像微服务的分布式集群或互联网本身，
它与这些系统的主要区别在于延迟，和可以建立什么样的信任程度以及达到的难度如何，以及关于位置，身份的各种假设等。
但[高度异步，分布式和I/O密集型系统](/2018/11/25/midori/3-asynchronous-everything/)中的故障必然会发生。
对此，我的看法是，很大程度上是因为宏内核的持续成功，使得整个世界还没有跃升到“作为分布式系统的操作系统”的洞察力。
但是，一旦你这么做了，很多的设计原则就会变得明显起来。

<!-- 
As with most distributed systems, our architecture assumed process failure was inevitable.  We went to great
length to defend against cascading failures, journal regularly, and to enable restartability of programs and services. 
-->
与大多数分布式系统一样，我们的架构假设进程失败是不可避免的，
尽管我们花了很长时间来防止级联故障发生，定期日志记录，以及实现程序和服务的可重启性。

<!-- 
You build things differently when you go in assuming this. 
-->
当你如此假设时，便会以不同的方式来构建整个系统。

<!-- 
In particular, isolation is critical.  Midori's process model encouraged lightweight fine-grained isolation.  As a
result, programs and what would ordinarily be "threads" in modern operating systems were independent isolated entities.
Safeguarding against failure of one such connection is far easier than when sharing mutable state in an address space. 
-->
特别地，隔离至关重要，Midori的进程模型有助于轻量级细粒度的隔离，
因此，程序和现代操作系统中通常称为“线程”的构造是独立的孤立实体。
对一个这样的系统中的网络连接失败进行保护比在地址空间中的共享可变状态下要容易得多。

<!-- 
Isolation also encourages simplicity.  Butler Lampson's classic [Hints on Computer System Design](
http://research.microsoft.com/pubs/68221/acrobat.pdf) explores this topic.  And I always loved this quote from Hoare: 
-->
隔离同样也有助于简单性，
Butler Lampson的经典文章[“Hints on Computer System Design”](http://research.microsoft.com/pubs/68221/acrobat.pdf)探索了这个主题。
我一直很喜欢Hoare的这句话：

<!-- 
> The unavoidable price of reliability is simplicity. (C. Hoare). 
-->
> 可靠性的无法避免的代价是简单性（C. Hoare）。

<!-- 
By keeping programs broken into smaller pieces, each of which can fail or succeed on its own, the state machines within
them stay simpler.  As a result, recovering from failure is easier.  In our language, the points of possible failure
were explicit, further helping to keep those internal state machines correct, and pointing out those connections with
the messier outside world.  In this world, the price of individual failure is not nearly as dire.  I can't
over-emphasize this point.  None of the language features I describe later would have worked so well without this
architectural foundation of cheap and ever-present isolation. 
-->
通过将程序分解成更小的部分，每个部分都可以自行失败或成功，
使得其中的每个状态机都能保持为更简单的形式，因此使得从故障中恢复变得更加容易。
在我们的语言中，可能的失败点是明确的，从而有利于进一步保持这些内部状态机的正确性，并指明了与混乱的外部世界的接口。
在这个世界上，单个个体失败的代价并不是那么可怕，所以我不能过分强调这一点。
如果没有廉价和永远在线的隔离的架构基础，我后面描述的语言功能都不会很好的工作。

<!-- 
Erlang has been very successful at building this property into the language in a fundamental way.  It, like Midori,
leverages lightweight processes connected by message passing, and encourages fault-tolerant architectures.  A common
pattern is the "supervisor," where some processes are responsible for watching and, in the event of failure, restarting
other processes.  [This article](http://ferd.ca/the-zen-of-erlang.html) does a terrific job articulating this philosophy
-- "let it crash" -- and recommended techniques for architecting reliable Erlang programs in practice. 
-->
Erlang非常成功地以一种基础的方式将这样的属性构建到语言中。
它与Midori一样，利用通过消息传递连接的轻量级进程并鼓励容错架构等措施来实现。
它采取的一种常见的模式是“监视器”模式：其中一些进程负责监测环境，并在其他进程发生故障时重启这些进程。
[这篇文章](http://ferd.ca/the-zen-of-erlang.html)在明确“让它崩溃（let it crash）”的哲学理念，
和关于在实践中构建可靠的Erlang程序的推荐技术上，做了很不错的工作。

<!-- 
The key thing, then, is not preventing failure per se, but rather knowing how and when to deal with it. 
-->
因此，关键不在于要防止失败本身，而是要知道如何以及何时处理它们。

<!-- 
Once you've established this architecture, you beat the hell out of it to make sure it works.  For us, this meant
week-long stress runs, where processes would come and go, some due to failures, to ensure the system as a whole kept
making good forward progress.  This reminds me of systems like Netflix's [Chaos Monkey](
https://github.com/Netflix/SimianArmy/wiki/Chaos-Monkey) which just randomly kills entire machines in your cluster to
ensure the service as a whole stays healthy. 
-->
一旦你构建了这种架构，你就会战胜它们并确保系统的运行。
对我们来说，这意味着为期一周的压力运行：进程不断启动和中止，
有些是由故障造成的，并同时确保整个系统良好地前进。
这让我想起了像Netflix的[Chaos Monkey](https://github.com/Netflix/SimianArmy/wiki/Chaos-Monkey)这样的系统，
它会随机终止集群中的某些机器的运行，以确保整个服务保持健康状态。

<!-- 
I expect more of the world to adopt this philosophy as the shift to more distributed computing happens.  In a cluster of
microservices, for example, the failure of a single container is often handled seamlessly by the enclosing cluster
management software (Kubernetes, Amazon EC2 Container Service, Docker Swarm, etc).  As a result, what I describe in this
post is possibly helpful for writing more reliable Java, Node.js/JavaScript, Python, and even Ruby services.  The
unfortunate news is you're likely going to be fighting your languages to get there.  A lot of code in your process is
going to work real damn hard to keep limping along when something goes awry. 
-->
随着更多的向分布式计算的转变不断发生，我期望更多的系统采用这种理念。
例如，在微服务集群中，单个容器的故障通常由封闭的集群管理软件
（如Kubernetes，Amazon EC2 Container Service和Docker Swarm等）无缝地处理。
因此，我在本文中描述的内容可能有助于编写更可靠的Java，Node.js/JavaScript，Python甚至是Ruby服务。
不幸的是，你很可能会与所使用的语言不断抗争来实现这样的目标，
因为很多进程的代码在出现问题时也会万分努力地艰难执行。


## Abandonment 


<!-- 
Even when processes are cheap and isolated and easy to recreate, it's still reasonable to think that abandoning an
entire process in the face of a bug is an overreaction.  Let me try to convince you otherwise. 
-->
即使进程是轻量级、隔离且易于重新创建的，你仍然有理由认为在面对错误时放弃整个进程是一种过度的反应。
那么下面让我试着来说服你。

<!-- 
Proceeding in the face of a bug is dangerous when you're trying to build a robust system.  If a programmer didn't expect
a given situation that's arisen, who knows whether the code will do the right thing anymore.  Critical data structures
may have been left behind in an incorrect state.  As an extreme (and possibly slightly silly) example, a routine that is
meant to round your numbers *down* for banking purposes might start rounding them *up*. 
-->
当你尝试构建一个健壮的系统时，在运行过程中遇到bug是危险的。
如果程序员没有预料到会出现特定的情况，没人知道代码下次是否还会再做正确的事情。
关键的数据结构可能在不正确的状态下被丢弃，
一个极端（也可能是稍微有点愚蠢）的例子是，一个用于银行业务的例程，
其本来目的是将你的存款数字向*下*舍入现在则可能变成了向*上*舍入。

<!-- 
And you might be tempted to whittle down the granularity of abandonment to something smaller than a process.  But that's
tricky.  To take an example, imagine a thread in your process encounters a bug, and fails.  This bug might have been
triggered by some state stored in a static variable.  Even though some other thread might *appear* to have been
unaffected by the conditions leading to failure, you cannot make this conclusion.  Unless some property of your system
-- isolation in your language, isolation of the object root-sets exposed to independent threads, or something else --
it's safest to assume that anything other than tossing the entire address space out the window is risky and unreliable. 
-->
你可能会试图将Abandonment的粒度减少到比进程更小的范围里，但这种方式是很棘手的。
举一个例子来说，假设你的进程中的一个线程遇到了一个bug，并且执行失败，
而且这个bug可能是由存储在静态变量中的某些状态所触发的。
即使其他线程似乎*看起来*不受导致故障的条件的影响，但你也无法对此确定。
除非你的系统有一些属性，被你的语言隔离，隔离暴露给独立线程或其他地方的对象根集合，
那么最安全的做法是，假设除了把整个地址空间全部销毁之外的任何操作都是有风险和不可靠的。

<!-- 
Thanks to the lightweight nature of Midori processes, abandoning a process was more like abandoning a single thread in a
classical system than a whole process.  But our isolation model let us do this reliably. 
-->
由于Midori进程的轻量级特性，放弃进程更像是在经典操作系统中放弃单个线程而不是整个进程。
但我们的隔离模型让我们可靠地做到了这一点。

<!-- 
I'll admit the scoping topic is a slippery slope.  Maybe all the data in the world has become corrupt, so how do you
know that tossing the process is even enough?!  There is an important distinction here.  Process state is transient by
design.  In a well designed system it can be thrown away and recreated on a whim.  It's true that a bug can corrupt
persistent state, but then you have a bigger problem on your hands -- a problem that must be dealt with differently. 
-->
我承认作用域主题是个滑坡谬误：也许环境中所有的数据都已经被破坏了，
那么你怎么知道中止掉这个进程就足够了？！
这里有一个重要的区别，也就是进程的状态在设计上是瞬态的。
在一个设计良好的系统中，它可以被丢弃并随意地被重建。
没错，一个bug会对持久性状态造成破坏，但是你手头上有一个更大的麻烦，
这个麻烦必须以不同的方式进行处理。

<!-- 
For some background, we can look to fault-tolerant systems design.  Abandonment (fail-fast) is already a common
technique in that realm, and we can apply much of what we know about these systems to ordinary programs and processes.
Perhaps the most important technique is regularly journaling and checkpointing precious persistent state.  Jim Gray's
1985 paper, [Why Do Computers Stop and What Can Be Done About It?](
http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.110.9127&rep=rep1&type=pdf), describes this concept nicely.
As programs continue moving to the cloud, and become aggressively decomposed into smaller independent services, this
clear separation of transient and persistent state is even more important.  As a result of these shifts in how software
is written, abandonment is far more achievable in modern architectures than it once was.  Indeed, abandonment can help
you avoid data corruption, because bugs detected before the next checkpoint prevent bad state from ever escaping. 
-->
对于某些背景，我们可以考虑容错的系统设计。
Abandonment（快速失败）已经是该领域的常用技术，
我们可以将我们对这些系统的大部分知识应用于普通程序和进程，
也许最重要的技术是定期记录和对宝贵的持久性状态做快照。
Jim Gray在1985年的论文，[“Why Do Computers Stop and What Can Be Done About It?”](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.110.9127&rep=rep1&type=pdf)，
很好地描述了这个概念。
随着程序持续地向云平台迁移，并且激进地分解为更小的独立服务的趋势，
瞬态和持久状态的这种明确区分甚至变得更为重要。
由于这种软件编写方式的转变，在现代架构中Abandonment比以前更容易实现。
实际上，放弃可以帮助你避免数据被损坏，
因为在下一个快照之前检测到的bug可以防止错误状态的逸出。

<!-- 
Bugs in Midori's kernel were handled differently.  A bug in the microkernel, for instance, is an entirely different
beast than a bug in a user-mode process.  The scope of possible damage was greater, and the safest response was to
abandon an entire "domain" (address space).  Thankfully, most of what you'd think of being classic "kernel"
functionality -- the scheduler, memory manager, filesystem, networking stack, and even device drivers -- was run
instead in isolated processes in user-mode where failures could be contained in the usual ways described above. 
-->
Midori内核中的bug处理方式也有所不同，因为微内核中的bug与用户进程中的bug就完全不同。
它可能造成的损害范围更大，因此最安全的反应是放弃整个“域”（地址空间）。
值得庆幸的是，大多数你认为经典的“内核”功能——调度器、内存管理器、
文件系统、网络堆栈甚至是设备驱动程序，都是在用户模式以隔离进程的方式运行，
所以故障以如上所述的通常方式被限定在隔离的进程中。

<!-- 
# Bugs: Abandonment, Assertions, and Contracts 
-->
# Bug：Abandonment，断言和合约

<!-- 
A number of kinds of bugs in Midori might trigger abandonment: 
-->
Midori中如下的一些bug可能会导致Abandonment：

<!-- 
* An incorrect cast.
* An attempt to dereference a `null` pointer.
* An attempt to access an array outside of its bounds.
* Divide-by-zero.
* An unintended mathematical over/underflow.
* Out-of-memory.
* Stack overflow.
* Explicit abandonment.
* Contract failures.
* Assertion failures. 
-->
* 不正确的强制类型转换；
* 试图解引用`null`指针；
* 试图越界访问数组；
* 除零；
* 意外的数学上/下溢；
* 内存不足；
* 栈溢出；
* 显式的放弃；
* 合约失败；
* 断言失败。
  
<!-- 
Our fundamental belief was that each is a condition the program cannot recover from.  Let's discuss each one. 
-->
我们的基本理念是：以上每种都是程序无法自动恢复的条件，让我们来依次讨论其中每一个。

<!-- 
## Plain Old Bugs 
-->
## 普通的旧Bug类型

<!-- 
Some of these situations are unquestionably indicative of a program bug. 
-->
一些情形毫无疑问表明程序存在bug。

<!-- 
An incorrect cast, attempt to dereference `null`, array out-of-bounds access, or divide-by-zero are clearly problems
with the program's logic, in that it attempted an undeniably illegal operation.  As we will see later, there are ways
out (e.g., perhaps you want NaN-style propagation for DbZ).  But by default we assume it's a bug. 
-->
不正确的强制转换，试图对`null`指针解引用，
数组越界访问或除零显然是程序逻辑的问题，
因为它试图进行无法否认的非法操作。
正如我们稍后将看到的，有一些是可以解决的
（例如对于除零操作而言，你可能想使用NaN风格的传播），
但默认情况下我们认为这是一个bug。

<!-- 
Most programmers were willing to accept this without question.   And dealing with them as bugs this way brought
abandonment to the inner development loop where bugs during development could be found and fixed fast.  Abandonment
really did help to make people more productive at writing code.  This was a surprise to me at first, but it makes sense. 
-->
大多数的程序员都毫无疑问地愿意接受这一点，
并且将它们以这种方式作为bug处理可将Abandonment带到内部开发循环中，
从而有助于快速找到并修复开发过程中的bug。
Abandonment确实有助于提高人们编写代码的效率，
起初这对我来说是一个意外惊喜，但它确实是有道理的。

<!-- 
Some of these situations, on the other hand, are subjective.  We had to make a decision about the default behavior,
often with controversy, and sometimes offer programmatic control. 
-->
另一方面，其他的一些情况则是比较主观的。
我们必须对这些情况的默认行为做出决定，
这通常会引起争议，所以有时还需提供对程序的控制。

<!-- 
### Arithmetic Over/Underflow 
-->

### 算术上/下溢出

<!-- 
Saying an unintended arithmetic over/underflow represents a bug is certainly a contentious stance.  In an unsafe system,
however, such things frequently lead to security vulnerabilities.  I encourage you to review the National Vulnerability
Database to see [the sheer number of these](
https://web.nvd.nist.gov/view/vuln/search-results?query=%22integer+overflow%22&search_type=all&cves=on). 
-->
如果说意外的算术上/下溢出是一种bug，这肯定是有争议的说法。
然而，在不安全的系统中，这种情况经常导致安全漏洞，
对此，我建议你打开国家漏洞数据库来看看[这种类型漏洞的绝对数量](https://web.nvd.nist.gov/view/vuln/search-results?query=%22integer+overflow%22&search_type=all&cves=on)。

<!-- 
In fact, the Windows TrueType Font parser, which we ported to Midori (with *gains* in performance), has suffered over a
dozen of them in the past few years alone.  (Parsers tend to be farms for security holes like this.) 
-->
事实上，我们移植到Midori（并获得性能*提升*）的Windows TrueType字体解析器，
仅在过去几年就遭受了十几次的上/下溢出（解析器往往是像这样的安全漏洞的发生地）。

<!-- 
This has given rise to packages like [SafeInt](https://safeint.codeplex.com/), which
essentially moves you away from your native language's arithmetic operations, in favor of checked library ones. 
-->
这就产生了像[SafeInt](https://safeint.codeplex.com/)这样的软件包，
它基本上使你远离了原生的算术运算，转而使用检查性的库来实现。

<!-- 
Most of these exploits are of course also coupled with an access to unsafe memory.  You could reasonably argue therefore
that overflows are innocuous in a safe language and therefore should be permitted.  It's pretty clear, however, based on
the security experience, that a program often does the wrong thing in the face of an unintended over/underflow.  Simply
put, developers frequently overlook the possibility, and the program proceeds to do unplanned things.  That's the
definition of a bug which is precisely what abandonment is meant to catch.  The final nail in the coffin on this one is
that philisophically, when there was any question about correctness, we tended to err on the side of explicit intent.
-->
这些漏洞中的大多数当然还伴随着对不安全内存的访问，
因此，你可以有理由的认为，溢出在安全语言中是无害的，所以应该允许出现。
然而，基于安全上的经验，很明显的是，程序在面临意外的上/下溢时经常会做错事。
简单地说，开发者经常忽略溢出的可能性，使得程序继续执行计划之外的事。
而这恰好是Abandonment试图捕捉的bug所定义的内容。
关于这一点的最致命一击是哲学上的，
当有任何关于正确性上的问题时，我们倾向于在明确的意图方面犯错误。

<!-- 
Hence, all unannotated over/underflows were considered bugs and led to abandonment.  This was similar to compiling
C# with [the `/checked` switch](https://msdn.microsoft.com/en-us/library/h25wtyxf.aspx), except that our compiler
aggressively optimized redundant checks away.  (Since few people ever think to throw this switch in C#, the
code-generators don't do nearly as aggressive a job in removing the inserted checks.)  Thanks to this language and
compiler co-development, the result was far better than what most C++ compilers will produce in the face of SafeInt
arithmetic.  Also as with C#, [the `unchecked` scoping construct](
https://msdn.microsoft.com/en-us/library/khy08726.aspx) could be used where over/underflow was intended. 
-->
因此，所有未注解的上/下溢出都被视为bug并会导致Abandonment。
除了我们的编译器会激进地优化掉冗余检查之外，
这基本上与使用[`/checked`选项](https://msdn.microsoft.com/en-us/library/h25wtyxf.aspx)来编译C#程序相类似
（因为很少有人考虑在C#中使用这个选项，
所欲代码生成器在删除冗余的插入检查时几乎没有那么积极地处理）。
多亏了这种语言和编译器共同开发的方式，
其结果远远好于大多数C++编译器面对SafeInt算法时所生成的代码。
与C#一样，[`unchecked`修饰的范围构造](https://msdn.microsoft.com/en-us/library/khy08726.aspx)
也可用于故意设定的上/下溢出的情况。

<!-- 
Although the initial reactions from most C# and C++ developers I've spoken to about this idea are negative about it, our
experience was that 9 times out of 10, this approach helped to avoid a bug in the program.  That remaining 1 time was
usually an abandonment sometime late in one of our 72 hour stress runs -- in which we battered the entire system with
browsers and multimedia players and anything else we could do to torture the system -- when some harmless counter
overflowed.  I always found it amusing that we spent time fixing these instead of the classical way products mature
through the stress program, which is to say deadlocks and race conditions.  Between you and me, I'll take the overflow
abandonments! 
-->
虽然大多数C#和C++开发者对我所说的这个想法最初反应都是负面的，
但我们的经验是：10次中有9次，这种方法都有助于避免程序中的bug，
而剩下的那一次通常是在我们72小时的压力运行
（我们用浏览器和多媒体播放器，以及我们认为可以给系统加压的任何应用来测试系统）
的后来某个时间点上，因为一些无害的计数器溢出时发生的Abandonment。
我们花时间修复这些而不是采用压力程序提升产品成熟度的经典方式——
也就是所谓的死锁和竞争条件，这一点上我总是觉得是非常有趣的。
在你我之间，我会将溢出采用Abandonment！

<!-- 
### Out-of-Memory and Stack Overflow 
-->
### 内存不足和栈溢出

<!-- 
Out-of-memory (OOM) is complicated.  It always is.  And our stance here was certainly contentious also. 
-->
内存不足（OOM）的情况总是很复杂，所以我们在这里的立场当然也是有争议的。

<!-- 
In environments where memory is manually managed, error code-style of checking is the most common approach: 
-->
在手动管理内存的环境中，错误码风格的检查方式是最常用的方法：

<!-- 
    X* x = (X*)malloc(...);
    if (!x) {
        // Handle allocation failure.
    } 
-->

    X* x = (X*)malloc(...);
    if (!x) {
        // 处理内存分配失败 
    } 

<!-- 
This has one subtle benefit: allocations are painful, require thought, and therefore programs that use this technique
are often more frugal and deliberate with the way they use memory.  But it has a huge downside: it's error prone and
leads to huge amounts of frequently untested code-paths.  And when code-paths are untested, they usually don't work. 
-->
这里的一个微妙的好处是：分配是痛苦的，并且需要思考，
因此使用这种技术的程序在使用内存的方式上通常更加节俭和慎重。
但它也有一个巨大的缺点：容易出错，并导致大量通常是未经测试的代码路径。
当代码路径未经测试时，它们通常都会出错。

<!-- 
Developers in general do a terrible job making their software work properly right at the edge of resource exhaustion.
In my experience with Windows and the .NET Framework, this is where egregious mistakes get made.  And it leads to
ridiculously complex programming models, like .NET's so-called [Constrained Execution Regions](
https://blogs.msdn.microsoft.com/bclteam/2005/06/13/constrained-execution-regions-and-other-errata-brian-grunkemeyer/).
A program limping along, unable to allocate even tiny amounts of memory, can quickly become the enemy of reliability.
[Chris Brumme's wondrous Reliability post](http://blogs.msdn.com/b/cbrumme/archive/2003/06/23/51482.aspx) describes this
and related challenges in all its gory glory. 
-->
一般而言，开发者在系统处于资源枯竭的边缘时，非常努力地试图使他们的软件正常工作，
但根据我使用Windows和.NET Framework的经验来看，这是一个令人震惊的错误。
它会导致非常复杂的编程模型，比如.NET的所谓的
[约束执行区域（Constrained Execution Region）](https://blogs.msdn.microsoft.com/bclteam/2005/06/13/constrained-execution-regions-and-other-errata-brian-grunkemeyer/)。
如果一个程序艰难地运行着，即使是少量的内存也无法分配，
那么这种情况很快就会成为可靠性的天敌，
[Chris Brumme奇妙的关于可靠性文章](http://blogs.msdn.com/b/cbrumme/archive/2003/06/23/51482.aspx)中描述了这种情形以及相关的挑战。

<!-- 
Parts of our system were of course "hardened" in a sense, like the lowest levels of the kernel, where abandonment's
scope would be necessarily wider than a single process.  But we kept this to as little code as possible. 
-->
在某种意义上，我们系统的某些部分当然是“硬化的”，就像内核的最低层次一样，
对这部分采用Abandonment其影响范围必然比单个进程要更宽，
但我们也尽量保持这部分代码尽量的小。

<!-- 
For the rest?  Yes, you guessed it: abandonment.  Nice and simple. 
-->
对于系统其余部分呢？是的你猜对了——Abandonment，这很不错也很简单。

<!-- 
It was surprising how much of this we got away with.  I attribute most of this to the isolation model.  In fact, we
could *intentionally* let a process suffer OOM, and ensuing abandonment, as a result of resource management policy, and
still remain confident that stability and recovery were built in to the overall architecture.
-->
令人惊讶的是我们侥幸逃脱了多少这种方式，对此我将大部分原因都归结于隔离模型。
实际上，由于资源管理的策略，我们可以*故意*让一个进程遭受OOM，然后对其中止的策略，
并且仍然对建立在整体架构上稳定性和恢复保持信心。


<!-- 
It was possible to opt-in to recoverable failure for individual allocations if you really wanted.  This was not common
in the slightest, however the mechanisms to support it were there.  Perhaps the best motivating example is this: imagine
your program wants to allocate a buffer of 1MB in size.  This situation is different than your ordinary run-of-the-mill
sub-1KB object allocation.  A developer may very well be prepared to think and explicitly deal with the fact that a
contiguous block of 1MB in size might not be available, and deal with it accordingly.  For example: 
-->
如果你真的需要的话，可以选择对单个分配采取可恢复故障的方式，
虽然这并不常见，但支持的机制是存在的。
也许最激发积极性的例子是：假设你的程序想要分配1MB大小的缓冲区，
这种情况与普通的1KB对象的分配是有所不同的。
开发者可能已经思考并准备好应对，那些可能无法获得1MB大小的连续块内存的情况，
并相应地对其进行处理。例如：


<!-- 
    var bb = try new byte[1024*1024] else catch;
    if (bb.Failed) {
        // Handle allocation failure.
    } 
-->

    var bb = try new byte[1024*1024] else catch;
    if (bb.Failed) {
        // 处理分配失败 
    } 

<!-- 
Stack overflow is a simple extension of this same philosophy.  Stack is just a memory-backed resource.  In fact, thanks
to our asynchronous linked stacks model, running out of stack was physically identical to running out of heap memory,
so the consistency in how it was dealt with was hardly surprising to developers.  Many systems treat stack overflow this
way these days. 
-->
栈溢出是这一理念的简单扩展，因为栈只是内存所支持下的一种资源。
实际上，由于我们的异步链接栈模型，
栈内存的溢出与堆内存的溢出在物理上是完全相同的，
因此开发者对其处理方式的一致性并不令人惊讶。
如今，许多系统也都以这种方式来处理栈溢出。

<!-- 
## Assertions 
-->
## 断言

<!-- 
An assertion was a manual check in the code that some condition held true, triggering abandonment if it did not.  As
with most systems, we had both debug-only and release code assertions, however unlike most other systems, we had more
release ones than debug.  In fact, our code was peppered liberally with assertions.  Most methods had multiple. 
-->
断言是代码中的手动检查某些条件是否成立，如果不成立则触发Abandonment的机制。
与大多数系统一样，我们同时具有调试（debug-only）版本和发布（release）版本的代码断言，
但与大多数其他系统不同的是，我们在发布版本中的断言数量多于调试版本。
事实上，我们的代码充满了断言，而大多数方法中存在多个断言。

<!-- 
This kept with the philosophy that it's better to find a bug at runtime than to proceed in the face of one.  And, of
course, our backend compiler was taught how to optimize them aggressively as with everything else.  This level of
assertion density is similar to what guidelines for highly reliable systems suggest.  For example, from NASA's paper,
[The Power of Ten -Rules for Developing Safety Critical Code](
http://pixelscommander.com/wp-content/uploads/2014/12/P10.pdf): 
-->
这样做的理念是，在运行时找到bug比在遇到错误时继续运行更好，
当然，我们的后端编译器也被实现为像其他方面一样激进地对断言进行优化。
这种断言的密度水平类似于高可靠性系统的指导原则所建议的一样，
例如，来自美国宇航局的论文，[“The Power of Ten -Rules for Developing Safety Critical Code”](http://pixelscommander.com/wp-content/uploads/2014/12/P10.pdf)是这样描述的：

<!-- 
> Rule: The assertion density of the code should average to a minimum of two assertions per function.  Assertions are
> used to check for anomalous conditions that should never happen in real-life executions.  Assertions must always be
> side-effect free and should be defined as Boolean tests.
> 
> Rationale: Statistics for industrial coding efforts indicate that unit tests often find at least one defect per 10 to
> 100 lines of code written.  The odds of intercepting defects increase with assertion density.  Use of assertions is
> often also recommended as part of strong defensive coding strategy. 
-->
> 规则：代码的断言密度应为平均每个函数最少两个断言。
> 断言用于检查在现实生活中不应发生的异常情况。
> 断言必须始终是无副作用的，并且应该定义为布尔测试。

> 理由：工业编码工作的统计数据表明，通常每写入10到100行代码，
> 单元测试就会发现至少一个缺陷。
> 而拦截缺陷的几率随着断言密度的增加而增加，
> 断言的使用通常也被推荐为强防御性编码策略的一部分。

<!-- 
To indicate an assertion, you simply called `Debug.Assert` or `Release.Assert`: 
-->
要表示断言，只需调用`Debug.Assert`或`Release.Assert`即可：

<!-- 
    void Foo() {
        Debug.Assert(something); // Debug-only assert.
        Release.Assert(something); // Always-checked assert.
    } 
-->

    void Foo() {
        Debug.Assert(something); // 仅对调试版本的断言 
        Release.Assert(something); // 始终检查性的断言 
    } 

<!-- 
We also implemented functionality akin to `__FILE__` and `__LINE__` macros like in C++, in addition to `__EXPR__` for
the text of the predicate expression, so that abandonments due to failed assertions contained useful information. 
-->
我们还实现了类似于C++中的`__FILE__`和`__LINE__`宏的功能，
以及谓词表达式文本的`__EXPR__`，因此由于断言失败而导致的Abandonment会包含有用的调试信息。

<!-- 
In the early days, we used different "levels" of assertions than these.  We had three levels, `Contract.Strong.Assert`,
`Contract.Assert`, and `Contract.Weak.Assert`.  The strong level meant "always checked," the middle one meant "it's up
to the compiler," and the weak one meant "only checked in debug mode."  I made the controversial decision to move away
from this model.  In fact, I'm pretty sure 49.99% of the team absolutely hated my choice of terminology (`Debug.Assert`
and `Release.Assert`), but I always liked them because it's pretty unambiguous what they do.  The problem with the old
taxonomy was that nobody ever knew exactly when the assertions would be checked; confusion in this area is simply not
acceptable, in my opinion, given how important good assertion discipline is to the reliability of one's program. 
-->
在早期，我们使用不同“级别”断言的方式。
断言有三个级别，分别是`Contract.Strong.Assert`，`Contract.Assert`和`Contract.Weak.Assert`。
最强的`Contract.Strong.Assert`级别意味着“始终检查”，中间的`Contract.Assert`级别意味这“是否检查取决于编译器”，
最弱的`Contract.Weak.Assert`级别意味着“只在调试模式下检查”。
我做出了有争议的决定，以放弃这种这种分类方式。
事实上，我非常确定团队49.99%的成员绝对讨厌我所选择的术语（`Debug.Assert`和`Release.Assert`），
但我总是喜欢这种方式，因为它们表示的意义非常明确。
旧的分类方法的问题在于，没有人确切知道何时会检查断言，
而在我看来，这个领域内的混乱根本是不可接受的，
因为好的断言规则对程序的可靠性非常重要。

<!-- 
As we moved contracts to the language (more on that soon), we tried making `assert` a keyword too.  However, we
eventually switched back to using APIs.  The primary reason was that assertions were *not* part of an API's signature
like contracts are; and given that assertions could easily be implemented as a library, it wasn't clear what we gained
from having them in the language.  Furthermore, policies like "checked in debug" versus "checked in release" simply
didn't feel like they belonged in a programming language.  I'll admit, years later, I'm still on the fence about this. 
-->
当我们将合约添加到语言中（很快会有更多的合约）时，我们也尝试将`assert`变成关键字。
但是，我们最终转而使用API的方式，其主要原因是断言不像合约那样，它*不是*API签名的一部分；
并鉴于断言可以很容易地作为一个库来实现，我们也不清楚加入到语言中能获得什么。
此外，像“checked in debug”和“checked in release”之类的策略根本不像是编程语言特性，
我承认，多年以后，我仍然对此持怀疑态度。

<!-- 
## Contracts 
-->
## 合约

<!-- 
Contracts were *the* central mechanism for catching bugs in Midori.  Despite us beginning with [Singularity](
https://en.wikipedia.org/wiki/Singularity_(operating_system)), which used Sing#, a variant of [Spec#](
http://research.microsoft.com/en-us/projects/specsharp/), we quickly moved away to vanilla C# and had to rediscover what
we wanted.  We ultimately ended up in a very different place after living with the model for years. 
-->
在Midori中，合约是捕获bug的*核心*机制。
尽管我们以使用了[Spec#](http://research.microsoft.com/en-us/projects/specsharp/)
变体Sing#的[Singularity](http://t.cn/EMtXJiW)作为开始，
但我们很快就转移到了普通C#并且不得不重新发现我们想要的东西。
在与此模型打交道多年以后，我们最终以和开始时非常不同的模样作为结束。

<!-- 
All contracts and assertions were proven side-effect free thanks to our language's understanding of immutability and
side-effects.  This was perhaps the biggest area of language innovation, so I'll be sure to write a post about it soon. 
-->
由于我们的语言对不变性和副作用的理解方式，所有的合约和断言都被证明是无副作用的，
这可能是语言创新的最大领域，所以我一定会尽快写一篇关于此的文章。

<!-- 
As with other areas, we were inspired and influenced by many other systems.  Spec# is the obvious one.  [Eiffel was
hugely influential](https://files.ifi.uzh.ch/rerg/amadeus/teaching/courses/ase_fs10/Meyer1992.pdf) especially as there
are many published case studies to learn from.  Research efforts like Ada-based [SPARK](
https://en.wikipedia.org/wiki/SPARK_(programming_language)) and proposals for realtime and embedded systems too.  Going
deeper into the theoretical rabbit's hole, programming logic like [Hoare's axiomatic semantics](
http://www.spatial.maine.edu/~worboys/processes/hoare%20axiomatic.pdf) provide the foundation for all of it.  For me,
however, the most philosophical inspiration came from CLU's, and later [Argus](
https://pdos.csail.mit.edu/archive/6.824-2009/papers/argus88.pdf)'s, overall approach to error handling. 
-->
与其他地方一样，关于合约，我们也受到许多其他系统的启发和影响。 
Spec#显然是其中之一，[Effiel对我们也有很大的影响力](https://files.ifi.uzh.ch/rerg/amadeus/teaching/courses/ase_fs10/Meyer1992.pdf)，
特别是因为他有许多已发表的案例研究可以学习，
另外基于Ada的[SPARK](http://t.cn/EMGjc0l)的相关研究工作以及实时和嵌入式系统的建议也是如此。
像[Hoare的公理语义](http://www.spatial.maine.edu/~worboys/processes/hoare%20axiomatic.pdf)这样的编程逻辑，深入研究理论上的未知领域，为所有这些打下了基础。
然而，对我来说，最有哲学意义上的灵感来自CLU以及后来的
[Argus](https://pdos.csail.mit.edu/archive/6.824-2009/papers/argus88.pdf)的整体错误处理方法。


<!-- 
### Preconditions and Postconditions 
-->
### 前置条件和后置条件

<!-- 
The most basic form of contract is a method precondition.  This states what conditions must hold for the method to be
dispatched.  This is most often used to validate arguments.  Sometimes it's used to validate the state of the target
object, however this was generally frowned upon, since modality is a tough thing for programmers to reason about.
A precondition is essentially a guarantee the caller provides to the callee. 
-->
最基本的合约形式是方法的前置条件，其申明了要指派的方法必须具备的条件，
并通常用于验证参数。
它有时也用于验证目标对象的状态，但这通常是不受欢迎的，
因为对于程序员来说，形态是很难推算的。
前置条件基本上是调用者向被调用者提供的一种保证。

<!-- 
In our final model, a precondition was stated using the `requires` keyword: 
-->
在我们的最终模型中，使用`requires`关键字声明前置条件：

<!-- 
    void Register(string name)
        requires !string.IsEmpty(name) {
        // Proceed, knowing the string isn't empty.
    } 
-->

    void Register(string name)
        requires !string.IsEmpty(name) {
        // 字符串不为空，并继续处理 
    } 

<!-- 
A slightly less common form of contract is a method postcondition.  This states what conditions hold *after* the method
has been dispatched.  This is a guarantee the callee provides to the caller. 
-->
一种稍微不太常见的合约形式是方法的后置条件，
它表明在指派完方法*之后*保持何种状态，这是被调用者向调用者提供的一种保证。

<!-- 
In our final model, a postcondition was stated using the `ensures` keyword: 
-->
在我们的最终模型中，使用`ensure`关键字声明后置条件：

<!-- 
    void Clear()
        ensures Count == 0 {
        // Proceed; the caller can be guaranteed the Count is 0 when we return.
    } 
-->

    void Clear()
        ensures Count == 0 {
        // 继续处理，并当函数返回时，调用者可以保证Count值是0 
    } 

<!-- 
It was also possible to mention the return value in the postcondition, through the special name `return`.  Old values --
such as necessary for mentioning an input in a post-condition -- could be captured through `old(..)`; for example: 
-->
也可以通过特殊名称`return`来声明后置条件中的返回值，
而旧的参数值，例如在后置条件中引用输入所必需的值，可以通过`old(..)`来捕获。比如说：


    int AddOne(int value)
        ensures return == old(value)+1 {
        ...
    } 


<!-- 
Of course, pre- and postconditions could be mixed.  For example, from our ring buffer in the Midori kernel: 
-->
当然，前置和后置条件也可能是混合出现的，
比如说下面是来自Midori内核中环形缓冲区的代码：

    public bool PublishPosition()
        requires RemainingSize == 0
        ensures UnpublishedSize == 0 {
        ...
    } 

<!-- 
This method could safely execute its body knowing that `RemainingSize` is `0` and callers could safely execute after
the return knowing that `UnpublishedSize` is also `0`. 
-->
此方法在知道`RemainingSize`的值为0时，可以安全地执行其函数体；
而调用者知道`UnpublishedSize`也为0后，可以在被调用函数返回后安全地执行。


<!-- 
If any of these contracts are found to be false at runtime, abandonment occurs. 
-->
如果在运行时发现这些合约中任何一个是错误的，则会执行Abandonment操作。

<!-- 
This is an area where we differ from other efforts.  Contracts have recently became popular as an expression of program
logics used in advanced proof techniques.  Such tools prove truths or falsities about stated contracts, often using
global analysis.  We took a simpler approach.  By default, contracts are checked at runtime.  If a compiler could prove
truth or falsehood at compile-time, it was free to elide runtime checks or issue a compile-time error, respectively. 
-->
该领域是我们与其他工作所不同之处。
合约作为高级证明技术中使用的程序逻辑表达最近变得流行起来，
这些工具通常使用全局分析来证明所述合约的真实性或虚假性。
我们采取了一种更简单的方法：在默认情况下，合约在运行被时检查，
但如果编译器可以在编译时证明其真或假，则可以自由地分别进行运行时检查或发出编译时错误。

<!-- 
Modern compilers have constraint-based analyses that do a good job at this, like the [range analysis](
https://en.wikipedia.org/wiki/Value_range_analysis) I mentioned in my last post.  These propagate facts and use them to
optimize code already.  This includes eliminating redundant checks: either explicitly encoded in contracts, or in
normal program logic.  And they are trained to perform these analyses in reasonable amounts of time, lest programmers
switch to a different, faster compiler.  The theorem proving techniques simply did not scale for our needs; our core
system module took over a day to analyze using the best in breed theorem proving analysis framework! 
-->
现代编译器具有基于约束的分析，并在这方面已经做得很好，
就像我在上一篇文章中提到的[范围分析](https://en.wikipedia.org/wiki/Value_range_analysis)一样。
分析器传播事实并使用它们来优化代码，
优化包括消除冗余检查，在合约或正常程序逻辑中显式地编码。
并且它们被训练地可以在合理的时间内执行这些分析，
避免程序员因等待时间过长而切换到其他更快的编译器上。
而定理证明技术根本无法满足在我们的规模上的需求，
我们的核心系统模块使用最优秀定理证明分析框架，也花了一天的时间来对其进行分析！


<!-- 
Furthermore, the contracts a method declared were part of its signature.  This meant they would automatically show up in
documentation, IDE tooltips, and more.  A contract was as important as a method's return and argument types.  Contracts
really were just an extension of the type system, using arbitrary logic in the language to control the shape of exchange
types.  As a result, all the usual subtyping requirements applied to them.  And, of course, this facilitated modular
local analysis which could be done in seconds using standard optimizing compiler techniques. 
-->
此外，方法声明的合约也是其签名的一部分，
这意味着它们会自动显示在文档和IDE的工具提示中，
所以合约与方法的返回值和参数类型一样重要。
合约实际上只是类型系统的扩展，使用语言中的任意逻辑来控制交换类型的形状，
因此，所有通常的子类型要求对于它们都是适用的。
当然，这有助于使用标准优化编译器技术在几秒钟内完成对局部分析的模块化。

<!-- 
90-something% of the typical uses of exceptions in .NET and Java became preconditions.  All of the
`ArgumentNullException`, `ArgumentOutOfRangeException`, and related types and, more importantly, the manual checks and
`throw`s were gone.  Methods are often peppered with these checks in C# today; there are thousands of these in .NET's
CoreFX repo alone.  For example, here is `System.IO.TextReader`'s `Read` method: 
-->
.NET和Java中90%的典型异常用法都变成了前置条件。所有的`ArgumentNullException`，
`ArgumentOutOfRangeException`和相关类型，以及更重要的的人工手动检查和`throw`都消失了。
如今，C#中的方法经常被这些检查所覆盖，仅在.NET的CoreFX代码仓库中就有数千个这样的检查。
例如，下面是`System.IO.TextReader`中的`Read`方法：

<!-- 
    /// <summary>
    /// ...
    /// </summary>
    /// <exception cref="ArgumentNullException">Thrown if buffer is null.</exception>
    /// <exception cref="ArgumentOutOfRangeException">Thrown if index is less than zero.</exception>
    /// <exception cref="ArgumentOutOfRangeException">Thrown if count is less than zero.</exception>
    /// <exception cref="ArgumentException">Thrown if index and count are outside of buffer's bounds.</exception>
    public virtual int Read(char[] buffer, int index, int count) {
        if (buffer == null) {
            throw new ArgumentNullException("buffer");
        }
        if (index < 0) {
            throw new ArgumentOutOfRangeException("index");
        }
        if (count < 0) {
            throw new ArgumentOutOfRangeException("count");
        }
        if (buffer.Length - index < count) {
            throw new ArgumentException();
        }
        ...
    } 
-->

    /// <summary>
    /// ...
    /// </summary>
    /// <exception cref="ArgumentNullException">如果buffer为null，则抛出该异常</exception> 
    /// <exception cref="ArgumentOutOfRangeException">如果index小于零，则抛出该异常</exception>
    /// <exception cref="ArgumentOutOfRangeException">如果count小于零，则抛出该异常</exception>
    /// <exception cref="ArgumentException">如果index和cout超出缓冲区的范围，则抛出该异常</exception>
    public virtual int Read(char[] buffer, int index, int count) {
        if (buffer == null) {
            throw new ArgumentNullException("buffer");
        }
        if (index < 0) {
            throw new ArgumentOutOfRangeException("index");
        }
        if (count < 0) {
            throw new ArgumentOutOfRangeException("count");
        }
        if (buffer.Length - index < count) {
            throw new ArgumentException();
        }
        ...
    } 

<!-- 
This is broken for a number of reasons.  It's laboriously verbose, of course.  All that ceremony!  But we have to go way
out of our way to document the exceptions when developers really ought not to ever catch them.  Instead, they should
find the bug during development and fix it.  All this exception nonsense encourages very bad behavior. 
-->
出于多种原因，这段代码已经不能编译。这样的代码当然是非常啰嗦的，充满了繁文缛节，
但当开发者真的不应该去捕捉他们时，我们必须尽力去将异常文档化，
相反，他们应该在开发过程中找到错误并修复它。
所有这些异常都会无意义地助长非常糟糕的行为。

<!-- 
If we use Midori-style contracts, on the other hand, this collapses to: 
-->
另一方面，如果我们使用Midori风格的合约，那么上面的代码则折叠成如下的形式：

    /// <summary>
    /// ...
    /// </summary>
    public virtual int Read(char[] buffer, int index, int count)
        requires buffer != null
        requires index >= 0
        requires count >= 0
        requires buffer.Length - index >= count {
        ...
    } 

<!-- 
There are a few appealing things about this.  First, it's more concise.  More importantly, however, it self-describes
the contract of the API in a way that documents itself and is easy to understand by callers.  Rather than requiring
programmers to express the error condition in English, the actual expressions are available for callers to read, and
tools to understand and leverage.  And it uses abandonment to communicate failure. 
-->
这种方式有一些吸引人之处。首先，它更加的简洁；
然而，更重要的是，它以一种记录自身并且调用者易于理解的方式自我描述API的合约。
其实际表达式也可供调用者来阅读，以及供工具来理解和利用，而不要求程序员用通俗语言来表达错误条件。
另外，它也使用Abandonment方式来传递失败。

<!-- 
I should also mention we had plenty of contracts helpers to help developers write common preconditions.  The above
explicit range checking is very messy and easy to get wrong.  Instead, we could have written: 
-->
我还提到我们有很多合约辅助工具来帮助开发者编写常见的前置条件。
上面的显式范围检查非常混乱，且容易出错，相反地，我们可以这样编写：

    public virtual int Read(char[] buffer, int index, int count)
        requires buffer != null
        requires Range.IsValid(index, count, buffer.Length) {
        ...
    } 

<!-- 
And, totally aside from the conversation at hand, coupled with two advanced features -- arrays as slices and non-null
types -- we could have reduced the code to the following, while preserving the same guarantees:
-->
另外除了交互之外，还有两个高级的功能：数组作为切片（slice）和非零类型。
我们可以将上面的代码简化到如下的形式，并同时保留相同的保证：

    public virtual int Read(char[] buffer) {
        ...
    } 

<!-- 
But I'm jumping way ahead ... 
-->
我们再向前迈了一步……

<!-- 
### Humble Beginnings 
-->
### 谦虚的开始

<!-- 
Although we landed on the obvious syntax that is very Eiffel- and Spec#-like -- coming full circle -- as I mentioned
earlier, we really didn't want to change the language at the outset.  So we actually began with a simple API approach: 
-->
虽然我们提到了语法与Eiffel和Spec#的一样清晰明了，但正如我之前所提到的那样，
我们真的不想在一开始就改变语言，所以实际上我们从一个简单的API方法作为开始：

    public bool PublishPosition() {
        Contract.Requires(RemainingSize == 0);
        Contract.Ensures(UnpublishedSize == 0);
        ...
    } 

<!-- 
There are a number of problems with this approach, as the [.NET Code Contracts](
http://research.microsoft.com/en-us/projects/contracts/) effort discovered the hard way. 
-->
这种方法存在许多问题，正如在[.NET Code Contracts](http://research.microsoft.com/en-us/projects/contracts/)的努力已经汲取了这方面的教训。

<!-- 
First, contracts written this way are part of the API's *implementation*, whereas we want them to be part of the
*signature*.  This might seem like a theoretical concern but it is far from being theoretical.  We want the resulting
program to contain built-in metadata so tools like IDEs and debuggers can display the contracts at callsites.  And we
want tools to be in a position to auto-generate documentation from the contracts.  Burying them in the implementation
doesn't work unless you somehow disassemble the method to extract them later on (which is a hack). 
-->

首先，以这种方式编写的合约是API*实现*的一部分，但我们却希望它们成为*签名*的一部分，
这似乎是一个理论上的问题，但它实际上远非理论上的。
我们希望生成的程序包含内置的元数据，使得IDE和调试器等工具可以在调用点上显示合约，
同时我们还希望工具能够从合约中自动生成文档。
除非你之后以某种反汇编手段从方法中提取它们（这是一种黑客的行为），否则将它们隐藏在实现中是行不通的。

<!-- 
This also makes it tough to integrate with a backend compiler which we found was necessary for good performance. 
-->
另外，这种方式也使得很难与后端编译器集成，而我们发现与后端的集成对于良好的性能是必要的。

<!-- 
Second, you might have noticed an issue with the call to `Contract.Ensures`.  Since `Ensures` is meant to hold on all
exit paths of the function, how would we implement this purely as an API?  The answer is, you can't.  One approach is
rewriting the resulting MSIL, after the language compiler emitted it, but that's messy as all heck.  At this point, you
begin to wonder, why not simply acknowledge that this is a language expressivity and semantics issue, and add syntax? 
-->
其次，你可能已经注意到对`Contract.Ensures`的调用存在问题。
由于`Ensures`意味着会保留函数的所有返回路径，那么我们如何将其仅仅实现为API的形式？答案是：这是做不到的。
一种方法是在语言编译器生成代码之后重写生成的MSIL，但这非常麻烦。
此时，你不禁开始怀疑，为什么不简单地承认这是语言表达性和语义问题，并添加相应的语法呢？

<!-- 
Another area of perpetual struggle for us was whether contracts are conditional or not.  In many classical systems,
you'd check contracts in debug builds, but not the fully optimized ones.  For a long time, we had the same three levels
for contracts that we did assertions mentioned earlier: 
-->
对我们来说，长期挣扎的另一个领域是合约是否是有条件的。
在许多经典系统中，只需要检查调试版本中的合约，而对完全优化版本的合约则不检查。
正如对前面的断言一样，在很长一段时间里，我们对合约也有三个相应的级别：

<!-- 
* Weak, indicated by `Contract.Weak.*`, meaning debug-only.
* Normal, indicated simply by `Contract.*`, leaving it as an implementation decision when to check them.
* Strong, indicated by `Contract.Strong.*`, meaning always checked. 
-->
* 最弱，由`Contract.Weak.*`表示，表示仅调试的合约
* 正常，简单地用`Contract.*`表示，留给实现决定何时检查它们
* 最强，由`Contract.Strong.*`表示，表示总是对其检查的合约

<!-- 
I'll admit, I initially found this to be an elegant solution.  Unfortunately, over time we found that there was constant
confusion about whether "normal" contracts were on in debug, release, or all of the above (and so people misused weak
and strong accordingly).  Anyway, when we began integrating this scheme into the language and backend compiler
toolchain, we ran into substantial issues and had to backpedal a little bit. 
-->
我承认，我最初认为这是一个优雅的解决方案。
但不幸的是，随着时间的推移，我们发现开发者在调试，
发布或上述所有内容中是否存在“正常”级别的合约时常常存在着混淆
（因此人们经常会误用最弱级别和最强级别）。
无论如何，当我们开始将这个方案集成到语言和后端编译器工具链中时，
我们遇到了很多问题，所以不得不稍微把目标退后一点。

<!-- 
First, if you simply translated `Contract.Weak.Requires` to `weak requires` and `Contract.Strong.Requires` to
`strong requires`, in my opinion, you end up with a fairly clunky and specialized language syntax, with more policy than
made me comfortable.  It immediately calls out for parameterization and substitutability of the `weak`/`strong` policies. 
-->
首先，如果你简单地将`Contract.Weak.Requires`翻译成`weak requires`，
以及将`Contract.Strong.Requires`翻译成`strong requires`，那么在我看来，
你最终得到一个相当笨重和专门化的语法，以及更多让我感到不舒服的策略。
所以这种方式会立即让人呼吁参数化和`weak`/`strong`策略的可替代性。

<!-- 
Next, this approach introduces a sort of new mode of conditional compilation that, to me, felt awkward.  In other
words, if you want a debug-only check, you can already say something like: 
-->
接下来，这种方法引入了一种对我来说感觉很尴尬的新条件编译模式，
换句话说，如果你想对仅调试版本进行检查，可以这样写：

    #if DEBUG
        requires X
    #endif 

<!-- 
Finally -- and this was the nail in the coffin for me -- contracts were supposed to be part of an API's signature.  What
does it even *mean* to have a conditional contract?  How is a tool supposed to reason about it?  Generate different
documentation for debug builds than release builds?  Moreover, as soon as you do this, you lose a critical guarantee,
which is that code doesn't run if its preconditions aren't met. 
-->
最后，对我来说最后的一击是，合约应该被当作API签名的一部分。
那么有条件的合约*意味*着什么？工具应该如何推理呢？为发布版本生成和调试版本不同的文档？
而且只要存在这样的合约，就会失去如果不满足其前置条件则代码将无法运行的关键性保证。

<!-- 
As a result, we nuked the entire conditional compilation scheme. 
-->
最终，我们完成了整个条件编译的方案。

<!-- 
We ended up with a single kind of contract: one that was part of an API's signature and checked all the time.  If a
compiler could prove the contract was satisfied at compile-time -- something we spent considerable energy on -- it was
free to elide the check altogether.  But code was guaranteed it would never execute if its preconditions weren't
satisfied.  For cases where you wanted conditional checks, you always had the assertion system (described above). 
-->
我们最终得到了单一类型的合约，它是API签名的一部分，并且在所有时刻都会被检查。
如果编译器在编译时可以证明合约永远会得到满足（我们花了相当大的精力在这上面），那么完全可以免除检查，
但是如果不满足其先决条件，则将保证代码永远不会执行。
对于需要进行条件检查的情况，则始终有断言系统可以利用（如上所述）。

<!-- 
I felt better about this bet when we deployed the new model and found that lots of people had been misusing the "weak"
and "strong" notions above out of confusion.  Forcing developers to make the decision led to healthier code. 
-->
当我们部署这样的新模型时，我感到这样的方式更好。
并且发现许多人因混淆而滥用上面的“weak”和“strong”概念，
因此，迫使开发者做出决定给它们带来更好的代码。

<!-- 
## Future Directions 
-->
## 未来的方向

<!-- 
A number of areas of development were at varying stages of maturity when our project wound down.
-->
当我们的项目结束时，许多领域的发展处于不同的成熟阶段。

<!-- 
### Invariants 
-->
### 不变量

<!-- 
We experimented **a lot** with invariants.  Anytime we spoke to someone versed in design-by-contract, they were
borderline appalled that we didn't have them from day one.  To be honest, our design did include them from the outset.
But we never quite got around to finishing the implementation and deploying it.  This was partly just due to engineering
bandwidth, but also because some difficult questions remained.  And honestly the team was almost always satisfied with
the combination of pre- and post-conditions plus assertions.  I suspect that in the fullness of time we'd have added
invariants for completeness, but to this day some questions remain for me.  I'd need to see it in action for a while. 
-->
我们在不变量上进行了*很多*的实验。
每当我们与熟悉合约设计的人交谈时，他们都会对我们从第一天起就没有使用不变量而感到宽慰。
说实话，我们的设计从一开始就包含了它们，但是却从来没有完成它的实现和部署，
这仅仅是由于工程的产出量不足和一些困难的问题仍然存在所导致的。
老实说，团队基本上满意于前置/后置条件和断言的结合，
因此我怀疑在充足的时间里我们是否应完成不变量的实现。
但到目前为止，我仍然对此有一些问题，所以我需要在行动中再看一段时间。

<!-- 
The approach we had designed was where an `invariant` becomes a member of its enclosing type; for example: 
-->
我们设计的方法是，`invariant`成为其封闭类型的成员。例如：

    public class List<T> {
        private T[] array;
        private int count;
        private invariant index >= 0 && index < array.Length;
    ...
    } 


<!-- 
Notice that the `invariant` is marked `private`.  An invariant's accessibility modifier controlled which members the
invariant was required to hold for.  For example, a `public invariant` only had to hold at the entry and exit of
functions with `public` accessibility; this allowed for the common pattern of `private` functions temporarily violating
invariants, so long as `public` entrypoints preserved them.  Of course, as in the above example, a class was free to
declare a `private invariant` too, which was required to hold at all function entries and exits. 
-->
请注意，`invariant`标记为`private`，不变量的访问性修饰符控制了需要保持不变性的成员。
例如，`public invariant`变量只需要在具有`public`访问性的函数的进入和退出时保持不变，
它允许`private`函数的常见模式暂时违反不变性，只要在`public`的入口点保持它们即可。
当然，如上例所示，类也可以自由声明为`private invariant`，
这需要在所有函数入口和出口保持不变性。

<!-- 
I actually quite liked this design, and I think it would have worked.  The primary concern we all had was the silent
introduction of checks all over the place.  To this day, that bit still makes me nervous.  For example, in the `List<T>`
example, you'd have the `index >= 0 && index < array.Length` check at the beginning and end of *every single function*
of the type.  Now, our compiler eventually got very good at recognizing and coalescing redundant contract checks; and
there were ample cases where the presence of a contract actually made code quality *better*.  However, in the extreme
example given above, I'm sure there would have been a performance penalty.  That would have put pressure on us changing
the policy for when invariants are checked, which would have possibly complicated the overall contracts model. 
-->
我其实很喜欢这样的设计，并且觉得它会有用。
我们所关心的主要问题是在所有地方静默地引入检查。
直到今天为止，这一点仍让我感到紧张，例如，在`List<T>`的示例中，
你将在类型的*每个函数*的开头和结尾进行`index > = 0 && index <array.Length`的检查。
现在，我们的编译器最终非常善于识别和合并冗余合约检查，并且在很多情况下，
合约的存在实际上使代码质量更好。
但是，在上面给出的极端例子中，我确信它会造成性能损失，
因此会对我们改变检查不变量的策略施加压力，从而可能会使整体合约模型变得复杂。

<!-- 
I really wish we had more time to explore invariants more deeply.  I don't think the team sorely missed not having them
-- certainly I didn't hear much complaining about their absence (probably because the team was so performance conscious)
-- but I do think invariants would have been a nice icing to put on the contracts cake. 
-->
我真的希望我们有更多的时间来更深入地探索不变量，
我不认为团队严重错过它们，当然我没有听到太多抱怨缺失他们的声音（可能是因为团队非常注重性能），
但我确实认为不变量会是合约系统上的闪光之处。

<!-- 
### Advanced Type Systems 
-->
### 高级类型系统

<!-- 
I always liked to say that contracts begin where the type system leaves off.  A type system allows you to encode
attributes of variables using types.  A type limits the expected range values that a variable might hold.  A contract
similarly checks the range of values that a variable holds.  The difference?  Types are proven at compile-time through
rigorous and composable inductive rules that are moderately inexpensive to check local to a function, usually, but not
always, aided by developer-authored annotations.  Contracts are proven at compile-time *where possible* and at runtime
otherwise, and as a result, permit far less rigorous specification using arbitrary logic encoded in the language itself. 
-->
我喜欢说合约始于类型系统触及不到之处。
类型系统允许你使用类型对变量的属性进行编码，对变量可能包含的预期范围值进行限制。
类似地，合约也对变量所持有值的范围进行检查。
那么它们的区别在什么地方？类型在编译时通过严格且可组合的归纳规则得以验证，
这些规则对于函数局部检查而言开销更小，但通常并不总是由开发者编写的注解所辅助。
合约在*可能的情况下*在编译时进行证明，否则在运行时被证明，
因此，合约被允许使用语言本身编码的任意逻辑进行远非严格的规范。

<!-- 
Types are preferable, because they are *guaranteed* to be compile-time checked; and *guaranteed* to be fast to check.
The assurances given to the developer are strong and the overall developer productivity of using them is better. 
-->
类型是一种可取的方式，因为它们*保证*在编译时进行检查，同时*保证*检查的快速。
它给开发者的保证很强，使得使用它们的整体效率更高。

<!-- 
Limitations in a type system are inevitable, however; a type system needs to leave *some* wiggle room, otherwise it
quickly grows unwieldly and unusable and, in the extreme, devolves into bi-value bits and bytes.  On the other hand, I
was always disappointed by two specific areas of wiggle room that required the use of contracts:
-->
然而，类型系统的局限也是不可避免的。类型系统需要留下*一些*协调空间，否则它会迅速膨胀并且可用性很差，
并且在极端情况下会转换为双值位和字节。
另一方面，我总是对需要使用合约的两个特定的协调空间区域感到失望：

<!-- 
1. Nullability.
2. Numeric ranges. 
-->
1. 空值性；
2. 数值范围。

<!-- 
Approximately 90% of our contracts fell into these two buckets.  As a result, we seriously explored more sophisticated
type systems to classify the nullability and ranges of variables using the type system instead of contracts. 
-->
我们大约90%的合约都属于这两者类型。
因此，我们认真研究了更复杂的类型系统，以使用类型系统而不是合约的方式对变量的空值性和范围进行分类。

<!-- 
To make it concrete, this was the difference between this code which uses contracts: 
-->
具体来说，这是与使用合约的代码之间的区别：

    public virtual int Read(char[] buffer, int index, int count)
        requires buffer != null
        requires index >= 0
        requires count >= 0
        requires buffer.Length - index < count {
        ...
    } 


<!-- 
And this code which didn't need to, and yet carried all the same guarantees, checked statically at compile-time: 
-->
下面这段代码在编译时静态检查，虽然不需要但仍然保留了所有相同的保证：

    public virtual int Read(char[] buffer) {
        ...
    } 


<!-- 
Placing these properties in the type system significantly lessens the burden of checking for error conditions.  Lets
say that for any given 1 producer of state there are 10 consumers.  Rather than having each of those 10 defend
themselves against error conditions, we can push the responsibility back onto that 1 producer, and either require a
single assertion that coerces the type, or even better, that the value is stored into the right type in the first place. 
-->
将这些属性放置在类型系统中可以显著地减轻错误条件检查带来的负担。
我们说，对于任何给定的一个状态的生产者，都有10个对应的消费者，
那么可以将责任推回到生产者身上，而不是让每个消费者自身来抵御错误条件。
这么做只需要一个强制类型的断言，或者首先将值存储到正确的类型这种更好的方式。

<!-- 
#### Non-Null Types 
-->
#### 非空类型

<!-- 
The first one's really tough: guaranteeing statically that variables do not take on the `null` value.  This is what
Tony Hoare has famously called his ["billion dollar mistake"](
http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare).  Fixing this for good is a
righteous goal for any language and I'm happy to see newer language designers tackling this problem head-on. 
-->
其中之一的非空性真的很难做到：保证静态变量不会使用`null`值，
而这就是Tony Hoare的所谓的
[“十亿美元的错误”](http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare)。
解决这个问题对于任何语言来说都是一个正确的目标，
我很高兴看到新的编程语言的设计师正在正面解决这个问题。

<!-- 
Many areas of the language fight you every step of the way on this one.  Generics, zero-initialization, constructors,
and more.  Retrofitting non-null into an existing language is tough! 
-->
语言的许多方面都会在这一步中与你发生冲突，比如说泛型、零初始化和构造函数等，在现有语言中实现非空性真的很难！

<!-- 
##### The Type System 
-->
##### 类型系统

<!-- 
In a nutshell, non-nullability boiled down to some simple type system rules: 
-->
简而言之，非空值性可以归结为一些简单的类型系统的规则：

<!-- 
1. All unadorned types `T` were non-null by default.
2. Any type could be modified with a `?`, as in `T?`, to mark it nullable.
3. `null` is an illegal value for variables of non-null types.
4. `T` implicitly converts to `T?`.  In a sense, `T` is a subtype of `T?` (although not entirely true).
5. Operators exist to convert a `T?` to a `T`, with runtime checks that abandoned on `null`. 
-->
1. 默认情况下，所有未加修饰的类型`T`都是非空的；
2. 任何类型都可以使用`?`进行修饰，如`T?`，以将其标记为可为空；
3. `null`对于任何非空类型变量来说都是非法值；
4. `T`可以隐式转换为`T?`，从某种意义上说，`T`是`T?`的子类型（尽管不完全如此）；
5. 存在运算符将`T?`转换为`T`，并进行运行时检查，如果值为`null`则触发放弃机制。

<!-- 
Most of this is probably "obvious" in the sense that there aren't many choices.  The name of the game is systematically
ensuring all avenues of `null` are known to the type system.  In particular, no `null` can ever "sneakily" become the
value of a non-null `T` type; this meant addressing zero-initialization, perhaps the hardest problem of all. 
-->
大多数情况可能是“显而易见的”，因为可以选择的方法并不多。
主要的原则是系统地确保所有`null`值对于类型系统都是可知的，
特别是，没有`null`可以“偷偷摸摸”成为非空的`T`类型值，
这意味着其解决了零初始化问题——可能是所有问题当中最困难的一个。

<!-- 
##### The Syntax 
-->
##### 语法

<!-- 
Syntactically, we offered a few ways to accomplish #5, converting from `T?` to `T`.  Of course, we discouraged this, and
preferred you to stay in "non-null" space as long as possible.  But sometimes it's simply not possible.  Multi-step
initialization happens from time to time -- especially with collections data structures -- and had to be supported. 
-->
从语法上来讲，我们提供了几种方法来完成任务第五项，也就是将`T?`转换为`T`的方法。
当然，我们不鼓励这样做，并且希望你尽可能长时间留在“非空”的环境中。
但有时它根本不可能做得到，多步骤初始化不时发生，
特别是对于集合数据结构来说更是如此，因此这必须得到支持。

<!-- 
Imagine for a moment we have a map: 
-->
设想一下，我们有如下的map数据结构：

    Map<int, Customer> customers = ...; 

<!-- 
This tells us three things by construction: 
-->
通过构造，我们知晓了三件事：

<!-- 
1. The `Map` itself is not null.
2. The `int` keys inside of it will not be `null`.
3. The `Customer` values inside of it will also not be null. 
-->

1.`Map`本身不为null；
2.其中的`int`类型的键不为`null`；
3.其中的`Customer`类型的值也不为`null`。

<!-- 
Let's now say that the indexer actually returns `null` to indicate the key was missing: 
-->
我们可以说，索引器实际上返回了`null`以表示`key`键不存在。


    public TValue? this[TKey key] {
        get { ... }
    } 

<!-- 
Now we need some way of checking at callsites whether the lookup succeeded.  We debated many syntaxes. 
-->
现在我们需要一些方法来检查调用是否成功，对此我们讨论了多种语法。

<!-- 
The easiest we landed on was a guarded check: 
-->
最容易想到的是“guarded check”：

<!-- 
    Customer? customer = customers[id];
    if (customer != null) {
        // In here, `customer` is of non-null type `Customer`.
    } 
-->

    Customer? customer = customers[id];
    if (customer != null) {
        // 这里的customer变量的类型是非null类型Customer 
    } 
    
<!-- 
I'll admit, I was always on the fence about the "magical" type coercions.  It annoyed me that it was hard to figure out
what went wrong when it failed.  For example, it didn't work if you compared `c` to a variable that held the `null`
value, only the literal `null`.  But the syntax was easy to remember and usually did the right thing. 
-->
我承认，我总是对“魔法”类型的强制转换持观望态度。但让我感到恼火的是，当它出现失败时很难弄清楚出了什么问题。
例如，如果将`c`与只持有字面`null`值的变量进行比较，那么它就不起作用了，但它的语法很容易记住，通常能够发挥作用。

<!-- 
These checks dynamically branch to a different piece of logic if the value is indeed `null`.  Often you'd want to simply
assert that the value is non-null and abandon otherwise.  There was an explicit type-assertion operator to do that: 
-->
如果值确实为`null`，则这些检查动态会分派到不同的逻辑上，
通常，你只想断言该值为非null并在断言失败则触发放弃。
有显式的类型断言运算符可以做到这一点：


    Customer? maybeCustomer = customers[id];
    Customer customer = notnull(maybeCustomer); 

<!-- 
The `notnull` operator turned any expression of type `T?` into an expression of type `T`. 
-->
除此之外，`notnull`操作符将类型为`T?`的任何表达式转换为`T`类型的表达式。

<!-- 
##### Generics 
-->
##### 泛型

<!-- 
Generics are hard, because there are multiple levels of nullability to consider.  Consider: 
-->
泛型很难，因为要考虑多个级别的空值性。
考虑如下的代码：

    class C {
        public T M<T>();
        public T? N<T>();
    }

    var a = C.M<object>();
    var b = C.M<object?>();
    var c = C.N<object>();
    var d = C.N<object?>(); 

<!-- 
The basic question is, what are the types of `a`, `b`, `c`, and `d`? 
-->
最基本问题是，`a`，`b`，`c`和`d`的类型是什么？

<!-- 
I think we made this one harder initially than we needed to largely because C#'s existing nullable is a pretty odd duck
and we got distracted trying to mimic it too much.  The good news is we finally found our way, but it took a while. 
-->
我认为我们最初将其变得比我们需要的更困难，
因为C#现有的可空系统非常的古怪，
而我们在试图模仿它上面分心太多。
不过，好消息是我们终于找到了方向，虽然这还需要一段时间。

<!-- 
To illustrate what I mean, let's go back to the example.  There are two camps: 
-->
为了说明我的所表达的意思，让我们回到这个例子。有如下的两个不同阵营：


<!-- 
* The .NET camp: `a` is `object`; `b`, `c`, and `d` are `object?`.
* The functional language camp: `a` is `object`; `b` and `c` are `object?`; `d` is `object??`.
-->
* .NET阵营：`a`是`object`，`b`、`c`和`d`是`object?`；
* 函数式语言阵营：`a`是`object`，`b`和`c`是`object?`，而`d`是`object??`。

<!-- 
In other words, the .NET camp thinks you should collapse any sequence of 1 or more `?`s into a single `?`.  The
functional language camp -- who understands the elegance of mathematical composition -- eschews the magic and lets the
world be as it is.  We eventually realized that the .NET route is incredibly complex, and requires runtime support. 
-->
换句话说，.NET阵营认为你应该将一个或更多`?`的序列折叠成一个单独的`?`；而函数式语言阵营，
由于了解组合数学的优雅，从而避免了魔法操作，并让整个代码方式也变得如此。 
我们最终意识到.NET的路线方式非常复杂，且需要运行时的支持。

<!-- 
The functional language route does bend your mind slightly at first.  For example, the map example from earlier: 
-->
函数语言的做法最初会稍微改变一下你的想法。例如，对于前面的map示例：

<!-- 
    Map<int, Customer?> customers = ...;
    Customer?? customer = customers[id];
    if (customer != null) {
        // Notice, `customer` is still `Customer?` in here, and could still be `null`!
    } 
-->

    Map<int, Customer?> customers = ...;
    Customer?? customer = customers[id];
    if (customer != null) {
        // 请注意，这里的customer仍然是“Customer?”类型，并且值依然可以是`null` 
    } 

<!-- 
In this model, you need to peel off one layer of `?` at a time.  But honestly, when you stop to think about it, that
makes sense.  It's more transparent and reflects precisely what's going on under here.  Best not to fight it. 
-->
在这个模型中，你需要一次剥掉一层`?`，
但老实说，当你停下来思考为什么需要这么做时，它是有道理的。
它更透明，并准确反映了正在发生的事情，因此最好不要排斥它。

<!-- 
There's also the question of implementation.  The easiest implementation is to expand `T?` into some "wrapper type,"
like `Maybe<T>`, and then inject the appropriate wrap and unwrap operations.  Indeed, that's a reasonable mental model
for how the implementation works.  There are two reasons this simple model doesn't work, however. 
-->
在实现上也有问题的。最简单的实现是将`T?`扩展为一些“包装（wrapper）类型”，如`Maybe<T>`，然后注入适当的包装和解包操作。
实际上，这是实现工作方式的符合心理模型，但存在两个原因，使得这个简单的模型不起作用。

<!-- 
First, for reference type `T`, `T?` must not carry a wasteful extra bit; a pointer's runtime representation can carry
`null` as a value already, and for a systems language, we'd like to exploit this fact and store `T?` as efficiently as
`T`.  This can be done fairly easily by specializing the generic instantiation.  But this does mean that non-null can
no longer simply be a front-end trick.  It requires back-end compiler support. 
-->
首先，对于引用类型`T`来说，`T?`一定不能占用额外的存储空间，
指针的运行时表示已经可以以`null`作为值。
并且对于系统语言而言，我们想利用这个事实来与`T`一样高效地存储`T?`，
这一点通过专门化泛型实例可以相当容易地完成。
但也确实意味着非空不再仅仅是一个前端技巧，它现在也需要后端编译器的支持。

<!-- 
(Note that this trick is not so easy to extend to `T??`!) 
-->
（注意，这个技巧不是那么容易能够扩展到`T??`！）

<!-- 
Second, Midori supported safe covariant arrays, thanks to our mutability annotations.  If `T` and `T?` have a different
physical representation, however, then converting `T[]` to `T?[]` is a non-transforming operation.  This was a minor
blemish, particularly since covariant arrays become far less useful once you plug the safety holes they already have. 
-->
其次，多亏了我们的可变性注解，Midori得以支持安全的协变数组。
但是如果`T`和`T?`具有不同的物理表示，那么将`T[]`转换为`T?[]`将是非变换操作。
但这仅算得上是微小的瑕疵，特别是因为协变数组在插入已有的安全孔后变得不那么有用了。

<!-- 
Anyway, we eventually burned the ships on .NET `Nullable<T>` and went with the more composable multi-`?` design. 
-->
无论如何，最终我们彻底放弃了.NET的`Nullable<T>`方式，转向了更多可组合的多`?`操作符设计。

<!-- 
##### Zero-Initialization 
-->
#### 零初始化

<!-- 
Zero-initialization is a real pain in the butt.  To tame it meant: 
-->
零初始化是真正的痛苦之处，征服它意味着要做到：

<!-- 
* All non-null fields of a class must be initialized at construction time.
* All arrays of non-null elements must be fully initialized at construction time. 
-->
* 必须在构造时初始化类的所有非空字段；
* 所有非空元素数组必须在构造时完全被初始化。

<!-- 
But it gets worse.  In .NET, value types are implicitly zero-initialized.  The initial rule was therefore: 
-->
但它变得更加糟糕。在.NET中，值的类型隐式地被零初始化。
因此，最初的规则变成了：

<!-- 
* All fields of a struct must be nullable. 
-->
* 结构的所有字段都必须是可空的。

<!-- 
But that stunk.  It infected the whole system with nullable types immediately.  My hypothesis was that nullability only
truly works if nullable is the uncommon (say 20%) case.  This would have destroyed that in an instant. 
-->
但是这是臭名昭著的，它会立刻以可空类型污染整个系统。
我的假设是，只有可空是不常见的情况（例如20%）中，可空性才真正起作用。
而上述的规则会在瞬间毁掉这样的条件。

<!-- 
So we went down the path of eliminating automatic zero-initialization semantics.  This was quite a large change.  (C# 6
went down the path of allowing structs to provide their own zero-arguments constructors and [eventually had to back it
out](https://github.com/dotnet/roslyn/issues/1029) due to the sheer impact this had on the ecosystem.)  It
could have been made to work but veered pretty far off course, and raised some other problems that we probably got too
distracted with.  If I could do it all over again, I'd just eliminate the value vs. reference type distinction
altogether in C#.  The rationale for that'll become clearer in an upcoming post on battling the garbage collector. 
-->
因此，我们沿着消除自动零初始化语义的道路走了下去，而这却是一个很大的变化。
（C# 6选择了允许结构提供零参数构造函数的路径走了下去，并最终因为它对生态系统产生了巨大的影响而[不得不支持它](https://github.com/dotnet/roslyn/issues/1029)）
它本可以很好的工作但是严重地偏离了路线，并引发了一些可能让我们分散注意力的问题。
如果我可以再来一次，我会在C#中完全消除了值与引用类型的区别。
在即将发布的关于与垃圾回收器做斗争的文章中，这个理由将被阐述的更加清晰。


<!-- 
##### The Fate of Non-Null Types 
-->
##### 非空类型的命运

<!-- 
We had a solid design, and several prototypes, but never deployed this one across the entire operating system.  The
reason why was tied up in our desired level of C# compatibility.  To be fair, I waffled on this one quite a bit, and I
suppose it was ultimately my decision.  In the early days of Midori, we wanted "cognitive familiarity."  In the later
days of the project, we actually considered whether all of the features could be done as "add on" extensions to C#.  It
was that later mindset that prevented us from doing non-null types in earnest.  My belief to this day is that additive
annotations just won't work; Spec# tried this with `!` and the polarity always felt inverted.  Non-null needs to be the
default for this to have the impact we desired. 
-->
对于非空类型而言，我们有坚实的设计和多个原型，但却从未在整个操作系统中部署非空类型。
之所以这样是因为被捆绑在我们期望的C#兼容性水平上，公平地说，我认为这一点最终是我的决定。
在Midori的早期，我们所需要的是“认知熟悉度”，
而在项目的后期，我们实际上考虑了是否所有功能都可以作为C#的“附加”扩展来完成。
正是后来的思维模式阻止了我们认真地完成非空类型。
现在，我对此的信念是，可加注解起不了作用，Spec#尝试利用`!`达到这个目的而极性总是被翻转。
非空必须成为默认值才能实现我们想要的影响力。

<!-- 
One of my biggest regrets is that we waited so long on non-null types.  We only explored it in earnest once contracts
were a known quantity, and we noticed the thousands of `requires x != null`s all over the place.  It would have been
complex and expensive, however this would have been a particularly killer combination if we nuked the value type
distinction at the same time.  Live and learn! 
-->
我最大的遗憾之一就是我们在非空类型上等待了很久，
在合约达到相当数量时，我们才对此进行过认真地探索，
并且我们注意到了在项目中有数千个`requires x != null`。
它本来是复杂而且高代价的，但如果我们同时确定值类型的区别，这将是相当杀手锏的组合。
活到老，学到老！

<!-- 
If we shipped our language as a standalone thing, different from C# proper, I'm convinced this would have made the cut. 
-->
如果我们把我们的语言作为一个独立的项目来交付，
而不同于C#本身，我相信这会有所成就的。

<!-- 
#### Range Types 
-->
#### 范围类型
<!-- 
We had a design for adding range types to C#, but it always remained one step beyond my complexity limit. 
-->
我们有用于向C#添加范围类型的设计，但它总是更进一步超过了我们的复杂性限制。

<!-- 
The basic idea is that any numeric type can be given a lower and upper bound type parameter.  For example, say you had
an integer that could only hold the numbers 0 through 1,000,000, exclusively.  It could be stated as
`int<0..1000000>`.  Of course, this points out that you probably should be using a `uint` instead and the compiler
would warn you.  In fact, the full set of numbers could be conceptually represented as ranges in this way: 
-->
其基本思想是，任何数字类型都可以给出一个下限和上限的类型参数。
例如，假设有一个整型，只能持有数字0到1,000,000之间，那么它可以表示为`int<0..1000000>`。
当然，应该指出的是你可能应该使用`uint`，所以编译器会对你发出警告。
实际上，完整的数字集合可以通过这种方式在概念上表示为范围：

<!-- 
    typedef byte number<0..256>;
    typedef sbyte number<-128..128>;
    typedef short number<-32768..32768>;
    typedef ushort number<0..65536>;
    typedef int number<-2147483648..2147483648>;
    typedef uint number<0..4294967295>;
    // And so on ... 
-->

    typedef byte number<0..256>;
    typedef sbyte number<-128..128>;
    typedef short number<-32768..32768>;
    typedef ushort number<0..65536>;
    typedef int number<-2147483648..2147483648>;
    typedef uint number<0..4294967295>;
    // 等等... 

<!-- 
The really "cool" -- but scary complicated -- part is to then use [dependent types](
https://en.wikipedia.org/wiki/Dependent_type) to permit symbolic range parameters.  For example, say I have an array and
want to pass an index whose range is guaranteed to be in-bounds.  Normally I'd write: 
-->
真正出彩但又是可怕的复杂的部分是，
使用[依赖类型](https://en.wikipedia.org/wiki/Dependent_type)来允许符号范围参数。
例如，假设我有一个数组，并希望传入一个索引，其范围保证是在范围之内的。
那么通常我会写成：

    T Get(T[] array, int index)
            requires index >= 0 && index < array.Length {
        return array[index];
    } 

<!-- 
Or maybe I'd use a `uint` to eliminate the first half of the check: 
-->
或者也许我会用`uint`来消除检查的前半部分：

    T Get(T[] array, uint index)
            index < array.Length {
        return array[index];
    } 

<!-- 
Given range types, I can instead associate the upper bound of the number's range with the array length directly:
-->
在范围类型的情形下，我可以直接将数字范围的上限与数组长度相关联：

    T Get(T[] array, number<0, array.Length> index) {
        return array[index];
    } 

<!-- 
Of course, there's no guarantee the compiler will eliminate the bounds check, if you somehow trip up its alias analysis.
But we would hope that it does no worse a job with these types than with normal contracts checks.  And admittedly this
approach is a more direct encoding of information in the type system. 
-->
当然，如果以某种方式欺骗了编译器的别名分析，则无法保证编译器会消除边界检查，
但我们希望这类型的工作不会比正常的合约检查更糟糕，
并且老实地说，这种方法是对类型系统中信息的更直接编码。


<!-- 
Anyway, I still chalk this one up to a cool idea, but one that's still in the realm of "nice to have but not critical." 
-->
无论如何，我仍然把这归结为很酷的想法，但是仍然处于“很不错但不是关键性”的范畴内。

<!-- 
The "not critical" aspect is especially true thanks to slices being first class in the type system.  I'd say 66% or more
of the situations where range checks were used would have been better written using slices.  I think mainly people were
still getting used to having them and so they'd write the standard C# thing rather than just using a slice.  I'll cover
slices in an upcoming post, but they removed the need for writing range checks altogether in most code. 
-->
由于切片（slice）在类型系统中是一等公民，因此其“非关键”方面是尤其突出的，
我可以说66%或更多使用范围检查的情况可以更好地使用切片来编写。
我认为主要是人们仍然习惯于拥有它们，因此他们会编写标准的C#而不仅仅是使用切片类型。
我将在即将发布的帖子中介绍切片，这样他们在大多数代码中都不再需要编写范围检查。

<!-- 
# Recoverable Errors: Type-Directed Exceptions 
-->
# 可恢复错误：类型导向的异常

<!-- 
Abandonment isn't the only story, of course.  There are still plenty of legitimate situations where an error the
programmer can reasonably recover from occurs.  Examples include: 
-->
当然，Abandonment不是唯一的主题，
程序中仍然存在程序员可以合理地从中恢复错误的大量合法情况。
这样的例子包括：

<!-- 
* File I/O.
* Network I/O.
* Parsing data (e.g., a compiler parser).
* Validating user data (e.g., a web form submission). 
-->
* 文件I/O
* 网络I/O
* 解析数据（例如，编译器解析器）
* 验证用户数据（例如，Web提交的表单）

<!-- 
In each of these cases, you usually don't want to trigger abandonment upon encountering a problem.  Instead, the
program expects it to occur from time to time, and needs to deal with it by doing something reasonable.  Often by
communicating it to someone: the user typing into a webpage, the administrator of the system, the developer using a
tool, etc.  Of course, abandonment is one method call away if that's the most appropriate action to take, but it's often
too drastic for these situations.  And, especially for IO, it runs the risk of making the system very brittle.  Imagine
if the program you're using decided to wink out of existence every time your network connection dropped a packet! 
-->
在所有的情况下，你通常不希望在一遇到问题时就触发Abandonment机制，
而相反地，该程序在预期上就可能会不时地发生错误，并需要通过一些合理的操作来对其进行处理。
这通常通过将错误交给其他对象处理：向网页中输入的用户、系统管理员和使用工具的开发者等等。
当然，如果Abandonment是最合适的行为，
那么它不失为一种做法，但它通常也被认为是这些情况下最为极为激烈的反应。
而且，特别是对于IO，它的存在使系统承担非常脆弱的风险。
想象一下，如果你使用的程序在每次网络连接丢包时都因Abandonment而终止，那这简直是不可想象的！

<!-- 
## Enter Exceptions 
-->
## 进入异常

<!-- 
We used exceptions for recoverable errors.  Not the unchecked kind, and not quite the Java checked kind, either. 
-->
对于可恢复错误，我们使用异常，这里的异常不是那种非检查性的类型，也不是Java那种检查性类型。

<!-- 
First thing's first: although Midori had exceptions, a method that wasn't annotated as `throws` could never throw one.
Never ever ever.  There were no sneaky `RuntimeException`s like in Java, for instance.  We didn't need them anyway,
because the same situations Java used runtime exceptions for were instead using abandonment in Midori. 
-->
首要的原则是：虽然Midori有异常机制，
但是一个没有注解为`throws`的方法永远不会抛出异常，永远永远不会。
例如，Midori没有在Java中那种悄悄抛出的`RuntimeException`，我们无论如何都不需要这样的方式。
因为在Java中使用运行时异常的相同情况下，Midori中采用的是Abandonment机制。

<!-- 
This led to a magical property of the result system.  90-something% of the functions in our system could not throw
exceptions!  By default, in fact, they could not.  This was a stark contrast to systems like C++ where you must go out
of your way to abstain from exceptions and state that fact using `noexcept`.  APIs could still fail due to abandonment,
of course, but only when callers fail meet the stated contract, similar to passing an argument of the wrong type. 
-->
这导致产生的系统具有神奇特性：我们系统中90%的函数都不会抛出异常！
事实上，默认情况下他们不能，这就与像C++这样的系统形成了鲜明对比。
在这些系统中，你必须不遗余力地避免异常并使用`noexcept`来申明这一事实。
当然，API仍可能因Abandonment机制而调用失败，
但只有当调用者未能满足所述合约时才会发生，
而这一点上类似于向函数传递了错误类型的参数。

<!-- 
Our choice of exceptions was controversial at the outset.  We had a mixture of imperative, procedural, object oriented,
and functional language perspective on the team.  The C programmers wanted to use error codes and were worried we would
recreate the Java, or worse, C# design.  The functional perspective would be to use dataflow for all errors, but
exceptions were very control-flow-oriented.  In the end, I think what we chose was a nice compromise between all of the
available recoverable error models available to us.  As we'll see later, we did offer a mechanism for treating errors as
first class values for that rare case where a more dataflow style of programming was what the developer wanted. 
-->
我们选择的异常在一开始时就存在争议，
我们在团队中融合了命令式、过程式、面向对象和函数式语言的视角。
C程序员想使用错误码的方式，并担心我们所做的却是重新创造了Java，或者更糟糕的C#设计。
函数式的视角是对所有错误使用数据流，但异常却是十分以控制流为导向的方式。
最后，我认为我们选择的是我们所有可用的可恢复错误模型之间的一种妥协。
正如我们稍后将看到的，我们确实提供了一种将错误视为一等类型的机制，
在这种情况下，开发者想要的是更多的数据流风格的编程。

<!-- 
Most importantly, however, we wrote a lot of code in this model, and it worked very well for us.  Even the functional
language guys came around eventually.  As did the C programmers, thanks to some cues we took from return codes. 
-->
然而最重要的是，我们在这个模型中编写了很多代码，它对我们来说非常有用。
即便是函数式语言的开发者也最终加入进来。
多亏了我们从返回码中获得的一些线索，C程序员也加入了。

<!-- 
### Language and Type System 
-->
### 语言和类型系统

<!-- 
At some point, I made a controversial observation and decision.  Just as you wouldn't change a function's return type
with the expectation of zero compatibility impact, you should not be changing a function's exception type with such an
expectation.  *In other words, an exception, as with error codes, is just a different kind of return value!* 
-->
在某个时间点上，我做了一个有争议的观察和决定：
正如你不会期望在零兼容性影响条件下更改函数的返回类型一样，
你也不应该以这样的期望方式更改函数的异常类型。
*换句话说，与错误代码一样，异常只是另一种不同的返回值！*

<!-- 
This has been one of the parroted arguments against checked exceptions.  My answer may sound trite, but it's simple:
too bad.  You're in a statically typed programming language, and the dynamic nature of exceptions is precisely the
reason they suck.  We sought to address these very problems, so therefore we embraced it, embellished strong typing,
and never looked back.  This alone helped to bridge the gap between error codes and exceptions.
-->
这是针对检查性异常的一个有争议的论点。
对此，我的回答可能听起来有点老生常谈，但其实很简单：这太糟糕了。
在静态类型的编程语言中，异常的动态性正是它们出问题的地方。
我们试图解决这些问题，因此我们拥抱了该决定，
用于为强类型提供辅助，并再也没有回头过。
仅此一点就有助于弥合错误码和异常之间的鸿沟。

<!-- 
Exceptions thrown by a function became part of its signature, just as parameters and return values are.  Remember,
due to the rare nature of exceptions compared to abandonment, this wasn't as painful as you might think.  And a lot of
intuitive properties flowed naturally from this decision. 
-->
函数抛出的异常成为其签名的一部分，就像参数和返回值一样。
请记住，由于异常与Abandonment相比的不常见性，这一点也并不像你想象的那么痛苦，
并且很多直观的属性也自然而然地也从这个决定中衍生出来。

<!-- 
The first thing is the [Liskov substitution principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle).  In
order to avoid the mess that C++ found itself in, all "checking" has to happen statically, at compile time.  As a
result, all of those performance problems mentioned in [the WG21 paper](
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2010/n3051.html) were not problems for us.  This type system must be
bulletproof, however, with no backdoors to defeat it.  Because we needed to address those performance challenges by
depending on `throws` annotations in our optimizing compiler, type safety hinged on this property. 
-->
因此，首要的原则就是[Liskov替代原则](https://en.wikipedia.org/wiki/Liskov_substitution_principle)：
为了避免在C++中所发现的混乱，所有的“检查”必须在编译时静态发生。
因此，[WG21的文章](
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2010/n3051.html)
中提到的所有这些性能问题对我们来说都不再是问题。
这种类型的系统必须是可以抵御攻击的，没有后门可以攻陷它，
因为我们需要依赖于优化编译器中的`throws`来解决这些性能挑战，
所以类型安全取决于该属性。

<!-- 
We tried many many different syntaxes.  Before we committed to changing the language, we did everything with C#
attributes and static analysis.  The user experience wasn't very good and it's hard to do a real type system that way.
Furthermore, it felt too bolted on.  We experimented with approaches from the Redhawk project -- what eventually became
.NET Native and [CoreRT](https://github.com/dotnet/corert) -- however, that approach also didn't leverage the language
and relied instead on static analysis, though it shares many similar principles with our final solution. 
-->
我们尝试了许多不同的语法。
在我们致力于改变语言之前，我们使用C#属性和静态分析完成了所有工作，
但是这种方式的用户体验不是很好，并且很难用这种方式实现一个真正的类型系统。
此外，感觉它太简单了。
我们尝试了Redhawk项目中，
也就是最终成为.NET Native和[CoreRT](https://github.com/dotnet/corert)的方法。
然而，尽管这种方法与我们的最终解决方案有许多类似的原则，
但它也没有利用语言而仅仅是依赖于静态分析。

<!-- 
The basic gist of the final syntax was to simply state a method `throws` as a single bit: 
-->
最终语法的基本要点是，简单地通过单个bit来声明`throws`方法：

    void Foo() throws {
        ...
    } 

<!-- 
(For many years, we actually put the `throws` at the beginning of the method, but that read wrong.) 
-->
（多年来，我们实际上把`throws`放在了方法的头部位置，但那没有区别）

<!-- 
At this point, the issue of substitutability is quite simple.  A `throws` function cannot take the place of a non-
`throws` function (illegal strengthening).  A non-`throws` function, on the other hand, can take the place of a `throws`
function (legal weakening).  This obviously impacts virtual overrides, interface implementation, and lambdas. 
-->
在这一点上，可替代性问题非常简单，有`throws`的函数不能代替无`throws`的函数（这叫做非法加强）；
而另一方面，无`throws`的函数可以代替有`throws`的函数（合法弱化）。
而这显然会对虚拟覆盖、接口实现和lambda带来影响。

<!-- 
Of course, we did the expected co- and contravariance substitution bells and whistles.  For example, if `Foo` were
virtual and you overrode it but didn't throw exceptions, you didn't need to state the `throws` contract.  Anybody
invoking such a function virtually, of course, couldn't leverage this but direct calls could. 
-->
当然，我们做了所期望的协同和逆变替代等华而不实的功能。
例如，如果`Foo`是虚拟函数并且它被没有抛出异常的函数覆盖，则不需要再声明`throws`。
当然，虚拟地调用这样一个函数的任何调用者都无法利用这个功能，但直接调用却是可以的。

<!-- 
For example, this is legal: 
-->
例如，下面的代码是合法的：

<!-- 
    class Base {
        public virtual void Foo() throws {...}
    }
    
    class Derived : Base {
        // My particular implementation doesn't need to throw:
        public override void Foo() {...}
    } 
-->

    class Base {
        public virtual void Foo() throws {...}
    }
    
    class Derived : Base {
        // 特定的实现不需要throws关键字： 
        public override void Foo() {...}
    } 

<!-- 
and callers of `Derived` could leverage the lack of `throws`; whereas this is wholly illegal: 
-->
`Derived`的调用者可以利用无`throws`的条件，而反之则是完全非法的：

    class Base {
        public virtual void Foo () {...}
    }
    
    class Derived : Base {
        public override void Foo() throws {...}
    } 

<!-- 
Encouraging a single failure mode was quite liberating.  A vast amount of the complexity that comes with Java's checked
exceptions evaporated immediately.  If you look at most APIs that fail, they have a single failure mode anyway (once all
bug failure modes are done with abandonment): IO failed, parsing failed, etc.  And many recovery actions a developer
tends to write don't actually depend on the specifics of *what exactly* failed when, say, doing an IO.  (Some do, and
for those, the keeper pattern is often the better answer; more on this topic shortly.)  Most of the information in
modern exceptions are *not* actually there for programmatic use; instead, they are for diagnostics. 
-->
对单一故障模式的鼓励是相当自由的。Java的检查性异常带来的大量复杂性将立即消失。
如果你观察大多数调用失败的API，它们无一例外都有单一的故障模式（一旦Abandonment处理完成所有bug故障模式），这包括IO故障，解析失败等等。
开发者倾向于编写的许多恢复操作，实际上并不依赖于在做IO时*究竟哪些地方*出现故障的具体细节。
（有些操作会这样做，对于它们来说，
守护者模式通常是更好的方案，
等一下就会有关于这个守护者模式的更多内容）。
现代异常中的大多数信息实际上并*不*适合程序使用，而相反它们是用于诊断的。

<!-- 
We stuck with just this "single failure mode" for 2-3 years.  Eventually I made the controversial decision to
support multiple failure modes.  It wasn't common, but the request popped up reasonably often from teammates, and the
scenarios seemed legitimate and useful.  It did come at the expense of type system complexity, but only in all the usual
subtyping ways.  And more sophisticated scenarios -- like aborts (more on that later) -- required that we do this. 
-->
我们坚持这种“单一故障模式”有2到3年，最终我做出了支持多种故障模式的有分歧的决定。
这种情况虽然并不常见，但是这样的请求经常会从团队成员中涌现出，
而这些使用情景似乎是合法且有用的。
它确实是以类型系统的复杂性为代价的，但仅限于所有常见的子类型方式。
在更复杂的，例如中止场景中（包括后面的更多场景），则要求我们这样做。

<!-- 
The syntax looked like this: 
-->
语法如下所示：

    int Foo() throws FooException, BarException {
        ...
    } 

<!-- 
In a sense, then, the single `throws` was a shortcut for `throws Exception`. 
-->
从某种意义上说，单个`throws`是`throws Exception`的快捷方式。

<!-- 
It was very easy to "forget" the extra detail if you didn't care.  For example, perhaps you wanted to bind a lambda to
the above `Foo` API, but didn't want callers to care about `FooException` or `BarException`.  That lambda must be marked
`throws`, of course, but no more detail was necessary.  This turned out to be a very common pattern: An internal system
would use typed exceptions like this for internal control flow and error handling, but translate all of them into just
plain `throws` at the public boundary of the API, where the extra detail wasn't required. 
-->
如果你不在乎的话，很容易“忘记”额外的细节。
例如，你可能希望将lambda绑定到上面的`Foo`API中，
但不希望调用者关心`FooException`或`BarException`。
当然，lambda必须标记为`throws`，但不需要更多细节。
这被证明是一种非常常见的模式：内部系统会使用类似的类型异常来进行内部控制流和错误处理，
但是将它们全部转换为API的公共边界上的普通`throws`时，其中额外的细节是非必需的。

<!-- 
All of this extra typing added great power to recoverable errors.  But if contracts outnumbered exceptions by 10:1,
then simple `throws` exceptional methods outnumbered multi-failure-mode ones by another 10:1. 
-->
所有这些额外的类型为可恢复的错误增加了强大的功能。
但是如果合约的数量超出了异常数量的10倍，
那么简单的`throws`异常方法的数量也将超过了多故障模式方法数量的10倍。

<!-- 
At this point, you may be wondering, what differentiated this from Java's checked exceptions? 
-->
在这一点上，你可能会疑惑的是，这与Java的检查性异常有何区别？

<!-- 
1. The fact that the lion's share of errors were expressed using abandonment meant most APIs didn't throw.

2. The fact that we encouraged a single mode of failure simplified the entire system greatly.  Moreover, we made it easy
   to go from the world of multiple modes, to just a single and back again. 
-->
1. 大部分错误使用Abandonment表达方式的事实意味着大多数API都没有抛出；
2. 我们鼓励单一故障模式的事实大大简化了整个系统；
   此外，还可以轻松地从多种模式过渡到单一故障模式，并再次回到多种模式。

<!-- 
The rich type system support around weakening and strengthening also helped, as did something else we did to that helped
bridge the gap between return codes and exceptions, improved code maintainability, and more ...
--> 
利用丰富的类型系统支持弱化和强化也有帮助的，
正如我们所做的其他事情一样，也有助于缩小返回码和异常之间的差距，提高代码可维护性等等……

<!-- 
### Easily Auditable Callsites 
-->
### 易于审核的调用点（Callsite）

<!-- 
At this point in the story, we still haven't achieved the full explicit syntax of error codes.  The declarations of
functions say whether they can fail (good), but callers of those functions still inherit silent control flow (bad). 
-->
在整个故事叙述到这个时候，我们仍然没有实现错误码的完整显式语法。
函数的声明说明它们是否可能出现故障（好的方面），
但这些函数的调用者仍然继承静默的控制流（坏的方面）。

<!-- 
This brings about something I always loved about our exceptions model.  A callsite needs to say `try`: 
-->
这给我带来了异常模型中最喜欢的一些东西。调用点需要声明`try`：

    int value = try Foo(); 

<!-- 
This invokes the function `Foo`, propagates its error if one occurs, and assigns the return value to `value` otherwise. 
-->
这将调用函数`Foo`，如果发生错误则传播其错误，否则将返回值赋值给`value`。

<!-- 
This has a wonderful property: all control flow remains explicit in the program.  You can think of `try` as a kind of
conditional `return` (or conditional `throw` if you prefer).  I *freaking loved* how much easier this made code
reviewing error logic!  For example, imagine a long function with a few `try`s inside of it; having the explicit
annotation made the points of failure, and therefore control flow, as easy to pick out as `return` statements: 
-->
它有一个很不错的属性：所有控制流在程序中都是显式的。
你可以将`try`视为条件`return`（如果你愿意，也可以认为是条件`throw`）。
我*非常喜欢*这种让代码审查错误逻辑变得更容易的方式！
例如，设想有一个长函数，里面有数个`try`语句，
具有显式注释使得失败点成为控制流，因此易于选择`return`语句：

<!-- 
    void doSomething() throws {
        blah();
        var x = blah_blah(blah());
        var y = try blah(); // <-- ah, hah! something that can fail!
        blahdiblahdiblahdiblahdi();
        blahblahblahblah(try blahblah()); // <-- another one!
        and_so_on(...);
    } 
-->

    void doSomething() throws {
        blah();
        var x = blah_blah(blah());
        var y = try blah(); // <-- 啊哈！可能产生故障的调用！ 
        blahdiblahdiblahdiblahdi(); 
        blahblahblahblah(try blahblah()); // <-- 另一个可能产生故障的调用！ 
        and_so_on(...);
    } 

<!-- 
If you have syntax highlighting in your editor, so the `try`s are bold and blue, it's even better. 
-->
如果你的编辑器中有语法高亮，那么`try`将是蓝色粗体的，这样的话就更好了。

<!-- 
This delivered many of the strong benefits of return codes, but without all the baggage. 
-->
这提供了许多返回码的强大好处，却没有它的所有包袱。

<!-- 
(Both Rust and Swift now support a similar syntax.  I have to admit I'm sad we didn't ship this to the general public
years ago.  Their implementations are very different, however consider this a huge vote of confidence in their syntax.) 
-->
（Rust和Swift现在都支持类似的语法。
我不得不承认，我对于我们没有在几年前将这种语法交付给公众而感到难过。
它们的实现方式非常不同，但对此的考虑给它们的语法带来了很大的信心。）

<!-- 
Of course, if you are `try`ing a function that throws like this, there are two possibilities: 
-->
当然，如果你正在尝试这样对如此抛出的函数采取`try`方法，就有一下两种可能：

<!-- 
* The exception escapes the calling function.
* There is a surrounding `try`/`catch` block that handles the error. 
-->
* 异常从被调用函数中逃逸出来；
* 周围有一个`try`/`catch`块来处理错误。

<!-- 
In the first case, you are required to declare that your function `throws` too.  It is up to you whether to propagate
strong typing information should the callee declare it, or simply leverage the single `throws` bit, of course. 
-->
在第一种情况下，你也需要对你的函数声明`throws`。
当然，是传播由被调用函数声明的强类型信息，
还是利用单个`throws`位，这是由你来决定的。


<!-- 
In the second case, we of course understood all the typing information.  As a result, if you tried to catch something
that wasn't declared as being thrown, we could give you an error about dead code.  This was yet another controversial
departure from classical exceptions systems.  It always bugged me that `catch (FooException)` is essentially hiding a
dynamic type test.  Would you silently permit someone to call an API that returns just `object` and automatically assign
that returned value to a typed variable?  Hell no!  So we didn't let you do that with exceptions either. 
-->
在第二种情况下，我们当然理解所有的输入类型信息，
因此，如果你尝试捕捉未声明被抛出的内容，我们可能会给你一个dead code的错误，
这是与传统异常系统的另一个有争议的背离。
它总是提示我`catch（FooException）`本质上隐藏了动态类型测试。 
你是否会默默地允许调用仅返回`object`的API，
并自动将这个返回值分配给具有类型的变量？那一定是不行的！ 
因此，我们也不会让你在异常上这样做。

<!--
Here too CLU influenced us.  Liskov talks about this in [A History of CLU](
http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.127.8460&rep=rep1&type=pdf):
-->
CLU也对我们产生了影响，Liskov在[一篇关于CLU历史的文章](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.127.8460&rep=rep1&type=pdf)中谈到：

<!--
> CLU's mechanism is unusual in its treatment of unhandled exceptions. Most mechanisms pass these through: if the caller
> does not handle an exception raised by a called procedure, the exception is propagated to its caller, and so on. We
> rejected this approach because it did not fit our ideas about modular program construction. We wanted to be able to
> call a procedure knowing just its specification, not its implementation. However, if exceptions are propagated
> automatically, a procedure may raise an exception not described in its specification. 
-->
> CLU在对待未处理的异常方面的机制是不寻常的。
> 大多数机制都通过这样的方式传递：
> 如果调用者没有处理被调用过程引发的异常，则异常将传播给它的调用者，依此类推下去。
> 而我们拒绝使用这种方法，因为它不符合我们关于模块化程序构建的思想。
> 我们希望能够通过只需知道其规范而不是其实现的方式来调用过程，
> 但如果异常自动地传播下去，则过程可能会触发其规范中未描述的异常。

<!-- 
Although we discouraged wide `try` blocks, this was conceptually a shortcut for propagating an error code.  To see what
I mean, consider what you'd do in a system with error codes.  In Go, you might say the following: 
-->
虽然我们不鼓励大范围的`try`块，但这是在概念上传播错误码的快捷方式。
如果要了解我的意思，请考虑在具有错误码的系统中你是怎么做的，比如在Go中，你可能会写出如下的代码：

    if err := doSomething(); err != nil {
        return err
    } 

<!-- 
In our system, you say: 
-->
而在我们的系统中，你需要这样编码：

    try doSomething(); 

<!-- 
But we used exceptions, you might say!  It's completely different!  Sure, the runtime systems differ.  But from a
language "semantics" perspective, they are isomorphic.  We encouraged people to think in terms of error codes and not
the exceptions they knew and loved.  This might seem funny: Why not just use return codes, you might wonder?  In an
upcoming section, I will describe the true isomorphism of the situation to try to convince you of our choice. 
-->
但你可能会说，我们使用了异常，这是完全不同的！
当然，运行时系统会不一样，但从语言“语义学”的角度来看，它们其实是同构的。
我们鼓励人们根据错误码来思考，而不是他们所熟悉和喜爱的异常。
这可能会有点意思：你可能想知道，为什么不使用返回码？
在接下来的部分中，我将描述真正的场景的同构，以试图说服你我们的选择。

<!-- 
### Syntactic Sugar 
-->
### 语法糖

<!-- 
We also offered some syntactic sugar for dealing with errors.  The `try`/`catch` block scoping construct is a bit
verbose, especially if you're following our intended best practices of handling errors as locally as possible.  It also
still retains a bit of that unfortunate `goto` feel for some, especially if you are thinking in terms of return codes.
That gave way to a type we called `Result<T>`, which was simply *either* a `T` value *or* an `Exception`. 
-->
我们还提供了一些处理错误的语法糖。
`try`/`catch`块作用域结构有点冗长，特别是如果你尽量局部地遵循我们处理错误的预期最佳实践。
对于某些地方来说，它仍然不幸地保留了一些`goto`的感觉，特别是如果就返回码考虑而言。
这种方式让位于我们称为`Result<T>`的类型，它只是一个简单的`T`值*或*`Exception`。

<!-- 
This essentially bridged from the world of control-flow to the world of dataflow, for scenarios in which the latter was
more natural.  Both certainly had their place, although most developers preferred the familiar control flow syntax. 
-->
对于数据流来说更自然的场景中，这基本上是从控制流世界到数据流世界之间的桥梁，
虽然大多数开发者更喜欢熟悉的控制流语法，但两者肯定都有自己的一席之地。

<!-- 
To illustrate common usage, imagine you want to log all errors that occur, before repropagating the exception.  Though
this is a common pattern, using `try`/`catch` blocks feels a little too control flow heavy for my taste: 
-->
为了说明常见的用法，假设你希望在重新传播异常之前记录发生的所有错误，
虽然这是一种常见的模式，但使用`try`/`catch`块会让我觉得有点过于控制流式：

<!-- 
    int v;
    try {
        v = try Foo();
        // Maybe some more stuff...
    }
    catch (Exception e) {
        Log(e);
        rethrow;
    }
    // Use the value `v`... 
-->

    int v;
    try {
        v = try Foo();
        // 或许有更多的语句...
    }
    catch (Exception e) {
        Log(e);
        rethrow;
    }
    // 使用v值... 

<!-- 
The "maybe some more stuff" bit entices you to squeeze more than you should into the `try` block.  Compare this to using
`Result<T>`, leading to a more return-code feel and more convenient local handling: 
-->
“或许有更多的语句”位置处会吸引你在`try`块处挤进应该比更多的东西，
将此与使用`Result<T>`进行比较，从而产生更多的返回码感觉和更方便的本地处理：

<!-- 
    Result<int> value = try Foo() else catch;
    if (value.IsFailure) {
        Log(value.Exception);
        throw value.Exception;
    }
    // Use the value `value.Value`... 
-->

    Result<int> value = try Foo() else catch;
    if (value.IsFailure) {
        Log(value.Exception);
        throw value.Exception;
    }
    // 使用值`value.Value` ...

<!-- 
The `try ... else` construct also permitted you to substitute your own value instead, or even trigger abandonment, in
response to failure: 
-->
作为对故障的响应，`try`/`else`构造还允许你替换成自己的值，甚至触发Abandonment机制：

    int value1 = try Foo() else 42;
    int value2 = try Foo() else Release.Fail(); 

<!-- 
We also supported NaN-style propagation of dataflow errors by lifting access to `T`s members out of the `Result<T>`.
For example, let's say I have two `Result<int>`s and want to add them together.  I can do so:
-->
我们还通过从`Result<T>`中抽取成员`T`的访问来支持NaN样式的数据流错误的传播。
例如，假设有两个`Result<int>`，并希望对它们求和，那么我可以这样做：


    Result<int> x = ...;
    Result<int> y = ...;
    Result<int> z = x + y; 

<!-- 
Notice that third line, where we added the two `Result<int>`s together, yielding a -- that's right -- third `Result<T>`.
This is the NaN-style dataflow propagation, similar to C#'s new `.?` feature. 
-->
注意第三行，我们将两个`Result<int>`加到一起，没错是的，产生了第三个`Result<T>`。
这是NaN风格的数据流传播，类似于C#的`.?`新功能。

<!-- 
This approach blends what I found to be an elegant mixture of exceptions, return codes, and dataflow error propagation. 
-->
我发现这种方法是异常、返回码和数据流错误传播的优雅混合。

<!-- 
## Implementation 
-->
## 实现

<!-- 
The model I just described doesn't have to be implemented with exceptions.  It's abstract enough to be reasonably
implemented using either exceptions or return codes.  This isn't theoretical.  We actually tried it.  And this is what
led us to choose exceptions instead of return codes for performance reasons. 
-->
我刚才所描述的模型不必用异常来实现，
因为它已经足够抽象，可以使用返回码来合理地实现。
这不仅仅是理论上可行性，我们实际上就这么尝试过，
这也是导致我们出于性能原因选择异常而不是返回码方式的原因。

<!-- 
To illustrate how the return code implementation might work, imagine some simple transformations: 
-->
为了说明返回码实现是如何工作的，设想如下的一些简单的转换：

    int foo() throws {
        if (...p...) {
            throw new Exception();
        }
        return 42;
    } 

<!-- 
becomes: 
-->
变成:


    Result<int> foo() {
        if (...p...) {
            return new Result<int>(new Exception());
        }
        return new Result<int>(42);
    } 

<!-- 
And code like this: 
-->
以及，该代码：

    int x = try foo(); 

<!-- 
becomes something more like this: 
-->
变成如下的代码：

    int x;
    Result<int> tmp = foo();
    if (tmp.Failed) {
        throw tmp.Exception;
    }
    x = tmp.Value; 

<!-- 
An optimizing compiler can represent this more efficiently, eliminating excessive copying.  Especially with inlining. 
-->
优化编译器可以对其进行更有效地表示，特别是通过内联，消除了过多的复制操作。

<!-- 
If you try to model `try`/`catch`/`finally` this same way, probably using `goto`, you'll quickly see why compilers have
a hard time optimizing in the presence of unchecked exceptions.  All those hidden control flow edges! 
-->
如果你通过`goto`的方式尝试以同样的方式来对`try`/`catch`/`finally`进行建模，
你会很快看到为什么编译器在存在非检查性异常时很难进行优化，所有都隐藏在控制流的边缘上！

<!-- 
Either way, this exercise very vividly demonstrates the drawbacks of return codes.  All that goop -- which is meant to
be rarely needed (assuming, of course, that failure is rare) -- is on hot paths, mucking with your program's golden path
performance.  This violates one of our most important principles. 
-->
无论以哪种方式，这个例子都非常生动地展示了返回码的缺点。
所有的跳转语句都处在热路径上，但它们其实是很少需要的（当然，假设故障是罕见的），
并混淆你程序的黄金路径上的性能，因此违反了我们最重要的原则之一。

<!-- 
I described the results of our dual mode experiment in [my last post](
http://joeduffyblog.com/2015/12/19/safe-native-code/).  In summary, the exceptions approach was 7% smaller and 4%
faster as a geomean across our key benchmarks, thanks to a few things: 
-->
我在[上一篇文章](/2019/02/17/midori/4-safe-native-code/)中描述了我们双模式实验的结果。
总之，由于以下几点，异常方法在我们的关键基准测试中，以几何平均的方式，代码缩小了7％，速度提高了4％：

<!-- 
* No calling convention impact.
* No peanut butter associated with wrapping return values and caller branching.
* All throwing functions were known in the type system, enabling more flexible code motion.
* All throwing functions were known in the type system, giving us novel EH optimizations, like turning try/finally
  blocks into straightline code when the try could not throw. 
-->
* 没有给调用约定带来影响；
* 没有与包装返回值和调用者分支相关联的“花生酱”开销；
* 所有抛出函数在类型系统中都是已知的，从而实现更灵活的代码移动；
* 所有抛出函数在类型系统中都是已知的，为我们提供了新颖的异常处理优化；
  例如在try无法抛出时将try/finally块转换为直接代码。

<!-- 
There were other aspects of exceptions that helped with performance.  I already mentioned that we didn't grovel the
callstack gathering up metadata as most exceptions systems do.  We left diagnostics to our diagnostics subsystem.
Another common pattern that helped, however, was to cache exceptions as frozen objects, so that each `throw` didn't
require an allocation: 
-->
还有其他方面的异常也有助于提高性能。
我已经提到过，我们并没有像大多数异常系统那样使用调用点来收集元数据，并将诊断留给了诊断子系统。
然而，另一个常见的模式是将异常缓存为冻结对象，因此每次`throw`都不需要再次分配：

    const Exception retryLayout = new Exception();
    ...
    throw retryLayout; 

<!-- 
For systems with high rates of throwing and catching -- as in our parser, FRP UI framework, and other areas -- this was
important to good performance.  And this demonstrates why we couldn't simply take "exceptions are slow" as a given.
-->
对于具有高概率抛出和捕获的系统，例如我们的解析器，FRP UI框架和其他领域来说，
这对于良好的性能至关重要，这也说明了为什么我们不能简单地将“异常很慢”视为理所应当的。

<!-- 
## Patterns 
-->
## 模式

<!-- 
A number of useful patterns came up that we embellished in our language and libraries. 
-->
许多有用的模式被用于修饰我们的语言和库。

<!-- 
### Concurrency 
-->
### 并发

<!-- 
Back in 2007, I wrote [this note about concurrency and exceptions](
http://joeduffyblog.com/2009/06/23/concurrency-and-exceptions/).  I wrote it mainly from the perspective of parallel,
shared memory computations, however similar challenges exist in all concurrent orchestration patterns.  The basic issue
is that the way exceptions are implemented assumes single, sequential stacks, with single failure modes.  In a
concurrent system, you have many stacks and many failure modes, where 0, 1, or many may happen "at once." 
-->
早在2007年，我就写了[关于并发和异常的注解](http://joeduffyblog.com/2009/06/23/concurrency-and-exceptions/)的文章，
虽然当时我主要是从并行共享内存计算的角度，但是所有并发编排模式都存在类似的挑战。
基本的问题是单一顺序栈和单一故障模式的假设下来实现异常的方式，
而在并发系统中，存在许多类型的栈和多种故障模式，可能会有0个，1个或多个模式“同时”发生。

<!-- 
A simple improvement that Midori made was simply ensuring all `Exception`-related infrastructure handled cases with
multiple inner errors.  At least then a programmer wasn't forced to decide to toss away 1/N'th of the failure
information, as most exceptions systems encourage today.  More than that, however, our scheduling and stack crawling
infrastructure fundamentally knew about cactus-style stacks, thanks to our asynchronous model, and what to do with them. 
-->
Midori所做的一个简单改进就是，
确保所有与`Exception`相关的基础架构处理具有多个内部错误的情况，
至少那时程序员没有被迫决定，像今天大多数异常系统所鼓励的那样，抛弃故障信息。
然而，更重要的是，由于我们的异步模型以及其交互方式，
调度和堆栈基础架构从根本了解cactus stack（仙人掌堆栈）。

<!-- 
At first, we didn't support exceptions across asynchronous boundaries.  Eventually, however, we extended the ability to
declare `throws`, along with optional typed exceptions clauses, across asynchronous process boundaries.  This brought a
rich, typed programming model to the asynchronous actors programming model and felt like a natural extension.  This
borrowed a page from CLU's successor, [Argus](https://en.wikipedia.org/wiki/Argus_(programming_language)). 
-->
起初，我们并不支持跨异步边界的异常，
但最终我们还是在跨异步进程边界上，扩展了声明`throws`以及可选类型化异常子句的能力。
这为异步actor编程模型带来了丰富的类型化编程模型并感觉这就像是一种自然的扩展，
而这一点从CLU的继任者[Argus](http://t.cn/EIkpXHd)那里获得的灵感。

<!-- 
Our diagnostics infrastructure embellished this to give developers debugging experiences with full-blown cross-process
causality in their stack views.  Not only are stacks cactuses in a highly concurrent system, but they are often smeared
across process message passing boundaries.  Being able to debug the system this way was a big time-saver. 
-->
我们的代码诊断基础设施对其进行了修饰，
以便为开发者在栈视图中提供全面的跨进程因果关系调试体验。
cactus stack不仅在高度并发的系统中存在，而且它们经常在进程消息传递边界上出现，
以这种方式调试系统可以节省大量时间。

<!-- 
### Aborts 
-->
### 中止

<!-- 
Sometimes a subsystem needs to "get the hell out of Dodge."  Abandonment is an option, but only in response to bugs.
And of course nobody in the process can stop it in its tracks.  What if we want to back out the callstack to some point,
know that no-one on the stack is going to stop us, but then recover and keep going within the same process? 
-->
有时一个子系统需要彻底从故障的环境中摆脱出来。
因此Abandonment也是一种选择，但它也只应为应对bug而存在，
并且在进程中没有什么可以阻止它。
但如果我们想要退回到调用栈的某个点上，并已知栈中没有其他会阻止我们，
然后恢复并继续在同一个进程中运行的话，那又会怎么样呢？

<!-- 
Exceptions were close to what we wanted here.  But unfortunately, code on the stack can catch an in-flight exception,
thereby effectively suppressing the abort.  We wanted something unsuppressable. 
-->
对于这种情况，异常更接近于我们所想要的。
但不幸的是，栈上的代码可以捕获正被抛出中的异常，从而有效地抑制中止的发生。
因此，我们想要的是一些不可被抑制的构造。

<!-- 
Enter aborts.  We invented aborts mainly in support of our UI framework which used Functional Reactive Programming
(FRP), although the pattern came up in a few spots.  As an FRP recalculation was happening, it's possible that events
would enter the system, or new discoveries got made, that invalidate the current recalculation.  If that happened --
typically deep within some calculation whose stack was an interleaving of user and system code -- the FRP engine needed
to quickly get back to top of its stack where it could safely begin a recalculation.  Thanks to all of that user code
on the stack being functionally pure, aborting it mid-stream was easy.  No errant side-effects would be left behind.
And all engine code that was traversed was audited and hardened thoroughly, thanks to typed exceptions, to ensure
invariants were maintained. 
-->
接着来聊聊中止（Abort）。
我们发明中止主要是为了支持我们使用函数反应式编程（FRP）的UI框架，
而FRP模式本身将会在未来的几篇文章中出现。
当FRP重计算（recalculation）发生时，事件可能会进入系统，
或者新的发现出现，从而使当前的重计算失效。
通常这种情况都深埋在用户和系统代码交织的计算中，如果这种情况发生，
FRP引擎需要快速返回其堆栈顶部，以便可以安全地开始重计算。
由于堆栈上的所有用户代码都是纯函数式的，
因此在流的中途进行中止很容易做到，并且不会留下任何导致错误的副作用。
并且多亏了类型化异常，
所有遍历的引擎代码都经过审核和彻底修改，以确保和维护不变性。

<!-- 
The abort design borrows a page from the [capability playbook](
http://joeduffyblog.com/2015/11/10/objects-as-secure-capabilities/).  First, we introduce a base type called
`AbortException`.  It may be used directly or subclassed.  One of these is special: nobody can catch-and-ignore it.  The
exception is reraised automatically at the end of any catch block that attempts to catch it.  We say that such
exceptions are *undeniable*. 
-->
中止的设计借用了[权能的设计](/2018/11/18/midori/2-objects-as-secure-capabilities/)。
首先，我们引入了一种名为`AbortException`的基础类型，它可以直接或以子类继承的方式使用。
但它有一点却是特殊的：在他被捕获后就无法再被忽略掉，
在尝试捕获它的任何catch块的末尾都会自动重新抛出该异常。
因此，我们可以说这种异常是*不可否认*的。

<!-- 
But someone's got to catch an abort.  The whole idea is to exit a context, not tear down the entire process a la
abandonment.  And here's where capabilities enter the picture.  Here's the basic shape of `AbortException`: 
-->
但总得有代码去捕捉abort。
为此，整体的想法就是离开所处的上下文，
而像Abandonment一样销毁整个进程，
所以这就是权能发挥作用的地方。
下面是`AbortException`的基本样子：

<!-- 
    public immutable class AbortException : Exception {
        public AbortException(immutable object token);
        public void Reset(immutable object token);
        // Other uninteresting members omitted...
    } 
-->    

    public immutable class AbortException : Exception {
        public AbortException(immutable object token);
        public void Reset(immutable object token);
        // 省略掉其他不关心的成员...
    } 

<!-- 
Notice that, at the time of construction, an immutable `token` is provided; in order to suppress the throw, `Reset` is
called, and a matching `token` must be provided.  If the `token` doesn't match, abandonment occurs.  The idea is that
the throwing and intended catching parties of an abort are usually the same, or at least in cahoots with one another,
such that sharing the `token` securely with one another is easy to do.  This is a great example of objects as
unforgeable capabilities in action.
-->

请注意，在调用构造函数时，需提供不可变的`token`，
而为了抑制抛出的异常，需调用`Reset`，并且必须提供相匹配的`token`，
一旦`token`不匹配，则会触发Abandonment。
这里的想法是，abort的抛出和预期的捕获方通常是相同的，
或者至少是彼此相关联，所以这样可以容易地安全地彼此共享`token`。
所以这是对象作为不可伪造的权能，在实践中一个很好的例子。

<!-- 
And yes, an arbitrary piece of code on the stack can trigger an abandonment, but such code could already do that by
simply dereferencing `null`.  This technique prohibits executing in the aborting context when it might not have been
ready for it. 
-->
是的，栈上的任意一段代码都可以触发Abandonment，
但是这样的代码已经可以简单地通过解引用`null`来实现。
该技术禁止在尚未准备好的情况下中止上下文中的代码执行。

<!-- 
Other frameworks have similar patterns.  The .NET Framework has `ThreadAbortException` which is also undeniable unless
you invoke `Thread.ResetAbort`; sadly, because it isn't capability-based, a clumsy combination of security annotations
and hosting APIs are required to stop unintended swallowing of aborts.  More often, this goes unchecked. 
-->
其他框架具有类似的模式，
.NET Framework中有`ThreadAbortException`类型的异常，
除非你调用`Thread.ResetAbort`，否则它也是不可否认的。
但遗憾的是，由于它不基于权能，
因此需要使用安全注释和托管API的笨拙组合来阻止Abort被透明地意外处理。
并且更常见的情况是，这是未经检查的异常。

<!-- 
Thanks to exceptions being immutable, and the `token` above being immutable, a common pattern was to cache these guys
in static variables and use singletons.  For example: 
-->
由于异常是不可变的，并且上面的`token`也是不可变的，
因此常见的模式是将这些它们缓存在静态变量中并以单例的方式使用。例如：

<!-- 
    class MyComponent {
        const object abortToken = new object();
        const AbortException abortException = new AbortException(abortToken);

        void Abort() throws AbortException {
            throw abortException;
        }

        void TopOfTheStack() {
            while (true) {
                // Do something that calls deep into some callstacks;
                // deep down it might Abort, which we catch and reset:
                let result = try ... else catch<AbortException>;
                if (result.IsFailed) {
                    result.Exception.Reset(abortToken);
                }
            }
        }
    } 
-->

    class MyComponent {
        const object abortToken = new object();
        const AbortException abortException = new AbortException(abortToken);

        void Abort() throws AbortException {
            throw abortException;
        }

        void TopOfTheStack() {
            while (true) {
                // 调用函数，使得调用栈变得很深， 
                // 在调用栈深处位置可能会发生Abort，这里进行捕获并将其重置： 
                let result = try ... else catch<AbortException>;
                if (result.IsFailed) {
                    result.Exception.Reset(abortToken);
                }
            }
        }
    } 
    
<!-- 
This pattern made aborts very efficient.  An average FRP recalculation aborted multiple times.  Remember, FRP was the
backbone of all UI in the system, so the slowness often attributed to exceptions was clearly not acceptable.  Even
allocating an exception object would have been unfortunate, due to the ensuing GC pressure. 
-->
这种模式使Abort非常高效，平均下来FRP的重计算会发生多次Abort。
请记住，FRP是系统中所有UI的主干部分，因此出现通常归因于异常的缓慢显然是不可接受的。
由于随之而来的GC压力，即使异常对象的分配也是一件不幸的事。

<!-- 
### Opt-in "Try" APIs 
-->
### 可选的“Try”API

<!-- 
I mentioned a number of operations that abandoned upon failure.  That included allocating memory, performing arithmetic
operations that overflowed or divided-by-zero, etc.  In a few of these instances, a fraction of the uses are appropriate
for dynamic error propagation and recovery, rather than abandonment.  Even if abandonment is better in the common case. 
-->
我提到过因故障而导致Abandonment的操作，这包括分配内存，算术溢出或除零等。
在其中一些实例中，一部分适用于动态错误传播和恢复，
而不是Abandonment，即使在通常情况下Abandonment是更好的选择。

<!-- 
This turned out to be a pattern.  Not terribly common, but it came up.  As a result, we had a whole set of arithmetic
APIs that used a dataflow-style of propagation should overflow, NaN, or any number of things happen. 
-->
结果证明这是一种模式，虽然不是非常普遍，但它确实出现了。
因此，我们有使用数据流方式传播溢出、NaN或任何数量的可能发生的错误的一整套算术API。

<!-- 
I also already mentioned a concrete instance of this earlier, which is the ability to `try new` an allocation, when OOM
yields a recoverable error rather than abandonment.  This was super uncommon, but could crop up if you wanted to, say,
allocate a large buffer for some multimedia operation. 
-->
我之前已经提到了一个具体的实例，即当OOM产生可恢复的错误而不是Abandonment时，
能够通过`try new`尝试进行新的分配。
这种情况非常罕见，但如果你想为某些多媒体操作分配一个大缓冲区时，它可能会发挥作用。

<!-- 
### Keepers 
-->
### 守护者
<!-- 
The last pattern I'll cover is called *the keeper pattern*. 
-->
我将介绍的最后一个模式，叫做*守护者（keeper）模式*。

<!-- 
In a lot of ways, the way recoverable exceptions are handled is "inside out."  A bunch of code is called, passing
arguments down the callstack, until finally some code is reached that deems that state unacceptable.  In the exceptions
model, control flow is then propagated back up the callstack, unwinding it, until some code is found that handles the
error.  At that point if the operation is to be retried, the sequence of calls must be reissued, etc. 
-->
在很多方面，处理可恢复异常的方式是“由内而外”的：
调用一堆代码，在调用栈中传递参数，直到最后达到一些被认为是不可接受的状态。
在异常模型中，控制流而后在调用栈中向上传播并展开，直到找到处理对应错误的代码。
这个时候如果需要重试，则必须进行重新调用。

<!-- 
An alternative pattern is to use a keeper.  The keeper is an object that understands how to recover from errors "in
situ," so that the callstack needn't be unwound.  Instead, the code that would have otherwise thrown an exception
consults the keeper, who instructs the code how to proceed.  A nice aspect of keepers is that often, when done as a
configured capability, surrounding code doesn't even need to know they exist -- unlike exceptions which, in our system,
had to be declared as part of the type system.  Another aspect of keepers is that they are simple and cheap. 
-->
另一种供选择的模式是使用keeper。
keeper是一个知道如何在“原地”从错误中恢复的对象，因此就不再需要展开调用栈，
而只需要抛出异常的代码询问keeper如何处理异常，keeper则告知代码如何继续执行。
Keeper的很好的优势是，当作为配置的功能时，代码甚至不需要知道它们存在，
在这一点上，不像我们的系统中必须被声明为类型系统的一部分的异常。
除此之外，Keeper的另一个优势是它们简单而开销较低。

<!-- 
Keepers in Midori could be used for prompt operations, but more often spanned asynchronous boundaries. 
-->
Midori中的Keeper可以被用作快速操作，但更常见的用法是作为跨越异步边界的异常。

<!-- 
The canonical example of a keeper is one guarding filesystem operations.  Accessing files and directories on a file
system typically has failure modes such as: 
-->
Keeper的规范化示例是保护文件系统的操作，
访问文件系统上的文件和目录通常具有以下的一些故障模式：

<!-- 
* Invalid path specification.
* File not found.
* Directory not found.
* File in use.
* Insufficient privileges.
* Media full.
* Media write-protected. 
-->
* 无效的路径规范
* 文件未找到
* 目录未找到
* 文件正在使用
* 权限不足
* 媒体已满
* 媒体写保护

<!-- 
One option is to annotate each filesystem API with a `throws` clause for each.  Or, like Java, to create an
`IOException` hierarchy with each of these as subclasses.  An alternative is to use a keeper.  This ensures the overall
application doesn't need to know or care about IO errors, permitting recovery logic to be centralized.  Such a keeper
interface might look like this: 
-->
一种选择是使用`throws`子句为每个文件系统API进行注解，
或者，像Java一样，创建一个`IOException`类型的层次结构，每种故障都作为其子类而存在。
另一种方法是使用Keeper模式，它可以确保整个应用程序无需知道或关心IO错误，也允许集中式地恢复逻辑。
这样的keeper接口可能像如下的形式：

    async interface IFileSystemKeeper {
        async string InvalidPathSpecification(string path) throws;
        async string FileNotFound(string path) throws;
        async string DirectoryNotFound(string path) throws;
        async string FileInUse(string path) throws;
        async Credentials InsufficientPrivileges(Credentials creds, string path) throws;
        async string MediaFull(string path) throws;
        async string MediaWriteProtected(string path) throws;
    } 

<!-- 
The idea is that, in each case, the relevant inputs are provided to the keeper when failure occurs.  The keeper is then
permitted to perform an operation, possibly asynchronous, to recover.  In many cases, the keeper can optionally return
updated arguments for the operation.  For example, `InsufficientPrivileges` could return alternative `Credentials` to
use.  (Maybe the program prompted the user and she switched to an account with write access.)  In each case shown, the
keeper could throw an exception if it didn't want to handle the error, although this part of the pattern was optional. 
-->
当发生故障时，在每种情况下相关输入被提供给keeper，
然后keeper执行可能异步的操作以进行恢复,
在许多情况下，keeper可以选择返回操作更新后的参数。
例如，`InsufficientPrivileges`可以返回将会用到的`Credentials`
（也许这时程序会提示用户，然后用户切换到具有写访问权限的帐户上）。
在以上显式的每种故障情况中，一种可选的操作是，
如果keeper不想处理该错误，则可以继续采取抛出异常的方式。

<!-- 
Finally, I should note that Windows's Structured Exception Handling (SEH) system supports "continuable" exceptions which
are conceptually attempting to achieve this same thing.  They let some code decide how to restart the faulting
computation.  Unfortunately, they're done using ambient handlers on the callstack, rather than first class objects in
the language, and so are far less elegant -- and significantly more error prone -- than the keepers pattern. 
-->
最后，注意到的是Windows的结构化异常处理（SEH）系统支持“可持续（continuable）”的异常，
这种异常在概念上试图实现同样的目的，它们让一些代码决定如何重新启动错误的计算。
不幸的是，它们是在调用栈上使用环境处理程序上完成的，
而不是作为语言中的一等对象，因此它远不如keeper模式那么优雅，而且更容易出错。

<!-- 
### Future Directions: Effect Typing 
-->
### 未来方向：效果类型化

<!-- 
Most people asked us about whether having `async` and `throws` as type system attributes bifurcated the entire universe
of libraries.  The answer was "No, not really."  But it sure was painful in highly polymorphic library code. 
-->
大多数人问我们是否将`async`和`throws`作为类型系统的属性分叉到整个库环境中，
我对此的答案是，“不，这不是真的”。
但在高度多态的库代码中这样肯定是痛苦的。

<!-- 
The most jarring example was combinators like map, filter, sort, etc.  In those cases, you often have arbitrary
functions and want the `async` and `throws` attributes of those functions to "flow through" transparently. 
-->
最令人震惊的例子是组合类型，如map、filter和sort等。
在这些情况下，你经常有任意函数，并希望这些函数的`async`和`throws`属性透明地“传播”。

<!-- 
The design we had to solve this was to let you parameterize over effects.  For instance, here is a universal mapping
function, `Map`, that propagates the `async` or `throws` effect of its `func` parameter: 
-->
我们必须解决的设计是让你对效果进行参数化。
例如，现在有一个通用的映射函数`Map`，它传播其`func`参数的`async`或`throws`效果：

    U[] Map<T, U, effect E>(T[] ts, Func<T, U, E> func) E {
        U[] us = new U[ts.Length];
        for (int i = 0; i < ts.Length; i++) {
            us[i] = effect(E) func(ts[i]);
        }
        return us;
    } 

<!-- 
Notice here that we've got an ordinary generic type, `E`, except that its declaration is prefixed by the keyword 
`effect`.  We then use `E` symbolically in place of the effects list of the `Map` signature, in addition to using it in
the "propagate" position via `effect(E)` when invoking `func`.  It's a pretty trivial exercise in substitution,
replacing `E` with `throws` and `effect(E)` with `try`, to see the logical transformation. 
-->
请注意，我们有一个普通的泛型类型`E`，除了它的声明以关键字`effect`为前缀之外。
然后除了在调用`func`时通过 `effect(E)`在“传播”位置使用它之外，
我们象征性地使用`E`来代替`Map`签名的效果列表。
这是一个非常简单的替代操作，用`effect(E)`替换`try`，以及用`E`替换`throws`，来看看逻辑的转换。

<!-- 
A legal invocation might be: 
-->
一种合法的调用如下：

    int[] xs = ...;
    string[] ys = try Map<int, string, throws>(xs, x => ...); 

<!-- 
Notice here that the `throws` flows through, so that we can pass a callback that throws exceptions. 
-->
请注意，这里的`throws`已经透明地传播下去，因此我们可以传递一个抛出异常的回调。

<!-- 
As a total aside, we discussed taking this further, and allowing programmers to declare arbitrary effects.  I've
[hypothesized about such a type system previously](
http://joeduffyblog.com/2010/04/25/from-simple-annotations-for-analysis-to-effect-typing/).  We were concerned, however,
that this sort of higher order programming might be gratuitously clever and hard to understand, no matter how powerful.
The simple model above probably would've been a sweet spot and I think we'd have done it given a few more months. 
-->
总而言之，我们将上述的讨论更进了一步，并允许程序员声明任意的效果。
我[以前曾经对这种类型的系统做过假设](
http://joeduffyblog.com/2010/04/25/from-simple-annotations-for-analysis-to-effect-typing/)，
然而我所担心的是，无论它多么强大，这种高阶编程都可能是过度的精巧且难以理解。
但上述的简单模型应该会是有意义的，我想如果多几个月时间，我们会将其加以实现。

<!-- 
# Retrospective and Conclusions 
-->
# 回顾与总结

<!-- 
We've reached the end of this particular journey.  As I said at the outset, a relatively predictable and tame outcome.
But I hope all that background helped to take you through the evolution as we sorted through the landscape of errors. 
-->
我们已经来到了这段特殊旅程的终点，正如我在一开始所说的，最终是一个相对可预测和温和的结果。
通过我们对错误的情况进行分类的基本原则，希望所有这些背景都能帮助你完成项目中错误处理的进化。

<!-- 
In summary, the final model featured: 
-->
总之，最终的模型具有以下的特点：

<!-- 
* An architecture that assumed fine-grained isolation and recoverability from failure.
* Distinguishing between bugs and recoverable errors.
* Using contracts, assertions, and, in general, abandonment for all bugs.
* Using a slimmed down checked exceptions model for recoverable errors, with a rich type system and language syntax.
* Adopting some limited aspects of return codes -- like local checking -- that improved reliability. 
-->
* 一种假设细粒度隔离和从故障中的可恢复性的体系结构；
* 区分bug和可恢复的错误；
* 使用合约、断言、以及面向所有bug的更通用的Abandonment；
* 对于可恢复的错误，使用向下精简的，具有丰富的类型系统和语言语法的检查性异常模型；
* 采用返回码的有限的某些特性，比如局部检查，以提高可靠性。

<!-- 
And, though this was a multi-year journey, there were areas of improvement we were actively working on right up until
our project's untimely demise.  I classified them differently because we didn't have enough experience using them to
claim success.  I would have hoped we'd have tidied up most of them and shipped them if we ever got that far.  In
particular, I'd have liked to put this one into the final model category: 
-->
虽然这是一个多年的旅程，但我们一直致力于多个领域中积极努力的改进，直到我们的项目过早地夭折。
因为我们没有使用它们的足够经验以宣称获得了成功，所以我对它们进行了不同的分类。
如果能走得更远，我希望我们能将大部分内容整理出来并将它们交付出去。
特别地，我想把下列原则放到最终模型的类别中：

<!-- 
* Leveraging non-null types by default to eliminate a large class of nullability annotations. 
-->
* 默认情况下，利用非空类型可以消除大量的可空性注解。

<!-- 
Abandonment, and the degree to which we used it, was in my opinion our biggest and most successful bet with the Error
Model.  We found bugs early and often, where they are easiest to diagnose and fix.  Abandonment-based errors outnumbered
recoverable errors by a ratio approaching 10:1, making checked exceptions rare and tolerable to the developer. 
-->
Abandonment以及我们对它的使用程度，在我看来是在错误模型上最大和最成功的赌注。
我们进程很早地发现bug，它们是最容易诊断和修复的，
基于Abandonment的错误数量超过可恢复错误的比例接近10:1，
因此，这样使得检查性异常很少出现并且可以被开发者所容忍。


<!-- 
Although we never had a chance to ship this, we have since brought some of these lessons learned to other settings. 
-->
虽然从未有机会将这些项目发布出来，
但我们已经将其中的一些经验教训带到了其他的项目中去。

<!-- 
During the Microsoft Edge browser rewrite from Internet Explorer, for example, we adopted abandonment in a few areas.
The key one, applied by a Midori engineer, was OOM.  The old code would attempt to limp along as I described earlier
and almost always did the wrong thing.  My understanding is that abandonment has found numerous lurking bugs, as was our
experience regularly in Midori when porting existing codebases.  The great thing too is that abandonment is more of an
architectural discipline that can be adopted in existing code-bases ranging in programming languages. 
-->
例如，在从Internet Explorer重写Microsoft Edge浏览器期间，我们在一些区域采用了Abandonment机制。
由Midori的工程师处理的一个关键领域是OOM。如前所述，旧代码会艰难地执行，并且几乎总是会出现错误，
而Abandonment发现了许多潜伏的错误，就像我们常常在Midori中移植现有代码库时所经历的那样。
更重要的是，Abandonment更多的是一种架构上的原则，可以在编程语言的现有代码库中所采用。

<!-- 
The architectural foundation of fine-grained isolation is critical, however many systems have an informal notion of this
architecture.  A reason why OOM abandonment works well in a browser is that most browsers devote separate processes to
individual tabs already.  Browsers mimic operating systems in many ways and here too we see this playing out. 
-->
细粒度隔离的架构基础至关重要，但许多系统都有这种架构的非正式概念。
面向OOM的Abandonment机制在浏览器中运行良好的原因是，大多数浏览器已经将单独进程专门用于各个选项卡，
浏览器正以多种方式模仿操作系统的行为，并且在这里我们也看到了同样的情况。

<!-- 
More recently, we've been exploring proposals to bring some of this discipline -- [including contracts](
https://www.youtube.com/watch?v=Hjz1eBx91g8) -- to C++.  There are also [concrete proposals](
https://github.com/dotnet/roslyn/issues/119) to bring some of these features to C# too.  We are actively iterating on
[a proposal that would bring some non-null checking to C#](https://github.com/dotnet/roslyn/issues/5032).  I have to
admit, I wish all of those proposals the best, however nothing will be as bulletproof as an entire stack written in the
same error discipline.  And remember, the entire isolation and concurrency model is essential for abandonment at scale. 
-->
最近，我们一直在探索将这些，[包括合约](https://www.youtube.com/watch?v=Hjz1eBx91g8)在内的原则带到C++的提议，
另外还有一些将这些功能带到C#中的[具体提议](https://github.com/dotnet/roslyn/issues/119)，
我们还正在积极地迭代出[给C#带来一些非空检查的提议](https://github.com/dotnet/roslyn/issues/5032)。
我不得不承认，我希望所有这些提案都是最好的，但是没有一个能像在同一个错误规则中写入整个栈那样具有防弹性。
请记住，整个隔离和并发模型对于大规模Abandonment至关重要。

<!-- 
I am hopeful that continued sharing of knowledge will lead to even more wide-scale adoption some of these ideas. 
-->
我希望继续分享这些知识，使得能够更广泛地采用其中的一些想法。

<!-- 
And, of course, I've mentioned that Go, Rust, and Swift have given the world some very good systems-appropriate error
models in the meantime.  I might have some minor nits here and there, but the reality is that they're worlds beyond
what we had in the industry at the time we began the Midori journey.  It's a good time to be a systems programmer! 
-->
当然，我已经提到，Go、Rust和Swift在此期间为业界提供了一些非常好的有关系统的错误模型。
我们的设计可能会在各个地方有一些微小的瑕疵，
但现实情况是，这些语言（Go、Rust和Swift）所诞生的环境，
已经超出了在我们开始Midori之旅时在行业中所具有的环境。
所以现在正是成为一名系统程序员的好时机！

<!-- 
Next time I'll talk more about the language.  Specifically, we'll see how Midori was able to tame the garbage collector
using a magical elixir of architecture, language support, and libraries.  I hope to see you again soon! 
-->
下一次我会更多地谈谈这门语言，具体来说将会看到Midori是如何利用架构、语言支持和库的万能钥匙来驯服垃圾收集器的
（*译者注：实际上作者后文中并没有写到GC*）。
我希望很快能和你们再次见面！
