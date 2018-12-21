---
title: Midori博客系列翻译（5）——关于并发的15年
date: 2018-12-19 15:58:00
tags: [操作系统, Midori, 翻译, 并发]
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
Midori是用基于C＃的提前编译的，类型安全的语言编写的。 除了我们的微内核之外，整个系统都是用它编写的，包括驱动程序，域内核和所有用户代码。 我一路上暗示了一些事情，现在是时候正面解决这些问题了。 整个语言是一个巨大的空间，需要一系列的帖子。 第一？ 错误模型。 传达和处理错误的方式是任何语言的基础，尤其是用于编写可靠操作系统的语言。 像我们在Midori所做的许多其他事情一样，“整个系统”的方法对于实现它是必要的，需要几年的几次迭代。 然而，我经常听到老队友的消息，这是他们最想念Midori的节目。 它也适合我。 所以，不用多说，让我们开始吧。

<!-- 
# Introduction 
-->
# 介绍

<!-- 
The basic question an Error Model seeks to answer is: how do "errors" get communicated to programmers and users of the
system?  Pretty simple, no?  So it seems. 
-->
错误模型试图回答的基本问题是：“错误”如何传达给程序员和系统用户？ 很简单，不是吗？ 所以它看起来。

<!-- 
One of the biggest challenges in answering this question turns out to be defining what an error actually *is*.  Most
languages lump bugs and recoverable errors into the same category, and use the same facilities to deal with them.  A
`null` dereference or out-of-bounds array access is treated the same way as a network connectivity problem or parsing
error.  This consistency may seem nice at first glance, but it has deep-rooted issues.  In particular, it is misleading
and frequently leads to unreliable code. 
-->
回答这个问题的最大挑战之一就是定义错误究竟是什么。 大多数语言将错误和可恢复的错误归为同一类别，并使用相同的工具来处理它们。 空取消引用或越界数组访问的处理方式与网络连接问题或解析错误的处理方式相同。 这种一致性乍一看似乎很不错，但它有根深蒂固的问题。 特别是，它具有误导性并经常导致不可靠的代码。

<!-- 
Our overall solution was to offer a two-pronged error model.  On one hand, you had fail-fast -- we called it
*abandonment* -- for programming bugs.  And on the other hand, you had statically checked exceptions for recoverable
errors.  The two were very different, both in programming model and the mechanics behind them.  Abandonment
unapologetically tore down the entire process in an instant, refusing to run any user code while doing so.  (Remember,
a typical Midori program had many small, lightweight processes.)  Exceptions, of course, facilitated recovery, but had
deep type system support to aid checking and verification. 
-->
我们的整体解决方案是提供双管齐下的错误模型。 一方面，你失败了 - 我们称之为放弃 - 用于编程错误。 另一方面，您已经静态检查了可恢复错误的异常。 这两者在编程模型和它们背后的机制方面都非常不同。 放弃无条件地在瞬间撕毁整个过程，拒绝运行任何用户代码。 （请记住，一个典型的Midori程序有许多小而轻的过程。）当然，例外有助于恢复，但有深入的系统支持以帮助检查和验证。

<!-- 
This journey was long and winding.  To tell the tale, I've broken this post into six major areas: 
-->
这段旅程漫长而曲折。 为了讲述这个故事，我将这篇文章分为六个主要方面：

<!-- 
* [Ambitions and Learnings](#ambitions-and-learnings)
* [Bugs Aren't Recoverable Errors!](#bugs-arent-recoverable-errors)
* [Reliability, Fault-Tolerance, and Isolation](#reliability-fault-tolerance-and-isolation)
* [Bugs: Abandonment, Assertions, and Contracts](#bugs-abandonment-assertions-and-contracts)
* [Recoverable Errors: Type-Directed Exceptions](#recoverable-errors-type-directed-exceptions)
* [Retrospective and Conclusions](#retrospective-and-conclusions) 
-->
* [野心和学习]（＃野心和学习）
* [错误不是可恢复的错误！]（＃bugs-arent-recoverable-errors）
* [可靠性，容错和隔离]（＃reliability-fault-tolerance-and-isolation）
* [错误：放弃，断言和合同]（＃bugs-abandonment-assertions-and-contracts）
* [可恢复的错误：类型导向的异常]（＃recoverable-errors-type-directed-exceptions）
* [回顾与结论]（＃retrospective-and-findings）

<!-- 
In hindsight, certain outcomes seem obvious.  Especially given modern systems languages like Go and Rust.  But some
outcomes surprised us.  I'll cut to the chase wherever I can but I'll give ample back-story along the way.  We tried out
plenty of things that didn't work, and I suspect that's even more interesting than where we ended up when the dust
settled. 
-->
事后看来，某些结果似乎很明显。 特别是现代系统语言，如Go和Rust。 但有些结果让我们感到惊讶 无论我在哪里，我都会尽力而为，但我会在此过程中提供充足的背景故事。 我们尝试了许多不起作用的东西，我怀疑它比尘埃落定的时候更有趣。

<!-- 
# Ambitions and Learnings 
-->
# 野心和学习

<!-- 
Let's start by examining our architectural principles, requirements, and learnings from existing systems. 
-->
让我们首先从现有系统中检查我们的架构原则，要求和学习。

<!-- 
## Principles 
-->
## 原则 

<!-- 
As we set out on this journey, we called out several requirements of a good Error Model: 
-->
在我们开始这个旅程时，我们提出了一个好的错误模型的几个要求：

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

* **可用**。开发人员面对错误时必须很容易做出“正确”的事情，几乎就像偶然一样。一位朋友和同事有名的称其为“成功之坑”。为了写出惯用的代码，模型不应该施加过多的仪式。理想情况下，我们的目标受众认知熟悉。

* **可靠**。误差模型是整个系统可靠性的基础。毕竟，我们正在构建一个操作系统，因此可靠性至关重要。你甚至可能指责我们痴迷于追求它的极端水平。我们对编程模型开发的大量指导是“正确的构造”。

* **高性能**。常见情况需要非常快。这意味着成功路径的开销尽可能接近零。失败路径的任何增加的成本必须完全是“付费游戏”。与许多愿意过度惩罚错误路径的现代系统不同，我们有几个性能关键的组件，这是不可接受的，因此错误必须也要快得多。

* **并发**。我们的整个系统是分布式的，高度并发。这引起了其他错误模型中通常存在的问题。他们需要成为我们的前沿和中心。

* **可诊断的**。无论是交互式还是事后调试失败都需要高效且简单。

* **可组合**。错误模型的核心是编程语言功能，位于开发人员代码表达的中心。因此，它必须提供熟悉的正交性和可组合性以及系统的其他功能。单独编写的组件必须是自然，可靠和可预测的。

<!-- 
It's a bold claim, however I do think what we ended up with succeeded across all dimensions. 
-->
这是一个大胆的主张，但我确实认为我们最终取得了成功的所有方面。

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
现有的错误模型不符合我们的上述要求。 至少不完全。 如果一个人在一个维度上做得很好，那么在另一个维度上表现不佳。 例如，错误代码可以具有良好的可靠性，但许多程序员发现它们易于使用; 进一步说，做错事很容易 - 比如忘记检查一个 - 这显然违反了“成功之门”的要求。

<!-- 
Given the extreme level of reliability we sought, it's little surprise we were dissatisfied with most models.
-->
鉴于我们追求的极高可靠性，我们对大多数模型不满意并不令人惊讶。

<!-- 
If you're optimizing for ease-of-use over reliability, as you might in a scripting language, your conclusions will
differ significantly.  Languages like Java and C# struggle because they are right at the crossroads of scenarios --
sometimes being used for systems, sometimes being used for applications -- but overall their Error Models were very
unsuitable for our needs. 
-->
如果您使用脚本语言优化易用性而非可靠性，那么您的结论将会有很大差异。 像Java和C＃这样的语言很困难，因为它们正处于场景的十字路口 - 有时用于系统，有时用于应用程序 - 但总体而言，它们的错误模型非常不适合我们的需求。

<!-- 
Finally, also recall that this story began in the mid-2000s timeframe, before Go, Rust, and Swift were available for our
consideration.  These three languages have done some great things with Error Models since then. -->
最后，还记得这个故事始于2000年代中期的时间框架，之后Go，Rust和Swift可供我们考虑。 从那以后，这三种语言在Error Models中做了一些很棒的事情。

<!-- 
### Error Codes 
-->
### 错误码

<!-- 
Error codes are arguably the simplest Error Model possible.  The idea is very basic and doesn't even require language
or runtime support.  A function just returns a value, usually an integer, to indicate success or failure: 
-->
错误代码可以说是最简单的错误模型。 
这个想法非常基础，甚至不需要语言或运行时支持。 
函数只返回一个值，通常是一个整数，表示成功或失败：

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
        // <在这里尝试一下>
        if (failed) {
            return 1;
        }
        return 0;
    } 

<!-- 
This is the typical pattern, where a return of `0` means success and non-zero means failure.  A caller must check it: 
-->
这是典型的模式，返回“0”表示成功，非零表示失败。 来电者必须检查：

<!-- 
    int err = foo();
    if (err) {
        // Error!  Deal with it.
    } 
-->
    int err = foo();
    if (err) {
        // 错误！ 处理它。
    } 

<!-- 
Most systems offer constants representing the set of error codes rather than magic numbers.  There may or may not be
functions you can use to get extra information about the most recent error (like `errno` in standard C and
`GetLastError` in Win32).  A return code really isn't anything special in the language -- it's just a return value. 
-->
大多数系统提供表示错误代码集的常量而不是幻数。 您可以使用或不使用函数来获取有关最新错误的额外信息（例如标准C中的errno和Win32中的GetLastError）。 
返回代码在语言中并不是特别的 - 它只是一个返回值。

<!-- 
C has long used error codes.  As a result, most C-based ecosystems do.  More low-level systems code has been written
using the return code discipline than any other.  [Linux does](http://linux.die.net/man/3/errno), as do countless
mission-critical and realtime systems.  So it's fair to say they have an impressive track record going for them! 
-->
C长期使用错误代码。 因此，大多数基于C的生态系统都可以。 使用返回代码规则编写的低级系统代码比任何其他代码都要多。 Linux确实如此，无数关键任务和实时系统也是如此。 因此，公平地说，他们有令人印象深刻的记录！

<!-- 
On Windows, `HRESULT`s are equivalent.  An `HRESULT` is just an integer "handle" and there are a bunch of constants and
macros in `winerror.h` like `S_OK`, `E_FAULT`, and `SUCCEEDED()`, that are used to create and check values.  The most
important code in Windows is written using a return code discipline.  No exceptions are to be found in the kernel.  At
least not intentionally. 
-->
在Windows上，HRESULT是等效的。 HRESULT只是一个整数“句柄”，winerror.h中有一堆常量和宏，如S_OK，E_FAULT和SUCCEEDED（），用于创建和检查值。 Windows中最重要的代码是使用返回代码规则编写的。 在内核中没有例外。 至少不是故意的。

<!-- 
In environments with manual memory management, deallocating memory on error is uniquely difficult.  Return codes can
make this (more) tolerable.  C++ has more automatic ways of doing this using [RAII](
https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization), but unless you buy into the C++ model whole hog
-- which a fair number of systems programmers don't -- then there's no good way to incrementally use RAII in your C
programs. 
-->
在具有手动内存管理的环境中，在出错时释放内存是非常困难的。 返回代码可以使这（更多）容忍。 C ++使用RAII有更多自动方式来做这件事，但除非你购买C ++模型整体猪 - 相当数量的系统程序员没有 - 然后没有好的方法在你的C程序中逐步使用RAII。

<!-- 
More recently, Go has chosen error codes.  Although Go's approach is similar to C's, it has been modernized with much
nicer syntax and libraries. 
-->
最近，Go选择了错误代码。 虽然Go的方法与C类似，但它已经过现代化，具有更好的语法和库。
<!-- 
Many functional languages use return codes disguised in monads and named things like `Option<T>`, `Maybe<T>`, or
`Error<T>`, which, when coupled with a dataflow-style of programming and pattern matching, feel far more natural.  This
approach removes several major drawbacks to return codes that we're about to discuss, especially compared to C.  Rust
has largely adopted this model but has dome some exciting things with it for systems programmers. 
-->
许多函数式语言使用monad中伪装的返回代码，并命名为Option <T>，Maybe <T>或Error <T>，它们与数据流式的编程和模式匹配相结合，感觉更自然。 这种方法消除了我们即将讨论的返回代码的几个主要缺点，特别是与C.相比，Rust已经在很大程度上采用了这个模型，但它为系统程序员提供了一些令人兴奋的东西。

<!-- 
Despite their simplicity, return codes do come with some baggage; in summary: 
-->
尽管它们简单，但返回代码确实带有一些包袱; 综上所述：

<!-- 
* Performance can suffer.
* Programming model usability can be poor.
* The biggie: You can accidentally forget to check for errors. 
-->
* 性能可能会受到影响
* 编程模型的可用性可能很差。
* 重要人物：您可能会意外忘记检查错误。

<!-- 
Let's discuss each one, in order, with examples from the languages cited above. 
-->
让我们依次用上面引用的语言中的例子来讨论每一个。

<!-- 
#### Performance 
-->
#### 性能

<!-- 
Error codes fail the test of "zero overhead for common cases; pay for play for uncommon cases": -->
错误代码未通过“常见情况的零开销;为非常见情况付费”的测试：

<!-- 
1. There is calling convention impact.  You now have *two* values to return (for non-`void` returning functions): the
   actual return value and the possible error.  This burns more registers and/or stack space, making calls less
   efficient.  Inlining can of course help to recover this for the subset of calls that can be inlined. 
-->
1. 呼吁会议有影响力。 您现在有两个值要返回（对于非void返回函数）：实际返回值和可能的错误。 这会烧掉更多寄存器和/或堆栈空间，从而降低调用效率。 内联当然可以帮助恢复可以内联的调用子集。

<!-- 
2. There are branches injected into callsites anywhere a callee can fail.  I call costs like this "peanut butter,"
   because the checks are smeared across the code, making it difficult to measure the impact directly.  In Midori we
   were able to experiment and measure, and confirm that yes, indeed, the cost here is nontrivial.  There is also a
   secondary effect which is, because functions contain more branches, there is more risk of confusing the optimizer. 
-->
2. 在被调用者失败的任何地方都有注入分支的分支。 我称之为“花生酱”的成本，因为检查在代码中被涂抹，使得难以直接衡量影响。 在Midori，我们能够进行实验和测量，并确认是的，确实，这里的成本是非常重要的。 还有一个次要影响，因为函数包含更多分支，更有可能混淆优化器。
<!-- 
This might be surprising to some people, since undoubtedly everyone has heard that "exceptions are slow."  It turns out
that they don't have to be.  And, when done right, they get error handling code and data off hot paths which increases
I-cache and TLB performance, compared to the overheads above, which obviously decreases them. 
-->
这对某些人来说可能是令人惊讶的，因为毫无疑问，每个人都听说过“例外情况很慢。”事实证明，他们并非如此。 并且，当完成正确时，他们从热路径获得错误处理代码和数据，与上面的开销相比，这增加了I-cache和TLB性能，这显然降低了它们。
<!-- 
Many high performance systems have been built using return codes, so you might think I'm nitpicking.  As with many
things we did, an easy criticism is that we took too extreme an approach.  But the baggage gets worse. 
-->
许多高性能系统都是使用返回码构建的，所以你可能会认为我在挑剔。 正如我们所做的许多事情一样，一个简单的批评是我们采取了极端的方法。 但行李变得更糟。

<!-- 
#### Forgetting to Check Them 
-->
#### 忘了检查他们
<!-- 
It's easy to forget to check a return code.  For example, consider a function: 
-->
忘记查看返回代码很容易。 例如，考虑一个函数：

    int foo() { ... } 


<!-- 
Now at the callsite, what if we silently ignore the returned value entirely, and just keep going?-->
现在在callsite上，如果我们完全默默地忽略返回的值，那就继续下去呢？

<!-- 
    foo();
    // Keep going -- but foo might have failed! 
-->
    foo();
    // 继续 - 但foo可能失败了！

<!-- 
At this point, you've masked a potentially critical error in your program.  This is easily the most vexing and
damaging problem with error codes.  As I will show later, option types help to address this for functional languages.
But in C-based languages, and even Go with its modern syntax, this is a real issue. 
-->
此时，您已经屏蔽了程序中可能存在的严重错误。 这很容易成为错误代码最棘手和最具破坏性的问题。 正如我稍后将要介绍的那样，选项类型有助于解决功能语言问题。 但是在基于C语言，甚至是Go的现代语法中，这是一个真正的问题。

<!-- 
This problem isn't theoretical.  I've encountered numerous bugs caused by ignoring return codes and I'm sure you have
too.  Indeed, in the development of this very Error Model, my team encountered some fascinating ones.  For example, when
we ported Microsoft's Speech Server to Midori, we found that 80% of Taiwan Chinese (`zh-tw`) requests were failing.  Not
failing in a way the developers immediately saw, however; instead, clients would get a gibberish response.  At first, we
thought it was our fault.  But then we discovered a silently swallowed `HRESULT` in the original code.  Once we got it
over to Midori, the bug was thrown into our faces, found, and fixed immediately after porting.  This experience
certainly informed our opinion about error codes. 
-->
这个问题不是理论上的。 我遇到了忽略返回代码导致的无数错误，我相信你也有。 实际上，在这个非常错误模型的开发中，我的团队遇到了一些令人着迷的问题。 例如，当我们将微软的语音服务器移植到Midori时，我们发现80％的台湾中文（zh-tw）请求失败了。 然而，开发人员不会立即看到失败; 相反，客户会得到一个胡言乱语的回应。 起初，我们认为这是我们的错。 但后来我们在原始代码中发现了一个静默吞噬的HRESULT。 一旦我们把它送到了Midori，这个虫就被扔到我们的脸上，找到并在移植后立即修复。 这种经验肯定告诉了我们对错误代码的看法。

<!-- 
It's surprising to me that Go made unused `import`s an error, and yet missed this far more critical one.  So close! 
-->
令我惊讶的是，Go使未使用的进口产生了错误，但却错过了这个更为关键的错误。 很近！

<!-- 
It's true you can add a static analysis checker, or maybe an "unused return value" warning as most commercial C++
compilers do.  But once you've missed the opportunity to add it to the core of the language, as a requirement, none of
those techniques will reach critical mass due to complaints about noisy analysis. 
-->
确实，您可以添加静态分析检查器，或者像大多数商业C ++编译器那样添加“未使用的返回值”警告。 但是，一旦你错过了将其添加到语言核心的机会，作为一项要求，由于对噪声分析的抱怨，这些技术都不会达到临界质量。

<!-- 
For what it's worth, forgetting to use return values in our language was a compile time error.  You had to explicitly
ignore them; early on we used an API for this, but eventually devised language syntax -- the equivalent of `>/dev/null`: 
-->
对于它的价值，忘记在我们的语言中使用返回值是编译时错误。 你必须明确地忽略它们; 早期我们使用了API，但最终设计了语言语法 - 相当于> / dev / null：


    ignore foo(); 

<!-- 
We didn't use error codes, however the inability to accidentally ignore a return value was important for the overall
reliability of the system.  How many times have you debugged a problem only to find that the root cause was a return
value you forgot to use?  There have even been security exploits where this was the root cause.  Letting developers say
`ignore` wasn't bulletproof, of course, as they could still do the wrong thing.  But it was at least explicit and
auditable. 
-->
我们没有使用错误代码，但无法意外忽略返回值对于系统的整体可靠性非常重要。 你调试一个问题多少次才发现根本原因是你忘记使用的返回值？ 甚至有安全漏洞，这是根本原因。 让开发人员说忽略不是防弹，当然，因为他们仍然可以做错事。 但它至少是明确的和可审计的。

<!-- 
#### Programming Model Usability 
-->
#### 编程模型可用性

<!-- 
In C-based languages with error codes, you end up writing lots of hand-crafted `if` checks everywhere after function
calls.  This can be especially tedious if many of your functions fail which, in C programs where allocation failures are
also communicated with return codes, is frequently the case.  It's also clumsy to return multiple values. 
-->
在带有错误代码的基于C语言中，如果在函数调用后的任何地方进行检查，则最终会编写大量手工制作的语言。 如果你的许多功能都失败了，这可能会特别繁琐，在C程序中，分配失败也与返回码通信，这种情况经常发生。 返回多个值也很笨拙。

<!-- 
A warning: this complaint is subjective.  In many ways, the usability of return codes is actually elegant.  You reuse
very simple primitives -- integers, returns, and `if` branches -- that are used in myriad other situations.  In my
humble opinion, errors are an important enough aspect of programming that the language should be helping you out. 
-->
警告：此投诉是主观的。 在许多方面，返回代码的可用性实际上是优雅的。 您可以重用非常简单的原语 - 整数，返回和if分支 - 在无数其他情况下使用。 在我看来，错误是编程的一个重要方面，语言应该帮助你。

<!-- 
Go has a nice syntactic shortcut to make the standard return code checking *slightly* more pleasant: 
-->
Go有一个很好的语法快捷方式，使标准的返回代码检查*稍微*更愉快：

<!-- 
    if err := foo(); err != nil {
        // Deal with the error.
    } 
-->
    if err := foo(); err != nil {
        // 处理错误。
    } 
    
<!-- 
Notice that we've invoked `foo` and checked whether the error is non-`nil` in one line.  Pretty neat. 
-->
请注意，我们已经调用了`foo`并在一行中检查错误是否为非“nil”。 很简约。

<!-- 
The usability problems don't stop there, however. 
-->
然而，可用性问题并不止于此。

<!-- 
It's common that many errors in a given function should share some recovery or remediation logic.  Many C programmers
use labels and `goto`s to structure such code.  For example: 
-->
通常，给定函数中的许多错误应该共享一些恢复或补救逻辑。 许多C程序员使用标签和`goto`s来构造这样的代码。 例如：

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

    // 正常函数退出。
    return 0;

    // ...
    Error:
    // 做一些关于`错误'的事情。
    return error; 

<!-- 
Needless to say, this is the kind of code only a mother could love. 
-->
不用说，这是只有母亲才能爱的代码。

<!-- 
In languages like D, C#, and Java, you have `finally` blocks to encode this "before scope exits" pattern more directly.
Similarly, Microsoft's proprietary extensions to C++ offer `__finally`, even if you're not fully buying into RAII and
exceptions.  And D provides `scope` and Go offers `defer`.  All of these help to eradicate the `goto Error` pattern. 
-->
在D，C＃和Java等语言中，您最终会直接对块“在范围退出之前”模式进行编码。 同样，微软对C ++的专有扩展提供__finally，即使你没有完全购买RAII和例外。 D提供范围，Go提供延迟。 所有这些都有助于根除goto错误模式。

<!-- 
Next, imagine my function wants to return a real value *and* the possibility of an error?  We've burned the return slot
already so there are two obvious possibilities: 
-->
接下来，假设我的函数想要返回一个真值和错误的可能性？ 我们已经烧掉了返回槽，因此有两个明显的可能性：

<!-- 
1. We can use the return slot for one of the two values (commonly the error), and another slot -- like a pointer
   parameter -- for the other of the two (commonly the real value).  This is the common approach in C.

2. We can return a data structure that carries the possibility of both in its very structure.  As we will see, this is
   common in functional languages.  But in a language like C, or Go even, that lacks parametric polymorphism, you lose
   typing information about the returned value, so this is less common to see.  C++ of course adds templates, so in
   principle it could do this, however because it adds exceptions, the ecosystem around return codes is lacking. 
-->
1. 我们可以将返回槽用于两个值中的一个（通常是错误），另一个槽 - 比如指针参数 - 用于两者中的另一个（通常是实际值）。 这是C中的常用方法。
2. 我们可以返回一个在其结构中具有两者可能性的数据结构。 我们将会看到，这在函数式语言中很常见。 但是在像C或Go等语言中，缺少参数多态，你会丢失有关返回值的输入信息，所以这种情况不太常见。 C ++当然会添加模板，因此原则上它可以做到这一点，但是因为它增加了异常，所以缺少返回代码的生态系统。

<!-- 
In support of the performance claims above, imagine what both of these do to your program's resulting assembly code. 
-->
为了支持上面的性能声明，想象一下这两者对程序生成的汇编代码有何影响。

<!-- 
##### Returning Values "On The Side" 
-->
##### 回归价值观“侧面”

<!-- 
An example of the first approach in C looks like this: 
-->
C语言中第一种方法的示例如下所示：

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
        // <在这里尝试一下>
        if (failed) {
            return 1;
        }
        *out = 42;
        return 0;
    } 
<!-- 
The real value has to be returned "on the side," making callsites clumsy: 
-->
真正的价值必须“在旁边”，使得呼叫笨拙：

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
        // 错误！ 处理它。
    }
    else {
        // 使用值...
    } 

<!-- 
In addition to being clumsy, this pattern perturbs your compiler's [definite assignment analysis](
https://en.wikipedia.org/wiki/Definite_assignment_analysis) which impairs your ability to get good warnings about
things like using uninitialized values.
-->
除了笨拙之外，这种模式还会扰乱编译器的明确赋值分析，这会削弱您对使用未初始化值等事情获得良好警告的能力。

<!-- 
Go also takes aim at this problem with nicer syntax, thanks to multi-valued returns: 
-->
由于多值返回，Go还可以通过更好的语法来解决这个问题：

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
因此，呼叫者更加清洁。 结合早期的单行功能，如果检查错误 - 一个微妙的扭曲，因为乍一看价值回报不在范围内，但它是 - 这得到更好的触摸：
<!-- 
    if value, err := foo(); err != nil {
        // Error!  Deal with it.
    } else {
        // Use value ...
    } 
-->
    if value, err := foo(); err != nil {
        // 错误！ 处理它。
    } else {
        // 使用值 ...
    } 

<!-- 
Notice that this also helps to remind you to check the error.  It's not bulletproof, however, because functions can
return an error and nothing else, at which point forgetting to check it is just as easy as it is in C. 
-->
请注意，这也有助于提醒您检查错误。 
然而，它不是防弹的，因为函数可以返回错误而没有别的，此时忘记检查它就像在C中一样容易。

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
正如我上面提到的，有些人会在可用性方面反对我。特别是Go的设计师，我怀疑。使用错误代码对Go的一个巨大吸引力是对当今景观中过于复杂的语言的反叛。我们已经失去了很多让C如此优雅的东西 - 你通常可以查看任何代码行并猜测它转换成的机器代码。我不反对这些观点。事实上，我非常喜欢Go的模型而不是未经检查的异常和Java的已检查异常的化身。即使在我写这篇文章的时候，最近写了很多Go，我看看Go的简单性和奇迹，我们对所有的尝试和需求都走得太远等等你很快就会看到？我不确定。 Go的错误模型往往是该语言最具分裂性的方面之一;它可能很大程度上是因为你不能像大多数语言那样马虎，但程序员确实喜欢在Midori中编写代码。最后，很难比较它们。我确信两者都可以用来编写可靠的代码。

<!-- 
##### Return Values in Data Structures 
-->
##### 数据结构中的返回值

<!-- 
Functional languages address many of the usability challenges by packaging up the possibility of *either* a value
*or* an error, into a single data structure.  Because you're forced to pick apart the error from the value if you want
to do anything useful with the value at the callsite -- which, thanks to a dataflow style of programming, you probably
will -- it's easy to avoid the killer problem of forgetting to check for errors. 
-->
功能语言通过将值或错误的可能性打包到单个数据结构中来解决许多可用性挑战。 因为如果你想对调用点上的值做任何有用的事情，你就不得不从值中挑出错误 - 这要归功于数据流编程风格，你可能会 - 很容易避免遗忘的杀手问题 检查错误。

<!-- 
For an example of a modern take on this, check out [Scala's `Option` type](
http://danielwestheide.com/blog/2012/12/19/the-neophytes-guide-to-scala-part-5-the-option-type.html).  The unfortunate
news is that some languages, like those in the ML family and even Scala (thanks to its JVM heritage), mix this elegant
model with the world of unchecked exceptions.  This taints the elegance of the monadic data structure approach. 
-->
有关此现代版本的示例，请查看Scala的Option类型。 不幸的消息是，有些语言，比如ML系列中的语言，甚至是Scala（由于其JVM传统），将这种优雅的模型与未经检查的例外世界混合在一起。 这玷污了monadic数据结构方法的优雅。

<!-- 
Haskell does something even cooler and [gives the illusion of exception handling while still using error values and
local control flow](https://wiki.haskell.org/Exception): 
-->
Haskell做了更酷的事情并且在仍然使用错误值和本地控制流时给出了异常处理的错觉：

<!-- 
> There is an old dispute between C++ programmers on whether exceptions or error return codes are the right way.  Niklas
> Wirth considered exceptions to be the reincarnation of GOTO and thus omitted them in his languages.  Haskell solves
> this problem a diplomatic way: Functions return error codes, but the handling of error codes does not uglify the code. 
-->

> C ++程序员之间就异常或错误返回代码是否正确的方式存在争议。 Niklas Wirth认为例外是GOTO的再生，因此在他的语言中省略了它们。 Haskell以外交方式解决了这个问题：函数返回错误代码，但错误代码的处理不会使代码失效。

<!-- 
The trick here is to support all the familiar `throw` and `catch` patterns, but using monads rather than control flow. 
-->
这里的技巧是支持所有熟悉的抛出和捕获模式，但使用monad而不是控制流。

<!-- 
Although [Rust also uses error codes](https://doc.rust-lang.org/book/error-handling.html) it is also in the style of
the functional error types.  For example, imagine we are writing a function named `bar` in Go: we'd like to call `foo`,
and then simply propagate the error to our caller if it fails: 
-->
虽然Rust也使用错误代码，但它也是功能错误类型的样式。 例如，假设我们在Go中编写了一个名为bar的函数：我们想调用foo，然后只是在错误时将错误传播给调用者：

    func bar() error {
        if value, err := foo(); err != nil {
            return err
        } else {
            // 使用值 ...
        }
    } 


<!-- 
The longhand version in Rust isn't any more concise.  It might, however, send C programmers reeling with its foreign
pattern matching syntax (a real concern but not a dealbreaker).  Anyone comfortable with functional programming,
however, probably won't even blink, and this approach certainly serves as a reminder to deal with your errors: 
-->
Rust中的速记版本不再简洁。 但是，它可能会让C程序员发现其外来模式匹配语法（一个真正的问题，但不是一个交易破坏者）。 然而，任何对功能编程感到满意的人都可能甚至不会眨眼，这种方法当然可以提醒您处理错误：

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
            Ok(value) => /* 使用值 ... */,
            Err(err) => return Err(err)
        }
    } 

<!-- 
But it gets better.  Rust has a [`try!` macro](https://doc.rust-lang.org/std/macro.try!.html) that reduces boilerplate
like the most recent example to a single expression: 
-->
但它变得更好。 Rust尝试过！ 将最新示例的样板减少到单个表达式的宏：

<!-- 
    fn bar() -> Result<(), Error> {
        let value = try!(foo);
        // Use value ...
    } 
-->
    fn bar() -> Result<(), Error> {
        let value = try!(foo);
        // 使用值 ...
    } 

<!-- 
This leads us to a beautiful sweet spot.  It does suffer from the performance problems I mentioned earlier, but does
very well on all other dimensions.  It alone is an incomplete picture -- for that, we need to cover fail-fast (a.k.a.
abandonment) -- but as we will see, it's far better than any other exception-based model in widespread use today. 
-->
这将我们带到一个美丽的甜蜜点。 它确实遭受了我之前提到的性能问题，但在所有其他方面都做得很好。 仅此一点是不完整的图片 - 为此，我们需要覆盖失败的快速（放弃放弃） - 但正如我们将看到的，它远远优于当今广泛使用的任何其他基于异常的模型。

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
例外的历史令人着迷。 在这段旅程中，我花了无数个小时来回顾行业的步伐。 这包括阅读一些原始论文 - 如Goodenough的1975年经典，异常处理：问题和建议的符号 - 除了查看几种语言的方法：Ada，Eiffel，Modula-2和3，ML，以及，最灵感 ，CLU。 许多论文比我总结漫长而艰辛的旅程做得更好，所以我不会在这里做。 相反，我将专注于哪些有效，哪些无法用于构建可靠的系统。

<!-- 
Reliability is the most important of our requirements above when developing the Error Model.  If you can't react
appropriately to failures, your system, by definition, won't be very reliable.  Operating systems generally speaking need
to be reliable.  Sadly, the most commonplace model -- unchecked exceptions -- is the worst you can do in this dimension. 
-->
在开发错误模型时，可靠性是我们上述要求中最重要的。 如果您无法对故障做出适当的反应，那么根据定义，您的系统将不会非常可靠。 通常说操作系统需要可靠。 可悲的是，最常见的模型 - 未经检查的例外 - 是您在此维度中可以做的最差的。

<!-- 
For these reasons, most reliable systems use return codes instead of exceptions.  They make it possible to locally
reason about and decide how best to react to error conditions.  But I'm getting ahead of myself.  Let's dig in. 
-->
在开发错误模型时，可靠性是我们上述要求中最重要的。 如果您无法对故障做出适当的反应，那么根据定义，您的系统将不会非常可靠。 通常说操作系统需要可靠。 可悲的是，最常见的模型 - 未经检查的例外 - 是您在此维度中可以做的最差的。

<!-- 
### Unchecked Exceptions 
-->
### 未经检查的例外情况

<!-- 
A quick recap.  In an unchecked exceptions model, you `throw` and `catch` exceptions, without it being part of the type
system or a function's signature.  For example: 
-->
快速回顾一下。 在未经检查的异常模型中，您抛出并捕获异常，而不是类型系统或函数签名的一部分。 例如：

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
            // 处理错误。
        }
    }

    // Baz也调用Foo，但不处理该异常：
    void Baz() {
        Foo(); // 让错误逃到我们的呼叫者。
    } 

<!-- 
In this model, any function call -- and sometimes any *statement* -- can throw an exception, transferring control
non-locally somewhere else.  Where?  Who knows.  There are no annotations or type system artifacts to guide your
analysis.  As a result, it's difficult for anyone to reason about a program's state at the time of the throw, the state
changes that occur while that exception is propagated up the call stack -- and possibly across threads in a concurrent
program -- and the resulting state by the time it gets caught or goes unhandled. 
-->
在这个模型中，任何函数调用 - 有时是任何语句 - 都可以抛出异常，在其他地方非本地传递控制。 哪里？ 谁知道。 没有注释或类型系统工件来指导您的分析。 因此，任何人都很难在抛出时推断程序的状态，当异常在调用堆栈中传播时可能会发生状态更改 - 并且可能跨并发程序中的线程 - 以及由此产生的状态 它被捕获或未处理的时间。

<!-- 
It's of course possible to try.  Doing so requires reading API documentation, doing manual audits of the code, leaning
heavily on code reviews, and a healthy dose of luck.  The language isn't helping you out one bit here.  Because failures
are rare, this tends not to be as utterly disastrous as it sounds.  My conclusion is that's why many people in the industry
think unchecked exceptions are "good enough."  They stay out of your way for the common success paths and, because most
people don't write robust error handling code in non-systems programs, throwing an exception *usually* gets you out of a
pickle fast.  Catching and then proceeding often works too.  No harm, no foul.  Statistically speaking, programs "work." 
-->
这当然可以尝试。 这样做需要阅读API文档，对代码进行手动审计，严重依赖代码审查以及健康的运气。 这里的语言没有帮助你。 因为失败是罕见的，所以这并不像听起来那样完全是灾难性的。 我的结论是，这就是为什么业内很多人认为未经检查的异常“足够好”。他们不会为了共同的成功路径而烦恼，因为大多数人都不会在非系统程序中编写强大的错误处理代码，抛出 一个例外通常会让你快速退出泡菜。 捕捉然后继续进行通常也有效。 没有伤害，没有犯规。 从统计上讲，节目“有效”。

<!-- 
Maybe statistical correctness is okay for scripting languages, but for the lowest levels of an operating system, or any
mission critical application or service, this isn't an appropriate solution.  I hope this isn't controversial. 
-->
也许统计正确性对于脚本语言是可行的，但对于操作系统的最低级别，或任何关键任务应用程序或服务，这不是一个合适的解决方案。 我希望这不是有争议的。

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
由于异步异常，.NET使情况更糟。 C ++也有所谓的“异步异常”：这些是由硬件故障触发的故障，例如访问冲突。 然而，它在.NET中变得非常讨厌。 任意线程几乎可以在代码中的任何一点注入失败。 即使在任务的RHS和LHS之间！ 因此，源代码中看起来像原子的东西不是。 我在10年前写过这篇文章并且挑战仍然存在，尽管风险已经减弱，因为.NET普遍认为线程中止是有问题的。 新的CoreCLR甚至缺少AppDomains，新的ASP.NET Core 1.0堆栈肯定不会像过去那样使用线程中止。 但API仍然存在。

<!-- 
There's a famous interview with Anders Hejlsberg, C#'s chief designer, called [The Trouble with Checked Exceptions](
http://www.artima.com/intv/handcuffs.html).  From a systems programmer's perspective, much of it leaves you scratching
your head.  No statement affirms that the target customer for C# was the rapid application developer more than this: 
-->
有一个着名的采访是C＃的首席设计师Anders Hejlsberg，名为The Trouble with Checked Exceptions。 从系统程序员的角度来看，大部分都会让你挠头。 没有声明确认C＃的目标客户是快速应用程序开发人员而不是：

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
> Bill Venners：但是你不是在这种情况下打破他们的代码，即使是没有经过检查的例外的语言吗？ 如果foo的新版本将抛出一个客户应该考虑处理的新异常，那么他们的代码是不是因为他们在编写代码时没有预料到异常？

> Anders Hejlsberg：不，因为在很多情况下，人们并不关心。 他们不会处理任何这些例外情况。 消息循环周围有一个底层异常处理程序。 那个处理程序只是打开一个对话框，说明出了什么问题并继续。 程序员通过在任何地方编写try finally来保护他们的代码，因此如果发生异常他们将正确退出，但他们实际上并不感兴趣处理异常。

<!-- 
This reminds me of `On Error Resume Next` in Visual Basic, and the way Windows Forms automatically caught and swallowed
errors thrown by the application, and attempted to proceed.  I'm not blaming Anders for his viewpoint here; heck, for
C#'s wild popularity, I'm convinced it was the right call given the climate at the time.   But this sure isn't the way
to write operating system code. 
-->
这让我想起了Visual Basic中的On Error Resume Next，以及Windows Forms自动捕获并吞下应用程序抛出的错误并尝试继续的方式。 我不是在这里责怪安德斯的观点; 哎呀，对于C＃的大受欢迎，我确信这是当时气候的正确选择。 但这肯定不是编写操作系统代码的方法。

<!-- 
C++ at least *tried* to offer something better than unchecked exceptions with its [`throw` exception specifications](
http://en.cppreference.com/w/cpp/language/except_spec).  Unfortunately, the feature relied on dynamic enforcement which
sounded its death knell instantaneously. 
-->
C ++至少尝试使用throw异常规范提供比未经检查的异常更好的东西。 不幸的是，这个功能依赖于动态执法，瞬间敲响了它的丧钟。

<!-- 
If I write a function `void f() throw(SomeError)`, the body of `f` is still free to invoke functions that throw things
other than `SomeError`.  Similarly, if I state that `f` throws no exceptions, using `void f() throw()`, it's still
possible to invoke things that throw.  To implement the stated contract, therefore, the compiler and runtime must ensure
that, should this happen, `std::unexpected` is called to rip the process down in response.
-->
如果我写一个函数void f（）throw（SomeError），f的主体仍然可以自由地调用抛出SomeError之外的函数。 类似地，如果我声明f抛出没有异常，使用void f（）throw（），它仍然可以调用抛出的东西。 因此，为了实现所述的合同，编译器和运行时必须确保，如果发生这种情况，则调用std :: unexpected来响应进程。

<!-- 
I'm not the only person to recognize this design was a mistake.  Indeed, `throw` is now deprecated.  A detailed WG21
paper, [Deprecating Exception Specifications](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2010/n3051.html),
describes how C++ ended up here, and has this to offer in its opening statement: 
-->
我不是唯一认识到这种设计是错误的人。 实际上，throw现已弃用。 详细的WG21论文，弃用例外规范，描述了C ++如何在这里结束，并在其开场陈述中提供：

<!-- 
> Exception specifications have proven close to worthless in practice, while adding a measurable overhead to programs. 
-->
> 事实证明，异常规范在实践中几乎毫无价值，同时为程序增加了可衡量的开销。

<!-- 
The authors list three reasons for deprecating `throw`.  Two of the three reasons were a result of the dynamic choice:
runtime checking (and its associated opaque failure mode) and runtime performance overheads.  The third reason, lack
of composition in generic code, could have been dealt with using a proper type system (admittedly at an expense). 
-->
作者列举了弃用throw的三个原因。 三个原因中的两个是动态选择的结果：运行时检查（及其关联的不透明故障模式）和运行时性能开销。 第三个原因是，通用代码中缺乏组成，可以使用适当的类型系统来处理（诚然需要付费）。

<!-- 
But the worst part is that the cure relies on yet another dynamically enforced construct -- the [`noexcept` specifier](
http://en.cppreference.com/w/cpp/language/noexcept_spec) -- which, in my opinion, is just as bad as the disease. 
-->
但最糟糕的是，治疗依赖于另一个动态强制的构造 -  noexcept说明符 - 在我看来，它与疾病一样糟糕。

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
“异常安全”是C ++社区中经常讨论的一种做法。这种方法巧妙地分类了函数如何从调用者的角度对故障，状态转换和内存管理的行为进行分类。功能属于四种类型之一：无投掷意味着前进进度得到保证，不会出现异常;强大的安全意味着状态转换以原子方式发生，而失败不会留下部分承诺的状态或破坏的不变量;基本安全意味着，虽然函数可能部分地提交状态更改，但不会破坏不变量并防止泄漏;最后，没有安全意味着一切皆有可能。这种分类法非常有用，我鼓励任何人对错误行为采取有意和严谨的态度，无论是使用这种方法还是类似的方法。即使您使用的是错误代码。问题是，在使用未经检查的异常的系统中遵循这些指南基本上是不可能的，除了叶节点数据结构调用一组小且易于审计的其他函数。试想一下：为了保证各地的安全性，您需要考虑所有函数调用的可能性，并相应地保护周围的代码。这或者意味着防御性编程，信任另一个函数的文档英文散文（没有被计算机检查），变得幸运，只调用noexcept函数，或者只是希望最好。感谢RAII，基本安全性的泄漏自由方面更容易实现 - 而且由于智能指针，这些天很常见 - 但即使破坏不变量也很难防止。 “异常处理：虚假的安全感”一文很好地总结了这一点。

<!-- 
For C++, the real solution is easy to predict, and rather straightforward: for robust systems programs, don't use
exceptions.  That's the approach [Embedded C++](https://en.wikipedia.org/wiki/Embedded_C%2B%2B) takes, in addition to
numerous realtime and mission critical guidelines for C++, including NASA's Jet Propulsion Laboratory's.
[C++ on Mars sure ain't using exceptions anytime soon](https://www.youtube.com/watch?v=3SdSKZFoUa8). 
-->
对于C ++，真正的解决方案很容易预测，而且非常简单：对于健壮的系统程序，不要使用异常。 这是嵌入式C ++的方法，除了C ++的许多实时和关键任务指南，包括NASA的喷气推进实验室。 火星上的C ++肯定不会很快使用异常。

<!-- 
So if you can safely avoid exceptions and stick to C-like return codes in C++, what's the beef? -->
因此，如果您可以安全地避免异常并坚持使用C ++中的C类返回代码，那么牛肉是什么？

<!-- 
The entire C++ ecosystem uses exceptions.  To obey the above guidance, you must avoid significant parts of the language
and, it turns out, significant chunks of the library ecosystem.  Want to use the Standard Template Library?  Too bad, it
uses exceptions.  Want to use Boost?  Too bad, it uses exceptions.  Your allocator likely throws `bad_alloc`.  And so
on.  This even causes insanity like people creating forks of existing libraries that eradicates exceptions.  The Windows
kernel, for instance, has its own fork of the STL that doesn't use exceptions.  This bifurcation of the ecosystem is
neither pleasant nor practical to sustain. 
-->
整个C ++生态系统使用异常。 要遵守上述指导原则，您必须避免使用该语言的重要部分，并且事实证明，这是图书馆生态系统的重要组成部分。 想使用标准模板库？ 太糟糕了，它使用异常。 想要使用Boost？ 太糟糕了，它使用异常。 您的分配器可能会抛出bad_alloc。 等等。 这甚至会导致疯狂，就像人们创建现有图书馆的叉子一样，可以消除异常。 例如，Windows内核有自己的STL分支，它不使用异常。 这种生态系统的分叉既不愉快也不实用。

<!-- 
This mess puts us in a bad spot.  Especially because many languages use unchecked exceptions.  It's clear that they are
ill-suited for writing low-level, reliable systems code.  (I'm sure I will make a few C++ enemies by saying this so
bluntly.)  After writing code in Midori for years, it brings me tears to go back and write code that uses unchecked
exceptions; even simply code reviewing is torture.  But "thankfully" we have checked exceptions from Java to learn and
borrow from ... Right? 
-->
这场混乱让我们陷入了困境。 特别是因为许多语言使用未经检查的异常。 很明显，它们不适合编写低级，可靠的系统代码。 （我肯定会直截了当地说出这几个C ++的敌人。）在Midori编写代码多年后，它让我流下眼泪回去编写使用未经检查的异常的代码; 即使简单的代码审查也是折磨。 但“幸运的是”我们已经检查了Java的异常以学习和借用......对吗？

<!-- 
### Checked Exceptions 
-->
### 已检查的异常
<!-- 
Ah, checked exceptions.  The rag doll that nearly every Java programmer, and every person who's observed Java from an
arm's length distance, likes to beat on.  Unfairly so, in my opinion, when you compare it to the unchecked exceptions
mess. 
-->
啊，检查异常。 几乎每个Java程序员以及每个从手臂长度距离观察Java的人都喜欢打败它。 在我看来，当你将它与未经检查的异常混乱进行比较时，这是不公平的。

<!-- 
In Java, you know *mostly* what a method might throw, because a method must say so: 
-->
在Java中，你知道*大多数*方法可能会抛出什么，因为方法必须这样说：

    void foo() throws FooException, BarException {
        ...
    } 


<!-- 
Now a caller knows that invoking `foo` could result in either `FooException` or `BarException` being thrown.  At
callsites a programmer must now decide: 1) propagate thrown exceptions as-is, 2) catch and deal with them, or 3) somehow
transform the type of exception being thrown (possibly even "forgetting" the type altogether).  For instance: 
-->
现在调用者知道调用`foo`可能导致抛出`FooException`或BarException`。 在callites中，程序员现在必须决定：1）按原样传播抛出的异常，2）捕获并处理它们，或者3）以某种方式转换抛出的异常类型（甚至可能“完全忘记”类型）。 例如：

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

    // 2) 抓住并处理它们：
    void bar() {
        try {
            foo();
        }
        catch (FooException e) {
            // 处理FooException错误条件。
        }
        catch (BarException e) {
            // 处理BarException错误条件。
        }
    }

    // 3) 转换抛出的异常类型：
    void bar() throws Exception {
        foo();
    } 

<!-- 
This is getting much closer to something we can use.  But it fails on a few accounts: 
-->
这与我们可以使用的东西越来越接近了。 但它在一些帐户上失败了：

<!-- 
1. Exceptions are used to communicate unrecoverable bugs, like null dereferences, divide-by-zero, etc.

2. You don't actually know *everything* that might be thrown, thanks to our little friend `RuntimeException`.  Because
   Java uses exceptions for all error conditions -- even bugs, per above -- the designers realized people would go mad
   with all those exception specifications.  And so they introduced a kind of exception that is unchecked.  That is, a
   method can throw it without declaring it, and so callers can invoke it seamlessly.

3. Although signatures declare exception types, there is no indication at callsites what calls might throw.

4. People hate them. 
-->
1. 异常用于传达不可恢复的错误，如空取消引用，除零等。
2. 由于我们的小朋友RuntimeException，你实际上并不知道可能抛出的所有内容。 因为Java使用了所有错误条件的异常 - 甚至是上面的错误 - 设计师意识到人们会对所有这些异常规范感到厌烦。 因此他们引入了一种未经检查的异常。 也就是说，一个方法可以抛出它而不声明它，因此调用者可以无缝地调用它。
3. 虽然签名声明了异常类型，但是没有迹象表明调用可能会抛出什么调用。
4. 人们讨厌他们。

<!-- 
That last one is interesting, and I shall return to it later when describing the approach Midori took.  In summary,
peoples' distaste for checked exceptions in Java is largely derived from, or at least significantly reinforced by, the
other three bullets above.  The resulting model seems to be the worst of both worlds.  It doesn't help you to write
bulletproof code and it's hard to use.  You end up writing down a lot of gibberish in your code for little perceived
benefit.  And versioning your interfaces is a pain in the ass.  As we'll see later, we can do better. 
-->
最后一个很有趣，稍后我将在描述Midori采用的方法时再回过头来。 总而言之，人们对Java中检查异常的厌恶主要源于上面其他三个子弹，或者至少得到了其他三个子弹的强化。 由此产生的模型似乎是两个世界中最糟糕的。 它无法帮助您编写防弹代码，并且很难使用。 你最终在你的代码中写下了很多乱码，几乎没有什么好处。 对你的界面进行版本控制是一件很痛苦的事。 我们稍后会看到，我们可以做得更好。

<!-- 
That versioning point is worth a ponder.  If you stick to a single kind of `throw`, then the versioning problem is no
worse than error codes.  Either a function fails or it doesn't.  It's true that if you design version 1 of your API to
have no failure mode, and then want to add failures in version 2, you're screwed.  As you should be, in my opinion.  An
API's failure mode is a critical part of its design and contract with callers.  Just as you wouldn't change the return
type of an API silently without callers needing to know, you shouldn't change its failure mode in a semantically
meaningful way.  More on this controversial point later on. 
-->
该版本控制点值得深思。 如果您坚持使用单一类型的抛出，那么版本控制问题并不比错误代码更糟糕。 功能失败或不失败。 确实，如果您将API的第1版设计为没有故障模式，然后想要在版本2中添加故障，那么您就搞砸了。 在我看来，就像你应该的那样。 API的故障模式是其设计和与呼叫者签订合同的关键部分。 正如您不会在没有调用者需要知道的情况下静默更改API的返回类型一样，您不应该以语义上有意义的方式更改其失败模式。 稍后将讨论这个有争议的问题。

<!-- 
CLU has an interesting approach, as described in this crooked and wobbly scan of a 1979 paper by Barbara Liskov,
[Exception Handling in CLU](http://csg.csail.mit.edu/pubs/memos/Memo-155/Memo-155-3.pdf).  Notice that they focus a lot
on "linguistics"; in other words, they wanted a language that people would love.  The need to check and repropagate all
errors at callsites felt a lot more like return values, yet the programming model had that richer and slightly
declarative feel of what we now know as exceptions.  And most importantly, `signal`s (their name for `throw`) were
checked.  There were also convenient ways to terminate the program should an unexpected `signal` occur. 
-->
CLU有一个有趣的方法，正如Barbara Liskov在1979年CLU中的异常处理这篇文章中所描述的那种弯曲和摇摆不定的扫描所描述的那样。 请注意，他们非常关注“语言学”; 换句话说，他们想要一种人们会喜欢的语言。 在callites上检查和重新传播所有错误的需要感觉更像是返回值，但编程模型对我们现在所知的异常有更丰富和略微声明的感觉。 最重要的是，检查了信号（它们的名字）。 如果发生意外信号，还有方便的方法来终止程序。

<!-- 
### Universal Problems with Exceptions 
-->
### 异常的普遍问题

<!-- 
Most exception systems get a few major things wrong, regardless of whether they are checked or unchecked. 
-->
大多数异常系统都会出现一些重大错误，无论它们是经过检查还是未经检查。

<!-- 
First, throwing an exception is usually ridiculously expensive.  This is almost always due to the gathering of a stack
trace.  In managed systems, gathering a stack trace also requires groveling metadata, to create strings of function
symbol names.  If the error is caught and handled, however, you don't even need that information at runtime!
Diagnostics are better implemented in the logging and diagnostics infrastructure, not the exception system itself.  The
concerns are orthogonal.  Although, to really nail the diagnostics requirement above, *something* needs to be able to
recover stack traces; never underestimate the power of `printf` debugging and how important stack traces are to it. 
-->
首先，抛出异常通常是非常昂贵的。 这几乎总是由于堆栈跟踪的收集。 在受管系统中，收集堆栈跟踪还需要groveling元数据，以创建函数符号名称字符串。 但是，如果捕获并处理了错误，您甚至不需要在运行时获取该信息！ 诊断更好地在日志记录和诊断基础结构中实现，而不是异常系统本身。 担忧是正交的。 虽然，为了真正确定上述诊断要求，某些东西需要能够恢复堆栈跟踪; 永远不要低估printf调试的强大功能以及堆栈跟踪对它的重要性。

<!-- 
Next, exceptions can significantly impair code quality.  I touched on this topic [in my last post](
http://joeduffyblog.com/2015/12/19/safe-native-code/), and there are [good papers on the topic in the context of C++](
http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.116.8337&rep=rep1&type=pdf).  Not having static type system
information makes it hard to model control flow in the compiler, which leads to overly conservative optimizers. 
-->
接下来，异常会严重影响代码质量。 我在上一篇文章中谈到了这个主题，并且在C ++的上下文中有关于该主题的很好的论文。 没有静态类型系统信息使得很难在编译器中建模控制流，这导致过于保守的优化器。

<!-- 
Another thing most exception systems get wrong is encouraging too coarse a granularity of handling errors.  Proponents
of return codes love that error handling is localized to a specific function call.  (I do too!)  In exception handling
systems, it's all too easy to slap a coarse-grained `try`/`catch` block around some huge hunk of code, without carefully
reacting to individual failures.  That produces brittle code that's almost certainly wrong; if not today, then after the
inevitable refactoring that will occur down the road.  A lot of this has to do with having the right syntaxes. 
-->
大多数异常系统出错的另一个原因是鼓励处理错误的粒度过于粗糙。 返回码的支持者喜欢将错误处理本地化为特定的函数调用。 （我也是这样！）在异常处理系统中，很容易在一些巨大的代码周围打一个粗粒度的try / catch块，而不会仔细地对单个故障做出反应。 这产生了几乎肯定错误的脆弱代码; 如果不是今天，那么在不可避免的重构之后将会发生在路上。 这很大程度上与拥有正确的语法有关。

<!-- 
Finally, control flow for `throw`s is usually invisible.  Even with Java, where you annotate method signatures, it's not
possible to audit a body of code and see precisely where exceptions come from.  Silent control flow is just as bad as
`goto`, or `setjmp`/`longjmp`, and makes writing reliable code very difficult. 
-->
最后，投掷的控制流程通常是不可见的。 即使使用Java来注释方法签名，也无法审核代码体并准确查看异常的来源。 静默控制流程与goto或setjmp / longjmp一样糟糕，并且使编写可靠代码变得非常困难。

<!-- 
## Where Are We? 
-->
## 我们在哪?

<!-- 
Before moving on, let's recap where we are: 
-->
在继续之前，让我们回顾一下我们所处的位置：

![Good-Bad-Ugly](/assets/img/2016-02-07-the-error-model-1.png)

<!-- 
Wouldn't it be great if we could take all of The Goods and leave out The Bads and The Uglies? 
-->
如果我们可以拿走所有的货物并遗漏The Bads和The Uglies，那不是很好吗？

<!-- 
This alone would be a great step forward.  But it's insufficient.  This leads me to our first big "ah-hah" moment that
shaped everything to come.  For a significant class of error, *none* of these approaches are appropriate! 
-->
仅这一点就是向前迈出的一大步。 但这还不够。 这让我想起了第一个影响未来一切的“啊哈”时刻。 对于一大类错误，这些方法都不合适！

<!-- 
# Bugs Aren't Recoverable Errors! 
-->
# Bugs不是可恢复的错误！

<!-- 
A critical distinction we made early on is the difference between recoverable errors and bugs: -->
我们早期做出的一个重要区别是可恢复错误和错误之间的区别：

<!-- 
* A *recoverable error* is usually the result of programmatic data validation.  Some code has examined the state of the
  world and deemed the situation unacceptable for progress.  Maybe it's some markup text being parsed, user input from a
  website, or a transient network connection failure.  In these cases, programs are expected to recover.  The developer
  who wrote this code must think about what to do in the event of failure because it will happen in well-constructed
  programs no matter what you do.  The response might be to communicate the situation to an end-user, retry, or abandon
  the operation entirely, however it is a *predictable* and, frequently, *planned* situation, despite being called an
  "error." 
-->
* 可恢复的错误通常是程序化数据验证的结果。一些代码检查了世界的状况，并认为这种情况对进展是不可接受的。也许它是一些正在解析的标记文本，来自网站的用户输入或瞬态网络连接故障。在这些情况下，预计计划将恢复。编写此代码的开发人员必须考虑在发生故障时该怎么做，因为无论您做什么，它都会在构造良好的程序中发生。响应可能是将情况传达给最终用户，重试或完全放弃操作，但这是一种可预测的，经常是计划的情况，尽管被称为“错误”。


<!-- 
* A *bug* is a kind of error the programmer didn't expect.  Inputs weren't validated correctly, logic was written wrong,
  or any host of problems have arisen.  Such problems often aren't even detected promptly; it takes a while until
  "secondary effects" are observed indirectly, at which point significant damage to the program's state might have
  occurred.  Because the developer didn't expect this to happen, all bets are off.  All data structures reachable by
  this code are now suspect.  And because these problems aren't necessarily detected promptly, in fact, a whole lot
  more is suspect.  Depending on the isolation guarantees of your language, perhaps the entire process is tainted. 
-->
* 错误是程序员没想到的一种错误。输入未正确验证，逻辑写错，或出现任何问题。这些问题通常甚至不能及时发现;需要一段时间才能间接观察到“次要影响”，此时可能会对程序状态造成重大损害。因为开发人员没想到会发生这种情况，所以所有的赌注都会被取消。此代码可访问的所有数据结构现在都是可疑的。而且因为这些问题不一定能及时发现，事实上，更多的是怀疑。根据您的语言的隔离保证，可能整个过程都受到污染。

<!-- 
This distinction is paramount.  Surprisingly, most systems don't make one, at least not in a principled way!  As we saw
above, Java, C#, and dynamic languages just use exceptions for everything; and C and Go use return codes.  C++ uses a
mixture depending on the audience, but the usual story is a project picks a single one and uses it everywhere.  You
usually don't hear of languages suggesting *two* different techniques for error handling, however. 
-->
这种区别至关重要。 令人惊讶的是，大多数系统都没有制造，至少不是原则性的！ 如上所述，Java，C＃和动态语言只使用异常; 和C和Go使用返回码。 C ++根据受众使用混合物，但通常的故事是项目选择一个并在任何地方使用它。 但是，您通常不会听到语言提示两种不同的错误处理技术。

<!-- 
Given that bugs are inherently not recoverable, we made no attempt to try.  All bugs detected at runtime caused
something called *abandonment*, which was Midori's term for something otherwise known as ["fail-fast"](
https://en.wikipedia.org/wiki/Fail-fast). 
-->
鉴于错误本质上是不可恢复的，我们没有尝试尝试。 在运行时检测到的所有错误都会导致称为放弃的东西，这是Midori的术语，也就是所谓的“失败快速”。

<!-- 
Each of the above systems offers abandonment-like mechanisms.  C# has `Environment.FailFast`; C++ has `std::terminate`;
Go has `panic`; Rust has `panic!`; and so on.  Each rips down the surrounding context abruptly and promptly.  The scope
of this context depends on the system -- for example, C# and C++ terminate the process, Go the current Goroutine, and
Rust the current thread, optionally with a panic handler attached to salvage the process.
-->
上述每个系统都提供类似放弃的机制。 C＃有Environment.FailFast; C ++有std :: terminate; Go有恐慌; Rust有恐慌！ 等等。 每一个都突然迅速地摧毁周围的环境。 此上下文的范围取决于系统 - 例如，C＃和C ++终止进程，转到当前Goroutine，并生成当前线程，可选择附加一个恐慌处理程序来抢救进程。

<!--
Although we did use abandonment in a more disciplined and ubiquitous way than is common, we certainly weren't the first
to recognize this pattern.  This [Haskell essay](https://wiki.haskell.org/Error_vs._Exception), articulates this
distinction quite well: 
-->
虽然我们确实使用放弃的方式比一般的更有纪律和无处不在，但我们当然不是第一个认识到这种模式的人。 这篇Haskell论文非常清楚地表达了这种区别：

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
> 我参与了用C ++编写的库的开发。其中一位开发人员告诉我，开发人员分为喜欢异常的人和喜欢返回代码的其他人。在我看来，返回代码的朋友赢了。但是，我得到的印象是他们辩论错误的观点：例外和返回代码同样具有表现力，但它们不应用于描述错误。实际上，返回码包含ARRAY_INDEX_OUT_OF_RANGE等定义。但我想知道：当它从子程序获得这个返回代码时，我的函数将如何反应？它应该向程序员发送邮件吗？它可以依次将此代码返回给其调用者，但它也不知道如何处理它。更糟糕的是，由于我不能对函数的实现做出假设，我不得不期望每个子例程都有一个ARRAY_INDEX_OUT_OF_RANGE。我的结论是ARRAY_INDEX_OUT_OF_RANGE是一个（编程）错误。它无法在运行时处理或修复，只能由其开发人员修复。因此，应该没有相应的返回代码，而应该有断言。

<!-- 
Abandoning fine grained mutable shared memory scopes is suspect -- like Goroutines or threads or whatever -- unless your
system somehow makes guarantees about the scope of the potential damage done.  However, it's great that these mechanisms
are there for us to use!  It means using an abandonment discipline in these languages is indeed possible. 
-->
放弃细粒度可变共享内存范围是可疑的 - 比如Goroutines或线程或其他 - 除非你的系统以某种方式保证潜在损坏的范围。 但是，我们可以使用这些机制！ 这意味着在这些语言中使用放弃纪律确实是可能的。
<!-- 
There are architectural elements necessary for this approach to succeed at scale, however.  I'm sure you're thinking
"If I tossed the entire process each time I had a null dereference in my C# program, I'd have some pretty pissed off
customers"; and, similarly, "That wouldn't be reliable at all!"  Reliability, it turns out, might not be what you think. 
-->
然而，这种方法必须具有大规模成功所必需的架构元素。 我相信你在想“如果我每次在我的C＃程序中取消引用时都会抛出整个过程，那么我会有一些非常生气的客户”; 同样地，“这根本不可靠！”事实证明，可靠性可能与您的想法不同。

<!-- 
# Reliability, Fault-Tolerance, and Isolation 
-->
# 可靠性, 容错和隔离

<!-- 
Before we get any further, we need to state a central belief: <s>Shi</s> Failure Happens. 
-->
在我们进一步讨论之前，我们需要说出一个中心信念：<s>Shi</s>失败发生。
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
常识是，通过系统地保证故障永远不会发生，您可以建立一个可靠的系统。 直观地说，这很有道理。 有一个问题：在极限情况下，这是不可能的。 如果您可以单独花费数百万美元购买此物业 - 就像许多关键任务，实时系统一样 - 那么您可以大大减少。 也许使用像SPARK这样的语言（一组基于合同的Ada扩展）来正式证明每行写的正确性。 然而，经验表明即使这种方法也不是万无一失的。

<!-- 
Rather than fighting this fact of life, we embraced it.  Obviously you try to eliminate failures where possible.  The
error model must make them transparent and easy to deal with.  But more importantly, you architect your system so that
the whole remains functional even when individual pieces fail, and then teach your system to recover those failing
pieces gracefully.  This is well known in distributed systems.  So why is it novel? 
-->
我们接受了它，而不是反对这一事实。 显然，你试图在可能的情况下消除失败。 错误模型必须使它们透明且易于处理。 但更重要的是，您构建系统，以便即使单个部件出现故障，整个系统仍然可以正常运行，然后教您的系统优雅地恢复那些失败的部分。 这在分布式系统中是众所周知的。 那为什么它是新颖的？

<!-- 
At the center of it all, an operating system is just a distributed network of cooperating processes, much like a
distributed cluster of microservices or the Internet itself.  The main differences include things like latency; what
levels of trust you can establish and how easily; and various assumptions about locations, identity, etc.  But failure
in [highly asynchronous, distributed, and I/O intensive systems](
http://joeduffyblog.com/2015/11/19/asynchronous-everything/) is just bound to happen.  My impression is that, largely
because of the continued success of monolithic kernels, the world at large hasn't yet made the leap to "operating system
as a distributed system" insight.  Once you do, however, a lot of design principles become apparent. 
-->
在这一切的中心，操作系统只是一个协作进程的分布式网络，就像微服务的分布式集群或互联网本身。 主要区别包括延迟; 你可以建立什么样的信任程度和容易程度; 关于位置，身份等的各种假设。但高度异步，分布式和I / O密集型系统的故障必然会发生。 我的印象是，很大程度上是因为单片内核的持续成功，整个世界还没有实现“作为分布式系统的操作系统”的洞察力。 但是，一旦你做了，很多设计原则就会变得明显。

<!-- 
As with most distributed systems, our architecture assumed process failure was inevitable.  We went to great
length to defend against cascading failures, journal regularly, and to enable restartability of programs and services. 
-->
与大多数分布式系统一样，我们的架构假设过程失败是不可避免的。 我们花了很长时间来防止级联故障，定期发布日志，以及实现程序和服务的可重启性。

<!-- 
You build things differently when you go in assuming this. 
-->
当你假设这个时，你会以不同的方式构建东西。

<!-- 
In particular, isolation is critical.  Midori's process model encouraged lightweight fine-grained isolation.  As a
result, programs and what would ordinarily be "threads" in modern operating systems were independent isolated entities.
Safeguarding against failure of one such connection is far easier than when sharing mutable state in an address space. 
-->
特别是隔离是至关重要的。 Midori的工艺模型鼓励轻量级细粒度隔离。 因此，程序和现代操作系统中通常为“线程”的程序是独立的孤立实体。 保护一个这样的连接失败比在地址空间中共享可变状态容易得多。

<!-- 
Isolation also encourages simplicity.  Butler Lampson's classic [Hints on Computer System Design](
http://research.microsoft.com/pubs/68221/acrobat.pdf) explores this topic.  And I always loved this quote from Hoare: 
-->
隔离也鼓励简单。 Butler Lampson关于计算机系统设计的经典提示探讨了这一主题。 我一直很喜欢霍尔的这句话：

<!-- 
> The unavoidable price of reliability is simplicity. (C. Hoare). 
-->
> 可靠的不可避免的价格是简单。（C. Hoare）。

<!-- 
By keeping programs broken into smaller pieces, each of which can fail or succeed on its own, the state machines within
them stay simpler.  As a result, recovering from failure is easier.  In our language, the points of possible failure
were explicit, further helping to keep those internal state machines correct, and pointing out those connections with
the messier outside world.  In this world, the price of individual failure is not nearly as dire.  I can't
over-emphasize this point.  None of the language features I describe later would have worked so well without this
architectural foundation of cheap and ever-present isolation. 
-->
通过将程序分解成更小的部分，每个部分都可以自行失败或成功，其中的状态机保持更简单。 因此，从故障中恢复更容易。 在我们的语言中，可能失败的要点是明确的，进一步帮助保持这些内部状态机正确，并指出与外部世界混乱的那些联系。 在这个世界上，个人失败的代价并不是那么可怕。 我不能过分强调这一点。 如果没有廉价和永远存在的隔离的架构基础，我后面描述的语言功能都不会很好。

<!-- 
Erlang has been very successful at building this property into the language in a fundamental way.  It, like Midori,
leverages lightweight processes connected by message passing, and encourages fault-tolerant architectures.  A common
pattern is the "supervisor," where some processes are responsible for watching and, in the event of failure, restarting
other processes.  [This article](http://ferd.ca/the-zen-of-erlang.html) does a terrific job articulating this philosophy
-- "let it crash" -- and recommended techniques for architecting reliable Erlang programs in practice. 
-->
Erlang非常成功地以一种基本的方式将这个属性构建到语言中。 它与Midori一样，利用通过消息传递连接的轻量级进程，并鼓励容错架构。 一种常见的模式是“主管”，其中一些进程负责观察，并且在发生故障时重新启动其他进程。 本文做了一个很棒的工作，明确了这个理念 - “让它崩溃” - 以及在实践中构建可靠的Erlang程序的推荐技术。

<!-- 
The key thing, then, is not preventing failure per se, but rather knowing how and when to deal with it. 
-->
因此，关键在于不是要防止失败本身，而是要知道如何以及何时处理它。

<!-- 
Once you've established this architecture, you beat the hell out of it to make sure it works.  For us, this meant
week-long stress runs, where processes would come and go, some due to failures, to ensure the system as a whole kept
making good forward progress.  This reminds me of systems like Netflix's [Chaos Monkey](
https://github.com/Netflix/SimianArmy/wiki/Chaos-Monkey) which just randomly kills entire machines in your cluster to
ensure the service as a whole stays healthy. 
-->
一旦你建立了这个架构，你就会打败它，以确保它的运行。 对我们来说，这意味着为期一周的压力运行，流程将来来去去，有些是由于失败，以确保整个系统保持良好的前进。 这让我想起了像Netflix的Chaos Monkey这样的系统，它会随机杀死集群中的整个机器，以确保整个服务保持健康。

<!-- 
I expect more of the world to adopt this philosophy as the shift to more distributed computing happens.  In a cluster of
microservices, for example, the failure of a single container is often handled seamlessly by the enclosing cluster
management software (Kubernetes, Amazon EC2 Container Service, Docker Swarm, etc).  As a result, what I describe in this
post is possibly helpful for writing more reliable Java, Node.js/JavaScript, Python, and even Ruby services.  The
unfortunate news is you're likely going to be fighting your languages to get there.  A lot of code in your process is
going to work real damn hard to keep limping along when something goes awry. 
-->
随着向更多分布式计算的转变发生，我期望更多的世界采用这种理念。 例如，在微服务集群中，单个容器的故障通常由封闭的集群管理软件（Kubernetes，Amazon EC2 Container Service，Docker Swarm等）无缝地处理。 因此，我在本文中描述的内容可能有助于编写更可靠的Java，Node.js / JavaScript，Python甚至Ruby服务。 不幸的消息是，你很可能会与你的语言作斗争来实现目标。 你的过程中的很多代码都会非常难以在出现问题时保持跛行。

<!-- 
## Abandonment 
-->
## 废弃

<!-- 
Even when processes are cheap and isolated and easy to recreate, it's still reasonable to think that abandoning an
entire process in the face of a bug is an overreaction.  Let me try to convince you otherwise. 
-->
即使流程便宜且孤立且易于重新创建，仍然有理由认为在面对错误时放弃整个流程是一种过度反应。 让我试着说服你。

<!-- 
Proceeding in the face of a bug is dangerous when you're trying to build a robust system.  If a programmer didn't expect
a given situation that's arisen, who knows whether the code will do the right thing anymore.  Critical data structures
may have been left behind in an incorrect state.  As an extreme (and possibly slightly silly) example, a routine that is
meant to round your numbers *down* for banking purposes might start rounding them *up*. 
-->
当您尝试构建一个健壮的系统时，面对错误进行是危险的。 如果程序员没有预料到会出现特定情况，谁知道代码是否会再做正确的事情了。 关键数据结构可能在不正确的状态下被遗忘。 作为一个极端（也可能是稍微愚蠢）的例子，一个旨在将您的数字用于银行业务的例程可能会开始四舍五入。

<!-- 
And you might be tempted to whittle down the granularity of abandonment to something smaller than a process.  But that's
tricky.  To take an example, imagine a thread in your process encounters a bug, and fails.  This bug might have been
triggered by some state stored in a static variable.  Even though some other thread might *appear* to have been
unaffected by the conditions leading to failure, you cannot make this conclusion.  Unless some property of your system
-- isolation in your language, isolation of the object root-sets exposed to independent threads, or something else --
it's safest to assume that anything other than tossing the entire address space out the window is risky and unreliable. 
-->
而且你可能会试图将放弃的粒度减少到比流程更小的东西。 但这很棘手。 举一个例子，假设你的进程中的一个线程遇到了一个bug，并且失败了。 该错误可能是由存储在静态变量中的某些状态触发的。 即使某些其他线程似乎不受导致失败的条件的影响，您也无法得出这个结论。 除非你的系统有一些属性 - 用你的语言隔离，隔离暴露给独立线程的对象根集，或者别的东西 - 最安全的做法是假设除了把整个地址空间抛到窗外之外的任何东西都是冒险和不可靠的。

<!-- 
Thanks to the lightweight nature of Midori processes, abandoning a process was more like abandoning a single thread in a
classical system than a whole process.  But our isolation model let us do this reliably. 
-->
由于Midori流程的轻量级特性，放弃流程更像是在经典系统中放弃单个线程而不是整个流程。 但我们的隔离模式让我们可靠地做到了这一点

<!-- 
I'll admit the scoping topic is a slippery slope.  Maybe all the data in the world has become corrupt, so how do you
know that tossing the process is even enough?!  There is an important distinction here.  Process state is transient by
design.  In a well designed system it can be thrown away and recreated on a whim.  It's true that a bug can corrupt
persistent state, but then you have a bigger problem on your hands -- a problem that must be dealt with differently. 
-->
我承认范围界定是一个滑坡。 也许世界上所有的数据都已经腐败了，所以你怎么知道抛弃这个过程就足够了？！ 这里有一个重要的区别。 过程状态是设计瞬态的。 在一个设计良好的系统中，它可以被抛弃并随意重建。 确实，一个bug可以破坏持久状态，但是你手上有一个更大的问题 - 一个必须以不同方式处理的问题。

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
对于某些背景，我们可以考虑容错系统设计。 放弃（快速失败）已经是该领域的常用技术，我们可以将我们对这些系统的大部分知识应用于普通程序和过程。 也许最重要的技术是定期记录和检查宝贵的持久状态。 吉姆格雷1985年的论文“为什么计算机停止以及可以做些什么？”很好地描述了这个概念。 随着程序继续迁移到云，并且积极地分解为更小的独立服务，瞬态和持久状态的这种明确分离甚至更为重要。 由于软件编写方式的这些转变，在现代架构中放弃比以前更容易实现。 实际上，放弃可以帮助您避免数据损坏，因为在下一个检查点之前检测到的错误可以防止错误的状态逃逸。

<!-- 
Bugs in Midori's kernel were handled differently.  A bug in the microkernel, for instance, is an entirely different
beast than a bug in a user-mode process.  The scope of possible damage was greater, and the safest response was to
abandon an entire "domain" (address space).  Thankfully, most of what you'd think of being classic "kernel"
functionality -- the scheduler, memory manager, filesystem, networking stack, and even device drivers -- was run
instead in isolated processes in user-mode where failures could be contained in the usual ways described above. 
-->
Midori内核中的错误处理方式不同。 例如，微内核中的错误与用户模式进程中的错误完全不同。 可能的损害范围更大，最安全的回应是放弃整个“域”（地址空间）。 值得庆幸的是，大多数你认为经典的“内核”功能 - 调度程序，内存管理器，文件系统，网络堆栈，甚至是设备驱动程序 - 都是在用户模式的隔离进程中运行的，其中故障可能包含在 通常的方式如上所述。

<!-- 
# Bugs: Abandonment, Assertions, and Contracts 
-->
# Bugs：放弃，断言和合约

<!-- 
A number of kinds of bugs in Midori might trigger abandonment: 
-->
Midori中的一些错误可能会导致放弃：

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
* 不正确的演员。
* 尝试取消引用“null”指针。
* 尝试访问其边界之外的数组。
* 除以零。
* 意外的数学上/下溢。
* 内存不足。
* 堆栈溢出。
* 明确放弃。
* 合同失败。
* 断言失败。
  
<!-- 
Our fundamental belief was that each is a condition the program cannot recover from.  Let's discuss each one. 
-->
我们的基本信念是每个都是程序无法恢复的条件。 我们来讨论每一个。

<!-- 
## Plain Old Bugs 
-->
## 普通老Bug

<!-- 
Some of these situations are unquestionably indicative of a program bug. 
-->
其中一些情况毫无疑问表明程序错误。

<!-- 
An incorrect cast, attempt to dereference `null`, array out-of-bounds access, or divide-by-zero are clearly problems
with the program's logic, in that it attempted an undeniably illegal operation.  As we will see later, there are ways
out (e.g., perhaps you want NaN-style propagation for DbZ).  But by default we assume it's a bug. 
-->
不正确的强制转换，尝试取消引用null，数组越界访问或被0除以显然是程序逻辑的问题，因为它试图进行无可否认的非法操作。 正如我们稍后将看到的，有一些方法可以解决（例如，你可能希望NaN风格的传播用于DbZ）。 但默认情况下我们认为这是一个错误。

<!-- 
Most programmers were willing to accept this without question.   And dealing with them as bugs this way brought
abandonment to the inner development loop where bugs during development could be found and fixed fast.  Abandonment
really did help to make people more productive at writing code.  This was a surprise to me at first, but it makes sense. 
-->
大多数程序员都愿意毫无疑问地接受这一点。 并且以这种方式将它们作为错误处理会使放弃内部开发循环，从而可以快速找到并修复开发过程中的错误。 放弃确实有助于提高人们编写代码的效率。 起初这对我来说是一个惊喜，但这是有道理的。

<!-- 
Some of these situations, on the other hand, are subjective.  We had to make a decision about the default behavior,
often with controversy, and sometimes offer programmatic control. 
-->
另一方面，其中一些情况是主观的。 我们必须对默认行为做出决定，通常会引起争议，有时会提供程序控制。

<!-- 
### Arithmetic Over/Underflow 
-->

### 算术上/下溢

<!-- 
Saying an unintended arithmetic over/underflow represents a bug is certainly a contentious stance.  In an unsafe system,
however, such things frequently lead to security vulnerabilities.  I encourage you to review the National Vulnerability
Database to see [the sheer number of these](
https://web.nvd.nist.gov/view/vuln/search-results?query=%22integer+overflow%22&search_type=all&cves=on). 
-->
说一个意外的算术上/下溢代表一个错误肯定是一个有争议的立场。 
然而，在不安全的系统中，这些事情经常导致安全漏洞。 
我建议您查看国家漏洞数据库，查看其中的绝对数量。

<!-- 
In fact, the Windows TrueType Font parser, which we ported to Midori (with *gains* in performance), has suffered over a
dozen of them in the past few years alone.  (Parsers tend to be farms for security holes like this.) 
-->
事实上，我们移植到Midori的Windows TrueType字体解析器
（性能提升）仅在过去几年就遭受了十几次解析。 
（解析器往往是像这样的安全漏洞的农场。）

<!-- 
This has given rise to packages like [SafeInt](https://safeint.codeplex.com/), which
essentially moves you away from your native language's arithmetic operations, in favor of checked library ones. 
-->
这就产生了像SafeInt这样的软件包，
它基本上使你远离了母语的算术运算，转而使用了已检查的库。

<!-- 
Most of these exploits are of course also coupled with an access to unsafe memory.  You could reasonably argue therefore
that overflows are innocuous in a safe language and therefore should be permitted.  It's pretty clear, however, based on
the security experience, that a program often does the wrong thing in the face of an unintended over/underflow.  Simply
put, developers frequently overlook the possibility, and the program proceeds to do unplanned things.  That's the
definition of a bug which is precisely what abandonment is meant to catch.  The final nail in the coffin on this one is
that philisophically, when there was any question about correctness, we tended to err on the side of explicit intent.
-->
这些漏洞中的大多数当然还伴随着对不安全内存的访问。 因此，您可以合理地争辩说，溢出在安全的语言中是无害的，因此应该被允许。 然而，基于安全经验，很明显，程序在面临意外的上溢/下溢时经常会做错事。 简单地说，开发人员经常忽略这种可能性，程序继续做无计划的事情。 这就是一个错误的定义，这正是放弃意味着捕获的错误。 关于这一点的棺材中的最后一个钉子是精神上的，当有任何关于正确性的问题时，我们倾向于在明确的意图方面犯错误。

<!-- 
Hence, all unannotated over/underflows were considered bugs and led to abandonment.  This was similar to compiling
C# with [the `/checked` switch](https://msdn.microsoft.com/en-us/library/h25wtyxf.aspx), except that our compiler
aggressively optimized redundant checks away.  (Since few people ever think to throw this switch in C#, the
code-generators don't do nearly as aggressive a job in removing the inserted checks.)  Thanks to this language and
compiler co-development, the result was far better than what most C++ compilers will produce in the face of SafeInt
arithmetic.  Also as with C#, [the `unchecked` scoping construct](
https://msdn.microsoft.com/en-us/library/khy08726.aspx) could be used where over/underflow was intended. 
-->
因此，所有未注释的上溢/下溢都被视为错误并导致放弃。 这类似于使用/ checked开关编译C＃，除了我们的编译器积极优化冗余检查。 （因为很少有人认为在C＃中抛出这个开关，代码生成器在删除插入的检查时几乎没有那么积极的工作。）由于这种语言和编译器共同开发，结果远远好于什么 大多数C ++编译器都会在面对SafeInt算法时生成。 与C＃一样，未经检查的范围构造也可用于预期上溢/下溢的情况。

<!-- 
Although the initial reactions from most C# and C++ developers I've spoken to about this idea are negative about it, our
experience was that 9 times out of 10, this approach helped to avoid a bug in the program.  That remaining 1 time was
usually an abandonment sometime late in one of our 72 hour stress runs -- in which we battered the entire system with
browsers and multimedia players and anything else we could do to torture the system -- when some harmless counter
overflowed.  I always found it amusing that we spent time fixing these instead of the classical way products mature
through the stress program, which is to say deadlocks and race conditions.  Between you and me, I'll take the overflow
abandonments! 
-->
虽然大多数C＃和C ++开发人员对这个想法所说的最初反应都是负面的，但我们的经验是10次中的9次，这种方法有助于避免程序中的错误。 在我们72小时的压力运行之一的某个时间段，剩下的1次通常是放弃 - 我们用浏览器和多媒体播放器以及我们可以用来折磨系统的任何其他任何东西来打击整个系统 - 当一些无害的计数器溢出时。 我总是觉得很有趣，我们花时间修复这些而不是通过压力程序成熟的经典方式产品，也就是死锁和竞争条件。 在你我之间，我会把溢出放弃！

<!-- 
### Out-of-Memory and Stack Overflow 
-->
### 内存不足和堆栈溢出

<!-- 
Out-of-memory (OOM) is complicated.  It always is.  And our stance here was certainly contentious also. 
-->
内存不足（OOM）很复杂。 它总是如此。 我们在这里的立场当然也是有争议的。

<!-- 
In environments where memory is manually managed, error code-style of checking is the most common approach: 
-->
在手动管理内存的环境中，错误代码检查方式是最常用的方法：

<!-- 
    X* x = (X*)malloc(...);
    if (!x) {
        // Handle allocation failure.
    } 
-->
    X* x = (X*)malloc(...);
    if (!x) {
        // 处理分配失败。
    } 

<!-- 
This has one subtle benefit: allocations are painful, require thought, and therefore programs that use this technique
are often more frugal and deliberate with the way they use memory.  But it has a huge downside: it's error prone and
leads to huge amounts of frequently untested code-paths.  And when code-paths are untested, they usually don't work. 
-->
这有一个微妙的好处：分配是痛苦的，需要思考，因此使用这种技术的程序通常更节俭和故意使用它们使用内存的方式。
但它有一个巨大的缺点：它容易出错并导致大量经常未经测试的代码路径。 当代码路径未经测试时，它们通常不起作用。

<!-- 
Developers in general do a terrible job making their software work properly right at the edge of resource exhaustion.
In my experience with Windows and the .NET Framework, this is where egregious mistakes get made.  And it leads to
ridiculously complex programming models, like .NET's so-called [Constrained Execution Regions](
https://blogs.msdn.microsoft.com/bclteam/2005/06/13/constrained-execution-regions-and-other-errata-brian-grunkemeyer/).
A program limping along, unable to allocate even tiny amounts of memory, can quickly become the enemy of reliability.
[Chris Brumme's wondrous Reliability post](http://blogs.msdn.com/b/cbrumme/archive/2003/06/23/51482.aspx) describes this
and related challenges in all its gory glory. 
-->
一般而言，开发人员在资源枯竭的边缘使他们的软件正常工作时做得非常糟糕。 根据我使用Windows和.NET Framework的经验，这是一个令人震惊的错误。 它导致了非常复杂的编程模型，比如.NET的所谓的约束执行区域。 一个程序一瘸一拐，无法分配甚至微小的内存，很快就会成为可靠性的敌人。 克里斯·布鲁姆（Chris Brumme）的奇妙可靠性帖子描述了这一点以及相关的挑战。

<!-- 
Parts of our system were of course "hardened" in a sense, like the lowest levels of the kernel, where abandonment's
scope would be necessarily wider than a single process.  But we kept this to as little code as possible. 
-->
在某种意义上，我们系统的某些部分当然是“硬化的”，就像内核的最低级别一样，放弃的范围必然比单个进程更宽。 但是我们尽量保持这个代码。

<!-- 
For the rest?  Yes, you guessed it: abandonment.  Nice and simple. 
-->
对于其余的？ 是的，你猜对了：放弃。 很好，很简单。

<!-- 
It was surprising how much of this we got away with.  I attribute most of this to the isolation model.  In fact, we
could *intentionally* let a process suffer OOM, and ensuing abandonment, as a result of resource management policy, and
still remain confident that stability and recovery were built in to the overall architecture.
-->
令人惊讶的是，我们侥幸逃脱了多少。 我将大部分内容归因于隔离模型。 实际上，由于资源管理政策，我们可能故意让一个流程遭受OOM，并随之放弃，并且仍然相信稳定性和恢复是建立在整体架构中的。


<!-- 
It was possible to opt-in to recoverable failure for individual allocations if you really wanted.  This was not common
in the slightest, however the mechanisms to support it were there.  Perhaps the best motivating example is this: imagine
your program wants to allocate a buffer of 1MB in size.  This situation is different than your ordinary run-of-the-mill
sub-1KB object allocation.  A developer may very well be prepared to think and explicitly deal with the fact that a
contiguous block of 1MB in size might not be available, and deal with it accordingly.  For example: 
-->
如果您真的想要，可以选择加入单个分配的可恢复故障。 这丝毫不常见，但支持它的机制就在那里。 也许最好的激励示例是：假设您的程序想要分配1MB大小的缓冲区。 这种情况与普通的普通的1KB对象分配不同。 开发人员可能已经准备好思考并明确地处理可能无法获得1MB大小的连续块的事实，并相应地处理它。 例如：


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
堆栈溢出是这一理念的简单扩展。 Stack只是一个内存支持的资源。 实际上，由于我们的异步链接堆栈模型，堆栈的运行与堆内存的运行完全相同，因此开发人员对其处理方式的一致性并不令人惊讶。 如今，许多系统以这种方式处理堆栈溢出。

<!-- 
## Assertions 
-->
## 断言

<!-- 
An assertion was a manual check in the code that some condition held true, triggering abandonment if it did not.  As
with most systems, we had both debug-only and release code assertions, however unlike most other systems, we had more
release ones than debug.  In fact, our code was peppered liberally with assertions.  Most methods had multiple. 
-->
断言是代码中的手动检查，某些条件成立，如果不成功则触发放弃。 与大多数系统一样，我们同时具有仅调试和释放代码断言，但与大多数其他系统不同，我们拥有的调试版本多于调试版本。 事实上，我们的代码充满了断言。 大多数方法有多个。

<!-- 
This kept with the philosophy that it's better to find a bug at runtime than to proceed in the face of one.  And, of
course, our backend compiler was taught how to optimize them aggressively as with everything else.  This level of
assertion density is similar to what guidelines for highly reliable systems suggest.  For example, from NASA's paper,
[The Power of Ten -Rules for Developing Safety Critical Code](
http://pixelscommander.com/wp-content/uploads/2014/12/P10.pdf): 
-->
这样做的理念是，在运行时找到错误比在面对错误时更好。 当然，我们的后端编译器也被教导如何像其他一切一样积极地优化它们。 这种断言密度水平类似于高可靠性系统的指导原则。 例如，来自美国宇航局的论文“发展安全关键代码的十大规则的力量”：

<!-- 
> Rule: The assertion density of the code should average to a minimum of two assertions per function.  Assertions are
> used to check for anomalous conditions that should never happen in real-life executions.  Assertions must always be
> side-effect free and should be defined as Boolean tests.
> 
> Rationale: Statistics for industrial coding efforts indicate that unit tests often find at least one defect per 10 to
> 100 lines of code written.  The odds of intercepting defects increase with assertion density.  Use of assertions is
> often also recommended as part of strong defensive coding strategy. 
-->
> 规则：代码的断言密度应平均为每个函数最少两个断言。 断言用于检查在现实生活中不应发生的异常情况。 断言必须始终是无副作用的，并且应该定义为布尔测试。

> 理由：工业编码工作的统计数据表明，单元测试通常每写入10到100行代码就会发现至少一个缺陷。 拦截缺陷的几率随着断言密度而增加。 断言的使用通常也被推荐为强防御性编码策略的一部分。

<!-- 
To indicate an assertion, you simply called `Debug.Assert` or `Release.Assert`: 
-->
要表示断言，只需调用Debug.Assert或Release.Assert：

<!-- 
    void Foo() {
        Debug.Assert(something); // Debug-only assert.
        Release.Assert(something); // Always-checked assert.
    } 
-->
    void Foo() {
        Debug.Assert(something); // 仅调试断言。
        Release.Assert(something); // 始终检查断言。
    } 

<!-- 
We also implemented functionality akin to `__FILE__` and `__LINE__` macros like in C++, in addition to `__EXPR__` for
the text of the predicate expression, so that abandonments due to failed assertions contained useful information. 
-->
我们还实现了类似于C ++中的__FILE__和__LINE__宏的功能，以及谓词表达式文本的__EXPR__，因此由于失败的断言而导致的放弃包含有用的信息。

<!-- 
In the early days, we used different "levels" of assertions than these.  We had three levels, `Contract.Strong.Assert`,
`Contract.Assert`, and `Contract.Weak.Assert`.  The strong level meant "always checked," the middle one meant "it's up
to the compiler," and the weak one meant "only checked in debug mode."  I made the controversial decision to move away
from this model.  In fact, I'm pretty sure 49.99% of the team absolutely hated my choice of terminology (`Debug.Assert`
and `Release.Assert`), but I always liked them because it's pretty unambiguous what they do.  The problem with the old
taxonomy was that nobody ever knew exactly when the assertions would be checked; confusion in this area is simply not
acceptable, in my opinion, given how important good assertion discipline is to the reliability of one's program. 
-->
在早期，我们使用与这些断言不同的“级别”。 我们有三个级别，Contract.Strong.Assert，Contract.Assert和Contract.Weak.Assert。 强大的水平意味着“始终检查”，中间的意思是“这取决于编译器”，弱的意味着“只在调试模式下检查。”我做出了有争议的决定，摆脱这种模式。 事实上，我非常确定49.99％的团队绝对讨厌我选择的术语（Debug.Assert和Release.Assert），但我总是喜欢它们，因为它们的确非常明确。 旧分类法的问题在于，没有人确切知道何时会检查断言; 在我看来，这个领域的混乱根本不可接受，因为好的断言纪律对一个人的计划的可靠性有多么重要。

<!-- 
As we moved contracts to the language (more on that soon), we tried making `assert` a keyword too.  However, we
eventually switched back to using APIs.  The primary reason was that assertions were *not* part of an API's signature
like contracts are; and given that assertions could easily be implemented as a library, it wasn't clear what we gained
from having them in the language.  Furthermore, policies like "checked in debug" versus "checked in release" simply
didn't feel like they belonged in a programming language.  I'll admit, years later, I'm still on the fence about this. 
-->
当我们将合同移到语言上时（很快就会更多），我们也尝试断言关键字。 但是，我们最终转而使用API。 主要原因是断言不是合同的API签名的一部分; 鉴于断言可以很容易地作为一个库实现，我们不清楚我们在语言中获得了什么。 此外，像“check in debug”和“checked in release”之类的策略根本不像是属于编程语言。 我承认，多年以后，我仍然对此持怀疑态度。

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
合同是在Midori中捕获bug的核心机制。尽管我们开始使用Singularity，它使用了Spec＃的变体Sing＃，但我们很快就转移到了香草C＃并且不得不重新发现我们想要的东西。在与模特生活多年后，我们最终在一个非常不同的地方结束了。

<!-- 
All contracts and assertions were proven side-effect free thanks to our language's understanding of immutability and
side-effects.  This was perhaps the biggest area of language innovation, so I'll be sure to write a post about it soon. 
-->
由于我们的语言对不变性和副作用的理解，所有合同和断言都被证明是无副作用的。这可能是语言创新的最大领域，所以我一定会尽快写一篇关于它的文章。

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
与其他领域一样，我们受到许多其他系统的启发和影响。 Spec＃是显而易见的。埃菲尔有很大的影响力，特别是因为有许多已发表的案例研究需要学习。研究工作，如基于Ada的SPARK以及实时和嵌入式系统的建议。深入研究理论上的兔子洞，像Hoare的公理语义这样的编程逻辑为所有这些提供了基础。然而，对我来说，最哲学的灵感来自CLU，以及后来的Argus，整体的错误处理方法。


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
最基本的合同形式是方法前提条件。 这说明了要分派的方法必须具备的条件。 这通常用于验证参数。 有时它用于验证目标对象的状态，但这通常是不受欢迎的，因为对于程序员来说，模态是很难的。 前提条件基本上是调用者向被调用者提供的保证。

<!-- 
In our final model, a precondition was stated using the `requires` keyword: 
-->
在我们的最终模型中，使用requires关键字声明了前提条件：

<!-- 
    void Register(string name)
        requires !string.IsEmpty(name) {
        // Proceed, knowing the string isn't empty.
    } 
-->
    void Register(string name)
        requires !string.IsEmpty(name) {
        // 继续，知道字符串不为空。
    } 

<!-- 
A slightly less common form of contract is a method postcondition.  This states what conditions hold *after* the method
has been dispatched.  This is a guarantee the callee provides to the caller. 
-->
一种稍微不太常见的合同形式是方法后置条件。 这表明在调度方法之后保持什么条件。 这是被叫方向呼叫者提供的保证。

<!-- 
In our final model, a postcondition was stated using the `ensures` keyword: 
-->
在我们的最终模型中，使用`ensure`关键字声明了后置条件：

<!-- 
    void Clear()
        ensures Count == 0 {
        // Proceed; the caller can be guaranteed the Count is 0 when we return.
    } 
-->
    void Clear()
        ensures Count == 0 {
        // 继续; 当我们返回时，可以保证调用者是0。
    } 
<!-- 
It was also possible to mention the return value in the postcondition, through the special name `return`.  Old values --
such as necessary for mentioning an input in a post-condition -- could be captured through `old(..)`; for example: 
-->
也可以通过特殊名称返回提及后置条件中的返回值。 旧的价值观 - 例如在后置条件中提及输入所必需的 - 可以通过旧的（...）捕获; 例如：


    int AddOne(int value)
        ensures return == old(value)+1 {
        ...
    } 


<!-- 
Of course, pre- and postconditions could be mixed.  For example, from our ring buffer in the Midori kernel: 
-->
当然，前后条件可能是混合的。 例如，来自Midori内核中的环形缓冲区：

    public bool PublishPosition()
        requires RemainingSize == 0
        ensures UnpublishedSize == 0 {
        ...
    } 

<!-- 
This method could safely execute its body knowing that `RemainingSize` is `0` and callers could safely execute after
the return knowing that `UnpublishedSize` is also `0`. 
-->
知道RemainingSize为0时，此方法可以安全地执行其身体，并且知道UnpublishedSize也为0后，调用者可以在返回后安全地执行。


<!-- 
If any of these contracts are found to be false at runtime, abandonment occurs. 
-->
如果在运行时发现任何这些合同是错误的，则会发生放弃。

<!-- 
This is an area where we differ from other efforts.  Contracts have recently became popular as an expression of program
logics used in advanced proof techniques.  Such tools prove truths or falsities about stated contracts, often using
global analysis.  We took a simpler approach.  By default, contracts are checked at runtime.  If a compiler could prove
truth or falsehood at compile-time, it was free to elide runtime checks or issue a compile-time error, respectively. 
-->
这是我们与其他努力不同的领域。最近，合同作为高级校对技术中使用的程序逻辑的表达而变得流行。这些工具通常使用全局分析来证明所述合同的真实性或虚假性。我们采取了一种更简单的方法。默认情况下，在运行时检查合同。如果编译器可以在编译时证明真或假，则可以自由地分别进行运行时检查或发出编译时错误。

<!-- 
Modern compilers have constraint-based analyses that do a good job at this, like the [range analysis](
https://en.wikipedia.org/wiki/Value_range_analysis) I mentioned in my last post.  These propagate facts and use them to
optimize code already.  This includes eliminating redundant checks: either explicitly encoded in contracts, or in
normal program logic.  And they are trained to perform these analyses in reasonable amounts of time, lest programmers
switch to a different, faster compiler.  The theorem proving techniques simply did not scale for our needs; our core
system module took over a day to analyze using the best in breed theorem proving analysis framework! 
-->
现代编译器具有基于约束的分析，在这方面做得很好，就像我在上一篇文章中提到的范围分析一样。这些传播事实并使用它们来优化代码。这包括消除冗余检查：在合同中明确编码，或在正常程序逻辑中。并且他们受过训练，可以在合理的时间内执行这些分析，以免程序员切换到不同的，更快的编译器。定理证明技术根本无法满足我们的需求;我们的核心系统模块花了一天的时间来分析使用最佳的品种定理证明分析框架！


<!-- 
Furthermore, the contracts a method declared were part of its signature.  This meant they would automatically show up in
documentation, IDE tooltips, and more.  A contract was as important as a method's return and argument types.  Contracts
really were just an extension of the type system, using arbitrary logic in the language to control the shape of exchange
types.  As a result, all the usual subtyping requirements applied to them.  And, of course, this facilitated modular
local analysis which could be done in seconds using standard optimizing compiler techniques. 
-->
此外，方法声明的合同是其签名的一部分。这意味着它们会自动显示在文档，IDE工具提示等中。契约与方法的返回和参数类型一样重要。合同实际上只是类型系统的扩展，使用语言中的任意逻辑来控制交换类型的形状。因此，所有通常的子类型要求都适用于它们。当然，这有助于模块化本地分析，可以使用标准优化编译器技术在几秒钟内完成。

<!-- 
90-something% of the typical uses of exceptions in .NET and Java became preconditions.  All of the
`ArgumentNullException`, `ArgumentOutOfRangeException`, and related types and, more importantly, the manual checks and
`throw`s were gone.  Methods are often peppered with these checks in C# today; there are thousands of these in .NET's
CoreFX repo alone.  For example, here is `System.IO.TextReader`'s `Read` method: 
-->
.NET和Java中90％的异常典型用法成为先决条件。所有的ArgumentNullException，ArgumentOutOfRangeException和相关类型，更重要的是，手动检查和抛出都消失了。今天C＃中的方法经常被这些检查所覆盖;仅在.NET的CoreFX repo中就有成千上万的这些。例如，这是System.IO.TextReader的Read方法：

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
    /// <exception cref="ArgumentNullException">如果buffer为null，则抛出该异常。</exception>
    /// <exception cref="ArgumentOutOfRangeException">如果index小于零则抛出。</exception>
    /// <exception cref="ArgumentOutOfRangeException">如果count小于零则抛出。</exception>
    /// <exception cref="ArgumentException">如果索引和计数超出缓冲区的范围，则抛出该异常。</exception>
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
出于多种原因，这已被打破。 当然，这是非常费力的。 所有的仪式！ 但是，当开发人员真的不应该抓住它们时，我们必须尽力记录异常。 相反，他们应该在开发过程中找到错误并修复它。 所有这些例外的废话都会鼓励非常糟糕的行为。

<!-- 
If we use Midori-style contracts, on the other hand, this collapses to: 
-->
另一方面，如果我们使用Midori式合约，则会崩溃：

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
这有一些吸引人的事情。 首先，它更简洁。 然而，更重要的是，它以一种记录自身并且呼叫者易于理解的方式自我描述API的合同。 而不是要求程序员用英语表达错误条件，实际表达式可供调用者阅读，以及用于理解和利用的工具。 它使用放弃来传达失败。

<!-- 
I should also mention we had plenty of contracts helpers to help developers write common preconditions.  The above
explicit range checking is very messy and easy to get wrong.  Instead, we could have written: 
-->
我还应该提到我们有很多合同助手来帮助开发人员编写常见的前提条件。 上面的显式范围检查非常混乱，容易出错。 相反，我们可以写：

    public virtual int Read(char[] buffer, int index, int count)
        requires buffer != null
        requires Range.IsValid(index, count, buffer.Length) {
        ...
    } 

<!-- 
And, totally aside from the conversation at hand, coupled with two advanced features -- arrays as slices and non-null
types -- we could have reduced the code to the following, while preserving the same guarantees:
-->
而且，除了手头的对话之外，还有两个高级功能 - 数组作为切片和非零
类型 - 我们可以将代码减少到以下，同时保留相同的保证：

    public virtual int Read(char[] buffer) {
        ...
    } 

<!-- 
But I'm jumping way ahead ... 
-->
但我正在向前迈进......

<!-- 
### Humble Beginnings 
-->
### 谦虚的开始

<!-- 
Although we landed on the obvious syntax that is very Eiffel- and Spec#-like -- coming full circle -- as I mentioned
earlier, we really didn't want to change the language at the outset.  So we actually began with a simple API approach: 
-->
虽然我们提到了明显的语法，这种语法非常像Eiffel-和Spec＃-like  - 正如我所提到的那样
之前，我们真的不想在一开始就改变语言。 所以我们实际上从一个简单的API方法开始：
    public bool PublishPosition() {
        Contract.Requires(RemainingSize == 0);
        Contract.Ensures(UnpublishedSize == 0);
        ...
    } 

<!-- 
There are a number of problems with this approach, as the [.NET Code Contracts](
http://research.microsoft.com/en-us/projects/contracts/) effort discovered the hard way. 
-->
这种方法存在许多问题，因为.NET Code Contracts努力发现了困难的方法。

<!-- 
First, contracts written this way are part of the API's *implementation*, whereas we want them to be part of the
*signature*.  This might seem like a theoretical concern but it is far from being theoretical.  We want the resulting
program to contain built-in metadata so tools like IDEs and debuggers can display the contracts at callsites.  And we
want tools to be in a position to auto-generate documentation from the contracts.  Burying them in the implementation
doesn't work unless you somehow disassemble the method to extract them later on (which is a hack). 
-->

首先，以这种方式编写的合同是API实现的一部分，而我们希望它们成为签名的一部分。这似乎是一个理论上的问题，但它远非理论上的。我们希望生成的程序包含内置元数据，因此IDE和调试器等工具可以在callites上显示合同。我们希望工具能够从合同中自动生成文档。除非你以某种方式反汇编方法以便在以后提取它们（这是一个黑客攻击），否则将它们埋在实现中是行不通的。

<!-- 
This also makes it tough to integrate with a backend compiler which we found was necessary for good performance. 
-->
这也使得很难与后端编译器集成，我们发现这对于良好的性能是必要的。

<!-- 
Second, you might have noticed an issue with the call to `Contract.Ensures`.  Since `Ensures` is meant to hold on all
exit paths of the function, how would we implement this purely as an API?  The answer is, you can't.  One approach is
rewriting the resulting MSIL, after the language compiler emitted it, but that's messy as all heck.  At this point, you
begin to wonder, why not simply acknowledge that this is a language expressivity and semantics issue, and add syntax? 
-->
其次，您可能已经注意到对Contract.Ensures的调用存在问题。由于Ensures旨在保留函数的所有退出路径，我们如何将其纯粹实现为API？答案是，你做不到。一种方法是在语言编译器发出它之后重写生成的MSIL，但这很麻烦。此时，您开始怀疑，为什么不简单地承认这是语言表达性和语义问题，并添加语法？

<!-- 
Another area of perpetual struggle for us was whether contracts are conditional or not.  In many classical systems,
you'd check contracts in debug builds, but not the fully optimized ones.  For a long time, we had the same three levels
for contracts that we did assertions mentioned earlier: 
-->
对我们来说，永久斗争的另一个领域是合同是否有条件。在许多经典系统中，您需要检查调试版本中的合同，而不是完全优化的合同。很长一段时间，我们对前面提到的断言的合同有三个相同的级别：

<!-- 
* Weak, indicated by `Contract.Weak.*`, meaning debug-only.
* Normal, indicated simply by `Contract.*`, leaving it as an implementation decision when to check them.
* Strong, indicated by `Contract.Strong.*`, meaning always checked. 
-->
* 弱，由`Contract.Weak。*`表示，表示仅调试。
* 正常，简单地用`Contract。*`表示，将其作为执行决定何时检查它们。
* 强，由`Contract.Strong。*`表示，意思是总是检查。

<!-- 
I'll admit, I initially found this to be an elegant solution.  Unfortunately, over time we found that there was constant
confusion about whether "normal" contracts were on in debug, release, or all of the above (and so people misused weak
and strong accordingly).  Anyway, when we began integrating this scheme into the language and backend compiler
toolchain, we ran into substantial issues and had to backpedal a little bit. 
-->
我承认，我最初认为这是一个优雅的解决方案。 不幸的是，随着时间的推移，我们发现在调试，发布或上述所有内容中是否存在“正常”合同时常常存在混淆（因此人们误用弱者和强者）。 无论如何，当我们开始将这个方案集成到语言和后端编译器工具链中时，我们遇到了很多问题，不得不稍微退一步。

<!-- 
First, if you simply translated `Contract.Weak.Requires` to `weak requires` and `Contract.Strong.Requires` to
`strong requires`, in my opinion, you end up with a fairly clunky and specialized language syntax, with more policy than
made me comfortable.  It immediately calls out for parameterization and substitutability of the `weak`/`strong` policies. 
-->
首先，如果你简单地将Contract.Weak.Requires翻译成弱要求和Contract.Strong要求强烈要求，在我看来，你最终得到一个相当笨重和专业的语言语法，更多的政策比让我感到舒服。 它立即呼吁弱/强政策的参数化和可替代性。

<!-- 
Next, this approach introduces a sort of new mode of conditional compilation that, to me, felt awkward.  In other
words, if you want a debug-only check, you can already say something like: 
-->
接下来，这种方法引入了一种新的条件编译模式，对我来说，感觉很尴尬。 换句话说，如果你想要一个只调试检查，你可以说：

    #if DEBUG
        requires X
    #endif 

<!-- 
Finally -- and this was the nail in the coffin for me -- contracts were supposed to be part of an API's signature.  What
does it even *mean* to have a conditional contract?  How is a tool supposed to reason about it?  Generate different
documentation for debug builds than release builds?  Moreover, as soon as you do this, you lose a critical guarantee,
which is that code doesn't run if its preconditions aren't met. 
-->
最后 - 这对我来说是棺材中的钉子 - 合同应该是API签名的一部分。拥有有条件合同甚至意味着什么？一个工具如何推理呢？为发布版本生成调试版本的不同文档？此外，只要执行此操作，就会失去一个关键保证，即如果不满足其前提条件，代码将无法运行。

<!-- 
As a result, we nuked the entire conditional compilation scheme. 
-->
结果，我们完成了整个条件编译方案。

<!-- 
We ended up with a single kind of contract: one that was part of an API's signature and checked all the time.  If a
compiler could prove the contract was satisfied at compile-time -- something we spent considerable energy on -- it was
free to elide the check altogether.  But code was guaranteed it would never execute if its preconditions weren't
satisfied.  For cases where you wanted conditional checks, you always had the assertion system (described above). 
-->
我们最终得到了一种合同：一种是API签名的一部分，并且一直在检查。如果编译器可以证明合同在编译时得到满足 - 我们花了相当大的精力 - 它可以完全免除检查。但是代码被保证，如果不满足其先决条件，它将永远不会执行。对于需要条件检查的情况，您始终拥有断言系统（如上所述）。

<!-- 
I felt better about this bet when we deployed the new model and found that lots of people had been misusing the "weak"
and "strong" notions above out of confusion.  Forcing developers to make the decision led to healthier code. 
-->
当我们部署新模型时，我觉得这个赌注更好，并且发现许多人因混淆而滥用上面的“弱”和“强”概念。迫使开发人员做出决定导致更健康的代码。

<!-- 
## Future Directions 
-->
## 未来发展方向

<!-- 
A number of areas of development were at varying stages of maturity when our project wound down.
-->
当我们的项目结束时，许多发展领域处于不同的成熟阶段。

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
我们用不变量进行了很多实验。 每当我们与熟悉合同设计的人交谈时，他们都会对我们从第一天起就没有它们感到宽慰。 说实话，我们的设计从一开始就包含了它们。 但是我们从来没有完成实施和部署它。 这部分仅仅是由于工程带宽，还因为一些困难的问题仍然存在。 老实说，团队几乎总是满意前后条件和断言的结合。 我怀疑在充足的时间里我们已经添加了不变量来完成，但到目前为止，我仍然有一些问题。 我需要在行动中看一段时间。

<!-- 
The approach we had designed was where an `invariant` becomes a member of its enclosing type; for example: 
-->
我们设计的方法是不变量成为其封闭类型的成员; 例如：

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
请注意，不变量标记为私有。不变量的可访问性修饰符控制了不变量需要保留的成员。例如，公共不变量只需要在具有公共可访问性的函数的进入和退出时保持;这允许私有函数的常见模式暂时违反不变量，只要公共入口点保留它们即可。当然，如上例所示，类也可以自由声明私有不变量，这需要保存所有函数入口和出口。

<!-- 
I actually quite liked this design, and I think it would have worked.  The primary concern we all had was the silent
introduction of checks all over the place.  To this day, that bit still makes me nervous.  For example, in the `List<T>`
example, you'd have the `index >= 0 && index < array.Length` check at the beginning and end of *every single function*
of the type.  Now, our compiler eventually got very good at recognizing and coalescing redundant contract checks; and
there were ample cases where the presence of a contract actually made code quality *better*.  However, in the extreme
example given above, I'm sure there would have been a performance penalty.  That would have put pressure on us changing
the policy for when invariants are checked, which would have possibly complicated the overall contracts model. 
-->

我其实很喜欢这个设计，我觉得它会有用。我们所关心的主要问题是在整个地方无声地引入支票。直到今天，这一点仍让我感到紧张。例如，在List <T>示例中，您将在类型的每个函数的开头和结尾检查`index > = 0 && index <array.Length`。现在，我们的编译器最终非常善于识别和合并冗余合同检查;在很多情况下，合同的存在实际上使代码质量更好。但是，在上面给出的极端例子中，我确信会有性能损失。这会对我们改变检查不变量的政策施加压力，这可能会使整体合约模式变得复杂。

<!-- 
I really wish we had more time to explore invariants more deeply.  I don't think the team sorely missed not having them
-- certainly I didn't hear much complaining about their absence (probably because the team was so performance conscious)
-- but I do think invariants would have been a nice icing to put on the contracts cake. 
-->

我真的希望我们有更多的时间来更深入地探索不变量。我不认为球队严重错过了没有他们 - 当然我没有听到太多抱怨他们缺席（可能是因为球队非常注重表现） - 但我确实认为不变量会是一个很好的结冰合同蛋糕。

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
我总是喜欢说合同从类型系统离开的地方开始。类型系统允许您使用类型对变量的属性进行编码。类型限制变量可能包含的预期范围值。合同类似地检查变量所持有的值的范围。区别？类型在编译时通过严格且可组合的归纳规则得到验证，这些规则对于函数本地检查而言价格便宜，通常但并不总是由开发人员编写的注释辅助。合同在可能的情况下在编译时证明，否则在运行时证明，因此，允许使用语言本身编码的任意逻辑进行严格的规范。

<!-- 
Types are preferable, because they are *guaranteed* to be compile-time checked; and *guaranteed* to be fast to check.
The assurances given to the developer are strong and the overall developer productivity of using them is better. 
-->
类型是可取的，因为它们保证是编译时检查的;并保证快速检查。给开发人员的保证很强，开发人员使用它们的整体效率更高。

<!-- 
Limitations in a type system are inevitable, however; a type system needs to leave *some* wiggle room, otherwise it
quickly grows unwieldly and unusable and, in the extreme, devolves into bi-value bits and bytes.  On the other hand, I
was always disappointed by two specific areas of wiggle room that required the use of contracts:
-->
然而，类型系统的局限是不可避免的;类型系统需要留下一些摆动空间，否则它会迅速增长并且无法使用，并且在极端情况下会转换为双值位和字节。另一方面，我总是对需要使用合同的两个特定的摆动空间区域感到失望：

<!-- 
1. Nullability.
2. Numeric ranges. 
-->
1. 空性。
2. 数字范围。

<!-- 
Approximately 90% of our contracts fell into these two buckets.  As a result, we seriously explored more sophisticated
type systems to classify the nullability and ranges of variables using the type system instead of contracts. 
-->
我们大约90％的合同属于这两个桶。因此，我们认真研究了更复杂的类型系统，使用类型系统而不是契约对变量的可空性和范围进行分类。

<!-- 
To make it concrete, this was the difference between this code which uses contracts: 
-->
为了使它具体化，这是使用合同的代码之间的区别：

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
并且这段代码在编译时静态检查，但不需要，但仍然保留了所有相同的保证：

    public virtual int Read(char[] buffer) {
        ...
    } 


<!-- 
Placing these properties in the type system significantly lessens the burden of checking for error conditions.  Lets
say that for any given 1 producer of state there are 10 consumers.  Rather than having each of those 10 defend
themselves against error conditions, we can push the responsibility back onto that 1 producer, and either require a
single assertion that coerces the type, or even better, that the value is stored into the right type in the first place. 
-->
将这些属性放置在类型系统中可以显着减轻检查错误条件的负担。 让我们说，对于任何给定的1个州的生产者，有10个消费者。 我们可以将责任推回到第一个生产者身上，而不是让每个人都自己抵御错误条件，并且需要一个强制类型的断言，或者更好的是，将值存储到正确的类型中。 第一名。

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
第一个真的很难：保证静态变量不会采用空值。 这就是Tony Hoare所谓的“十亿美元错误”。 解决这个问题对于任何语言来说都是一个正确的目标，我很高兴看到新的语言设计师正面解决这个问题。

<!-- 
Many areas of the language fight you every step of the way on this one.  Generics, zero-initialization, constructors,
and more.  Retrofitting non-null into an existing language is tough! 
-->
语言的许多方面都会在这一步中与你争吵。 泛型，零初始化，构造函数等。 将非null转换为现有语言很难！

<!-- 
##### The Type System 
-->
##### 类型系统

<!-- 
In a nutshell, non-nullability boiled down to some simple type system rules: 
-->
简而言之，非可空性归结为一些简单的类型系统规则：
<!-- 
1. All unadorned types `T` were non-null by default.
2. Any type could be modified with a `?`, as in `T?`, to mark it nullable.
3. `null` is an illegal value for variables of non-null types.
4. `T` implicitly converts to `T?`.  In a sense, `T` is a subtype of `T?` (although not entirely true).
5. Operators exist to convert a `T?` to a `T`, with runtime checks that abandoned on `null`. 
-->
1. 默认情况下，所有未加修饰的类型`T`都是非空的。
2. 任何类型都可以使用`？`进行修改，如`T？`，以将其标记为可为空。
3. “null”是非null类型变量的非法值。
4. `T`隐含地转换为`T？`。 从某种意义上说，`T`是`T？'的子类型（尽管不完全正确）。
5. 运算符的存在将`T？`转换为`T`，运行时检查放弃在`null`上。

<!-- 
Most of this is probably "obvious" in the sense that there aren't many choices.  The name of the game is systematically
ensuring all avenues of `null` are known to the type system.  In particular, no `null` can ever "sneakily" become the
value of a non-null `T` type; this meant addressing zero-initialization, perhaps the hardest problem of all. 
-->
大多数情况可能是“显而易见的”，因为选择的选择并不多。 游戏的名称系统地确保类型系统知道所有null的途径。 特别是，没有null可以“偷偷摸摸”成为非null T类型的值; 这意味着解决零初始化问题，这可能是最困难的问题。

<!-- 
##### The Syntax 
-->
##### 语法

<!-- 
Syntactically, we offered a few ways to accomplish #5, converting from `T?` to `T`.  Of course, we discouraged this, and
preferred you to stay in "non-null" space as long as possible.  But sometimes it's simply not possible.  Multi-step
initialization happens from time to time -- especially with collections data structures -- and had to be supported. 
-->
从语法上讲，我们提供了几种方法来完成＃5，从T转换？ 当然，我们不鼓励这样做，并且希望你尽可能长时间留在“非空”空间。 但有时它根本不可能。 
多步骤初始化不时发生 - 特别是对于集合数据结构 - 并且必须得到支持。

<!-- 
Imagine for a moment we have a map: 
-->
想象一下，我们有一张地图：

    Map<int, Customer> customers = ...; 

<!-- 
This tells us three things by construction: 
-->
这通过构造告诉我们三件事：

<!-- 
1. The `Map` itself is not null.
2. The `int` keys inside of it will not be `null`.
3. The `Customer` values inside of it will also not be null. 
-->
1.“地图”本身不是空的。
2.里面的`int`键不是'null`。
3.其中的“Customer”值也不会为null。

<!-- 
Let's now say that the indexer actually returns `null` to indicate the key was missing: 
-->
现在让我们说索引器实际上返回“null”以指示密钥丢失：


    public TValue? this[TKey key] {
        get { ... }
    } 

<!-- 
Now we need some way of checking at callsites whether the lookup succeeded.  We debated many syntaxes. 
-->
现在我们需要一些方法来检查调用是否成功。 我们讨论了许多语法。

<!-- 
The easiest we landed on was a guarded check: 
-->
我们最容易登陆的是一张看守：

<!-- 
    Customer? customer = customers[id];
    if (customer != null) {
        // In here, `customer` is of non-null type `Customer`.
    } 
-->
    Customer? customer = customers[id];
    if (customer != null) {
        // 在这里，`customer`是非null类型`Customer`。
    } 
<!-- 
I'll admit, I was always on the fence about the "magical" type coercions.  It annoyed me that it was hard to figure out
what went wrong when it failed.  For example, it didn't work if you compared `c` to a variable that held the `null`
value, only the literal `null`.  But the syntax was easy to remember and usually did the right thing. 
-->
我承认，我总是对“魔法”类型的强制进行抨击。 让我感到恼火的是，当它失败时很难弄清楚出了什么问题。 例如，如果将c与保持null值的变量进行比较，那么它就不起作用，只有文字null。 但语法很容易记住，通常做对了。

<!-- 
These checks dynamically branch to a different piece of logic if the value is indeed `null`.  Often you'd want to simply
assert that the value is non-null and abandon otherwise.  There was an explicit type-assertion operator to do that: 
-->
如果值确实为空，则这些检查动态分支到不同的逻辑。 通常，您只想断言该值为非null，否则放弃。 有一个显式的类型断言运算符来做到这一点：


    Customer? maybeCustomer = customers[id];
    Customer customer = notnull(maybeCustomer); 

<!-- 
The `notnull` operator turned any expression of type `T?` into an expression of type `T`. 
-->
`notnull`操作符将类型为`T？`的任何表达式转换为`T`类型的表达式。

<!-- 
##### Generics 
-->
##### 泛型

<!-- 
Generics are hard, because there are multiple levels of nullability to consider.  Consider: 
-->
泛型很难，因为要考虑多个级别的可空性。 考虑：
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
基本问题是，`a`，`b`，`c`和`d`的类型是什么？

<!-- 
I think we made this one harder initially than we needed to largely because C#'s existing nullable is a pretty odd duck
and we got distracted trying to mimic it too much.  The good news is we finally found our way, but it took a while. 
-->
我认为我们最初比我们需要的更难，因为C＃现有的可空系统是一个非常奇怪的鸭子，我们分心试图模仿它太多。 好消息是我们终于找到了方向，但需要一段时间。

<!-- 
To illustrate what I mean, let's go back to the example.  There are two camps: 
-->
为了说明我的意思，让我们回到这个例子。 有两个阵营：


功能语言路线最初会稍微改变一下你的想法。 例如，前面的地图示例：
<!-- 
* The .NET camp: `a` is `object`; `b`, `c`, and `d` are `object?`.
* The functional language camp: `a` is `object`; `b` and `c` are `object?`; `d` is `object??`.
-->
* .NET阵营：一个是对象; b，c和d是对象吗？
* 功能语言阵营：一个是对象; b和c是对象吗？ d是对象??。

<!-- 
In other words, the .NET camp thinks you should collapse any sequence of 1 or more `?`s into a single `?`.  The
functional language camp -- who understands the elegance of mathematical composition -- eschews the magic and lets the
world be as it is.  We eventually realized that the .NET route is incredibly complex, and requires runtime support. 
-->
换句话说，.NET阵营认为你应该将1个或更多的序列折叠成一个单独的？ 功能语言阵营 - 了解数学作品的优雅 - 避开了魔法，让世界变得如此。 我们最终意识到.NET路由非常复杂，需要运行时支持。

<!-- 
The functional language route does bend your mind slightly at first.  For example, the map example from earlier: 
-->
功能语言路线最初会稍微改变一下你的想法。 例如，前面的地图示例：

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
        // 请注意，`customer`仍然是`Customer？`在这里，仍然可以是`null`！
    } 

<!-- 
In this model, you need to peel off one layer of `?` at a time.  But honestly, when you stop to think about it, that
makes sense.  It's more transparent and reflects precisely what's going on under here.  Best not to fight it. 
-->
在这个模型中，你需要剥掉一层？ 一次。 但老实说，当你停下来思考它时，这是有道理的。 它更透明，并准确反映了这里正在发生的事情。 最好不要打它。

<!-- 
There's also the question of implementation.  The easiest implementation is to expand `T?` into some "wrapper type,"
like `Maybe<T>`, and then inject the appropriate wrap and unwrap operations.  Indeed, that's a reasonable mental model
for how the implementation works.  There are two reasons this simple model doesn't work, however. 
-->
还有实施问题。 最简单的实现是将`T？`扩展为一些“包装类型”，如`Maybe <T>`，然后注入适当的包装和解包操作。 实际上，这是实施工作方式的合理心理模型。 但是，这个简单的模型不起作用有两个原因。

<!-- 
First, for reference type `T`, `T?` must not carry a wasteful extra bit; a pointer's runtime representation can carry
`null` as a value already, and for a systems language, we'd like to exploit this fact and store `T?` as efficiently as
`T`.  This can be done fairly easily by specializing the generic instantiation.  But this does mean that non-null can
no longer simply be a front-end trick.  It requires back-end compiler support. 
-->
首先，对于参考类型T，T？ 一定不要浪费额外的钱; 指针的运行时表示可以携带null作为值，对于系统语言，我们想利用这个事实并存储T？ 与T一样有效。通过专门化泛型实例化，可以相当容易地完成。 但这确实意味着非null不再仅仅是一个前端技巧。 它需要后端编译器支持。

<!-- 
(Note that this trick is not so easy to extend to `T??`!) 
-->
（注意，这个技巧不是那么容易扩展到T ??！）

<!-- 
Second, Midori supported safe covariant arrays, thanks to our mutability annotations.  If `T` and `T?` have a different
physical representation, however, then converting `T[]` to `T?[]` is a non-transforming operation.  This was a minor
blemish, particularly since covariant arrays become far less useful once you plug the safety holes they already have. 
-->
其次，由于我们的可变性注释，Midori支持安全协变数组。 如果T和T？ 然而，具有不同的物理表示，然后将T []转换为T？[]是非变换操作。 这是一个轻微的瑕疵，特别是因为协变阵列在插入已有的安全孔后变得不那么有用了。

<!-- 
Anyway, we eventually burned the ships on .NET `Nullable<T>` and went with the more composable multi-`?` design. 
-->
无论如何，我们最终在.NET Nullable <T>上烧毁了这些船只并使用了更具组合性的多线程？ 设计。

<!-- 
##### Zero-Initialization 
-->
#### 零初始化

<!-- 
Zero-initialization is a real pain in the butt.  To tame it meant: 
-->
零初始化是一个真正的痛苦。 驯服它意味着：

<!-- 
* All non-null fields of a class must be initialized at construction time.
* All arrays of non-null elements must be fully initialized at construction time. 
-->
* 必须在构造时初始化类的所有非空字段。
* 所有非空元素数组必须在构造时完全初始化。
<!-- 
But it gets worse.  In .NET, value types are implicitly zero-initialized.  The initial rule was therefore: 
-->
但它变得更糟。 在.NET中，值类型隐式为零初始化。 因此，最初的规则是：

<!-- 
* All fields of a struct must be nullable. 
-->
* 结构的所有字段都必须是可空的。

<!-- 
But that stunk.  It infected the whole system with nullable types immediately.  My hypothesis was that nullability only
truly works if nullable is the uncommon (say 20%) case.  This would have destroyed that in an instant. 
-->
但那个臭名昭着。 它立即以可空类型感染整个系统。 我的假设是，如果可以为空的是不常见（例如20％）的情况，那么可空性才真正起作用。 这会在瞬间毁掉它。

<!-- 
So we went down the path of eliminating automatic zero-initialization semantics.  This was quite a large change.  (C# 6
went down the path of allowing structs to provide their own zero-arguments constructors and [eventually had to back it
out](https://github.com/dotnet/roslyn/issues/1029) due to the sheer impact this had on the ecosystem.)  It
could have been made to work but veered pretty far off course, and raised some other problems that we probably got too
distracted with.  If I could do it all over again, I'd just eliminate the value vs. reference type distinction
altogether in C#.  The rationale for that'll become clearer in an upcoming post on battling the garbage collector. 
-->
因此，我们沿着消除自动零初始化语义的道路走下去。这是一个很大的变化。 （C＃6走下了允许结构提供自己的零参数构造函数的路径，并最终因为它对生态系统产生了巨大的影响而不得不支持它。）它本可以工作但是偏离了很远的路线，并提出了一些我们可能太分心的问题。如果我可以重新做一遍，我只是在C＃中完全消除了值与引用类型的区别。在即将发布的关于与垃圾收集器作战的文章中，这个理由将变得更加清晰。


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
我们有一个坚实的设计和几个原型，但从未在整个操作系统中部署这个原型。之所以被捆绑在我们期望的C＃兼容性水平上。公平地说，我对这一点感到非常担忧，我想这最终是我的决定。在Midori的早期，我们想要“认知熟悉度”。在项目的后期，我们实际上考虑了是否所有功能都可以作为C＃的“附加”扩展来完成。正是后来的思维模式阻止我们认真地做非空类型。我对今天的信念是，加法注释不起作用; Spec＃尝试了这个！而极性总是感觉倒转。非null必须是默认值才能产生我们想要的影响。

<!-- 
One of my biggest regrets is that we waited so long on non-null types.  We only explored it in earnest once contracts
were a known quantity, and we noticed the thousands of `requires x != null`s all over the place.  It would have been
complex and expensive, however this would have been a particularly killer combination if we nuked the value type
distinction at the same time.  Live and learn! 
-->
我最大的遗憾之一就是我们在非null类型上等了很久。一旦合同成为已知数量，我们才会认真地进行探索，并且我们注意到数千个需要x！= null的地方。它本来是复杂而昂贵的，但如果我们同时确定价值类型的区别，这将是一个特别杀手的组合。活到老，学到老！

<!-- 
If we shipped our language as a standalone thing, different from C# proper, I'm convinced this would have made the cut. 
-->
如果我们把我们的语言作为一个独立的东西运输，不同于C＃本身，我相信这会有所成就。

<!-- 
#### Range Types 
-->
#### 范围类型
<!-- 
We had a design for adding range types to C#, but it always remained one step beyond my complexity limit. 
-->
我们有一个设计用于向C＃添加范围类型，但它总是比我的复杂性限制更进一步。

<!-- 
The basic idea is that any numeric type can be given a lower and upper bound type parameter.  For example, say you had
an integer that could only hold the numbers 0 through 1,000,000, exclusively.  It could be stated as
`int<0..1000000>`.  Of course, this points out that you probably should be using a `uint` instead and the compiler
would warn you.  In fact, the full set of numbers could be conceptually represented as ranges in this way: 
-->
基本思想是任何数字类型都可以给出一个下限和上限类型参数。 例如，假设您有一个整数，只能保存数字0到1,000,000。 它可以表示为int <0..1000000>。 当然，这指出你可能应该使用uint而编译器会警告你。 实际上，完整的数字集可以通过这种方式在概念上表示为范围：

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
    // 等等 ... 

<!-- 
The really "cool" -- but scary complicated -- part is to then use [dependent types](
https://en.wikipedia.org/wiki/Dependent_type) to permit symbolic range parameters.  For example, say I have an array and
want to pass an index whose range is guaranteed to be in-bounds.  Normally I'd write: 
-->
真正“酷” - 但可怕的复杂 - 部分是使用依赖类型来允许符号范围参数。 例如，假设我有一个数组，并希望传递一个索引，其范围保证是入站的。 通常我会写：

    T Get(T[] array, int index)
            requires index >= 0 && index < array.Length {
        return array[index];
    } 

<!-- 
Or maybe I'd use a `uint` to eliminate the first half of the check: 
-->
或者也许我会用'uint`来消除检查的前半部分：

    T Get(T[] array, uint index)
            index < array.Length {
        return array[index];
    } 

<!-- 
Given range types, I can instead associate the upper bound of the number's range with the array length directly:
-->
给定范围类型，我可以直接将数字范围的上限与数组长度相关联：

    T Get(T[] array, number<0, array.Length> index) {
        return array[index];
    } 

<!-- 
Of course, there's no guarantee the compiler will eliminate the bounds check, if you somehow trip up its alias analysis.
But we would hope that it does no worse a job with these types than with normal contracts checks.  And admittedly this
approach is a more direct encoding of information in the type system. 
-->
当然，如果您以某种方式查询其别名分析，则无法保证编译器将消除边界检查。 但我们希望这些类型的工作不会比正常的合同检查更糟糕。 诚然，这种方法是对类型系统中信息的更直接的编码。


<!-- 
Anyway, I still chalk this one up to a cool idea, but one that's still in the realm of "nice to have but not critical." 
-->
无论如何，我仍然把这个想法归结为一个很酷的想法，但是仍然处于“很高兴但没有批评”的领域。

<!-- 
The "not critical" aspect is especially true thanks to slices being first class in the type system.  I'd say 66% or more
of the situations where range checks were used would have been better written using slices.  I think mainly people were
still getting used to having them and so they'd write the standard C# thing rather than just using a slice.  I'll cover
slices in an upcoming post, but they removed the need for writing range checks altogether in most code. 
-->
由于切片在类型系统中是第一类，因此“非关键”方面尤其如此。 我会说66％或更多使用范围检查的情况可以更好地使用切片编写。 我认为主要是人们仍然习惯于拥有它们，因此他们会编写标准的C＃而不仅仅是使用切片。 我将在即将发布的帖子中介绍切片，但他们在大多数代码中都不再需要编写范围检查。

<!-- 
# Recoverable Errors: Type-Directed Exceptions 
-->
# 可恢复的错误：类型导向的异常
<!-- 
Abandonment isn't the only story, of course.  There are still plenty of legitimate situations where an error the
programmer can reasonably recover from occurs.  Examples include: 
-->
当然，放弃不是唯一的故事。 仍然存在大量合法情况，程序员可以合理地从中恢复错误。 例子包括：

<!-- 
* File I/O.
* Network I/O.
* Parsing data (e.g., a compiler parser).
* Validating user data (e.g., a web form submission). 
-->
* 文件I/O
* 网络I / O.
* 解析数据（例如，编译器解析器）。
* 验证用户数据（例如，Web表单提交）。

<!-- 
In each of these cases, you usually don't want to trigger abandonment upon encountering a problem.  Instead, the
program expects it to occur from time to time, and needs to deal with it by doing something reasonable.  Often by
communicating it to someone: the user typing into a webpage, the administrator of the system, the developer using a
tool, etc.  Of course, abandonment is one method call away if that's the most appropriate action to take, but it's often
too drastic for these situations.  And, especially for IO, it runs the risk of making the system very brittle.  Imagine
if the program you're using decided to wink out of existence every time your network connection dropped a packet! 
-->
在每种情况下，您通常不希望在遇到问题时触发放弃。 相反，该程序期望它不时发生，并需要通过做一些合理的事情来处理它。 通常通过将其传达给某人：用户输入网页，系统管理员，开发人员使用工具等等。当然，如果这是最合适的行动，放弃是一种方法，但它通常也是 对这些情况极为激烈。 而且，特别是对于IO，它存在使系统变得非常脆弱的风险。 想象一下，如果您使用的程序在每次网络连接丢弃数据包时决定不再存在！

<!-- 
## Enter Exceptions 
-->
## 输入例外
<!-- 
We used exceptions for recoverable errors.  Not the unchecked kind, and not quite the Java checked kind, either. 
-->
我们使用了可恢复错误的例外情况。 不是那种未经检查的类型，也不是Java检查类型。

<!-- 
First thing's first: although Midori had exceptions, a method that wasn't annotated as `throws` could never throw one.
Never ever ever.  There were no sneaky `RuntimeException`s like in Java, for instance.  We didn't need them anyway,
because the same situations Java used runtime exceptions for were instead using abandonment in Midori. 
-->
第一件事是第一件事：虽然Midori有例外，但是一个没有注释为抛出的方法永远不会抛出一个。永远不会。例如，在Java中没有偷偷摸摸的RuntimeExceptions。我们无论如何都不需要它们，因为Java使用运行时异常的相同情况是在Midori中使用放弃。

<!-- 
This led to a magical property of the result system.  90-something% of the functions in our system could not throw
exceptions!  By default, in fact, they could not.  This was a stark contrast to systems like C++ where you must go out
of your way to abstain from exceptions and state that fact using `noexcept`.  APIs could still fail due to abandonment,
of course, but only when callers fail meet the stated contract, similar to passing an argument of the wrong type. 
-->
这导致了结果系统的神奇属性。我们系统中90％的函数不能抛出异常！事实上，默认情况下，他们不能。这与像C ++这样的系统形成鲜明对比，在这些系统中，你必须不遗余力地避免异常并使用noexcept来陈述这一事实。当然，API仍可能因放弃而失败，但只有当调用者未能满足所述合同时，类似于传递错误类型的参数。

<!-- 
Our choice of exceptions was controversial at the outset.  We had a mixture of imperative, procedural, object oriented,
and functional language perspective on the team.  The C programmers wanted to use error codes and were worried we would
recreate the Java, or worse, C# design.  The functional perspective would be to use dataflow for all errors, but
exceptions were very control-flow-oriented.  In the end, I think what we chose was a nice compromise between all of the
available recoverable error models available to us.  As we'll see later, we did offer a mechanism for treating errors as
first class values for that rare case where a more dataflow style of programming was what the developer wanted. 
-->
我们选择的例外在一开始就存在争议。我们在团队中融合了命令式，程序性，面向对象和功能语言的视角。 C程序员想要使用错误代码，并担心我们会重新创建Java，或者更糟糕的是，C＃设计。功能视角是对所有错误使用数据流，但异常非常以控制流为导向。最后，我认为我们选择的是我们可用的所有可用可恢复错误模型之间的妥协。正如我们稍后将看到的，我们确实提供了一种将错误视为一流值的机制，在这种情况下，更多的数据流编程风格是开发人员想要的。

<!-- 
Most importantly, however, we wrote a lot of code in this model, and it worked very well for us.  Even the functional
language guys came around eventually.  As did the C programmers, thanks to some cues we took from return codes. 
-->
然而，最重要的是，我们在这个模型中编写了很多代码，它对我们来说非常有用。即便是功能语言的人最终也会出现。正如C程序员一样，感谢我们从返回代码中获得的一些线索。

<!-- 
### Language and Type System 
-->
### 语言和类型系统
<!-- 
At some point, I made a controversial observation and decision.  Just as you wouldn't change a function's return type
with the expectation of zero compatibility impact, you should not be changing a function's exception type with such an
expectation.  *In other words, an exception, as with error codes, is just a different kind of return value!* 
-->
在某些时候，我做了一个有争议的观察和决定。正如您不会在期望零兼容性影响的情况下更改函数的返回类型一样，您不应该以这样的期望更改函数的异常类型。换句话说，与错误代码一样，异常只是一种不同的返回值！

<!-- 
This has been one of the parroted arguments against checked exceptions.  My answer may sound trite, but it's simple:
too bad.  You're in a statically typed programming language, and the dynamic nature of exceptions is precisely the
reason they suck.  We sought to address these very problems, so therefore we embraced it, embellished strong typing,
and never looked back.  This alone helped to bridge the gap between error codes and exceptions.
-->
这是针对已检查例外的一个有争议的论点。我的回答可能听起来很陈旧，但很简单：太糟糕了。你是一个静态类型的编程语言，异常的动态性正是他们吮吸的原因。我们试图解决这些问题，因此我们接受了它，点缀了强烈的打字，从不回头。仅这一点就有助于弥合错误代码和异常之间的差距。

<!-- 
Exceptions thrown by a function became part of its signature, just as parameters and return values are.  Remember,
due to the rare nature of exceptions compared to abandonment, this wasn't as painful as you might think.  And a lot of
intuitive properties flowed naturally from this decision. 
-->
函数抛出的异常成为其签名的一部分，就像参数和返回值一样。请记住，由于与放弃相比，例外的罕见性，这并不像你想象的那么痛苦。并且很多直观的属性自然而然地从这个决定中流出。

<!-- 
The first thing is the [Liskov substitution principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle).  In
order to avoid the mess that C++ found itself in, all "checking" has to happen statically, at compile time.  As a
result, all of those performance problems mentioned in [the WG21 paper](
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2010/n3051.html) were not problems for us.  This type system must be
bulletproof, however, with no backdoors to defeat it.  Because we needed to address those performance challenges by
depending on `throws` annotations in our optimizing compiler, type safety hinged on this property. 
-->
第一件事是Liskov替代原则。为了避免C ++发现的混乱，所有“检查”必须在编译时静态发生。因此，WG21文件中提到的所有这些性能问题对我们来说都不是问题。这种类型的系统必须是防弹的，但没有后门可以打败它。因为我们需要通过依赖于优化编译器中的throws注释来解决这些性能挑战，所以类型安全取决于此属性。

<!-- 
We tried many many different syntaxes.  Before we committed to changing the language, we did everything with C#
attributes and static analysis.  The user experience wasn't very good and it's hard to do a real type system that way.
Furthermore, it felt too bolted on.  We experimented with approaches from the Redhawk project -- what eventually became
.NET Native and [CoreRT](https://github.com/dotnet/corert) -- however, that approach also didn't leverage the language
and relied instead on static analysis, though it shares many similar principles with our final solution. 
-->
我们尝试了许多不同的语法。在我们致力于改变语言之前，我们使用C＃属性和静态分析完成了所有工作。用户体验不是很好，很难用这种方式做一个真正的类型系统。此外，它感觉太闩上了。我们尝试了Redhawk项目的方法 - 最终成为.NET Native和CoreRT  - 然而，这种方法也没有利用语言而是依赖于静态分析，尽管它与我们的最终解决方案共享许多类似的原则。

<!-- 
The basic gist of the final syntax was to simply state a method `throws` as a single bit: 
-->
最终语法的基本要点是简单地声明一个方法抛出一个位：

    void Foo() throws {
        ...
    } 

<!-- 
(For many years, we actually put the `throws` at the beginning of the method, but that read wrong.) 
-->
（多年来，我们实际上把'throws`放在方法的开头，但读错了。）

<!-- 
At this point, the issue of substitutability is quite simple.  A `throws` function cannot take the place of a non-
`throws` function (illegal strengthening).  A non-`throws` function, on the other hand, can take the place of a `throws`
function (legal weakening).  This obviously impacts virtual overrides, interface implementation, and lambdas. 
-->
在这一点上，可替代性问题非常简单。 投掷功能不能取代非投掷功能（非法加强）。 另一方面，非投掷函数可以取代抛出函数（合法弱化）。 这显然会影响虚拟覆盖，接口实现和lambda。

<!-- 
Of course, we did the expected co- and contravariance substitution bells and whistles.  For example, if `Foo` were
virtual and you overrode it but didn't throw exceptions, you didn't need to state the `throws` contract.  Anybody
invoking such a function virtually, of course, couldn't leverage this but direct calls could. 
-->
当然，我们做了预期的共同和逆变替代花哨。 例如，如果Foo是虚拟的并且您覆盖它但没有抛出异常，则您不需要声明抛出契约。 当然，虚拟地调用这样一个函数的任何人都无法利用这个功能，但直接调用可以。

<!-- 
For example, this is legal: 
-->
例如，这是合法的：

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
        // 我的特定实现不需要抛出：
        public override void Foo() {...}
    } 

<!-- 
and callers of `Derived` could leverage the lack of `throws`; whereas this is wholly illegal: 
-->
“衍生”的来电者可以利用“抛出”的缺乏; 这完全是非法的：

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
鼓励单一失败模式是相当自由的。 Java检查异常带来的大量复杂性立即消失。如果您查看大多数失败的API，它们无论如何都会有一个失败模式（一旦放弃完成所有错误失败模式）：IO失败，解析失败等等。开发人员倾向于编写的许多恢复操作实际上并不依赖关于在做IO时究竟失败了什么的具体细节。 （有些人会这样做，对于那些人来说，守门员模式通常是更好的答案;不久就会更多关于这个主题。）现代例外中的大多数信息实际上并不适合程序化使用;相反，它们用于诊断。

<!-- 
We stuck with just this "single failure mode" for 2-3 years.  Eventually I made the controversial decision to
support multiple failure modes.  It wasn't common, but the request popped up reasonably often from teammates, and the
scenarios seemed legitimate and useful.  It did come at the expense of type system complexity, but only in all the usual
subtyping ways.  And more sophisticated scenarios -- like aborts (more on that later) -- required that we do this. 
-->
我们坚持这种“单一故障模式”已有2至3年了。最终我做出了有争议的决定，支持多种失败模式。这种情况并不常见，但是这个请求经常会从队友中弹出，这些情景似乎合法且有用。它确实是以类型系统复杂性为代价的，但仅限于所有常见的子类型方式。更复杂的场景 - 比如中止（后面会有更多） - 要求我们这样做。

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
从某种意义上说，单次抛出是抛出异常的捷径。

<!-- 
It was very easy to "forget" the extra detail if you didn't care.  For example, perhaps you wanted to bind a lambda to
the above `Foo` API, but didn't want callers to care about `FooException` or `BarException`.  That lambda must be marked
`throws`, of course, but no more detail was necessary.  This turned out to be a very common pattern: An internal system
would use typed exceptions like this for internal control flow and error handling, but translate all of them into just
plain `throws` at the public boundary of the API, where the extra detail wasn't required. 
-->
如果你不在乎，很容易“忘记”额外的细节。例如，您可能希望将lambda绑定到上面的Foo API，但不希望调用者关心FooException或BarException。当然，lambda必须标记为抛出，但不需要更多细节。这被证明是一种非常常见的模式：内部系统会使用类似的类型异常来进行内部控制流和错误处理，但是将它们全部转换为API的公共边界上的普通抛出，其中额外的细节不是需要。

<!-- 
All of this extra typing added great power to recoverable errors.  But if contracts outnumbered exceptions by 10:1,
then simple `throws` exceptional methods outnumbered multi-failure-mode ones by another 10:1. 
-->
所有这些额外的打字为可恢复的错误增加了很大的力量。但是如果合同的数量超过了10：1的例外，那么简单的抛出异常方法的数量超过了多失效模式的另外10：1。

<!-- 
At this point, you may be wondering, what differentiated this from Java's checked exceptions? 
-->
在这一点上，您可能想知道，这与Java的已检查异常有何区别？

<!-- 
1. The fact that the lion's share of errors were expressed using abandonment meant most APIs didn't throw.

2. The fact that we encouraged a single mode of failure simplified the entire system greatly.  Moreover, we made it easy
   to go from the world of multiple modes, to just a single and back again. 
-->
1. 使用放弃表达大部分错误的事实意味着大多数API都没有抛出。
2. 我们鼓励单一故障模式的事实大大简化了整个系统。此外，我们可以轻松地从多种模式的世界，再到单一的，再次回归。

<!-- 
The rich type system support around weakening and strengthening also helped, as did something else we did to that helped
bridge the gap between return codes and exceptions, improved code maintainability, and more ...
--> 
丰富的类型系统支持弱化和强化也有帮助，我们所做的其他事情也有助于缩小返回代码和异常之间的差距，提高代码可维护性等等......

<!-- 
### Easily Auditable Callsites 
-->
### 易于审核的Callsites
<!-- 
At this point in the story, we still haven't achieved the full explicit syntax of error codes.  The declarations of
functions say whether they can fail (good), but callers of those functions still inherit silent control flow (bad). 
-->
在故事的这一点上，我们仍然没有实现错误代码的完整显式语法。 函数的声明说明它们是否可以失败（好），但这些函数的调用者仍然继承静默控制流（坏）。

<!-- 
This brings about something I always loved about our exceptions model.  A callsite needs to say `try`: 
-->
这带来了我一直喜欢的异常模型。 一个callsite需要说试试：

    int value = try Foo(); 

<!-- 
This invokes the function `Foo`, propagates its error if one occurs, and assigns the return value to `value` otherwise. 
-->
这将调用函数Foo，如果发生错误则传播其错误，否则将返回值赋值给value。

<!-- 
This has a wonderful property: all control flow remains explicit in the program.  You can think of `try` as a kind of
conditional `return` (or conditional `throw` if you prefer).  I *freaking loved* how much easier this made code
reviewing error logic!  For example, imagine a long function with a few `try`s inside of it; having the explicit
annotation made the points of failure, and therefore control flow, as easy to pick out as `return` statements: 
-->
这有一个很棒的属性：所有控制流程在程序中都是明确的。 您可以将try视为一种条件返回（如果您愿意，可以选择条件抛出）。 我非常喜欢这让代码审查错误逻辑变得容易多了！ 例如，想象一个长函数，里面有几个trys; 具有显式注释使得失败点成为控制流，因此易于选择返回语句：

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
        var y = try blah(); // <-- 啊，哈！ 可能会失败的东西！
        blahdiblahdiblahdiblahdi();
        blahblahblahblah(try blahblah()); // <-- 另一个！
        and_so_on(...);
    } 

<!-- 
If you have syntax highlighting in your editor, so the `try`s are bold and blue, it's even better. 
-->
如果你的编辑器中有语法高亮，那么`try`s是粗体和蓝色的，它甚至更好。

<!-- 
This delivered many of the strong benefits of return codes, but without all the baggage. 
-->
这提供了许多返回代码的强大好处，但没有所有的包袱。

<!-- 
(Both Rust and Swift now support a similar syntax.  I have to admit I'm sad we didn't ship this to the general public
years ago.  Their implementations are very different, however consider this a huge vote of confidence in their syntax.) 
-->
（Rust和Swift现在都支持类似的语法。我不得不承认，我很难过几年前我们没有将它发送给普通大众。他们的实现非常不同，但是考虑到这对他们的语法有很大的信心。）

<!-- 
Of course, if you are `try`ing a function that throws like this, there are two possibilities: 
-->
当然，如果你正在尝试这样抛出的函数，有两种可能：

<!-- 
* The exception escapes the calling function.
* There is a surrounding `try`/`catch` block that handles the error. 
-->
* 异常转义了调用函数。
* 周围有一个`try` /`catch`块来处理错误。

<!-- 
In the first case, you are required to declare that your function `throws` too.  It is up to you whether to propagate
strong typing information should the callee declare it, or simply leverage the single `throws` bit, of course. 
-->
在第一种情况下，您需要声明您的函数也会抛出。 当被调用者声明它时，是否传播强类型信息，或者只是利用单个投掷位，由你来决定。


<!-- 
In the second case, we of course understood all the typing information.  As a result, if you tried to catch something
that wasn't declared as being thrown, we could give you an error about dead code.  This was yet another controversial
departure from classical exceptions systems.  It always bugged me that `catch (FooException)` is essentially hiding a
dynamic type test.  Would you silently permit someone to call an API that returns just `object` and automatically assign
that returned value to a typed variable?  Hell no!  So we didn't let you do that with exceptions either. 
-->
在第二种情况下，我们当然理解所有的输入信息。 因此，如果您尝试捕获未声明被抛出的内容，我们可能会给您一个关于死代码的错误。 这是与传统例外系统的另一个有争议的背离。 总是告诉我catch（FooException）本质上隐藏了动态类型测试。 您是否会默默地允许某人调用仅返回对象的API并自动将返回的值分配给类型变量？ 一定不行！ 因此，除了例外，我们也没有让你这样做。

<!--
Here too CLU influenced us.  Liskov talks about this in [A History of CLU](
http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.127.8460&rep=rep1&type=pdf):
-->
CLU也影响了我们。 Liskov在CLU的历史中谈到这个：

<!--
> CLU's mechanism is unusual in its treatment of unhandled exceptions. Most mechanisms pass these through: if the caller
> does not handle an exception raised by a called procedure, the exception is propagated to its caller, and so on. We
> rejected this approach because it did not fit our ideas about modular program construction. We wanted to be able to
> call a procedure knowing just its specification, not its implementation. However, if exceptions are propagated
> automatically, a procedure may raise an exception not described in its specification. 
-->
> CLU的机制在处理未处理的异常方面是不寻常的。 大多数机制都通过这些机制：如果调用者没有处理被调用过程引发的异常，则异常将传播给其调用者，依此类推。 我们拒绝这种方法，因为它不符合我们关于模块化程序构建的想法。 我们希望能够只知道其规范而不是其实现来调用过程。 但是，如果异常自动传播，则过程可能会引发其规范中未描述的异常。

<!-- 
Although we discouraged wide `try` blocks, this was conceptually a shortcut for propagating an error code.  To see what
I mean, consider what you'd do in a system with error codes.  In Go, you might say the following: 
-->
虽然我们不鼓励广泛的尝试块，但这在概念上是传播错误代码的捷径。 要了解我的意思，请考虑在具有错误代码的系统中执行的操作。 在Go中，您可能会说以下内容：

    if err := doSomething(); err != nil {
        return err
    } 

<!-- 
In our system, you say: 
-->
在我们的系统中，您说：

    try doSomething(); 

<!-- 
But we used exceptions, you might say!  It's completely different!  Sure, the runtime systems differ.  But from a
language "semantics" perspective, they are isomorphic.  We encouraged people to think in terms of error codes and not
the exceptions they knew and loved.  This might seem funny: Why not just use return codes, you might wonder?  In an
upcoming section, I will describe the true isomorphism of the situation to try to convince you of our choice. 
-->
但你可能会说，我们使用例外！ 这完全不同！ 当然，运行时系统不同。 但从语言“语义学”的角度来看，它们是同构的。 我们鼓励人们根据错误代码进行思考，而不是他们所熟悉和喜爱的例外情况。 这可能看起来很有趣：为什么不使用返回码，你可能想知道？ 在接下来的部分中，我将描述情境的真正同构，试图说服你我们的选择。

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
我们还提供了一些处理错误的语法糖。 try / catch块作用域结构有点冗长，特别是如果您遵循我们在本地处理错误的预期最佳实践。 对于某些人来说，它仍然保留了一些不幸的goto感觉，特别是如果你正在考虑返回代码。 这让位于我们称为Result <T>的类型，它只是一个T值或一个Exception。

<!-- 
This essentially bridged from the world of control-flow to the world of dataflow, for scenarios in which the latter was
more natural.  Both certainly had their place, although most developers preferred the familiar control flow syntax. 
-->
这基本上是从控制流世界到数据流世界的桥梁，对于后者更自然的场景。 虽然大多数开发人员更喜欢熟悉的控制流语法，但两者肯定都有自己的位

<!-- 
To illustrate common usage, imagine you want to log all errors that occur, before repropagating the exception.  Though
this is a common pattern, using `try`/`catch` blocks feels a little too control flow heavy for my taste: 
-->
为了说明常见用法，假设您希望在重新传播异常之前记录发生的所有错误。 虽然这是一种常见的模式，但使用try / catch模块会让我觉得有点过于控制流量：

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
        // 也许更多的东西...
    }
    catch (Exception e) {
        Log(e);
        rethrow;
    }
    // 使用值“v” ... 

<!-- 
The "maybe some more stuff" bit entices you to squeeze more than you should into the `try` block.  Compare this to using
`Result<T>`, leading to a more return-code feel and more convenient local handling: 
-->
“也许更多的东西”有点吸引你挤进比try块更多的东西。 
将此与使用Result <T>进行比较，从而产生更多的返回代码感觉和更方便的本地处理：

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
try ... else构造还允许您替换自己的值，甚至触发放弃，以响应失败：

    int value1 = try Foo() else 42;
    int value2 = try Foo() else Release.Fail(); 

<!-- 
We also supported NaN-style propagation of dataflow errors by lifting access to `T`s members out of the `Result<T>`.
For example, let's say I have two `Result<int>`s and want to add them together.  I can do so:
-->
我们还通过从Result <T>中提升对Ts成员的访问来支持NaN样式的数据流错误传播。 例如，假设我有两个Result <int>，并希望将它们一起添加。 我可以这样做：

    Result<int> x = ...;
    Result<int> y = ...;
    Result<int> z = x + y; 

<!-- 
Notice that third line, where we added the two `Result<int>`s together, yielding a -- that's right -- third `Result<T>`.
This is the NaN-style dataflow propagation, similar to C#'s new `.?` feature. 
-->
注意第三行，我们将两个Result <int>一起添加到一起，产生一个 - 那是正确的 - 第三个结果<T>。 这是NaN风格的数据流传播，类似于C＃的新增功能。 特征。

<!-- 
This approach blends what I found to be an elegant mixture of exceptions, return codes, and dataflow error propagation. 
-->
这种方法融合了我发现的异常，返回代码和数据流错误传播的优雅混合。

<!-- 
## Implementation 
-->
## 实现

<!-- 
The model I just described doesn't have to be implemented with exceptions.  It's abstract enough to be reasonably
implemented using either exceptions or return codes.  This isn't theoretical.  We actually tried it.  And this is what
led us to choose exceptions instead of return codes for performance reasons. 
-->
我刚才描述的模型不必用例外实现。 它足够抽象，可以使用异常或返回代码合理地实现。 
这不是理论上的。 我们实际上尝试过。 这是导致我们出于性能原因选择异常而不是返回代码的原因。

<!-- 
To illustrate how the return code implementation might work, imagine some simple transformations: 
-->
为了说明返回代码实现如何工作，想象一些简单的转换：

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
代码如下：

    int x = try foo(); 

<!-- 
becomes something more like this: 
-->
变成这样的东西：

    int x;
    Result<int> tmp = foo();
    if (tmp.Failed) {
        throw tmp.Exception;
    }
    x = tmp.Value; 

<!-- 
An optimizing compiler can represent this more efficiently, eliminating excessive copying.  Especially with inlining. 
-->
优化编译器可以更有效地表示这一点，消除了过多的复制。 特别是内联。

<!-- 
If you try to model `try`/`catch`/`finally` this same way, probably using `goto`, you'll quickly see why compilers have
a hard time optimizing in the presence of unchecked exceptions.  All those hidden control flow edges! 
-->
如果你尝试以同样的方式模拟`try` /`catch` /`finally`，可能使用`goto`，你会很快看到为什么编译器有
存在未经检查的异常时很难进行优化。 所有那些隐藏的控制流边缘！

<!-- 
Either way, this exercise very vividly demonstrates the drawbacks of return codes.  All that goop -- which is meant to
be rarely needed (assuming, of course, that failure is rare) -- is on hot paths, mucking with your program's golden path
performance.  This violates one of our most important principles. 
-->
无论哪种方式，这个练习都非常生动地展示了返回码的缺点。 所有那些goop  - 这是很少需要的（假设，当然，失败是罕见的） - 是在热门的道路上，与你的程序的黄金路径表现相混淆。 这违反了我们最重要的原则之一。

<!-- 
I described the results of our dual mode experiment in [my last post](
http://joeduffyblog.com/2015/12/19/safe-native-code/).  In summary, the exceptions approach was 7% smaller and 4%
faster as a geomean across our key benchmarks, thanks to a few things: 
-->
我在上一篇文章中描述了双模式实验的结果。 总之，由于以下几点，例外方法在我们的关键基准测试中作为几何图形缩小了7％，速度提高了4％：

<!-- 
* No calling convention impact.
* No peanut butter associated with wrapping return values and caller branching.
* All throwing functions were known in the type system, enabling more flexible code motion.
* All throwing functions were known in the type system, giving us novel EH optimizations, like turning try/finally
  blocks into straightline code when the try could not throw. 
-->
* 没有呼叫约定的影响。
* 没有花生酱与包装返回值和调用者分支相关联。
* 所有抛出函数在类型系统中都是已知的，从而实现更灵活的代码运动。
* 所有抛出函数在类型系统中都是已知的，为我们提供了新颖的EH优化，例如在try无法抛出时将try / finally块转换为直线代码。

<!-- 
There were other aspects of exceptions that helped with performance.  I already mentioned that we didn't grovel the
callstack gathering up metadata as most exceptions systems do.  We left diagnostics to our diagnostics subsystem.
Another common pattern that helped, however, was to cache exceptions as frozen objects, so that each `throw` didn't
require an allocation: 
-->
还有其他方面的例外有助于提高绩效。 我已经提到过，我们并没有像大多数异常系统那样使用callstack来收集元数据。 我们将诊断程序留给了诊断子系统。 然而，另一个常见的模式是将异常缓存为冻结对象，因此每次抛出都不需要分配：

    const Exception retryLayout = new Exception();
    ...
    throw retryLayout; 

<!-- 
For systems with high rates of throwing and catching -- as in our parser, FRP UI framework, and other areas -- this was
important to good performance.  And this demonstrates why we couldn't simply take "exceptions are slow" as a given.
-->
对于具有高投掷和捕获率的系统 - 如我们的解析器，FRP UI框架和其他领域 - 这对于良好的性能非常重要。 这说明了为什么我们不能简单地将“异常缓慢”视为给定。

<!-- 
## Patterns 
-->
## 模式

<!-- 
A number of useful patterns came up that we embellished in our language and libraries. 
-->
我们在语言和图书馆中修饰了许多有用的模式。
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
早在2007年，我就写了关于并发和异常的说明。我主要是从并行共享内存计算的角度编写它，但是所有并发编排模式都存在类似的挑战。基本问题是实现异常的方式假设单个，顺序堆栈，具有单一故障模式。在并发系统中，您有许多堆栈和许多故障模式，其中0,1或多个可能“同时”发生。

<!-- 
A simple improvement that Midori made was simply ensuring all `Exception`-related infrastructure handled cases with
multiple inner errors.  At least then a programmer wasn't forced to decide to toss away 1/N'th of the failure
information, as most exceptions systems encourage today.  More than that, however, our scheduling and stack crawling
infrastructure fundamentally knew about cactus-style stacks, thanks to our asynchronous model, and what to do with them. 
-->
Midori所做的一个简单改进就是确保所有与Exception相关的基础架构处理具有多个内部错误的案例。至少那时程序员没有被迫决定抛弃失败信息的1 / N，正如大多数例外系统今天所鼓励的那样。然而，更重要的是，由于我们的异步模型以及如何处理它们，我们的调度和堆栈爬行基础架构从根本上了解了仙人掌式堆栈。

<!-- 
At first, we didn't support exceptions across asynchronous boundaries.  Eventually, however, we extended the ability to
declare `throws`, along with optional typed exceptions clauses, across asynchronous process boundaries.  This brought a
rich, typed programming model to the asynchronous actors programming model and felt like a natural extension.  This
borrowed a page from CLU's successor, [Argus](https://en.wikipedia.org/wiki/Argus_(programming_language)). 
-->
起初，我们不支持跨异步边界的异常。但是，最终，我们扩展了跨异步进程边界声明throws以及可选类型化例外子句的能力。这为异步演员编程模型带来了丰富的类型化编程模型，感觉就像是一个自然的扩展。这借了CLU的继任者阿古斯的一页。

<!-- 
Our diagnostics infrastructure embellished this to give developers debugging experiences with full-blown cross-process
causality in their stack views.  Not only are stacks cactuses in a highly concurrent system, but they are often smeared
across process message passing boundaries.  Being able to debug the system this way was a big time-saver. 
-->
我们的诊断基础设施对其进行了修饰，以便为开发人员在堆栈视图中提供全面的跨进程因果关系调试体验。堆栈cactuses不仅在高度并发的系统中，而且它们经常在流程消息传递边界上涂抹。能够以这种方式调试系统可以节省大量时间。

<!-- 
### Aborts 
-->
### 中止

<!-- 
Sometimes a subsystem needs to "get the hell out of Dodge."  Abandonment is an option, but only in response to bugs.
And of course nobody in the process can stop it in its tracks.  What if we want to back out the callstack to some point,
know that no-one on the stack is going to stop us, but then recover and keep going within the same process? 
-->
有时一个子系统需要“彻底摆脱道奇。”放弃是一种选择，但只是为了应对错误。当然，在这个过程中没有人可以阻止它。如果我们想要在某个时候退出callstack，知道堆栈中没有人会阻止我们，但是然后恢复并继续在同一个进程中运行会怎么样？

<!-- 
Exceptions were close to what we wanted here.  But unfortunately, code on the stack can catch an in-flight exception,
thereby effectively suppressing the abort.  We wanted something unsuppressable. 
-->
例外情况接近我们想要的。但不幸的是，堆栈上的代码可以捕获飞行中的异常，从而有效地抑制中止。我们想要一些不可抑制的东西。

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
输入中止。我们发明了中止主要是为了支持我们使用功能反应式编程（FRP）的UI框架，尽管这种模式出现在一些地方。当FRP重新计算正在进行时，事件可能会进入系统，或者发现新的发现，从而使当前的重新计算无效。如果发生这种情况 - 通常深度在某些计算中，其堆栈是用户和系统代码的交错 -  FRP引擎需要快速返回其堆栈顶部，以便可以安全地开始重新计算。由于堆栈上的所有用户代码在功能上都是纯粹的，因此在流中流中止很容易。不会留下任何错误的副作用。并且由于类型异常，所有遍历的引擎代码都经过审核和彻底修改，以确保维护不变量。

<!-- 
The abort design borrows a page from the [capability playbook](
http://joeduffyblog.com/2015/11/10/objects-as-secure-capabilities/).  First, we introduce a base type called
`AbortException`.  It may be used directly or subclassed.  One of these is special: nobody can catch-and-ignore it.  The
exception is reraised automatically at the end of any catch block that attempts to catch it.  We say that such
exceptions are *undeniable*. 
-->
中止设计借用了能力手册中的页面。首先，我们介绍一种名为AbortException的基类型。它可以直接使用或子类化。其中一个是特殊的：没有人可以捕捉并忽视它。在尝试捕获它的任何catch块的末尾自动重新引发该异常。我们说这种例外是不可否认的。

<!-- 
But someone's got to catch an abort.  The whole idea is to exit a context, not tear down the entire process a la
abandonment.  And here's where capabilities enter the picture.  Here's the basic shape of `AbortException`: 
-->
但有人必须抓住中止。整个想法是退出一个环境，而不是放弃整个过程。这是功能进入图片的地方。这是AbortException的基本形状：

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
        // 其他无趣的成员省略了...
    } 

<!-- 
Notice that, at the time of construction, an immutable `token` is provided; in order to suppress the throw, `Reset` is
called, and a matching `token` must be provided.  If the `token` doesn't match, abandonment occurs.  The idea is that
the throwing and intended catching parties of an abort are usually the same, or at least in cahoots with one another,
such that sharing the `token` securely with one another is easy to do.  This is a great example of objects as
unforgeable capabilities in action.
-->
请注意，在构造时，提供了不可变的令牌;为了抑制抛出，调用Reset，并且必须提供匹配的标记。如果令牌不匹配，则发生放弃。这个想法是，中止的投掷和预期的捕获方通常是相同的，或者至少是彼此相互关联，这样可以容易地彼此安全地共享令牌。这是对象的一个​​很好的例子，它具有不可伪造的功能。

<!-- 
And yes, an arbitrary piece of code on the stack can trigger an abandonment, but such code could already do that by
simply dereferencing `null`.  This technique prohibits executing in the aborting context when it might not have been
ready for it. 
-->
是的，堆栈上的任意一段代码都可以触发放弃，但是这样的代码已经可以通过简单地解除引用null来实现。该技术禁止在可能尚未准备好的情况下在中止上下文中执行。

<!-- 
Other frameworks have similar patterns.  The .NET Framework has `ThreadAbortException` which is also undeniable unless
you invoke `Thread.ResetAbort`; sadly, because it isn't capability-based, a clumsy combination of security annotations
and hosting APIs are required to stop unintended swallowing of aborts.  More often, this goes unchecked. 
-->
其他框架具有类似的模式。 .NET Framework具有ThreadAbortException，除非您调用Thread.ResetAbort，否则它也是不可否认的;遗憾的是，由于它不是基于功能的，因此需要使用安全注释和托管API的笨拙组合来阻止意外吞咽中止。更常见的是，这是不受控制的。

<!-- 
Thanks to exceptions being immutable, and the `token` above being immutable, a common pattern was to cache these guys
in static variables and use singletons.  For example: 
-->
由于异常是不可变的，并且上面的标记是不可变的，因此常见的模式是将这些人缓存在静态变量中并使用单例。例如：

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
                // 做一些深入调用一些callstacks的东西;
                // 在内心深处它可能会中止，我们捕获并重置：
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
这种模式使中止非常有效。 平均FRP重新计算多次中止。 
请记住，FRP是系统中所有UI的主干，因此通常归因于异常的缓慢显然是不可接受的。 
由于随之而来的GC压力，即使分配异常对象也是不幸的。

<!-- 
### Opt-in "Try" APIs 
-->
### 选择“尝试”API
<!-- 
I mentioned a number of operations that abandoned upon failure.  That included allocating memory, performing arithmetic
operations that overflowed or divided-by-zero, etc.  In a few of these instances, a fraction of the uses are appropriate
for dynamic error propagation and recovery, rather than abandonment.  Even if abandonment is better in the common case. 
-->
我提到了一些在失败时放弃的操作。 这包括分配内存，执行溢出或被零除的算术运算等。在其中一些实例中，一小部分用途适用于动态错误传播和恢复，而不是放弃。 即使在普通情况下放弃更好。

<!-- 
This turned out to be a pattern.  Not terribly common, but it came up.  As a result, we had a whole set of arithmetic
APIs that used a dataflow-style of propagation should overflow, NaN, or any number of things happen. 
-->
结果证明这是一种模式。 不是非常普遍，但它出现了。 因此，我们有一整套算术API，它们使用数据流传播方式，如果溢出，NaN或任何数量的事情发生。

<!-- 
I also already mentioned a concrete instance of this earlier, which is the ability to `try new` an allocation, when OOM
yields a recoverable error rather than abandonment.  This was super uncommon, but could crop up if you wanted to, say,
allocate a large buffer for some multimedia operation. 
-->
我之前已经提到了一个具体的实例，即当OOM产生可恢复的错误而不是放弃时，能够尝试新的分配。 这种情况非常罕见，但如果你想为某些多媒体操作分配一个大缓冲区，可能会突然出现。

<!-- 
### Keepers 
-->
### 守护者
<!-- 
The last pattern I'll cover is called *the keeper pattern*. 
-->
我将介绍的最后一个模式叫做守护者模式*。

<!-- 
In a lot of ways, the way recoverable exceptions are handled is "inside out."  A bunch of code is called, passing
arguments down the callstack, until finally some code is reached that deems that state unacceptable.  In the exceptions
model, control flow is then propagated back up the callstack, unwinding it, until some code is found that handles the
error.  At that point if the operation is to be retried, the sequence of calls must be reissued, etc. 
-->
在很多方面，处理可恢复异常的方式是“由内而外”。调用一堆代码，在callstack中传递参数，直到最后达到一些认为该状态不可接受的代码。在异常模型中，控制流然后在callstack中向上传播，展开它，直到找到一些处理错误的代码。此时，如果要重试该操作，则必须重新发出呼叫序列等。

<!-- 
An alternative pattern is to use a keeper.  The keeper is an object that understands how to recover from errors "in
situ," so that the callstack needn't be unwound.  Instead, the code that would have otherwise thrown an exception
consults the keeper, who instructs the code how to proceed.  A nice aspect of keepers is that often, when done as a
configured capability, surrounding code doesn't even need to know they exist -- unlike exceptions which, in our system,
had to be declared as part of the type system.  Another aspect of keepers is that they are simple and cheap. 
-->
另一种模式是使用守门员。守护者是一个了解如何“原位”从错误中恢复的对象，因此不需要解开callstack。相反，否则会抛出异常的代码会咨询守护者，守护者指示代码如何继续。守护者的一个很好的方面是，当作为配置的功能完成时，周围的代码甚至不需要知道它们存在 - 不像在我们的系统中必须被声明为类型系统的一部分的异常。守护者的另一个方面是它们简单而便宜。

<!-- 
Keepers in Midori could be used for prompt operations, but more often spanned asynchronous boundaries. 
-->
Midori中的守护者可用于快速操作，但更常见的是跨越异步边界。

<!-- 
The canonical example of a keeper is one guarding filesystem operations.  Accessing files and directories on a file
system typically has failure modes such as: 
-->
守护者的规范示例是保护文件系统操作。访问文件系统上的文件和目录通常具有以下故障模式：

<!-- 
* Invalid path specification.
* File not found.
* Directory not found.
* File in use.
* Insufficient privileges.
* Media full.
* Media write-protected. 
-->
* 无效的路径规范。
* 文件未找到。
* 找不到目录。
* 文件正在使用中。
* 权限不足。
* 媒体已满。
* 媒体写保护。

<!-- 
One option is to annotate each filesystem API with a `throws` clause for each.  Or, like Java, to create an
`IOException` hierarchy with each of these as subclasses.  An alternative is to use a keeper.  This ensures the overall
application doesn't need to know or care about IO errors, permitting recovery logic to be centralized.  Such a keeper
interface might look like this: 
-->
一种选择是使用throws子句为每个文件系统API注释。 或者，像Java一样，创建一个IOException层次结构，每个层次结构都作为子类。 另一种方法是使用守门员。 这可确保整个应用程序不需要知道或关心IO错误，从而允许集中恢复逻辑。 这样的守护者界面可能如下所示：

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
这个想法是，在每种情况下，当发生故障时，相关输入被提供给保持器。然后允许守护者执行可能异步的操作以进行恢复。在许多情况下，守护者可以选择返回操作的更新参数。例如，InsufficientPrivileges可以返回要使用的备用凭据。 （也许程序提示用户并且她切换到具有写访问权限的帐户。）在显示的每种情况下，如果管理员不想处理错误，则可以抛出异常，尽管该部分模式是可选的。

<!-- 
Finally, I should note that Windows's Structured Exception Handling (SEH) system supports "continuable" exceptions which
are conceptually attempting to achieve this same thing.  They let some code decide how to restart the faulting
computation.  Unfortunately, they're done using ambient handlers on the callstack, rather than first class objects in
the language, and so are far less elegant -- and significantly more error prone -- than the keepers pattern. 
-->
最后，我应该注意到Windows的结构化异常处理（SEH）系统支持“可持续”异常，这些异常在概念上试图实现同样的目的。他们让一些代码决定如何重新启动错误计算。不幸的是，它们是在callstack上使用环境处理程序完成的，而不是语言中的第一类对象，因此远不如守护者模式那么优雅 - 而且更容易出错。

<!-- 
### Future Directions: Effect Typing 
-->
### 未来方向：效果打字
<!-- 
Most people asked us about whether having `async` and `throws` as type system attributes bifurcated the entire universe
of libraries.  The answer was "No, not really."  But it sure was painful in highly polymorphic library code. 
-->
大多数人问我们是否将async和throws作为类型系统属性分叉整个库的整个世界。 答案是“不，不是真的。”但在高度多态的库代码中肯定是痛苦的。

<!-- 
The most jarring example was combinators like map, filter, sort, etc.  In those cases, you often have arbitrary
functions and want the `async` and `throws` attributes of those functions to "flow through" transparently. 
-->
最令人震惊的例子是组合，如map，filter，sort等。在这些情况下，你经常有任意函数，并希望这些函数的async和throws属性透明地“流过”。

<!-- 
The design we had to solve this was to let you parameterize over effects.  For instance, here is a universal mapping
function, `Map`, that propagates the `async` or `throws` effect of its `func` parameter: 
-->
我们必须解决的设计是让你参数化效果。 例如，这是一个通用映射函数Map，它传播其func参数的异步或抛出效果：

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
请注意，我们有一个普通的泛型类型E，除了它的声明以关键字effect为前缀。 然后我们象征性地使用E来代替Map签名的效果列表，除了在调用func时通过效果（E）在“传播”位置使用它。 这是一个非常简单的替代练习，用try替换E和效果（E），看看逻辑转换。

<!-- 
A legal invocation might be: 
-->
法律援引可能是：

    int[] xs = ...;
    string[] ys = try Map<int, string, throws>(xs, x => ...); 

<!-- 
Notice here that the `throws` flows through, so that we can pass a callback that throws exceptions. 
-->
请注意，`throws`会流过，这样我们就可以传递一个抛出异常的回调。

<!-- 
As a total aside, we discussed taking this further, and allowing programmers to declare arbitrary effects.  I've
[hypothesized about such a type system previously](
http://joeduffyblog.com/2010/04/25/from-simple-annotations-for-analysis-to-effect-typing/).  We were concerned, however,
that this sort of higher order programming might be gratuitously clever and hard to understand, no matter how powerful.
The simple model above probably would've been a sweet spot and I think we'd have done it given a few more months. 
-->
总而言之，我们讨论了进一步，并允许程序员声明任意效果。我以前曾经假设过这种类型的系统。然而，我们担心的是，无论多么强大，这种高阶编程都可能是无懈可击且难以理解的。上面的简单模型可能会是一个甜蜜点，我想我们已经做了几个月。

<!-- 
# Retrospective and Conclusions 
-->
# 回顾与结论

<!-- 
We've reached the end of this particular journey.  As I said at the outset, a relatively predictable and tame outcome.
But I hope all that background helped to take you through the evolution as we sorted through the landscape of errors. 
-->
我们已经走完了这个特殊的旅程。正如我在一开始所说，一个相对可预测和温和的结果。但是我希望所有这些背景都能帮助你完成进化过程，因为我们对错误的情况进行了分类。

<!-- 
In summary, the final model featured: 
-->
总之，最终的模型特色：

<!-- 
* An architecture that assumed fine-grained isolation and recoverability from failure.
* Distinguishing between bugs and recoverable errors.
* Using contracts, assertions, and, in general, abandonment for all bugs.
* Using a slimmed down checked exceptions model for recoverable errors, with a rich type system and language syntax.
* Adopting some limited aspects of return codes -- like local checking -- that improved reliability. 
-->
* 一种体系结构，它承担了故障时的细粒度隔离和可恢复性。
* 区分错误和可恢复的错误。
* 使用合同，断言，以及一般而言，放弃所有错误。
* 使用精简的向下检查异常模型来获取可恢复的错误，使用丰富的类型系统和语言语法。
* 采用返回代码的某些有限方面 - 如本地检查 - 提高了可靠性。

<!-- 
And, though this was a multi-year journey, there were areas of improvement we were actively working on right up until
our project's untimely demise.  I classified them differently because we didn't have enough experience using them to
claim success.  I would have hoped we'd have tidied up most of them and shipped them if we ever got that far.  In
particular, I'd have liked to put this one into the final model category: 
-->
而且，虽然这是一个多年的旅程，但我们正在积极努力，直到我们的项目不合时宜地消亡。我对它们进行了不同的分类，因为我们没有足够的经验使用它们来宣称成功。如果我们走得那么远，我希望我们已经整理了大部分并运送了它们。特别是，我想把这个放到最终的模型类别中：

<!-- 
* Leveraging non-null types by default to eliminate a large class of nullability annotations. 
-->
* 默认情况下，利用非null类型可以消除大量的可空性注释。

<!-- 
Abandonment, and the degree to which we used it, was in my opinion our biggest and most successful bet with the Error
Model.  We found bugs early and often, where they are easiest to diagnose and fix.  Abandonment-based errors outnumbered
recoverable errors by a ratio approaching 10:1, making checked exceptions rare and tolerable to the developer. 
-->
放弃，以及我们使用它的程度，在我看来是我们在错误模型中最大和最成功的赌注。我们很早就经常发现错误，它们最容易诊断和修复。基于放弃的错误数量超过可恢复错误的比例接近10：1，使得检查异常很少并且可以被开发人员容忍。


<!-- 
Although we never had a chance to ship this, we have since brought some of these lessons learned to other settings. 
-->
虽然我们从来没有机会发货，但我们已经将其中的一些经验教训带到了其他环境中。

<!-- 
During the Microsoft Edge browser rewrite from Internet Explorer, for example, we adopted abandonment in a few areas.
The key one, applied by a Midori engineer, was OOM.  The old code would attempt to limp along as I described earlier
and almost always did the wrong thing.  My understanding is that abandonment has found numerous lurking bugs, as was our
experience regularly in Midori when porting existing codebases.  The great thing too is that abandonment is more of an
architectural discipline that can be adopted in existing code-bases ranging in programming languages. 
-->
例如，在从Internet Explorer重写Microsoft Edge浏览器期间，我们在一些领域采用了放弃。由Midori工程师应用的关键一个是OOM。如前所述，旧代码会试图跛行，几乎总是做错了。我的理解是放弃已经发现了许多潜伏的错误，就像我们在移植现有代码库时经常在Midori中经历的那样。最重要的是，放弃更多的是一种建筑学科，可以在编程语言的现有代码库中采用。

<!-- 
The architectural foundation of fine-grained isolation is critical, however many systems have an informal notion of this
architecture.  A reason why OOM abandonment works well in a browser is that most browsers devote separate processes to
individual tabs already.  Browsers mimic operating systems in many ways and here too we see this playing out. 
-->
细粒度隔离的架构基础至关重要，但许多系统都有这种架构的非正式概念。 OOM放弃在浏览器中运行良好的原因是大多数浏览器已经将单独的进程专用于各个选项卡。浏览器以多种方式模仿操作系统，在这里我们也看到了这种情况。

<!-- 
More recently, we've been exploring proposals to bring some of this discipline -- [including contracts](
https://www.youtube.com/watch?v=Hjz1eBx91g8) -- to C++.  There are also [concrete proposals](
https://github.com/dotnet/roslyn/issues/119) to bring some of these features to C# too.  We are actively iterating on
[a proposal that would bring some non-null checking to C#](https://github.com/dotnet/roslyn/issues/5032).  I have to
admit, I wish all of those proposals the best, however nothing will be as bulletproof as an entire stack written in the
same error discipline.  And remember, the entire isolation and concurrency model is essential for abandonment at scale. 
-->
最近，我们一直在探索将这些学科（包括合同）带入C ++的建议。还有一些具体的建议将这些功能带到C＃中。我们正在积极地迭代一个会给C＃带来一些非空检查的提议。我不得不承认，我希望所有这些提案都是最好的，但是没有什么能像在同一个错误规则中编写的整个堆栈那样具有防弹性。请记住，整个隔离和并发模型对于大规模放弃至关重要。

<!-- 
I am hopeful that continued sharing of knowledge will lead to even more wide-scale adoption some of these ideas. 
-->
我希望继续分享知识将导致更广泛地采用其中的一些想法。

<!-- 
And, of course, I've mentioned that Go, Rust, and Swift have given the world some very good systems-appropriate error
models in the meantime.  I might have some minor nits here and there, but the reality is that they're worlds beyond
what we had in the industry at the time we began the Midori journey.  It's a good time to be a systems programmer! 
-->
当然，我已经提到Go，Rust和Swift在此期间为世界提供了一些非常好的系统适当的错误模型。我可能会在这里和那里有一些小小的痘痘，但现实是，在我们开始Midori之旅时，它们的世界已经超出了我们在行业中所拥有的世界。现在是成为系统程序员的好时机！

<!-- 
Next time I'll talk more about the language.  Specifically, we'll see how Midori was able to tame the garbage collector
using a magical elixir of architecture, language support, and libraries.  I hope to see you again soon! 
-->
下次我会更多地谈论这门语言。具体来说，我们将看到Midori如何使用一种神奇的建筑，语言支持和库的灵丹妙药来驯服垃圾收集器。我希望很快能在见到你！
