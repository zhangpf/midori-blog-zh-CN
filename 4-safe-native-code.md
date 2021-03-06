---
title: Midori博客系列翻译（4）——安全的原生代码
date: 2019-2-17 14:46:00
tags: [操作系统, Midori, 翻译, 安全, 编译]
categories: 中文
---

<!-- 
In my [first Midori post](http://joeduffyblog.com/2015/11/03/a-tale-of-three-safeties/), I described how safety was the
foundation of everything we did.  I mentioned that we built an operating system out of safe code, and yet stayed
competitive with operating systems like Windows and Linux written in C and C++.  In many ways, system architecture
played a key role, and I will continue discussing how in future posts.  But, at the foundation, an optimizing compiler
that often eeked out native code performance from otherwise "managed", type- and memory-safe code, was one of our most
important weapons.  In this post, I'll describe some key insights and techniques that were essential to our success. -->

在我的[第一篇Midori文章](/2018/10/24/midori/1-a-tale-of-three-safeties/)中，
我描述了安全是我们所做的一切的基础。 
我提到我们使用安全代码构建了操作系统，
但仍然保持与使用C和C++编写的Windows和Linux等操作系统相比有竞争力的性能。 
系统架构在许多方面发挥了关键作用，关于这点我将在未来的帖子中继续讨论。 
但是，在系统的基石部分，一个从“托管”的，强类型的和内存安全的源代码中榨取原生代码性能的优化编译器，
是我们最重要的武器之一。 
在本篇文章中，我将就此描述一些对我们成功至关重要的关键思考和技巧。

<!-- 
# Overview 
-->
# 概览

<!-- 
When people think of C#, Java, and related languages, they usually think of [Just-In-Time (JIT) compilation](
https://en.wikipedia.org/wiki/Just-in-time_compilation).  Especially back in the mid-2000s when Midori began.  But
Midori was different, using more C++-like [Ahead-Of-Time (AOT) compilation](
https://en.wikipedia.org/wiki/Ahead-of-time_compilation) from the outset. 
-->
当人们想到C#，Java及相关语言时，他们通常会想到[Just-In-Time（JIT）编译](https://en.wikipedia.org/wiki/Just-in-time_compilation)，
尤其是在Midori开始的2000年代中期。 
但是Midori却采用了不同的方式，它一开始便使用类[C++的Ahead-Of-Time（AOT）](https://en.wikipedia.org/wiki/Ahead-of-time_compilation) 编译方式。

<!-- 
AOT compiling managed, [garbage collected code](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))
presents some unique challenges compared to C and C++.  As a result, many AOT efforts don't achieve parity with their
native counterparts.  .NET's NGEN technology is a good example of this.  In fact, most efforts in .NET have exclusively
targeted startup time; this is clearly a key metric, but when you're building an operating system and everything on top,
startup time just barely scratches the surface. 
-->

与C和C++相比，AOT编译托管的[垃圾回收代码](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))存有一些独特的挑战。 
因此，许多关于AOT的努力与其对应的原生代码相比并没有获得相匹配的性能，
.NET的NGEN技术就是一个很好的例子。
实际上，.NET中的大部分工作都专门针对于启动时间再进行优化，这显然是算的上一个关键指标。
但是当构建一个操作系统及其之上的所有模块时，启动时间几乎仅仅只是皮毛而已。

<!-- 
Over the course of 8 years, we were able to significantly narrow the gap between our version of C# and classical C/C++
systems, to the point where basic code quality, in both size of speed dimensions, was seldom the deciding factor when
comparing Midori's performance to existing workloads.  In fact, something counter-intuitive happened.  The ability to
co-design the language, runtime, frameworks, operating system, and the compiler -- making tradeoffs in one area to gain
advantages in other areas -- gave the compiler far more symbolic information than it ever had before about the program's
semantics and, so, I dare say, was able to exceed C and C++ performance in a non-trivial number of situations. 
-->

在8年的时间里，我们显著地缩小我们版本的C#与经典C/C++系统之间的差距，
当同时在两个不同的速度维度上利用现有的任务上比较Midori的性能时，基本的代码质量都很少成为决定性因素。
事实上，反直觉的事情发生了——
协同设计语言、运行时、框架、操作系统和编译器的能力——
使得在一个方面上的折衷可以在其他方面获得优势，
这也为编译器提供了比以往更多的关于程序语义的符号信息。
因此，我敢说，我们的语言能够在相当多的情况下超过C和C++的性能。

<!-- 
Before diving deep, I have to put in a reminder.  The architectural decisions -- like [Async Everywhere](
http://joeduffyblog.com/2015/11/19/asynchronous-everything/) and Zero-Copy IO (coming soon) -- had more to do with us
narrowing the gap at a "whole system" level.  Especially the less GC-hungry way we wrote systems code.  But the
foundation of a highly optimizing compiler, that knew about and took advantage of safety, was essential to our results. 
-->

在深入探究编译器之前，我必须有所提醒。
架构上的决策，如[无处不在的Async](/2018/11/25/midori/3-asynchronous-everything/)和
（即将推出的）Zero-Copy IO（零拷贝IO）文章，
在“整个系统”级别缩小性能差距上与我们更加相关， 
特别是在我们编写缺乏GC的系统代码时。
但是，高度优化的编译器打下的基础，了解并利用其安全性，对我们的结果至关重要。

<!-- 
I would also be remiss if I didn't point out that the world has made considerable inroads in this area alongside us.
[Go](https://golang.org/) has straddled an elegant line between systems performance and safety.  [Rust](
http://rust-lang.org/) is just plain awesome.  The [.NET Native](
https://msdn.microsoft.com/en-us/vstudio/dotnetnative.aspx) and, related, [Android Runtime](
https://en.wikipedia.org/wiki/Android_Runtime) projects have brought a nice taste of AOT to C# and Java in a more
limited way, as a "silent" optimization technique to avoid mobile application lag caused by JITting.  Lately, we've
been working on bringing AOT to a broader .NET setting with the [CoreRT project](https://github.com/dotnet/corert).
Through that effort I hope we can bring some of the lessons learned below to a real-world setting.  Due to the
delicate balance around breaking changes it remains to be seen how far we can go.  It took us years to get
everything working harmoniously, measured in man-decades, however, so this transfer of knowledge will take time. 
-->
如果我没有指出这个领域中其他团队与我们同时取得了巨大进展的工作，那么会是我的失职。
这里的进展包括，[Go语言](https://golang.org/)跨越了系统性能和安全性之间的优雅界限；
另外，[Rust](http://rust-lang.org/)在这方面也做的很棒。 
[.NET Native](https://msdn.microsoft.com/en-us/vstudio/dotnetnative.aspx)和相关的[Android Runtime](https://en.wikipedia.org/wiki/Android_Runtime)
项目以更强限制的方式为C#和Java带来了AOT编译，并将其作为一种“静默”优化技术，
以避免在移动应用程序上由JIT引起的延迟。 
最近，我们一直致力于通过[CoreRT项目](https://github.com/dotnet/corert)以广泛地方式将AOT带入.NET环境中，
通过这项努力，我希望我们能够将下面的一些经验教训带到真实世界的编程环境中。 
虽然由于突破性变化之间的微妙平衡，我们还能走多远还有待观察，
然而，我们花了几年时间，数十人年的工作量，
才能使所有事情和谐有序地工作，因此这种知识的转移需要一定的时间。


<!-- 
First thing's first.  Let's quickly recap: What's the difference between native and managed code, anyway? 
-->
首先，让我们简要地回顾一个问题：原生代码和托管代码之间到底有什么区别？

<!-- 
## What's the same 
-->
## 相同点

<!-- 
I despise the false dichotomy "native and managed," so I must apologize for using it.  After reading this article, I
hope to have convinced you that it's a continuum.  C++ is [safer these days than ever before](
https://github.com/isocpp/CppCoreGuidelines), and likewise, C# performant.  It's amusing how many of these lessons apply
directly to the work my team is doing on Safe C++ these days. 
-->
由于我对将“原生和托管”进行错误的二分的方式有所鄙夷，因此我必须为使用这种说法而道歉。
在阅读完这篇文章后，我希望能说服你它们其实是连续统一体。 
[C++现在比以往任何时候都更安全](https://github.com/isocpp/CppCoreGuidelines)，
同样地，C#表现地更好。 
有趣的是，这些大量的教训直接适用于我们团队最近在安全C++上所做的工作。

<!-- 
So let's begin by considering what's the same. 
-->
所以让我们首先讨论它们的共同点。

<!-- 
All the basic [dragon book](https://en.wikipedia.org/wiki/Principles_of_Compiler_Design) topics apply to managed as
much as they do native code. 
-->
所有[龙书](https://en.wikipedia.org/wiki/Principles_of_Compiler_Design)中的基础主题，
只要适用于原生代码，对于托管代码都同样适用。

<!-- 
In general, compiling code is a balancing act between, on one hand emitting the most efficient instruction sequences
for the target architecture, to execute the program quickly; and on the other hand emitting the smallest encoding of
instructions for the target architecture, to store the program compactly and effectively use the memory system on the
target device.  Countless knobs exist on your favorite compiler to dial between the two based on your scenario.  On
mobile, you probably want smaller code, whereas on a multimedia workstation, you probably want the fastest. 
-->
通常，编译代码的过程是下列两种行为的平衡，一方面为目标体系结构产生最高效的指令序列，以快速执行程序；另一方面，为目标体系结构发出最小的指令编码，以便在目标设备上紧凑且有效地使用存储系统来存储程序。 
你最喜欢的编译器上存在无数个旋钮，可根据你的场景在两者之间进行切换。 
或许在移动设备上，你需要更小的代码体积，而在多媒体工作站上，可能需要最快的代码。

<!-- 
The choice of managed code doesn't change any of this.  You still want the same flexibility.  And the techniques you'd
use to achieve this in a C or C++ compiler are by and large the same as what you use for a safe language. 
-->
选择托管代码不会改变这种平衡关系的任何地方，你仍然需要相同的灵活性。 
在C或C++编译器中用于实现此目的的技术基本上与用在安全语言的技术相同。

<!-- 
You need a great [inliner](https://en.wikipedia.org/wiki/Inline_expansion).  You want [common subexpression
elimination (CSE)](https://en.wikipedia.org/wiki/Common_subexpression_elimination), [constant propagation and folding](
https://en.wikipedia.org/wiki/Constant_folding), [strength reduction](https://en.wikipedia.org/wiki/Strength_reduction),
and an excellent [loop optimizer](https://en.wikipedia.org/wiki/Loop_optimization).  These days, you probably want to
use [static single assignment form (SSA)](https://en.wikipedia.org/wiki/Static_single_assignment_form), and some unique
SSA optimizations like [global value numbering](https://en.wikipedia.org/wiki/Global_value_numbering) (although you need
to be careful about working set and compiler throughput when using SSA everywhere).  You will need specialized machine
dependent optimizers for the target architectures that are important to you, including [register allocators](
https://en.wikipedia.org/wiki/Register_allocation).  You'll eventually want a global analyzer that does interprocedural
optimizations, link-time code-generation to extend those interprocedural optimizations across passes, a [vectorizer](
https://en.wikipedia.org/wiki/Automatic_vectorization) for modern processors (SSE, NEON, AVX, etc.), and most definitely
[profile guided optimizations (PGO)](https://en.wikipedia.org/wiki/Profile-guided_optimization) to inform all of the
above based on real-world scenarios. 
-->
你需要一个出色的[内联器](https://en.wikipedia.org/wiki/Inline_expansion)，同时还需要[公共子表达式消除（CSE）](https://en.wikipedia.org/wiki/Common_subexpression_elimination)、[常量传播和折叠](https://en.wikipedia.org/wiki/Constant_folding)，[强度折减](https://en.wikipedia.org/wiki/Strength_reduction)以及出色的[循环优化器](https://en.wikipedia.org/wiki/Loop_optimization)。 
目前，你可能还希望使用[静态单赋值形式（SSA）](https://en.wikipedia.org/wiki/Static_single_assignment_form)和一些独特的SSA优化方法，如[全局值编号](https://en.wikipedia.org/wiki/Global_value_numbering)（尽管在任何地方使用SSA时都需注意工作集和编译器的吞吐量）。 
除此之外，对你来说很重要的目标体系结构，还需要专门的机器相关的优化器，包括[寄存器分配器](https://en.wikipedia.org/wiki/Register_allocation)等。 
最后，你还需要一个进行过程间优化的全局分析器，以跨pass的方式来扩展过程间优化的链接时代码生成，和现代处理器的[矢量化器](https://en.wikipedia.org/wiki/Automatic_vectorization)
（SSE，NEON，AVX等），以及可根据实际情况通知所有以上优化部件的[配置文件引导优化（PGO）](https://en.wikipedia.org/wiki/Profile-guided_optimization)。

<!-- 
Although having a safe language can throw some interesting curveballs your way that are unique and interesting -- which
I'll cover below -- you'll need all of the standard optimizing compiler things. 
-->
虽然拥有一种安全的语言可以以一种独特而有趣的方式带来意想不到的变化，
对此我将在下面进行介绍，但你同时也需要所有以上这些标准的编译器优化方法。

<!-- 
I hate to say it, but doing great at all of these things is "table stakes."  Back in the mid-2000s, we had to write
everything by hand.  Thankfully, these days you can get an awesome off-the-shelf optimizing compiler like [LLVM](
http://llvm.org) that has most of these things already battle tested, ready to go, and ready for you to help improve. 
-->
虽然我讨厌这样说，但在所有这些优化技术上做得很好就是你的“筹码”。
早在2000年代中期，我们不得不以手工方式实现所有这些方法。 
值得庆幸的是，现在已经有现成的非常棒优化编译器，如[LLVM](http://llvm.org)，其中大部分的功能已经通过了测试，并随时准备帮助你进行改进。

<!-- 
## What's different 
-->
## 不同点

<!-- 
But, of course, there are differences.  Many.  This article wouldn't be very interesting otherwise. 
-->
但当然，托管和原生在许多地方是存在差异的，
否则，这篇文章也不会很有趣。

<!-- 
The differences are more about what "shapes" you can expect to be different in the code and data structures thrown at
the optimizer.  These shapes come in the form of different instruction sequences, logical operations in the code that
wouldn't exist in the C++ equivalent (like more bounds checking), data structure layout differences (like extra object
headers or interface tables), and, in most cases, a larger quantity of supporting runtime data structures.
-->
差异更多体现在优化器所生成的代码和数据结构的“形状”上，
这些“形状”如下的形式出现：不同的指令序列、不存在对应C++等价代码的逻辑操作（如更多的边界检查）、数据结构布局差异（如额外的对象头信息或接口表）以及最大的不同之处在于大量的支撑运行时的数据结构。

<!-- 
Objects have "more to them" in most managed languages, compared to frugal data types in, say, C.  (Note that C++ data
structures are not nearly as frugal as you might imagine, and are probably closer to C# than your gut tells you.)  In
Java, every object has a vtable pointer in its header.  In C#, most do, although structs do not.  The GC can impose
extra layout restrictions, such as padding and a couple words to do its book-keeping.  Note that none of this is really
specific to managed languages -- C and C++ allocators can inject their own words too, and of course, many C++ objects
also carry vtables -- however it's fair to say that most C and C++ implementations tend to be more economical in these
areas.  In most cases, for cultural reasons more than hard technical ones.  Add up a few thousand objects in a heap,
especially when your system is built of many small processes with isolated heaps, like Midori, and it adds up quickly. 
-->
与C语言中的简单的数据类型相比，对象在大多数托管语言中都有“比实际更大的体积”
（请注意：C++数据结构并不像你想象的那样简单，并且可能比你的直觉更接近于C#）。
在Java中，每个对象的头部中都有一个vtable指针，而在C#中，尽管结构除外，但大多数情况也都是如此。 
GC可以对布局施加额外的限制，例如填充和增加几个字节来于记录信息。 
请注意，这些都不是特定于托管语言的特征——C和C++分配器也可能注入额外的信息，当然，许多C++对象也都带有vtable。
但是可以公正地说，大多数C和C++在这些方面的实现上往往更经济。 
在大多数情况下，出于文化不是技术原因，在堆中添加几千个对象，尤其是当系统由带有隔离堆空间的大量轻量级进程（如Midori中）组成时，内存消耗会快速增加。

<!-- 
In Java, you've got a lot more virtual dispatch, because methods are virtual by default.  In C#, thankfully, methods
are non-virtual by default.  (We even made classes sealed by default.)  Too much virtual dispatch can totally screw
inlining which is a critical optimization to have for small functions.  In managed languages you tend to have more
small functions for two reasons: 1) properties, and 2) higher level programmers tend to over-use abstraction. 
-->
另外，在Java中有更多的虚拟调度（virtual dispatch），因为默认情况下方法都是虚拟的，
而在C#中，幸运的是，默认情况下方法是非虚拟的（我们甚至默认将类密封）。
太多的虚拟调度完全可以内联化，而这对于小型函数来讲也是关键的优化技术。 
在托管语言中，代码则倾向于具有更多的小型函数，其原因出于以下两个：1）属性值；2）更高级别的程序员倾向于过度使用抽象。

<!-- 
Although it's seldom described this formally, there's an "ABI" ([Application Binary Interface](
https://en.wikipedia.org/wiki/Application_binary_interface)) that governs interactions between code and the runtime.
The ABI is where the rubber meets the road.  It's where things like calling conventions, exception handling, and, most
notably, the GC manifest in machine code.  This is *not* unique to managed code!  C++ has a "runtime" and therfore an
ABI too.  It's just that it's primarily composed of headers, libraries like allocators, and so on, that are more
transparently linked into a program than with classical C# and Java virtual machines, where a runtime is non-negotiable
(and in the JIT case, fairly heavy-handed).  Thinking of it this way has been helpful to me, because the isomorphisms
with C++ suddenly become immediately apparent. 
-->
尽管很少有这方面的形式化描述，
但也存在“ABI”（[应用程序二进制接口](https://en.wikipedia.org/wiki/Application_binary_interface)）用于管理代码和运行时之间的交互。 
ABI可以看做是车辆与道路之间接触部分，它包含了调用约定，异常处理，尤其是机器代码中的GC清单（manifest）。 
虽然这*不是*托管代码所特有的概念，比如C++同样存在运行时和ABI， 
只是C++中主要由对象头部信息，以及像分配器这样的库等组成，并以更透明的方式链接到程序，
而不像传统的C#和Java的虚拟机，其运行时却是不可协商的（JIT情况下也是相当粗暴的）。 
由于托管代码和C++的同构关系突然变得明显起来，因此以这种方式思考对我来说是有帮助的。

<!-- 
The real biggie is array bounds checks.  A traditional approach is to check that the index is within the bounds of an
array before accessing it, either for loading or storing.  That's an extra field fetch, compare, and conditional
branch.  [Branch prediction](https://en.wikipedia.org/wiki/Branch_predictor) these days is quite good, however it's just
plain physics that if you do more work, you're going to pay for it.  Interestingly, the work we're doing with C++'s
`array_view<T>` incurs all these same costs. 
-->
真正的大问题是数组边界检查。
传统的做法是在访问索引之前检查索引是否在数组范围内，以便加载或存储，
因此存在一个额外的字段提取、比较和条件分支。
当前的[分支预测](https://en.wikipedia.org/wiki/Branch_predictor)已经做的相当不错，
但存在一个简单的道理是：如果你想完成更多的工作，则需要更大的开销。 
有趣的是，我们在C++的`array_view<T>`上所做的工作会产生所有这些相同的开销。

<!-- 
Related to this, there can be null checks where they didn't exist in C++.  If you perform a method dispatch on a null
object pointer in C++, for example, you end up running the function anyway.  If that function tries to access `this`,
it's bound to [AV](https://en.wikipedia.org/wiki/Segmentation_fault), but in Java and .NET, the compiler is required
(per specification) to explicitly check and throw an exception in these cases, before the call even occurs.  These
little branches can add up too.  We eradicated such checks in favor of C++ semantics in optimized builds. 
-->
与此相关的是，还需要对托管代码中不存在于C++的null类型进行检查。 
例如，如果在C++中对空对象指针执行方法调度，则无论如何都会运行该函数，
但如果该函数试图访问`this`，则它将会产生[内存错误](https://en.wikipedia.org/wiki/Segmentation_fault)，
但在Java和.NET中，编译器需要（根据规范）甚至在调用发生之前，显式地进行检查，并在这些情况下抛出异常。
这些细小的分支同样可以叠加到一起，而我们将根据optimized编译版本的C++语义消除这样的检查。

<!-- 
In Midori, we compiled with overflow checking on by default.  This is different from stock C#, where you must explicitly
pass the `/checked` flag for this behavior.  In our experience, the number of surprising overflows that were caught,
and unintended, was well worth the inconvenience and cost.  But it did mean that our compiler needed to get really good
at understanding how to eliminate unnecessary ones. 
-->
在Midori中，我们在编译中默认使用溢出检查，
这与现行的C#中必须为此行为显式传递`/checked`标志有所不同。 
根据我们的经验，捕获的意外和无意的溢出的数量，相比于带来不便和开销来讲是非常值得的，
但这确实意味着我们的编译器需要非常善于理解如何消除不必要的检查。

<!-- 
Static variables are very expensive in Java and .NET.  Way more than you'd expect.  They are mutable and so cannot be
stored in the readonly segment of an image where they are shared across processes.  And my goodness, the amount of
lazy-initialization checking that gets injected into the resulting source code is beyond belief.  Switching from
`preciseinit` to `beforefieldinit` semantics in .NET helps a little bit, since the checks needn't happen on every
access to a static member -- just accesses to the static variable in question -- but it's still disgusting compared to
a carefully crafted C program with a mixture of constant and intentional global initialization. 
-->

另外，在Java和.NET中，静态变量开销非常大，甚至超出了你所期望的。 
由于它们是可变的，因此不能存储于用作进程间共享的可执行文件只读段中。 
而且令人惊讶的是，被注入到生成代码中的延迟初始化检查数量超出了你的想象。
虽然从.NET的`preciseinit`切换到`beforefieldinit`语义能够带来一些帮助，
因为每次访问静态成员都不需要再进行检查，而只需检查可能有问题的静态变量，
但与精心设计的，具有常量和定向的全局初始化的C程序相比，它仍然具有很大的开销。

<!-- 
The final major area is specific to .NET: structs.  Although structs help to alleviate GC pressure and hence are a
good thing for most programs, they also carry some subtle problems.  The CLI specifies surprising behavior around their
initialization, for example.  Namely if an exception happens during construction, the struct slot must remain zero-
initialized.  The result is that most compilers make defensive copies.  Another example is that the compiler must
make a defensive copy anytime you call a function on a readonly struct.  It's pretty common for structs to be copied
all over the place which, when you're counting cycles, hurts, especially since it often means time spent in `memcpy`.
We had a lot of techniques for addressing this and, funny enough, I'm pretty sure when all was said and done, our
code quality here was *better* than C++'s, given all of its RAII, copy constructor, destructor, and so on, penalties. 
-->

最后的主要内容则特定于.NET：结构。 
虽然结构有助于缓解GC压力，这对大多数程序来说都是好事，但它们也带有一些微妙的问题。 
例如，CLI指定了初始化时一些奇怪的的行为，例如如果在构造期间发生异常，则结构的空位必须初始化为全零，
其结果导致大多数编译器都需制作防御性的副本。 
而另一个相似的例子是，只要调用readonly结构上的函数，编译器就必须制作防御性副本。 
将结构复制到整个其他地方是非常常见的，因此当你计算程序执行周期数时则会出现偏差，尤其是因为它通常意味着花费在`memcpy`中的时间开销，
我们提出了很多技术解决这个问题。有趣的是，当所有问题被提出和解决时，
我很确定的一点是，考虑到它的RAII，复制构造函数，析构函数以及开销等所有因素，我们的代码质量是*优于*C++的。

<!-- 
# Compilation Architecture 
-->
# 编译架构

<!-- 
Our architecture involved three major components: 
-->
我们的架构涉及到三个主要的部件：

<!-- 
* [C# Compiler](https://github.com/dotnet/roslyn): Performs lexing, parsing, and semantic analysis.  Ultimately
  translates from C# textual source code into a [CIL](https://en.wikipedia.org/wiki/Common_Intermediate_Language)-based
  [intermediate representation (IR)](https://en.wikipedia.org/wiki/Intermediate_language).
* [Bartok](https://en.wikipedia.org/wiki/Bartok_(compiler)): Takes in said IR, does high-level MSIL-based analysis,
  transformations, and optimizations, and finally lowers this IR to something a bit closer to a more concrete machine
  representation.  For example, generics are gone by the time Bartok is done with the IR.
* [Phoenix](https://en.wikipedia.org/wiki/Phoenix_(compiler_framework)): Takes in this lowered IR, and goes to town on
  it.  This is where the bulk of the "pedal to the metal" optimizations happen.  The output is machine code. 
-->
* [C#编译器](https://github.com/dotnet/roslyn)：执行词法分析，语法解析和语义分析，最终将C#源代码文本转换为基于[CIL](https://en.wikipedia.org/wiki/Common_Intermediate_Language)的[中间表示（IR）](https://en.wikipedia.org/wiki/Intermediate_language)。
* [Bartok](http://t.cn/EVXZ53S)：以前一阶段的IR作为输入，进行基于MSIL的高层次分析，转换和优化，最后将IR削弱到更具体的机器表示形式。
  例如，当Bartok处理完IR时，泛型就已不存在了。
* [Phoenix](http://t.cn/EVXZCoJ)：以前一阶段的低层次IR作为输入进行频繁的处理。
  这就是大多数“把油门踩到底”的相关优化的地方，其最后输出的结果是机器代码。

<!-- 
The similarities here with Swift's compiler design, particularly [SIL](
http://llvm.org/devmtg/2015-10/slides/GroffLattner-SILHighLevelIR.pdf), are evident.  The .NET Native project also
mirrors this architecture somewhat.  Frankly, most AOT compilers for high level languages do. 
-->
这与Swift编译器的设计（特别是[SIL](http://llvm.org/devmtg/2015-10/slides/GroffLattner-SILHighLevelIR.pdf)）的相似之处是显而易见的，
.NET Native项目也在某种程度上体现了这种架构。
坦率地说，大多数高级语言的AOT编译器都是如此。

<!-- 
In most places, the compiler's internal representation leveraged [static single assignment form (SSA)](
https://en.wikipedia.org/wiki/Static_single_assignment_form).  SSA was preserved until very late in the compilation.
This facilitated and improved the use of many of the classical compiler optimizations mentioned earlier. 
-->
在大多数地方，编译器的内部表示都利用了
[静态单赋值形式（SSA）](https://en.wikipedia.org/wiki/Static_single_assignment_form)，
SSA形式一直保持到编译的最后阶段，这促成并改进了前面提到的许多经典编译器优化的使用。

<!-- 
The goals of this architecture included: 
-->
该架构的目标包括：

<!-- 
* Facilitate rapid prototyping and experimentation.
* Produce high-quality machine code on par with commerical C/C++ compilers.
* Support debugging optimized machine code for improved productivity.
* Facilitate profile-guided optimizations based on sampling and/or instrumenting code.
* Suitable for self-host:
    - The resulting compiled compiler is fast enough.
    - It is fast enough that the compiler developers enjoy using it.
    - It is easy to debug problems when the compiler goes astray. 
-->
* 促进快速原型设计和实验；
* 生成与商业C/C++编译器相当的高质量机器代码；
* 支持调试优化的机器代码以提高生产率；
* 基于采样和检测代码，提升配置文件引导优化；
* 适合自托管（self-host）：
    - 生成的编译器编译足够快；
    - 足够快，因此编译器开发人员乐于使用它；
    - 当编译器出现bug时，很容易对问题进行调试。
  
<!-- 
Finally, a brief warning.  We tried lots of stuff.  I can't remember it all.  Both Bartok and Phoenix existed for years
before I even got involved in them.  Bartok was a hotbed of research on managed languages -- ranging from optimizations
to GC to software transactional memory -- and Phoenix was meant to replace the shipping Visual C++ compiler.  So,
anyway, there's no way I can tell the full story.  But I'll do my best. 
-->
最后是一个简要警告：我们进行了很多方面的尝试，多到我已无法全部回忆起来。
在我参与之前，Bartok和Phoenix都已存在多年： 
Bartok是托管语言研究的温床，其包含从优化到GC再到软件事务内存的各个方面；
而Phoenix本来是作为取代已发布的Visual C++编译器而存在。 
所以，无论怎样我都无法讲完所有的故事，但我会尽我所能。

<!-- 
# Optimizations 
-->
# 优化

<!-- 
Let's go deep on a few specific areas of classical compiler optimizations, extended to cover safe code. 
-->
让我们深入研究一些扩展到涵盖安全代码的经典编译器优化方法的特定领域。

<!-- 
## Bounds check elimination 
-->
## 边界检查消除

<!-- 
C# arrays are bounds checked.  So were ours.  Although it is important to eliminate superfluous bounds checks in regular
C# code, it was even more so in our case, because even the lowest layers of the system used bounds checked arrays.  For
example, where in the bowels of the Windows or Linux kernel you'd see an `int*`, in Midori you'd see an `int[]`. 
-->
C#数组是带有边界检查的，而我们的也同样如此。 
虽然在常规C#代码中消除多余的边界检查很重要，但在我们的环境下更需如此，
因为即使是使用系统的最低层也需使用已检查边界的数组。 
例如，在Windows或Linux内核的中，你看到的是`int*`类型，
同样地，在Midori中你看到的是`int[]`类型。

<!-- 
To see what a bounds check looks like, consider a simple example:
-->
为了说明边界检查长什么样子的，请考虑如下的一个简单例子：

    var a = new int[100];
    for (int i = 0; i < 100; i++) {
        ... a[i] ...;
    }

<!-- 
Here's is an example of the resulting machine code for the inner loop array access, with a bounds check: 
-->
这是带有边界检查的循环内数组访问所生成的机器代码的示例：

<!-- 
    ; First, put the array length into EAX:
    3B15: 8B 41 08        mov         eax,dword ptr [rcx+8]
    ; If EDX >= EAX, access is out of bounds; jump to error:
    3B18: 3B D0           cmp         edx,eax
    3B1A: 73 0C           jae         3B28
    ; Otherwise, access is OK; compute element's address, and assign:
    3B1C: 48 63 C2        movsxd      rax,edx
    3B1F: 8B 44 81 10     mov         dword ptr [rcx+rax*4+10h],r8d
    ; ...
    ; The error handler; just call a runtime helper that throws:
    3B28: E8 03 E5 FF FF  call        2030 
-->
```
; 首先，将数组长度放入EAX：
3B15: 8B 41 08        mov         eax,dword ptr [rcx+8]
; 如果EDX>=EAX，则访问超出范围，跳转到错误处理例程：
3B18: 3B D0           cmp         edx,eax
3B1A: 73 0C           jae         3B28
; 否则，访问正常；计算元素的地址，并赋值：
3B1C: 48 63 C2        movsxd      rax,edx
3B1F: 8B 44 81 10     mov         dword ptr [rcx+rax*4+10h],r8d
; ...
; 错误处理，仅仅是调用抛出异常的运行时辅助程序：
3B28: E8 03 E5 FF FF  call        2030 
```
<!-- 
If you're doing this bookkeeping on every loop iteration, you won't get very tight loop code.  And you're certianly not
going to have any hope of vectorizing it.  So, we spent a lot of time and energy trying to eliminate such checks. 
-->
如果你在每次循环迭代中采用如此方式做记录，那么将不会得到非常紧凑的循环代码，
而且你肯定不会有对其进行矢量化的可能性。 
因此，我们花了很多时间和精力试图消除这样的检查。

<!-- 
In the above example, it's obvious to a human that no bounds checking is necessary.  To a compiler, however, the
analysis isn't quite so simple.  It needs to prove all sorts of facts about ranges.  It also needs to know that `a`
isn't aliased and somehow modified during the loop body.  It's surprising how hard this problem quickly becomes. 
-->
在上面的例子中，对于人类来讲显然不需要进行边界检查，
然而对于编译器而言，分析并不是那么简单。
它需要证明关于范围的各种事实，
还需要证明`a`在循环体中没有别名或以某种方式被修改过。 
这个问题变得如此困难的速度之快，着实令人惊讶。

<!-- 
Our system had multiple layers of bounds check eliminations. 
-->
在我们的系统中，有多层次的边界检查消除方法。


<!-- 
First it's important to note that CIL severely constraints an optimizer by being precise in certain areas.  For example,
accessing an array out of bounds throws an `IndexOutOfRangeException`, similar to Java's `ArrayOutOfBoundsException`.
And the CIL specifies that it shall do so at precisely the exception that threw it.  As we will see later on, our
error model was more relaxed.  It was based fail-fast and permitted code motion that led to inevitable failures
happening "sooner" than they would have otherwise.  Without this, our hands would have been tied for much of what I'm
about to discuss. 
-->
首先，重要的是要注意到CIL在某些区域中的精确处理的需求严格限制了优化器的作用。
例如，访问数组越界会抛出`IndexOutOfRangeException`异常，
类似于Java的`ArrayOutOfBoundsException`，并且CIL指定了它应该在抛出异常时准确地这样做。
而正如在稍后将看到的那样，我们的错误模型更加轻松，
它基于快速失败（fail-fast）和允许代码外提（code motion），使得不可避免的失败能够比其他系统更快地“发生”。 
如果没有实现这一点，我们的双手将会被我即将讨论的大部分内容束缚在一起。

<!-- 
At the highest level, in Bartok, the IR is still relatively close to the program input.  So, some simple patterns could
be matched and eliminated.  Before lowering further, the [ABCD algorithm](
http://www.cs.virginia.edu/kim/courses/cs771/papers/bodik00abcd.pdf) -- a straightforward value range analysis based on
SSA -- then ran to eliminate even more common patterns using a more principled approach than pattern matching.  We were
also able to leverage ABCD in the global analysis phase too, thanks to inter-procedural length and control flow fact
propagation. 
-->
在最高的一层中，在Bartok中，IR仍然与程序输入相对接近，
因此，可以匹配和消除一些简单的模式。 
在进一步降低层次之前，[ABCD算法](http://citeseerx.ist.psu.edu/viewdoc/download;jsessionid=993D087A3E775ED0476B70E3D7CDD52B?doi=10.1.1.118.9991&rep=rep1&type=pdf)
（即基于SSA的直接值范围分析）能够采用比模式匹配原则性更强的方法来消除更常见的模式。 
幸亏有程序间长度和控制流的事实传播，我们也能够在全局分析阶段使用上ABCD算法。


<!-- 
Next up, the Phoenix Loop Optimizer got its hands on things.  This layer did all sorts of loop optimizations and, most
relevant to this section, range analysis.  For example: 
-->

接下来，Phoenix循环优化器开始发挥作用，这一层处理的是各种循环优化，
以及和这一节内容最相关的范围分析。
例如：

<!-- 
* Loop materialization: this analysis actually creates loops.  It recognizes repeated patterns of code that would be
  more ideally represented as loops, and, when profitable, rewrites them as such.  This includes unrolling hand-rolled
  loops so that a vectorizer can get its hands on them, even if they might be re-unrolled later on.
* Loop cloning, unrolling, and versioning: this analysis creates copies of loops for purposes of specialization.  That
  includes loop unrolling, creating architectural-specific versions of a vectorized loop, and so on.
* [Induction](https://en.wikipedia.org/wiki/Induction_variable) range optimization: this is the phase we are most
  concerned with in this section.  It uses induction range analysis to remove unnecessary checks, in addition to doing
  classical induction variable optimizations such as widening.  As a byproduct of this phase, bounds checks were
  eliminated and coalesced by hoisting them outside of loops. 
-->
* 循环实体化（Loop Materialization）：
  该处分析实际上创建了循环。它能够识别理想地表示形式为循环的重复代码模式，并且在有性能收益时，将其重写为这样的形式。 
  这包括手动展开循环，以便矢量化器可以处理它们，即使它们以后可能重新被展开；
* 循环克隆，展开和版本控制：该处分析创建循环副本以用作特殊化处理，这包括循环展开，创建矢量化循环的体系结构特定版本等等；
* [归纳（Induction）](https://en.wikipedia.org/wiki/Induction_variable)范围优化：这是我们在本节中最关注的处理阶段。 
  除了进行经典的归纳变量优化（如加宽）之外，还使用归纳范围分析来删除不必要的检查。 
  作为该阶段的副产品，边界检查通过将它们外提到循环之外的方式被消除和合并。


<!-- 
This sort of principled analysis was more capable than what was shown earlier.  For example, there are ways to write
the earlier loop that can easily "trick" the more basic techniques discussed earlier: 
-->

这种原则性强的分析比之前介绍的方法处理能力更强。 
例如，存在如下的一些做法可以编写更早期的循环，达到“欺骗”前面讨论的更基础技术的目的：

<!-- 
    var a = new int[100];
    
    // Trick #1: use the length instead of constant.
    for (int i = 0; i < a.length; i++) {
        a[i] = i;
    }
    
    // Trick #2: start counting at 1.
    for (int i = 1; i <= a.length; i++) {
        a[i-1] = i-1;
    }
    
    // Trick #3: count backwards.
    for (int i = a.length - 1; i >= 0; i--) {
        a[i] = i;
    }
    
    // Trick #4: don't use a for loop at all.
    int i = 0;
    next:
    if (i < a.length) {
        a[i] = i;
        i++;
        goto next;
    } 
-->
    var a = new int[100];
    
    // Trick #1：使用变量length而不是常量。
    for (int i = 0; i < a.length; i++) {
        a[i] = i;
    }
    
    // Trick #2：从1开始计数
    for (int i = 1; i <= a.length; i++) {
        a[i-1] = i-1;
    }
    
    // Trick #3：后向计数
    for (int i = a.length - 1; i >= 0; i--) {
        a[i] = i;
    }
    
    // Trick #4：根本不使用for循环。
    int i = 0;
    next:
    if (i < a.length) {
        a[i] = i;
        i++;
        goto next;
    } 

<!-- 
You get the point.  Clearly at some point you can screw the optimizer's ability to do anything, especially if you
start doing virtual dispatch inside the loop body, where aliasing information is lost.  And obviously, things get more
difficult when the array length isn't known statically, as in the above example of `100`.  All is not lost, however,
if you can prove relationships between the loop bounds and the array.  Much of this analysis requires special knowledge
of the fact that array lengths in C# are immutable. 
-->
你已经发现了，很明显，在某些时候可以采用某种方式阻止优化器执行任何操作，
特别是如果在循环体内进行虚拟调度，其中的别名信息也将丢失。 
显然，当无法静态地知道数组长度时，如上面长度为`100`的例子所示，事情将会变得更加困难。
但如果可以证明循环边界和数组之间的关系时，那所有信息都不会丢失。
同时在C#中，大部分的分析方法都需要数组长度是不可变的。

<!-- 
At the end of the day, doing a good job at optimizing here is the difference between this: 
-->
不管怎么说，做好优化的区别体现在如下的原始版本：
<!-- 
    ; Initialize induction variable to 0:
    3D45: 33 C0           xor         eax,eax
    ; Put bounds into EDX:
    3D58: 8B 51 08        mov         edx,dword ptr [rcx+8]
    ; Check that EAX is still within bounds; jump if not:
    3D5B: 3B C2           cmp         eax,edx
    3D5D: 73 13           jae         3D72
    ; Compute the element address and store into it:
    3D5F: 48 63 D0        movsxd      rdx,eax
    3D62: 89 44 91 10     mov         dword ptr [rcx+rdx*4+10h],eax
    ; Increment the loop induction variable:
    3D66: FF C0           inc         eax
    ; If still < 100, then jump back to the loop beginning:
    3D68: 83 F8 64        cmp         eax,64h
    3D6B: 7C EB           jl          3D58
    ; ...
    ; Error routine:
    3D72: E8 B9 E2 FF FF  call        2030 
-->
```
; 将归纳变量初始化为0：
3D45: 33 C0           xor         eax,eax
; 将边界值放入EDX：
3D58: 8B 51 08        mov         edx,dword ptr [rcx+8]
; 检查EAX是否仍然在边界之内，如果没有则跳转：
3D5B: 3B C2           cmp         eax,edx
3D5D: 73 13           jae         3D72
; 计算元素的地址并将其存入其中：
3D5F: 48 63 D0        movsxd      rdx,eax
3D62: 89 44 91 10     mov         dword ptr [rcx+rdx*4+10h],eax
; 增加循环归纳变量的值：
3D66: FF C0           inc         eax
; 如果变量值仍然小于100，则跳回到循环开始位置：
3D68: 83 F8 64        cmp         eax,64h
3D6B: 7C EB           jl          3D58
; ...
; 错误处理例程：
3D72: E8 B9 E2 FF FF  call        2030 
```
<!-- 
And the following, completely optimized, bounds check free, loop: 
-->
以及如下完全优化且无边界检查的循环版本之间：

<!-- 
    ; Initialize induction variable to 0:
    3D95: 33 C0           xor         eax,eax
    ; Compute the element address and store into it:
    3D97: 48 63 D0        movsxd      rdx,eax
    3D9A: 89 04 91        mov         dword ptr [rcx+rdx*4],eax
    ; Increment the loop induction variable:
    3D9D: FF C0           inc         eax
    ; If still < 100, then jump back to the loop beginning:
    3D9F: 83 F8 64        cmp         eax,64h
    3DA2: 7C F3           jl          3D97 
-->
```
; 将归纳变量初始化为0：
3D95: 33 C0           xor         eax,eax
; 计算元素的地址并将其存入其中：
3D97: 48 63 D0        movsxd      rdx,eax
3D9A: 89 04 91        mov         dword ptr [rcx+rdx*4],eax
; 增加循环归纳变量的值：
3D9D: FF C0           inc         eax
; 如果变量值仍然小于100，则回到循环开始位置：
3D9F: 83 F8 64        cmp         eax,64h
3DA2: 7C F3           jl          3D97 
```
<!-- 
It's amusing that I'm now suffering deja vu as we go through this same exercise with C++'s new `array_view<T>` type.
Sometimes I joke with my ex-Midori colleagues that we're destined to repeat ourselves, slowly and patiently, over the
course of the next 10 years.  I know that sounds arrogant.  But I have this feeling on almost a daily basis. 
-->
有趣的是，如今当我们使用C++新的`array_view<T>`类型进行同样的实现时，
遭遇到了似曾相识的问题。 
有时候我和Midori的前同事开玩笑说，我们注定要在接下来的10年里慢慢地，有耐心地重复自己之前做过的事情。 
我知道这听起来很傲慢，但却几乎每天都有这种感觉。


<!-- 
## Overflow checking 
-->
## 溢出检查

<!-- 
As mentioned earlier, in Midori we compiled with checked arithmetic by default (by way of C#'s `/checked` flag).  This
eliminated classes of errors where developers didn't anticipate, and therefore code correctly for, overflows.  Of
course, we kept the explicit `checked` and `unchecked` scoping constructs, to override the defaults when appropriate,
but this was preferable because a programmer declared her intent. 
-->
如前所述，Midori默认使用检查后运算（通过C#的`/checked`标志）进行编译，
这消除了开发者没有预料到错误类别并因此能够为溢出问题正确的编码。
当然，我们保留了显式的`checked`和`unchecked`的作用域构造方式，并在适当时覆盖默认值，这是一种更可取的方式，因为程序员声明了其对于程序的意图。

<!-- 
Anyway, as you might expect, this can reduce code quality too. 
-->
不管怎样，正如你所料到的，这也会降低代码的质量。

<!-- 
For comparison, imagine we're adding two variables: 
-->
为了便于比较，假设添加了以下两个变量：

    int x = ...;
    int y = ...;
    int z = x + y; 


<!-- 
Now imagine `x` is in `ECX` and `y` is in `EDX`.  Here is a standard unchecked add operation: 
-->
现假设`x`在`ECX`寄存器中，`y`在`EDX`中，这是一个标准的未经检查的求和操作：

    03 C2              add         ecx,edx 


<!-- 
Or, if you want to get fancy, one that uses the `LEA` instruction to also store the result in the `EAX` register using
a single instruction, as many modern compilers might do: 
-->
或者，如果你显得更花哨，那么使用单个`LEA`指令也可将结果存储在`EAX`寄存器中，正如许多现代编译器的做法：

    8D 04 11           lea         eax,[rcx+rdx] 


<!-- 
Well, here's the equivalent code with a bounds check inserted into it: 
-->
好了，下面是插入边界检查的等价代码：

    3A65: 8B C1              mov         eax,ecx
    3A67: 03 C2              add         eax,edx
    3A69: 70 05              jo          3A70
    ; ...
    3A70: E8 B3 E5 FF FF     call        2028 


<!-- 
More of those damn conditional jumps (`JO`) with error handling routines (`CALL 2028`). 
-->
大多是那些该死的条件跳转指令（`JO`）和错误处理例程（`CALL 2028`）。

<!-- 
It turns out a lot of the analysis mentioned earlier that goes into proving bounds checks redundant also apply to
proving that overflow checks are redundant.  It's all about proving facts about ranges.  For example, if you can prove
that some check is [dominated by some earlier check](https://en.wikipedia.org/wiki/Dominator_(graph_theory)), and that
furthermore that earlier check is a superset of the later check, then the later check is unnecessary.  If the opposite
is true -- that is, the earlier check is a subset of the later check, then if the subsequent block postdominates the
earlier one, you might move the stronger check to earlier in the program. 
-->
事实证明，前面提到的很多证明边界检查是多余的方法也同样适用于溢出检查，所有这一切都是为了证明关于范围的一些事实。
例如，如果你可以证明某些检查[由某些前期检查所支配](http://t.cn/RqvLACK)，
并且前期检查是后续检查的超集，那么后续检查就没有必要了。 
如果相反的情况成立，也就是说，前期的检查是后续检查的子集，则可以将更强的检查移至程序的前面。

<!-- 
Another common pattern is that the same, or similar, arithmetic operation happens multiple times near one another:
-->
另一种常见模式是相同或类似的算术运算在彼此相邻的位置多次发生：

    int p = r * 32 + 64;
    int q = r * 32 + 64 - 16; 


<!-- 
It is obvious that, if the `p` assignment didn't overflow, then the `q` one won't either. 
-->
很明显，如果`p`的赋值没有溢出，那么`q`的赋值也不会溢出。

<!-- 
There's another magical phenomenon that happens in real world code a lot.  It's common to have bounds checks and
arithmetic checks in the same neighborhood.  Imagine some code that reads a bunch of values from an array: 
-->
在真实世界中的代码可能发生了另一种神奇的现象——
在同一邻域中进行边界检查和算术检查是相当常见的。 
假设有从数组中读取一堆值的如下代码：

<!-- 
    int data0 = data[dataOffset + (DATA_SIZE * 0)];
    int data1 = data[dataOffset + (DATA_SIZE * 1)];
    int data2 = data[dataOffset + (DATA_SIZE * 2)];
    int data3 = data[dataOffset + (DATA_SIZE * 3)];
    .. and so on ... 
-->
    int data0 = data[dataOffset + (DATA_SIZE * 0)];
    int data1 = data[dataOffset + (DATA_SIZE * 1)];
    int data2 = data[dataOffset + (DATA_SIZE * 2)];
    int data3 = data[dataOffset + (DATA_SIZE * 3)];
    ... 等等 ... 

<!-- 
Well C# arrays cannot have negative bounds.  If a compiler knows that `DATA_SIZE` is sufficiently small that an
overflowed computation won't wrap around past `0`, then it can eliminate the range check in favor of the bounds check. 
-->
良定义的C#数组不存在值为负的边界，
如果编译器知道`DATA_SIZE`足够小以至于可能发生溢出的计算不会小于`0`，
那么它可以消除边界检查所需的范围检查。

<!-- 
There are many other patterns and special cases you can cover.  But the above demonstrates the power of a really good
range optimizer that is integrated with loops optimization.  It can cover a wide array of scenarios, array bounds and
arithmetic operations included.  It takes a lot of work, but it's worth it in the end. 
-->
还有许多其他模式和特殊情况可以涵盖，但是前面以及展示了与循环优化集成的非常好的范围优化器所展现的强大功能。 
它可以覆盖各种场景，包括数组边界和算术运算。 
虽然花费了大量的工作，但最终还是值得的。

<!-- 
## Inlining 
-->
## 内联

<!-- 
For the most part, [inlining](https://en.wikipedia.org/wiki/Inline_expansion) is the same as with true native code.  And
just as important.  Often more important, due to C# developers' tendency to write lots of little methods (like property
accessors).  Because of many of the topics throughout this article, getting small code can be more difficult than in
C++ -- more branches, more checks, etc. -- and so, in practice, most managed code compilers inline a lot less than
native code compilers, or at least need to be tuned very differently.  This can actually make or break performance. 
-->
对于大多数的部分，[内联](https://en.wikipedia.org/wiki/Inline_expansion)与真正的原生代码是一样的。
内联也是同等重要，并且由于C#开发者倾向于编写许多微小方法（如属性访问器等），所以显得其更加重要。 
由于本文中的许多主题，获取小代码可能比在C++中更加困难——具有更多分支和检查等。
因此，在实践中，大多数的托管代码编译器比本机代码编译器所内联的要少得多，或者至少需要以非常不同的方式进行调整，这实际上对性能起到了决定性作用。

<!-- 
There are also areas of habitual bloat.  The way lambdas are encoded in MSIL is unintelligable to a naive backend
compiler, unless it reverse engineers that fact.  For example, we had an optimization that took this code: 
-->
还有一些习惯性膨胀的领域，例如对于原生后端编译器来讲，lambda算子在MSIL中编码的方式是不可理解的，除非其对此事实进行逆向工程。

例如，对如下的代码进行优化：

    void A(Action a) {
        a();
    }

    void B() {
        int x = 42;
        A(() => x++);
        ...
    } 


<!-- 
and, after inlining, was able to turn B into just: 
-->
在进行内联之后，能够将B转变为如下的形式：

    void B() {
        int x = 43;
        ...
    } 


<!-- 
That `Action` argument to `A` is a lambda and, if you know how the C# compiler encodes lambdas in MSIL, you'll
appreciate how difficult this trick was.  For example, here is the code for B: 
-->
类型为`Action`的参数`A`是一个lambda算子，
并且如果你知道C#编译器是如何在MSIL中编码lambda的，那么你将明白该技巧有多么的困难。 
例如，如下是B的MSIL代码：

<!-- 
    .method private hidebysig instance void
        B() cil managed
    {
        // Code size       36 (0x24)
        .maxstack  3
        .locals init (class P/'<>c__DisplayClass1' V_0)
        IL_0000:  newobj     instance void P/'<>c__DisplayClass1'::.ctor()
        IL_0005:  stloc.0
        IL_0006:  nop
        IL_0007:  ldloc.0
        IL_0008:  ldc.i4.s   42
        IL_000a:  stfld      int32 P/'<>c__DisplayClass1'::x
        IL_000f:  ldarg.0
        IL_0010:  ldloc.0
        IL_0011:  ldftn      instance void P/'<>c__DisplayClass1'::'<B>b__0'()
        IL_0017:  newobj     instance void [mscorlib]System.Action::.ctor(object,
                                                                      native int)
        IL_001c:  call       instance void P::A(class [mscorlib]System.Action)
        IL_0021:  nop
        IL_0022:  nop
        IL_0023:  ret
    } 
-->
    .method private hidebysig instance void
        B() cil managed
    {
        // 代码大小       36 (0x24)
        .maxstack  3
        .locals init (class P/'<>c__DisplayClass1' V_0)
        IL_0000:  newobj     instance void P/'<>c__DisplayClass1'::.ctor()
        IL_0005:  stloc.0
        IL_0006:  nop
        IL_0007:  ldloc.0
        IL_0008:  ldc.i4.s   42
        IL_000a:  stfld      int32 P/'<>c__DisplayClass1'::x
        IL_000f:  ldarg.0
        IL_0010:  ldloc.0
        IL_0011:  ldftn      instance void P/'<>c__DisplayClass1'::'<B>b__0'()
        IL_0017:  newobj     instance void [mscorlib]System.Action::.ctor(object,
                                                                      native int)
        IL_001c:  call       instance void P::A(class [mscorlib]System.Action)
        IL_0021:  nop
        IL_0022:  nop
        IL_0023:  ret
    } 
<!-- 
To get the magic result required constant propagating the `ldftn`, recognizing how delegate construction works
(`IL_0017`), leveraging that information to inline `B` and eliminate the lambda/delegate altogether, and then, again
mostly through constant propagation, folding the arithmetic into the constant `42` initialization of `x`.  I always
found it elegant that this "fell out" of a natural composition of multiple optimizations with separate concerns. 
-->
为了获得这种神奇结果，需要常量传播`ldftn`，识别委托构造的工作方式（`IL_0017`），利用该信息内联`B`并同时消除了lambda和delegate，然后再次主要通过常量传播，将算术折叠成利用常量`42`初始化`x`。 
我总觉得这种多种不同考量的优化的自然组合的方式是非常“优雅”的。

<!-- 
As with native code, profile guided optimization made our inlining decisions far more effective. 
-->
与原生代码一样，配置文件引导优化使我们的内联决策更加有效。

<!-- 
## Structs 
-->
## 结构

<!-- 
CLI structs are almost just like C structs.  Except they're not.  The CLI imposes some semantics that incur overheads.
These overheads almost always manifest as excessive copying.  Even worse, these copies are usually hidden from your
program.  It's worth noting, because of copy constructors and destructors, C++ also has some real issues here, often
even worse than what I'm about to describe. 
-->
CLI的结构几乎和C的结构很相似，但它们却不完全一样。
CLI强加了一些会产生开销的语义，而这些开销几乎总是表现为过度的复制，更糟糕的是，副本通常隐藏于程序中。 
值得注意的是，由于复制构造函数和析构函数，C++在这方面也存在一些实际问题，通常甚至比我将要描述的更加糟糕。

<!-- 
Perhaps the most annoying is that initializing a struct the CLI way requires a defensive copy.  For example, consider
this program, where the initialzer for `S` throws an exception: 
-->
但也许最令人讨厌的是，CLI初始化结构的方式需要防御性的副本。 
例如，考虑如下的程序，其中`S`的初始化方法抛出了异常：


    class Program {
        static void Main() {
            S s = new S();
            try {
                s = new S(42);
            }
            catch {
                System.Console.WriteLine(s.value);
            }
        }
    }

    struct S {
        public int value;
        public S(int value) {
            this.value = value;
            throw new System.Exception("Boom");
        }
    } 


<!-- 
The program behavior here has to be that the value `0` is written to the console.  In practice, that means that the
assignment operation `s = new S(42)` must first create a new `S`-typed slot on the stack, construct it, and *then* and
only then copy the value back over the `s` variable.  For single-`int` structs like this one, that's not a huge deal.
For large structs, that means resorting to `memcpy`.  In Midori, we knew what methods could throw, and which could not,
thanks to our error model (more later), which meant we could avoid this overhead in nearly all cases. 
-->
此处的程序预期行为是：必须将值`0`写入控制台中。 
实际上，这意味着赋值操作`s = new S(42)`必须首先在栈空间上创建一个新的`S`类型的slot并对其进行构造，*然后*仅仅将它的值赋给`s`变量。 
虽然对于像这样的单个`int`的结构来讲并不是什么大问题，但对于大型的结构，这意味着要使用`memcpy`函数进行赋值。
而在Midori中，我们知道哪些方法可以抛出异常，哪些方法无法抛出，
而这些都要归功于我们的错误模型（将在后面进行介绍），
所以也意味着我们几乎可以在所有的情况下避免这样的开销。

<!-- 
Another annoying one is the following: 
-->
另一个令人讨厌的地方出现在如下形式的代码中：

    struct S {
        // ...
        public int Value { get { return this.value; } }
    }
    
    static readonly S s = new S(); 


<!-- 
Every single time we read from `s.Value`: 
-->
每次从`s.Value`读取值：

    int x = s.Value; 

<!-- 
we are going to get a local copy.  This one's actually visible in the MSIL.  This is without `readonly`: 
-->
都将得到数据的局部副本。 
它实际上在MSIL中可见，且没有`readonly`关键字：

    ldsflda    valuetype S Program::s
    call       instance int32 S::get_Value() 


<!-- 
And this is with it: 
-->
也就是如下的形式：

    ldsfld     valuetype S Program::s
    stloc.0
    ldloca.s   V_0
    call       instance int32 S::get_Value() 


<!-- 
Notice that the compiler elected to use `ldsfld` followed by `lodloca.s`, rather than loading the address directly,
by way of `ldsflda` in the first example.  The resulting machine code is even nastier.  I also can't pass the struct
around by-reference which, as I mention later on, requires copying it and again can be problematic. 
-->
请注意，编译器选择使用`ldsfld`并由`lodloca.s`指令紧随其后，
而不是通过第一个示例中的`ldsflda`指令直接加载地址，
因此其生成的机器代码将会更加糟糕。
这里我们也无法通过引用传递结构体，正如我后面将要提到的那样，它需要数据的复制并且可能再次出现问题。

<!-- 
We solved this in Midori because our compiler knew about methods that didn't mutate members.  All statics were immutable
to begin with, so the above `s` wouldn't need defensive copies.  Alternatively, or in addition to this, the struct could
have beem declared as `immutable`, as follows: 
-->
由于我们的编译器知道方法不会改变成员的值，因此在Midori中，我们对此问题进行了解决。
因为所有的静态值都是不可变（`immutable`）的，所以上面的`s`不再需要防御性的副本。 
除此之外的另一种方式，结构体也同样可声明为`immutable`，如下所示：

<!-- 
    immutable struct S {
        // As above ...
    }  
-->
    immutable struct S {
        // 如上 ...
    }  

<!-- 
Or because all static values were immutable anyway.  Alternatively, the properties or methods in question could have
been annotated as `readable` meaning that they couldn't trigger mutations and hence didn't require defensive copies. -->

或者可采用另一种方式：由于所有静态值在任何情况下都是不可变的，因此存有疑问的属性或方法可注解为`readable`，这意味着它们不能被改变，所以也就不再需要防御性的副本。

<!-- 
I mentioned by-reference passing.  In C++, developers know to pass large structures by-reference, either using `*` or
`&`, to avoid excessive copying.  We got in the habit of doing the same.  For example, we had `in` parameters, as so: 
-->
我在前面已经提到了引用传递。 
在C++中，开发者知道使用`*`或`&`的方式，通过引用传递大型的结构，
从而避免过度的复制。
我们也养成了同样做法的习惯，例如，我们有名为`in`的参数：

<!-- 
    void M(in ReallyBigStruct s) {
        // Read, but don't assign to, s ...
    } 
-->   
    void M(in ReallyBigStruct s) {
        // 可读但不可写s的值 ...
    } 

<!-- 
I'll admit we probably took this to an extreme, to the point where our APIs suffered.  If I could do it all over again,
I'd go back and eliminate the fundamental distinction between `class` and `struct` in C#.  It turns out, pointers aren't
that bad after all, and for systems code you really do want to deeply understand the distinction between "near" (value)
and "far" (pointer).  We did implement what amounted to C++ references in C#, which helped, but not enough.  More on
this in my upcoming deep dive on our programming language. 
-->
我承认我们可能把这个问题推向极端，直到对我们的API产生了影响。
如果我能从头再来一次，我会回过头来消除C#中`class`和`struct`之间的根本区别。 
事实证明，指针毕竟没那么糟糕，并且对于系统代码而言，你真的需要深入理解“近”（值）和“远”（指针）之间的区别。 
我们确实在C#中实现了C++引用的功能，这能够带来一些帮助，但还不够。 
关于此更多内容将出现在未来对编程语言的深入研究中。

<!-- 
## Code size 
-->
## 代码体积

<!-- 
We pushed hard on code size.  Even more than some C++ compilers I know. 
-->
我们还努力推动减少代码体积，在此花费的努力甚至比我所知道的一些C++编译器还要多。

<!-- 
A generic instantiation is just a fancy copy-and-paste of code with some substitutions.  Quite simply, that means an
explosion of code for the compiler to process, compared to what the developer actually wrote.  [I've covered many of the
performance challenges with generics in the past.](
http://joeduffyblog.com/2011/10/23/on-generics-and-some-of-the-associated-overheads/)  A major problem there is the
transitive closure problem.  .NET's straightforward-looking `List<T>` class actually creates 28 types in its transitive
closure!  And that's not even speaking to all the methods in each type.  Generics are a quick way to explode code size. 
-->
泛型的实例化只是一些带有替换的代码复制和粘贴。 
很明显这意味着与开发人员实际编写的代码量相比，编译器要处理的代码数量将会激增。
[我在之前的文章中已经介绍了泛型的许多性能挑战](
http://joeduffyblog.com/2011/10/23/on-generics-and-some-of-the-associated-overheads/)，
一个主要问题就是传递闭包问题。
.NET中直观上的`List<T>`类实际上在其传递闭包中创建了多大28种类型！ 
而且甚至这还没把每种类型的所有方法包括进去。
因此，泛型是一种使代码体积爆炸的快速方法。


<!-- 
I never forgot the day I refactored our LINQ implementation.  Unlike in .NET, which uses extension methods, we made all
LINQ operations instance methods on the base-most class in our collection type hierarchy.  That meant 100-ish nested
classes, one for each LINQ operation, *for every single collection instantiated*!  Refactoring this was an easy way for
me to save over 100MB of code size across the entire Midori "workstation" operating system image.  Yes, 100MB! 
-->
我永远不会忘记重构实现LINQ的那段日子， 
与在.NET中使用扩展方法不同，我们在集合类型层次结构中的最底层基类上创建了所有LINQ操作的实例方法。 
*对于实例化的每个集合*，这意味着大约100个嵌套类！
每个LINQ操作对应于其中一个。
对其进行重构是一种可以在整个Midori“工作站”操作系统文件中节省超过100MB空间的简单方法。 
是的没错，节省了100MB！

<!-- 
We learned to be more thoughtful about our use of generics.  For example, types nested inside an outer generic are
usually not good ideas.  We also aggressively shared generic instantiations, even more than [what the CLR does](
http://blogs.msdn.com/b/joelpob/archive/2004/11/17/259224.aspx).  Namely, we shared value type generics, where the
GC pointers were at the same locations.  So, for example, given a struct S: 
-->
我们学会了更加周到地使用泛型，比如说在外部泛型内的嵌套类型通常不是什么好主意。
除此之外还积极地共享通用实例，甚至比[CLR所做](http://blogs.msdn.com/b/joelpob/archive/2004/11/17/259224.aspx)的更多。 
也就是说，我们共享了GC指针位于相同位置的值类型的泛型。
所以说，如果给定一个结构S：

    struct S {
        int Field;
    } 


<!-- 
we would share the same code representation of `List<int>` with `List<S>`.  And, similarly, given: 
-->
那么`List<S>`将共享与`List<int>`相同的代码表示。 
并且同样地，假定有如下的结构：


    struct S {
        object A;
        int B;
        object C;
    } 

    struct T {
        object D;
        int E;
        object F;
    } 


<!-- 
we would share instantiations between `List<S>` and `List<T>`. 
-->
那么`List<S>`和`List<T>`之间将共享实例。

<!-- 
You might not realize this, but C# emits IL that ensures `struct`s have `sequential` layout: 
-->
另外，你可能没有意识到的一点是，C#生成了确保`struct`结构具有`sequential`特性布局的IL：


    .class private sequential ansi sealed beforefieldinit S
        extends [mscorlib]System.ValueType
    {
        ...
    } 

<!-- 
As a result, we couldn't share `List<S>` and `List<T>` with some     `List<U>`: 
-->
结果是，我们不能与一些假设的`List<U>`共享`List<S>`和`List<T>`：

    struct U {
        int G;
        object H;
        object I;
    } 


<!-- 
For this, among other reasons -- like giving the compiler more flexibility around packing, cache alignment, and so on
-- we made `struct`s `auto` by default in our language.  Really, `sequential` only matters if you're doing unsafe code,
which, in our programming model, wasn't even legal. 
-->
为此，除了其他原因例如让编译器在打包，缓存对齐等方面具有更大的灵活性之外，我们在语言的`struct`中默认使用`auto`。 
实际上，`sequential`只对你进行不安全代码时很重要，而在我们的编程模型中，这样的代码甚至都是不合法的。

<!-- 
We did not support reflection in Midori.  In principle, we had plans to do it eventually, as a purely opt-in feature.
In practice, we never needed it.  What we found is that code generation was always a more suitable solution.  We shaved
off at least 30% of the best case C# image size by doing this.  Significantly more if you factor in systems where the
full MSIL is retained, as is usually the case, even for NGen and .NET AOT solutions. 
-->
我们没有在Midori中支持反射机制。
原则上，我们最终计划将其作为纯粹的可选功能项，而在实践中从来都不需要它。 
我们发现代码生成始终都是更合适的解决方案，通过这种做法，对于C#的文件体积，我们在最佳情况下至少减少了30%。 
如果你考虑的是保留完整MSIL的系统，正如通常情况下的做法，即使对于NGen和.NET的AOT解决方案，将能够进一步减少代码的体积。

<!-- 
In fact, we removed significant pieces of `System.Type` too.  No `Assembly`, no `BaseType`, and yes, even no `FullName`.
The .NET Framework's mscorlib.dll contains about 100KB of just type names.  Sure, names are useful, but our eventing
framework leveraged code generation to produce just those you actually needed to be around at runtime. 
-->
实际上，我们也删除了很多`System.Type`，使得没有`Assembly`，没有`BaseType`，是的，甚至没有`FullName`类型。 
.NET Framework的mscorlib.dll中仅类型名称就有大约100KB。
当然，名称也是很有用的，但我们的事件框架利用代码生成来产生仅在运行时实际需要的那部分代码。

<!-- 
At some point, we realized 40% of our image sizes were [vtable](https://en.wikipedia.org/wiki/Virtual_method_table)s.
We kept pounding on this one relentlessly, and, after all of that, we still had plenty of headroom for improvements. 
-->
在某个时刻，我们意识到生成的可执行文件大小的40%都是
[vtable](https://en.wikipedia.org/wiki/Virtual_method_table)。
我们一直在坚持不懈地减少这部分的大小，毕竟由于所述的一切，我们仍然有足够的改进空间。

<!-- 
Each vtable consumes image space to hold pointers to the virtual functions used in dispatch, and of course has a runtime
representation.  Each object with a vtable also has a vtable pointer embedded within it.  So, if you care about size
(both image and runtime), you are going to care about vtables. 
-->
每个vtable都使用文件中的部分空间来保存指向调度中使用的虚函数的指针，
当然同时还有一个运行时的表示。
另外，具有vtable的每个对象也同时具有嵌入其中的vtable指针， 
所以，如果你关心（整个映像和运行时）的文件大小，你就会对vtable有所关注。

<!-- 
In C++, you only get a vtable if your type is [polymorphic](http://www.cplusplus.com/doc/tutorial/typecasting/).  In
languages like C# and Java, on the other hand, you get a vtable even if you don't want, need, or use it.  In C#, at
least, you can use a `struct` type to elide them.  I actually love this aspect of Go, where you get a virtual dispatch-
like thing, via interfaces, without needing to pay for vtables on every type; you only pay for what you use, at the
point of coercing something to an interface. 
-->
在C++中，如果类型是[多态的](http://www.cplusplus.com/doc/tutorial/typecasting/)，
那么它只会有一个vtable。 
另一方面，在C#和Java等语言中，即使不想要或不需要使用它，类型也具有vtable，虽然说在C#中，你可以使用`struct`结构类型来忽略它们。 
我真的很喜欢Go的这个方面的做法，因为你可以通过接口获得类似虚拟调度的功能，
而只需为某些类型强制实现接口，而无需为每种类型带来vtable的开销。

<!-- 
Another vtable problem in C# is that all objects inherit three virtuals from `System.Object`: `Equals`, `GetHashCode`,
and `ToString`.  Besides the point that these generally don't do the right thing in the right way anyways -- `Equals`
requires reflection to work on value types, `GetHashCode` is nondeterministic and stamps the object header (or sync-
block; more on that later), and `ToString` doesn't offer formatting and localization controls -- they also bloat every
vtable by three slots.  This may not sound like much, but it's certainly more than C++ which has no such overhead. 
-->
C#中vtable的另一个问题是，所有对象都从`System.Object`继承了三个虚拟类：`Equals`，`GetHashCode`和`ToString`。 
除了这些类通常不会以正确的方式做正确的事情之外，还存在其他的问题：
`Equals`需要反射机制来处理值类型，
`GetHashCode`是非确定性的并且标记了对象头（或同步块，稍后会有更多内容的讨论），
而`ToString`不提供格式化和本地化控制。
它们的存在也会使每个vtable增加三个位置，这虽然听起来可能不是很多，但它肯定比没有这些开销的C++体积要更大。

<!-- 
The main source of our remaining woes here was the assumption in C#, and frankly most OOP languages like C++ and Java,
that [RTTI](https://en.wikipedia.org/wiki/Run-time_type_information) is always available for downcasts.  This was
particularly painful with generics, for all of the above reasons.  Although we aggressively shared instantiations, we
could never quite fully fold together the type structures for these guys, even though disparate instantiations tended
to be identical, or at least extraordinarily similar.  If I could do it all over agan, I'd banish RTTI.  In 90% of the
cases, type discriminated unions or pattern matching are more appropriate solutions anyway. 
-->
我们剩下的困扰主要来自于基于C#的假设，坦率地说，对于大多数OOP语言（如C++和Java）而言，[RTTI](https://en.wikipedia.org/wiki/Run-time_type_information)始终可用于向下转换，因此对于泛型来讲，这尤其痛苦。
虽然我们激进地采取了共享实例方法，但永远无法完全共享所有这些类型的结构，即使不同的实例往往是相同的或者至少是非常相似的。 
如果我能再来一次，我会放弃RTTI，
因为在90%的情况下，无论如何类型区分联合体或模式匹配都是更合适的解决方案。

<!-- 
## Profile guided optimizations (PGO) 
-->
## 配置文件引导优化（PGO）

<!-- 
I've mentioned [profile guided optimization](https://en.wikipedia.org/wiki/Profile-guided_optimization) (PGO) already.
This was a critical element to "go that last mile" after mostly everything else in this article had been made
competitive.  This gave our browser program boosts in the neighborhood of 30-40% on benchmarks like [SunSpider](
https://webkit.org/perf/sunspider/sunspider.html) and [Octane](https://developers.google.com/octane/). 
-->
在前文中，我已经提到过[配置文件引导优化（PGO）](https://en.wikipedia.org/wiki/Profile-guided_optimization)。
而在本文中的大部分方法都具有竞争力之后，PGO成为“走完最后一英里”的关键因素，
它也使我们的浏览器程序在[SunSpider](https://webkit.org/perf/sunspider/sunspider.html)和[Octane](https://developers.google.com/octane/)等基准测试中增加了30%—40%的性能。

<!-- 
Most of what went into PGO was similar to classical native profilers, with two big differences. 
-->
PGO的大部分工作与传统的原生profiler类似，但有两个地方有很大的不同。

<!-- 
First, we tought PGO about many of the unique optimizations listed throughout this article, such as asynchronous stack
probing, generics instantiations, lambdas, and more.  As with many things, we could have gone on forever here. 
-->
首先，我们向PGO新增了本文中列出的许多独特优化方法，例如异步堆栈探测，泛型实例化和lambda等。
和其他许多地方一样，我们可以在PGO上永远无休止地优化下去。

<!-- 
Second, we experimented with sample profiling, in addition to the ordinary instrumented profiling.  This is much nicer
from a developer perspective -- they don't need two builds -- and also lets you collect counts from real, live running
systems in the data center.  A good example of what's possible is outlined in [this Google-Wide Profiling (GWP) paper](
http://static.googleusercontent.com/media/research.google.com/en/us/pubs/archive/36575.pdf). 
-->
其次，除了普通性能测量分析之外，我们还尝试了样本分析。 
从开发这的角度上来看，这样做可能会更好，因为它们不需要两次构建，并且还允许从数据中心的实际运行系统中收集和计数。
在[Google-Wide分析方法（GWP）](http://static.googleusercontent.com/media/research.google.com/en/us/pubs/archive/36575.pdf)一文中所概述的，可能是一个不错的例子。


<!-- 
# System Architecture 
-->
# 系统结构

<!-- 
The basics described above were all important.  But a number of even more impactful areas required deeper architectural
co-design and co-evolution with the language, runtime, framework, and operating system itself.  I've written about [the
immense benefits of this sort of "whole system" approach before](
http://joeduffyblog.com/2014/09/10/software-leadership-7-codevelopment-is-a-powerful-thing/).  It was kind of magical. 
-->
上述的基础知识都是重要的，但在一些更具影响力的领域，
则需要与语言、运行时、框架和操作系统本身进行更深层次的架构协同设计和协同演进。 
我之前写过[关于这种“整个系统”方法的巨大好处](http://joeduffyblog.com/2014/09/10/software-leadership-7-codevelopment-is-a-powerful-thing/)，它也是有几分神奇的。

<!-- 
## GC 
-->
## GC

<!-- 
Midori was garbage collected through-and-through.  This was a key element of our overall model's safety and
productivity.  In fact, at one point, we had 11 distinct collectors, each with its own unique characteristics.  (For
instance, see [this study](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.353.9594&rep=rep1&type=pdf).)  We
had some ways to combat the usual problems, like long pause times.  I'll go through those in a future post, however.
For now, let's stick to the realm of code quality. 
-->
Midori在各个方面采用了垃圾回收机制，这是我们在整体模型的安全性和生产率上的关键因素。 
事实上，在一个收集点上，我们有11个不同的收集器，每个收集器都有自己独特的特征（比如，参考[这项研究](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.353.9594&rep=rep1&type=pdf)）。
我们有一些方法可以解决诸如长时间停顿等常见问题。 
不过，我会在以后的文章中介绍这些内容，
现在，让我们回到代码质量方面。

<!-- 
The first top-level decision is: *conservative* or *precise*?  A conserative collector is easier to wedge into an
existing system, however it can cause troubles in certain areas.  It often needs to scan more of the heap to get the
same job done.  And it can falsely keep objects alive.  We felt both were unacceptable for a systems programming
environment.  It was an easy, quick decision: we sought precision. 
-->
首要的顶层的决定是：*保守*还是*精确*？ 
一个保守的垃圾收集器更容易融入现有系统，但它可能会在某些地方带来麻烦：
它通常需要扫描更多的堆才能完成相同的工作， 
并且可能错误地使对象保持存活。 
我们认为这两者对于系统编程环境都是不可接受的，
所以，我们做出了简单且快速的决定：我们追求精确。

<!-- 
Precision costs you something in the code generators, however.  A precise collector needs to get instructions where to
find its root set.  That root set includes field offsets in data structures in the heap, and also places on the stack
or, even in some cases, registers.  It needs to find these so that it doesn't miss an object and erroneously collect it
or fail to adjust a pointer during a relocation, both of which would lead to memory safety problems.  There was no magic
trick to making this efficient other than close integration between runtime and code generator, and being thoughtful. 
-->
但是，精确会给代码生成器带来开销。
精确的垃圾收集器需要告知它在哪里能够找到根集合（root set），
该根集合包括堆中的数据结构的字段偏移，并且还包括栈，甚至在某些情况下还包括寄存器。 
所以需要找到它们的全部，使得避免错过任何一个对象、
错误地将其回收或者在重定位期间调整指针失败，而这些情况都会导致内存安全问题。
而除了运行时和代码生成器之间的紧密集成和周密考虑之外，没有任何神奇的技巧可以使这个过程变得高效。


<!-- 
This brings up the topic of *cooperative* versus *preemptive*, and the notion of GC safe-points.  A GC operating in
cooperative mode will only collect when threads have reached so-called "safe-points."  A GC operating in preemptive
mode, on the other hand, is free to stop threads in their tracks, through preemption and thread suspension, so that it
may force a collection.  In general, preemptive requires more bookkeeping, because the roots must be identifiable at
more places, including things that have spilled into registers.  It also makes certain low-level code difficult to
write, of the ilk you'll probably find in an operating system's kernel, because objects are subject to movement between
arbitrary instructions.  It's difficult to reason about.  (See [this file](
https://github.com/dotnet/coreclr/blob/master/src/vm/eecontract.h), and its associated uses in the CLR codebase, if you
don't believe me.)  As a result, we used cooperative mode as our default.  We experimented with automatic safe-point
probes inserted by the compiler, for example on loop back-edges, but opted to bank the code quality instead.  It did
mean GC "livelock" was possible, but in practice we seldom ran into this. 
-->
这就进入是*协作式*还是*抢占式*的主题以及GC安全点的概念。
以协作式运行的GC操作只会在线程达到所谓的“安全点”时进行收集，
而以抢占式运行的GC可以通过抢占和线程暂停等方式自由地阻止部分线程，因此可能会强制性回收。
一般来说，抢占式需要更多的信息记录，因为它必须在更多的地方识别根集合，包括已经溢出到寄存器中的数据。
同时由于对象可能会在任意指令之间移动，
它还可能导致某些在操作系统内核中出现的底层代码难以编写，而这些情况都很难进行推断
（如果你对此存有怀疑，请参考此[代码](https://github.com/dotnet/coreclr/blob/master/src/vm/eecontract.h)
及其在CLR代码库中的相关用法），因此，我们使用了协作式GC作为默认手段。
我们以保证代码质量作为目的，尝试使用编译器插入的自动安全点探针，例如在循环后沿上。
这种方式确实意味着GC“活锁”是有可能的，但在实践中我们很少遇到这种情况。


<!-- 
We used a *generational* collector.  This has the advantage of reducing pause times because less of the heap needs to be
inspected upon a given collection.  It does come with one disadvantage from the code generator's perspective, which is
the need to insert write barriers into the code.  If an older generation object ever points back at a younger generation
object, then the collector -- which would have normally preferred to limit its scope to younger generations -- must know
to look at the older ones too.  Otherwise, it might miss something. 
-->
我们使用*分代*收集器，它具有减少暂停时间的优点，因为在给定集合上只需检查较少的堆。 
从代码生成器的角度来看，它确实存在一个缺点，即需要在代码中插入写屏障。 
如果老年代的对象曾经指向一个新生代的对象，那么收集器通常倾向于将其范围限制在新生代中，
这么一来收集器也要检查老年代的对象，否则可能就会错过一些对象的收集。

<!-- 
Write barriers show up as extra instructions after certain writes; e.g., note the `call`: 
-->
写屏障以跟随特定写入指令的额外指令的形式出现。
比如说，注意如下的`call`指令：

    48 8D 49 08        lea         rcx,[rcx+8]
    E8 7A E5 FF FF     call        0000064488002028 


<!-- 
That barrier simply updates an entry in the card table, so the GC knows to look at that segment the next time it scans
the heap.  Most of the time this ends up as inlined assembly code, however it depends on the particulars of the
situation.  See [this code](https://github.com/dotnet/coreclr/blob/master/src/vm/amd64/JitHelpers_Fast.asm#L462) for an
example of what this looks like for the CLR on x64. 
-->
该屏障只是简单地更新表中的项，使得GC知道下次扫描堆时要检查该段， 
大多数情况下，它们最终都形成了内联汇编代码，但也取决于具体情况。 
对于x64的CLR在这方面的示例做法，请参考
[该代码](https://github.com/dotnet/coreclr/blob/master/src/vm/amd64/JitHelpers_Fast.asm#L462)。

<!-- 
It's difficult for the compiler to optimize these away because the need for write barriers is "temporal" in nature.  We
did aggressively eliminate them for stack allocated objects, however.  And it's possible to write, or transform code,
into less barrier hungry styles.  For example, consider two ways of writing the same API: 
-->
编译器难以对其进行优化的原因是因为对写屏障的需求在本质上是“暂时的”。 
但是，我们确实在积极地为栈分配对象消除它们，
并且可以将代码编写或转换为更少屏障的风格。
比如说，考虑如下相同API的方法：

    bool Test(out object o);
    object Test(out bool b); 

<!-- 
In the resulting `Test` method body, you will find a write barrier in the former, but not the latter.  Why?  Because the
former is writing a heap object reference (of type `object`), and the compiler has no idea, when analyzing this method
in isolation, whether that write is to another heap object.  It must be conservative in its analysis and assume the
worst.  The latter, of course, has no such problem, because a `bool` isn't something the GC needs to scan. 
-->
在生成的`Test`方法的函数体中，在前者中能找到写屏障指令，而在后者中则不会，这是为什么呢？ 
因为前者正在向堆对象引用（类型为`object`）写入，并且编译器在单独分析此方法时，无法知道写入的是否是另一个堆对象，
所以这里的分析必须是保守式，并且需要做最坏的假设。 
当然后者没有这样的问题，因为`bool`不是GC需要扫描的类型。

<!-- 
Another aspect of GC that impacts code quality is the optional presence of more heavyweight concurrent read and write
barriers, when using concurrent collection.  A concurrent GC does some collection activities concurrent with the user
program making forward progress.  This is often a good use of multicore processors and it can reduce pause times and
help user code make more forward progress over a given period of time. 
-->
GC影响代码质量的另一个方面问题是，在使用并发收集时可选地保存更重量级的并发读写屏障。 
并发GC是在用户程序运行的同时进行垃圾回收活动，这种方法通常可以很好地利用多核处理器，
减少暂停时间，并帮助用户代码在给定的时间段内运行更多的代码。

<!-- 
There are many challenges with building a concurrent GC, however one is that the cost of the resulting barriers is high.
The original [concurrent GC by Henry Baker](
http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.23.5878&rep=rep1&type=pdf) was a copying GC and had the notion
of "old" versus "new" space.  All reads and writes had to be checked and, anything operation against the old space had
to be forwarded to the new space.  Subsequent research for the DEC Firefly used hardware memory protection to reduce the
cost, but the faulting cases were still exceedingly expensive.  And, worst of all, access times to the heap were
unpredictable.  There has been [a lot of good research into solving this problem](
http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.69.1875&rep=rep1&type=pdf), however we abandoned copying. 
-->
构建并发GC存在诸多的挑战，但其中一个问题是它产生屏障的开销非常高。 
[由Henry Baker提出的原始并发GC方法](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.23.5878&rep=rep1&type=pdf)采用复制GC方式，
具有空间“旧”与“新”的概念，所有的读和写都必须进行检查，
并且必须将任何针对旧空间的操作都需要转移到新空间中。 
DEC Firefly的后续研究使用硬件内存保护来降低开销，但故障情况的处理仍然非常耗时，
而且，最糟糕的问题是，堆的访问时间是不可预测的。 
为了解决这个问题已经进行了[很多很好的研究](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.69.1875&rep=rep1&type=pdf)，
但我们最终还是放弃了复制的做法。

<!-- 
Instead, we used a concurrent mark-sweep compacting collector.  This means only write barriers are needed under normal
program execution, however some code was cloned so that read barriers were present when programs ran in the presence of
object movement.  [Our primary GC guy's research was published](
http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.108.322&rep=rep1&type=pdf), so you can read all about it.  The
CLR also has a concurrent collector, but it's not quite as good.  It uses copying to collect the youngest generation,
mark-sweep for the older ones, and the mark phase is parallelized.  There are unfortunately a few conditions that can
lead to sequential pauses (think of this like a big "lock"), sometimes over 10 milliseconds: 1) all threads must be
halted and scanned, an operation that is bounded only by the number of threads and the size of their stacks; 2) copying
the youngest generation is bounded only by the size of that generation (thankfully, in normal configurations, this is
small); and 3) under worst case conditions, compaction and defragmentation, even of the oldest generation, can happen. 
-->
相反，我们采用了并发标记-清除压缩收集方法。
这意味着在程序正常执行期间只需加入写屏障，但是某些代码需要被克隆，以便在存在对象移动时具有读屏障。
[我们的主要GC开发人员的研究已经发表](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.108.322&rep=rep1&type=pdf)，你可以阅读所有相关的内容。
CLR也有一个并发的收集器，但它并非优秀，它主要使用复制方式来收集最年轻的一代，对老年代则采用标记清除方法并将标记阶段并行化处理。
遗憾的是，有一些条件会导致顺序性的暂停（想象一下，这就像一把大“锁”），有时甚至超过10毫秒：
1）所有线程必须暂停和扫描，这个操作只受线程数和堆大小的限制；
2）复制最年轻的一代只受那一代大小的限制（幸运的是，在正常配置下该代内存数量很少）；
3）在最坏的情况下，即使是最老的一代也可能发生压缩和碎片整理操作。


<!-- 
## Separate compilation 
-->
## 独立编译

<!-- 
The basic model to start with is static linking.  In this model, you compile everything into a single executable.  The
benefits of this are obvious: it's simple, easy to comprehend, conceptually straightforward to service, and less work
for the entire compiler toolchain.  Honestly, given the move to Docker containers as the unit of servicing, this model
makes more and more sense by the day.  But at some point, for an entire operating system, you'll want separate
compilation.  Not just because compile times can get quite long when statically linking an entire operating system, but
also because the working set and footprint of the resulting processes will be bloated with significant duplication. 
-->
这部分以最基本模型——静态链接作为开始。
在此模型中，所有的内容被编译成单个可执行文件， 
这样做的好处是显而易见的：它简单易懂，服务起来概念简单，整个编译器工具链的工作量更少。 
老实说，考虑到正在发生的将Docker容器作为服务单元的运动，这种模式在当前变得越来越有意义。 
但在某些时候，对于整个操作系统而言，需要的可能是独立编译，
这不仅仅是因为静态链接编译整个操作系统的时间会很长，
而且是因为生成的进程工作集和占用空间存在大量的重复。

<!-- 
Separately compiling object oriented APIs is hard.  To be honest, few people have actually gotten it to work.  Problems
include the [fragile base class problem](https://en.wikipedia.org/wiki/Fragile_base_class), which is a real killer for
version resilient libraries.  As a result, most real systems use a dumbed down ["C ABI"](
https://en.wikipedia.org/wiki/Application_binary_interface) at the boundary between components.  This is why Windows,
for example, has historically used flat C Win32 APIs and, even in the shift to more object orientation via WinRT, uses
COM underneath it all.  At some runtime expense, the ObjectiveC runtime addressed this challenge.  As with most things
in computer science, virtually all problems can be solved with an extra level of indirection; [this one can be too](
http://www.sealiesoftware.com/blog/archive/2009/01/27/objc_explain_Non-fragile_ivars.html). 
-->
独立编译面向对象的API是一件很难的事情，说实话，很少有人真正将其搞定。
这里的问题包括[脆弱的基类问题](https://en.wikipedia.org/wiki/Fragile_base_class)，这也是版本有弹性的库的真正杀手。 
因此，大多数真实系统在组件之间的边界处使用了笨拙的
[“C ABI”](https://en.wikipedia.org/wiki/Application_binary_interface)。 
这就是为什么Windows在历史上使用普通C语言的Win32 API，即使在底层使用COM并通过WinRT转到更多面向对象的情况下也依然如此。 
在花费一定运行时的开销的条件下，Objective C的运行时解决了这一挑战。
而与计算机科学中的大多数事物一样，几乎所有问题都可以通过额外的间接抽象来解决，[这个问题也依然如此](http://www.sealiesoftware.com/blog/archive/2009/01/27/objc_explain_Non-fragile_ivars.html)。

<!-- 
The design pivot we took in Midori was that whole processes were sealed.  There was no dynamic loading, so nothing that
looked like classical DLLs or SOs.  For those scenarios, we used the [Asynchronous Everything](
http://joeduffyblog.com/2015/11/19/asynchronous-everything/) programming model, which made it easy to dynamically
connect to and use separately compiled and versioned processes. 
-->
我们在Midori中采用的设计思路是所有进程都是密封的（sealed），
没有动态加载，所以没有看起来像经典的DLL或SO的库文件。 
对于这些场景，我们使用了[一切皆异步](/2018/11/25/midori/3-asynchronous-everything/)的编程模型，
这使得它和使用独立编译和版本化的进程动态连接变得容易。

<!-- 
We did, however, want separately compiled binaries, purely as a developer productivity and code sharing (working set)
play.  Well, I lied.  What we ended up with was incrementally compiled binaries, where a change in a root node triggered
a cascading recompilation of its dependencies.  But for leaf nodes, such as applications, life was beautiful.  Over
time, we got smarter in the toolchain by understanding precisely which sorts of changes could trigger cascading
invaliation of images.  A function that was known to never have been inlined across modules, for example, could have its
implementation -- but not its signature -- changed, without needing to trigger a rebuild.  This is similar to the
distinction between headers and objects in a classical C/C++ compilation model. 
-->
但是，我们确实需要独立编译的二进制文件，纯粹是由于开发者的工作效率和代码共享（工作集）所导致。
好吧，我承认我之前说谎了。 
我们最终得到的是增量编译的二进制文件，其中根节点的更改会触发其依赖项级联式的重新编译，
但对于叶节点，比如说应用程序而言，情况却要好得多。 
随着时间的推移，我们通过精确了解哪种类型的更改可以触发映像文件的级联失效，使得工具链变得更加智能。 
例如，一个从未在模块之间发生内联的函数如果它的实现（而不是函数签名）发生了更改，则无需触发重建，
这类似于传统C/C++编译模型中头文件和对象之间的区别。

<!-- 
Our compilation model was very similar to C++'s, in that there was static and dynamic linking.  The runtime model, of
course, was quite different.  We also had the notion of "library groups," which let us cluster multiple logically
distinct, but related, libraries into a single physical binary.  This let us do more aggressive inter-module
optimizations like inlining, devirtualization, async stack optimizations, and more. 
-->
因为我们的编译模型也有静态和动态链接，所以与C++的非常相似，当然运行时模型则是完全不同的。
我们还有“库分组”的概念，它允许我们将多个逻辑上不同但相关的库集中到一个物理的二进制文件中，
这让我们可以进行更激进的模块间优化，如内联，虚拟化和异步堆栈优化等。

<!-- 
## Parametric polymorphism (a.k.a., generics) 
-->
## 参数多态（也就是泛型）

<!-- 
That brings me to generics.  They throw a wrench into everything. 
-->
上面的内容让我想到了泛型（generics），它是能把一切都搞砸的特性。

<!-- 
The problem is, unless you implement an [erasure
model](https://docs.oracle.com/javase/tutorial/java/generics/erasure.html) -- which utterly stinks performance-wise due
to boxing allocations, indirections, or both -- there's no way for you to possibly pre-instantiate all possible versions
of the code ahead-of-time.  For example, say you're providing a `List<T>`.  How do you know whether folks using your
library will want a `List<int>`, `List<string>`, or `List<SomeStructYouveNeverHeardOf>`? 
-->
这里的问题是，除非你实现一个因为Box分配，间接取值或两者同时具备的，完全以性能为代价的
[擦除模型](https://docs.oracle.com/javase/tutorial/java/generics/erasure.html)，你是没有办法预先实例化代码的所有可能版本。 
比如说，假设你提供了`List<T>`，你怎么知道使用库的人需要的是`List<int>`，`List<string>`还是`List<SomeStructYouveNeverHeardOf>`？

<!-- 
Solutions abound: 
-->
解决的方案有很多：

<!-- 
1. Do not specialize.  Erase everything.
2. Specialize only a subset of instantiations, and create an erased instantiation for the rest.
3. Specialize everything.  This gives the best performance, but at some complexity. 
-->
1. 不专门处理，擦除一切；
2. 仅专门实例化其中的一个子集，并为其余实例创建擦除后的实例。
3. 专门处理所有实例，它能够提供最佳的性能，但同时也有些复杂。

<!--
Java uses #1 (in fact, erasure is baked into the language).  Many ML compilers use #2.  .NET's NGen compilation
model is sort of a variant of #2, where things that can be trivially specialized are specialized, and everything else is
JIT compiled.  .NET Native doesn't yet have a solution to this problem, which means 3rd party libraries, separate
compilation, and generics are a very big TBD.   As with everything in Midori, we picked the hardest path, with the most
upside, which meant #3.  Actually I'm being a little glib; we had several ML compiler legends on the team, and #2 is
fraught with peril; just dig a little into [some papers](
http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.98.2165&rep=rep1&type=pdf) on how hard (and clever) this can
get.  It's difficult to know a priori which instantiations are going to be performance critical to a program.  My own
experience trying to get C# code into the heart of Windows back in the Longhorn days also reinforced this; we didn't
want JIT'ting and the rules for what generics you could and couldn't use in that world were so mind boggling they
eventually led to greek formulas. 
-->
Java使用的是方案1（事实上，擦除模型已经合并到其语言中），而许多ML编译器使用方案2。
.NET的NGen编译模型是方案2的变体，其中可以简单地专门化处理的都已专门处理，而其他都是通过JIT编译的。 
.NET Native还没有这个问题的解决方案，这意味着第三方库，独立编译和泛型是存在着非常大的TBD。
和Midori其他一切内容一样，我们选择了最艰难但却最具有上升空间的道路，这里就意味着是方案3。
实际上，我说的有点夸张，我们的团队中有几个ML编译器的传奇人物，所以知道方案2充满着危险。
如果能稍微深入了解一下[这篇论文](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.98.2165&rep=rep1&type=pdf)
就知道这个方案有多难（和聪明），因为很难先验地知道哪些实例化对程序至关重要。
我自己在Longhorn（其正式的名称是Vista）的年代试图将C#代码置入Windows核心的经验也强化了这样的信念：
我们不想要JIT，哪些泛型可以使用和哪些泛型不可使用的规则是如此令人难以置信，使得其最终转变为希腊公式。

<!-- 
Anyway, Midori's approach turned out to be harder than it sounded at first. 
-->
无论如何，Midori的方法比最初听起来更加困难。

<!-- 
Imagine you have a diamond.  Library A exports a `List<T>` type, and libraries B and C both instantiate `List<int>`.  A
program D then consumes both B and C and maybe even passes `List<T>` objects returned from one to the other.  How do we
ensure that the versions of `List<int>` are compatible? 
-->
设想一下你有如下的菱形关系。 
库A导出了`List<T>`类型，库B和C都实例化了`List<int>`，而程序D则同时使用了B和C，甚至将`List<T>`对象从一个库传递到另一个库。
那么我们应如何确保`List<int>`的版本是兼容的？

<!-- 
We called this problem the *potentially multiply instantiated*, or PMI for short, problem. 
-->
我们称此问题为*潜在的多次实例化*，或简称为PMI问题。

<!-- 
The CLR handles this problem by unifying the instantiations at runtime.  All RTTI data structures, vtables, and whatnot,
are built and/or aggressively patched at runtime.  In Midori, on the other hand, we wanted all such data structures to
be in readonly data segments and hence shareable across processes, wherever possible. 
-->
CLR通过在运行时统一实例化来处理此问题，
所有的RTTI数据结构，vtable和诸如此类的东西都在运行时构建和/或激进地进行修补。 
而另一方面，在Midori中，我们希望所有这些数据结构都在只读数据段中出现，因此可以尽可能地在各个进程之间共享。

<!-- 
Again, everything can be solved with an indirection.  But unlike solution #2 above, solution #3 permits you to stick
indirections only in the rare places where you need them.  And for purposes of this one, that meant RTTI and accessing
static variables of just those generic types that might have been subject to PMI.  First, that affected a vast subset of
code (versus #2 which generally affects even loading of instance fields).  Second, it could be optimized away for
instantiations that were known not to be PMI, by attaching state and operations to the existing generic dictionary that
was gets passed around as a hidden argument already.  And finally, because of all of this, it was pay for play. 
-->
再一次地，一切都可以通过间接抽象来解决。 
但与上面的方案#2不同的是，方案#3允许只在需要使用它们的地方加入间接抽象。 
就本文而言，这意味着RTTI和访问那些可能受PMI影响的泛型类型的静态变量。 
首先，它影响了大量代码（相对而言，方案#2通常影响的是实例字段的加载）；
其次，它可以通过将状态和操作附加到已经作为隐藏参数传递的，现有的泛型字典来优化明确不是PMI的实例化。 
最后，因为以上所有的这一切，也将付出相应的性能上的代价。

<!-- 
But damn was it complex. 
-->
但该死的是它过于复杂。

<!-- 
It's funny, but C++ RTTI for template instantiations actually suffers from many of the same problems.  In fact, the
Microsoft Visual C++ compiler resorts to a `strcmp` of the type names, to resolve diamond issues!  (Thankfully there are
[well-known, more efficient ways to do this](
http://www6.open-std.org/JTC1/SC22/WG21/docs/papers/1992/WG21%201992/X3J16_92-0068%20WG21_N0145.pdf), which we are
actively pursuing for the next release of VC++.) 
-->
很有意思的是，用于模板实例化的C++ RTTI实际上遇到了许多相同的问题。 
事实上上，Microsoft Visual C++编译器在类名上使用`strcmp`字符串比较的方法，以解决菱形问题 
（值得庆幸的是，有一些[众所周知的，更有效的方法可以做到这一点](http://www6.open-std.org/JTC1/SC22/WG21/docs/papers/1992/WG21%201992/X3J16_92-0068%20WG21_N0145.pdf)，我们正在积极关注下一版VC++）！

<!-- 
## Virtual dispatch 
-->
## 虚拟调度（Virtual Dispatch） 

<!-- 
Although I felt differently when first switching from Java to C#, Midori made me love that C# made methods non-virtual
by default.  I'm sure we would have had to change this otherwise.  In fact, we went even further and made classes
`sealed` by default, requiring that you explicitly mark them `virtual` if you wanted to facilitate subclasses. 
-->
虽然在我首次从Java切换到C#时感觉很不一样，但Midori让我喜欢上了C#默认情况将方法以非虚拟的方式存在的做法，所以我相信我们不得不改变这一点。
事实上我们更进了一步，即在默认情况下使用`sealed`类型的类，
如果你想便利地使用子类，则需要明确地使用`virtual`标记它们。

<!-- 
Aggressive devirtualization, however, was key to good performance.  Each virtual means an indirection.  And more
impactfully, a lost opportunity to inline (which for small functions is essential).  We of course did global
intra-module analysis to devirtualize, but also extended this across modules, using whole program compilation, when
multiple binaries were grouped together into a library group. 
-->
然而，激进的去虚拟化是获得优异性能的关键，
那是因为，每个虚拟方法意味着一次抽象，而影响更大的是，虚函数使得其失去了内联的机会（而这对于小型函数来说是必不可少的优化方法）。 
当然，我们还进行了全局模块内分析以支持去虚拟化，但当为了编译整个程序而需将多个二进制文件组合成一整个库时，也扩展到模块之间的分析。

<!-- 
Although our defaults were right, my experience with C# developers is that they go a little hog-wild with virtuals and
overly abstract code.  I think the ecosystem of APIs that exploded around highly polymorphic abstractions, like LINQ and
Reactive Extensions, encouraged this and instilled a bit of bad behavior ("gratuitous over-abstraction").  I guess you
could make similar arguments about highly templated code in C++.  As you can guess, there wasn't very much of it in the
lowest levels of our codebase -- where every allocation and instruction mattered -- but in higher level code, especially
in applications that tended to be dominated by high-latency asynchronous operations, the overheads were acceptable and
productivity benefits high.  A strong culture around identifying and trimming excessive fat helped to ensure features
like this were used appropriately, via code reviews, benchmarks, and aggressive static analysis checking. 
-->
尽管我们的默认设置是正确的，但对C#开发者来说，我的经验是他们对虚拟和过于抽象的代码存有兴趣。 
围绕高度多态的抽象（如LINQ和Reactive Extensions）的API生态系统的剧增助长了这种情况的发生，并灌输了一些的不良行为（比如“无偿的过度抽象”），
我想你可以在C++中高度模板化的代码上得出类似的观点。 
正如你所猜测的那样，在我们的每个分配和指令都很重要的代码库最低层，并没有很多这样的问题；
但在更高层次的代码中，特别是在那些倾向于由高延迟异步操作主导的，可以接受一定的开销，且非常注重开发效率的应用程序中大量的出现。 
而通过代码审查，基准测试和激进的静态分析检查，围绕识别和修剪过多冗余代码的强大文化有助于确保适当地使用此类功能。

<!-- 
Interfaces were a challenge. 
-->
接口是一个挑战。

<!-- 
There are just some poorly designed, inefficient patterns in the .NET Framework.  `IEnumerator<T>` requires *two*
interface dispatches simply to extract the next item!  Compare that to C++ iterators which can compile down a pointer
increment plus dereference.  Many of these problems could be addressed simply with better library designs.  (Our final
design for enumeration didn't even invole interfaces at all.) 
-->
.NET Framework中有一些设计不良，效率低下的模式。 
比如说`IEnumerator<T>`仅仅只是为了获取下一个项的操作，就需要*两个*接口的指派！ 
与其相比较的是，C++迭代器可以编译成只使用指针的递增外加一次解除引用的简单方式。 
对于许多类似的问题，都可以通过更好的库设计来解决（比如说，我们最终设计的枚举器甚至根本没有接口的介入）。

<!-- 
Plus invoking a C# interface is tricky.  Existing systems do not use pointer adjustment like
C++ does so usually an interface dispatch requires a table search.  First a level of indirection to get to the vtable,
then another level to find the interface table for the interface in question.  Some systems attempt to do callsite
caching for monomorphic invocations; that is, caching the latest invocation in the hope that the same object kind passes
through that callsite time and time again.  This requires mutable stubs, however, not to mention an [incredibly complex
system of thunks and whatnot](https://github.com/dotnet/coreclr/blob/master/src/vm/virtualcallstub.cpp).  In Midori, we
never ever ever violated [W^X](https://en.wikipedia.org/wiki/W%5EX); and we avoided mutable runtime data structures,
because they inhibit sharing, both in terms of working set, but also amortizing TLB and data cache pressure. 
-->
除此之外，调用C#接口是一件很棘手的事情。 
现有的系统不像C++那样使用指针调整，因此通常接口的指派需要进行表内搜索。 
首先为了获取vtable，需要一层间接的跳转；然后为了获取接口表，又是另一层间接的跳转。 
有些系统尝试对单态调用进行callsite缓存，也就是说，缓存最新的调用并期望相同的对象类再次进入该callsite。
这种方式需要可变的stub，更不用说一个[异常复杂的thunk和诸如此类的东西的系统](https://github.com/dotnet/coreclr/blob/master/src/vm/virtualcallstub.cpp)。 
在Midori，我们从未违反过[W^X](https://en.wikipedia.org/wiki/W%5EX)原则，并且由于其不利于共享也避免了可变的运行时数据结构，
使得减小了工作集的大小，同时分摊了TLB和数据缓存的压力。 

<!-- 
Our solution took advantage of the memory ordering model earlier.  We used so-called "fat" interface pointers.  A fat
interface pointer was two words: the first, a pointer to the object itself; the second, a pointer to the interface
vtable for that object.  This made conversion to interfaces slightly slower -- because the interface vtable lookup had
to happen -- but for cases where you are invoking it one or more times, it came out a wash or ahead.  Usually,
significantly.  Go does something like this, but it's slightly different for two reasons.  First, they generate the
interface tables on the fly, because interfaces are duck typed.  Second, fat interface pointers are subject to tearing
and hence can violate memory safety in Go, unlike Midori thanks to our strong concurrency model. 
-->
我们的解决方案是更早地利用了内存排序模型，使用了所谓的“胖”接口指针。 
胖接口指针由两个单字组成：第一个是指向对象本身的指针，第二个是指向该对象的接口的vtable的指针。 
因为必须在接口vtable中进行查找，所以这使得转换到接口的速度变得稍慢。
但对于你一次或几次调用的地方，缓存可以有效的解决，并且通常效果是相当显著的。
Go做了类似的事情，但由于以下两个原因而略有不同： 
首先，由于其接口都是duck typed，因此采用了动态生成接口表的方式；
其次，由于Go没有我们Midori强大的并发模型，胖接口指针会遭到破坏从而可能会违反内存安全性。

<!-- 
The finally challenge in this category was *generic virtual methods*, or GVMs.  To cut to the chase, we banned them.
Even if you NGen an image in .NET, all it takes is a call to the LINQ query `a.Where(...).Select(...)`, and you're
pulling in the JIT compiler.  Even in .NET Native, there is considerable runtime data structure creation, lazily, when
this happens.  In short, there is no known way to AOT compile GVMs in a way that is efficient at runtime.  So, we didn't
even bother offering them.  This was a slightly annoying limitation on the programming model but I'd have done it all
over again thanks to the efficiencies that it bought us.  It really is surprising how many GVMs are lurking in .NET. 
-->
虚拟调度的最终挑战是*泛型虚方法*，或者称之为GVM。 
我们的方法概括起来就是，禁止了他们的使用。 
即使在.NET中使用NGen生成映像文件，所有它需要仅仅是一次LINQ查询`a.Where(...).Select(...)`的函数调用，然后便进入到了JIT编译器。 
另外即使是对于.NET Native，当发生这种情况时，也会惰性地创建了相当多的运行时数据结构。 
简而言之，尚未有已知的方法使得AOT以一种在运行时中高效的方式编译GVM，
因此我们甚至都不提供GVM。 
这对编程模型来说是一个有点烦人的限制，但由于这样的做法确实我们带来了效率，所以我们还是选择这样做。 
另外，令人吃惊的是，在.NET中潜伏了大量的GVM。

<!-- 
## Statics 
-->
## 静态值

<!-- 
I was astonished the day I learned that 10% of our code size was spent on static initialization checks. 
-->
当我知道10%的代码体积用于存储静态初始化检查时，这让我感到十分惊讶。

<!-- 
Many people probably don't realize that the [CLI specification](
http://www.ecma-international.org/publications/standards/Ecma-335.htm) offers two static initialization modes.  There
is the default mode and `beforefieldinit`.  The default mode is the same as Java's.  And it's horrible. The static
initializer will be run just prior to accessing any static field on that type, any static method on that type, any
instance or virtual method on that type (if it's a value type), or any constructor on that type.  The "when" part
doesn't matter as much as what it takes to make this happen; *all* of those places now need to be guarded with explicit
lazy initialization checks in the resulting machine code! 
-->
许多人可能没有意识到[CLI规范](http://www.ecma-international.org/publications/standards/Ecma-335.htm)提供了两种静态初始化模式：
默认模式和`beforefieldinit`模式。 
默认模式与Java的做法相同，这听起来太可怕了。 
静态初始化程序将在访问该类型上的任何静态字段，该类型上的任何静态方法，
该类型上的任何实例或虚方法（如果它是值类型）或该类型上的任何构造函数的执行之前运行。 
“何时”部分与实现这一目标所需要的一样重要，因为现在*所有*这些地方都需要在生成的机器代码中进行显式的延迟初始化检查！

<!-- 
The `beforefieldinit` relaxation is weaker.  It guarantees the initializer will run sometime before actually accessing
a static field on that type.  This gives the compiler a lot of leeway in deciding on this placement.  Thankfully the
C# compiler will pick `beforefieldinit` automatically for you should you stick to using field initializers only.  Most
people don't realize the incredible cost of choosing instead to use a static constructor, however, especially for value
types where suddenly all method calls now incur initialization guards.  It's just the difference between: 
-->
`beforefieldinit`的松弛就显得约束比较弱，
它保证了初始化程序将在实际访问该类型的静态字段之前运行，这为编译器在决定何时初始时提供了很大的余地。 
值得庆幸的是，如果你只坚持使用字段初始化器，C#编译器会自动选择`beforefieldinit`模式。 
然而，大多数人并没有意识到选择使用静态构造函数的不可思议的成本，特别是对于所有方法调用现在都会进行初始化保护的值类型来说。 
因此，这是如下两者之间的区别：

    struct S {
        static int Field = 42;
    } 


<!-- 
and: 
-->

    struct S {
        static int Field;
        static S() {
            Field = 42;
        }
    } 


<!-- 
Now imagine the struct has a property: 
-->
现假设结构有如下的属性值：

<!-- 
    struct S {
        // As above...
        int InstanceField;
        public int Property { get { return InstanceField; } }
    } 
-->
    struct S {
        // 如上 ...
        int InstanceField;
        public int Property { get { return InstanceField; } }
    } 

<!-- 
Here's the machine code for `Property` if `S` has no static initializer, or uses `beforefieldinit` (automatically
injected by C# in the the field initializer example above): 
-->
如果`S`没有静态初始化程序或者使用了beforefieldinit`模式（在上面的字段初始化程序示例中由C#自动注入），那么这将是产生如下的`Property`的机器代码：

<!-- 
    ; The struct is one word; move its value into EAX, and return it:
    8B C2                mov         eax,edx
    C3                   ret 
-->
```
; 结构是一个单字；将其值移入EAX中，并返回：
8B C2                mov         eax,edx
C3                   ret 
```
<!-- 
And here's what happens if you add a class constructor: 
-->
如果添加类的构造函数，会发生以下情况：

<!-- 
    ; Big enough to get a frame:
    56                   push        rsi
    48 83 EC 20          sub         rsp,20h
    ; Load the field into ESI:
    8B F2                mov         esi,edx
    ; Load up the cctor's initialization state:
    48 8D 0D 02 D6 FF FF lea         rcx,[1560h]
    48 8B 09             mov         rcx,qword ptr [rcx]
    BA 03 00 00 00       mov         edx,3
    ; Invoke the conditional initialization helper:
    E8 DD E0 FF FF       call        2048
    ; Move the field from ESI into EAX, and return it:
    8B C6                mov         eax,esi
    48 83 C4 20          add         rsp,20h
    5E                   pop         rsi 
-->
```
; 大到足以装下整个frame：
56                   push        rsi
48 83 EC 20          sub         rsp,20h
; 将字段加载到ESI中：
8B F2                mov         esi,edx
; 加载cctor的初始化状态：
48 8D 0D 02 D6 FF FF lea         rcx,[1560h]
48 8B 09             mov         rcx,qword ptr [rcx]
BA 03 00 00 00       mov         edx,3
; 调用条件初始化函数：
E8 DD E0 FF FF       call        2048
; 将字段从ESI移动到EAX，而后返回：
8B C6                mov         eax,esi
48 83 C4 20          add         rsp,20h
5E                   pop         rsi 
```
<!-- 
On every property access! 
-->
对于每个属性的访问都是如此！

<!-- 
Of course, all static members still incur these checks, even if `beforefieldinit` is applied. 
-->
当然，即使应用了`beforefieldinit`，所有的静态成员仍会执行这些检查。

<!-- 
Although C++ doesn't suffer this same problem, it does have mind-bending [initialization ordering semantics](
http://en.cppreference.com/w/cpp/language/initialization).  And, like C# statics, C++11 introduced thread-safe
initialization, by way of the ["magic statics" feature](
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2660.htm). 
-->
尽管C++没有遇到同样的问题，但它确实存在令人费解的[初始化排序语义](http://en.cppreference.com/w/cpp/language/initialization)。 
并且就像C#的静态值一样，C++11通过[“魔法静态”功能](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2660.htm)
引入了线程安全的初始化方法。

<!-- 
We virtually eliminated this entire mess in Midori. 
-->
我们几乎消除了Midori在这个方面的整个混乱局面。

<!-- 
I mentioned offhandedly earlier that Midori had no mutable statics.  More accurately, we extended the notion of `const`
to cover any kind of object.  This meant that static values were evaluated at compile-time, written to the readonly
segment of the resulting binary image, and shared across all processes.  More importantly for code quality, all runtime
initialization checks were removed, and all static accesses simply replaced with a constant address. 
-->
之前已经提到过，Midori没有可变的静态值。 
更准确地说，我们扩展了`const`的概念以涵盖任何种类的对象， 
这意味着静态的值在编译时会进行求值，将结果写入生成的二进制映像文件的只读段，并在所有进程中共享。 
而对于代码质量更重要的是，所有运行时初始化检查都被移除，
并且所有的静态访问都被固定的地址所替换。

<!-- 
There were still mutable statics at the core of the system -- in the kernel, for example -- but these did not make their
way up into user code.  And because they were few and far between, we did not rely on the classical C#-style lazy
initialization checks for them.  They were manually initialized on system startup. 
-->
在系统的核心部分，比如内核中，仍然存在可变的静态值，但用户代码中却不存在可变静态值。 
因为这样值的数量很少，所以我们未使用它们的经典C#风格的延迟初始化检查，
而是在系统启动时手动进行初始化的。

<!-- 
As I said earlier, a 10% reduction in code size, and lots of speed improvements.  It's hard to know exactly how much
saved this was than a standard C# program because by the time we made the change, developers were well aware of the
problems and liberally applied our `[BeforeFieldInit]` attribute all over their types, to avoid some of the overheads.
So the 10% number is actually a lower bound on the savings we realized throughout this journey. 
-->
正如我之前所提到的，整个镜像代码体积减少了10%，并且速度提升了很多。 
不过很难确切知道这比标准C#程序节省了多少空间，
因为当我们着手进行更改时，开发者已经很清楚这些问题，
并且在他们的类型中自由地使用`[BeforeFieldInit]`属性以避免开销。 
因此，10%实际上是我们在整个过程中实现的代码体积节省的下限值。

<!-- 
## Async model 
-->
## Async模型

<!-- 
I already wrote a lot about [our async model](http://joeduffyblog.com/2015/11/19/asynchronous-everything/).  I won't
rehash all of that here.  I will reiterate one point: the compiler was key to making linked stacks work. 
-->
我已经写了很多关于[异步模型](/2018/11/25/midori/3-asynchronous-everything/)的内容，
这里我将不再赘述。 
不过我将重申一点：编译器是使链接运行栈运行的关键。

<!-- 
In a linked stacks model, the compiler needs to insert probes into the code that check for available stack space.  In
the event there isn't enough to perform some operation -- make a function call, dynamically allocate on the stack, etc.
-- the compiler needs to arrange for a new link to get appended, and to switch to it.  Mostly this amounts to some
range checking, a conditional call to a runtime function, and patching up `RSP`.  A probe looked something like: 
-->
在链接栈模型中，编译器需要将探针插入到检查可用栈空间的代码中。
如果出现没有足够空间来执行某些操作情况，比如进行函数调用或在栈上动态分配等，
编译器则需要安排新的链接附加到内存中，并切换到它的位置。
大多数情况下，这相当于一些范围检查，对运行时函数的条件调用以及对`RSP`的修补。 
探针看起来像是如下的代码：

<!-- 
    ; Check amount of stack space:
        lea     rax, [rsp-250h]
        cmp     rax, qword ptr gs:[0]
        ja      prolog
    ; If insufficient stack, link a new segment:
        mov     eax, 10029h
        call    ?g_LinkNewStackTrampoline
    prolog:
    ; The real code goes here... 
-->
```
; 检查栈空间使用量：
  lea     rax, [rsp-250h]
  cmp     rax, qword ptr gs:[0]
  ja      prolog
; 如果栈空间不足，则链接到新的段：
  mov     eax, 10029h
  call    ?g_LinkNewStackTrampoline
prolog:
; 真正的运行代码出现在这里 ...
```
<!-- 
Needless to say, you want to probe as little as possible, for two reasons.  First, they incur runtime expense.  Second,
they chew up code size.  There are a few techniques we used to eliminate probes. 
-->
不用说的是，出于以下两个原因，你希望尽可能少地进行探测操作：
首先，它们会产生运行时开销；其次，他们会增加代码体积。
因此，我们使用一些技术来消除探针。

<!-- 
The compiler of course knew how to compute stack usage of functions.  As a result, it could be smart about the amount of
memory to probe for.  We incorporated this knowledge into our global analyzer.  We could coalesce checks after doing
code motion and inlining.  We hoisted checks out of loops.  For the most part, we optimized for eliminating checks,
sometimes at the expense of using a little more stack. 
-->
编译器当然知道如何计算函数运行栈的使用量，
因此，对于探测的内存量实际上还可以更聪明一些。 
我们将这些知识融入我们的全局分析器中，
并可以在代码移动和内联后进行合并检查。 
我们将检查外提到循环之外，
并在大多数情况下，对消除检查进行了优化，
有时甚至以使用更多栈空间作为代价。

<!-- 
The most effective technique we used to eliminate probes was to run synchronous code on a classical stack, and to teach
our compiler to elide probes altogether for them.  This took advantage of our understanding of async in the type system.
Switching between the classical stack and back again again amounted to twiddling `RSP`: 
-->
为了消除探针，我们最有效的技术是在经典栈上运行同步代码，并指导我们的编译器完全省略掉探针，这利用了我们对类型系统中异步的理解。
而经典的运行栈之间的切换只需简单地更改`RSP`的值：

<!-- 
    ; Switch to the classical stack:
    move    rsp, qword ptr gs:[10h]
    sub     rsp, 20h

    ; Do some work (like interop w/ native C/C++ code)...

    ; Now switch back:
    lea     rsp, [rbp-50h] 
-->
```
; 切换到经典栈：
move    rsp, qword ptr gs:[10h]
sub     rsp, 20h

; 完成一些任务（比如和原生的C/C++进行互操作）...

; 再切换回来：
lea     rsp, [rbp-50h] 
```
<!-- 
I know Go abandoned linked stacks because of these switches.  At first they were pretty bad for us, however after about
a man year or two of effort, the switching time faded away into the sub-0.5% noise. 
-->
我知道由于这样的切换，Go放弃了链接栈。 
并且起初对于我们来说，这种方式也是非常糟糕，但经过大约一年或两年的努力，
切换时间逐渐降低到低于总开销的0.5%。

<!-- 
## Memory ordering model 
-->
## 内存顺序模型

<!-- 
Midori's stance on [safe concurrency](http://joeduffyblog.com/2015/11/03/a-tale-of-three-safeties/) had truly one
amazing benefit: you get a [sequentially consistent](https://en.wikipedia.org/wiki/Sequential_consistency) memory
ordering model *for free*.  You may wish to read that again.  Free! 
-->
Midori对[安全并发](/2018/10/24/midori/1-a-tale-of-three-safeties/)
的态度确实有一个惊人的好处：你可以*免费*获得[顺序一致](https://en.wikipedia.org/wiki/Sequential_consistency)的内存排序模型。
现在，你可能希望再次阅读那篇安全并发的文章，那么请随意吧！

<!-- 
Why is this so?  First, Midori's [process model](http://joeduffyblog.com/2015/11/19/asynchronous-everything/) ensured
single-threaded execution by default.  Second, any fine-grained parallelism inside of a process was governed by a finite
number of APIs, all of which were race-free.  The lack of races meant we could inject a fence at fork and join points,
selectively, without a developer needing to care or know. 
-->
为什么会是这样呢？
首先，Midori的[进程模型](/2018/11/25/midori/3-asynchronous-everything/)在默认情况下确保以单线程方式执行。 
其次，进程内部的任何细粒度并行都由有限数量的API控制，所有这些都是无竞争的。 
缺少竞争意味着我们可以有选择性地在fork和join位置注入屏障指令，而无需开发者注意或了解其细节。

<!-- 
Obviously this had incredible benefits to developer productivity.  The fact that Midori programmers never got bitten by
memory reordering problems was certainly one of my proudest outcomes of the project. 
-->
显然，这对开发者的工作效率有着不可思议的好处。 
因此，Midori程序员从未被内存重排序问题所困扰，而这也无疑是该项目最值得骄傲的结果之一。

<!-- 
But it also meant the compiler was free to make [more aggressive code motion optimizations](
https://www.cs.princeton.edu/courses/archive/fall10/cos597C/docs/memory-models.pdf), without any sacrifices to this
highly productive programming model.  In other words, we got the best of both worlds. 
-->
同时这也意味着编译器可以自由地进行[更激进的代码移动优化](https://www.cs.princeton.edu/courses/archive/fall10/cos597C/docs/memory-models.pdf)，
而不会牺牲编程模型的高效性。换句话说，我们实现了两全其美。

<!-- 
A select few kernel developers had to think about the memory ordering model of the underlying machine.  These were the
people implementing the async model itself.  For that, we eliminated C#'s notion of `volatile` -- which is [utterly
broken anyway](http://joeduffyblog.com/2010/12/04/sayonara-volatile/) -- in favor of something more like C++
[`atomic`s](http://en.cppreference.com/w/cpp/atomic).  That model is quite nice for two reasons.  First, what kind of
fence you need is explicit for every read and write, where it actually matters.  (ences affect the uses of a variable,
not its declaration.  Second, the explicit model tells the compiler more information about what optimizations can or
cannot take place, again at a specific uses, where it matters most. 
-->
不过少数的内核开发者还是不得不考虑底层机器的内存排序模型，因为他们是实现异步模型本身的那群人。 
为此，我们消除了C#中[无论如何都会被完全破坏](http://joeduffyblog.com/2010/12/04/sayonara-volatile/)的`volatile`概念，
并支持类似于C++中[`atomic`](http://en.cppreference.com/w/cpp/atomic)的机制。 
出于以下两个原因，该模型是非常不错的。 
首先，对于每一次读写操作，你需要什么样的内存屏障是明确的，实际上也是重要的（因为这会影响变量的使用，而不是它的声明）；
其次，显式模型告诉编译器关于哪些可以优化而哪些不行的更多信息，对于具有特定用途的场景中，则才是最重要的。

<!-- 
## Error model 
-->
## 错误模型

<!-- 
Our error model journey was a long one and will be the topic of a future post.  In a nutshell, however, we experimented
with two ends of the spectrum -- exceptions and return codes -- and lots of points in betweeen. 
-->
我们的错误模型之旅很长，并且将成为未来一篇文章的主题。 
然而，简而言之，我们试验了错误处理频谱的两个相对的端点——异常和返回错误代码，以及两者之间的许多折衷点。

<!-- 
Here is what we found from a code quality perspective. 
-->
以下是我们从代码质量角度的一些发现。

<!-- 
Return codes are nice because the type system tells you an error can happen.  A developer is thus forced to deal with
them (provided they don't ignore return values).  Return codes are also simple, and require far less "runtime magic"
than exceptions or related mechanisms like setjmp/longjmp.  So, lots to like here. 
-->
返回错误代码是不错的方式，因为从类型系统就能告知该处代码可能会发生错误，
所以开发者被迫对其进行处理（前提是他们不忽略返回值）。 
返回错误代码也是一种简单的方法，并且只需要比异常或相关机制（如setjmp/longjmp）少得多的“运行时戏法”。 
因此我很喜欢这样的特性。

<!-- 
From a code quality persective, however, return codes suck.  They force you to execute instructions in hot paths that
wouldn't have otherwise been executed, including when errors aren't even happening.  You need to return a value from
your function -- occupying register and/or stack space -- and callers need to perform branches to check the results.
Granted, we hope that these are predicted correctly, but the reality is, you're just doing more work. 
-->
然而，从代码质量的角度来看，返回代码是相当糟糕，
因为它们会强制在热点路径中执行本来不会执行的指令，包括甚至不会发生的错误。 
而且还需要从函数返回一个值，导致占用寄存器和/或栈空间，以及调用者需要执行分支来检查结果。 
当然，我们希望代码都能正确预测，但事实上后果是开发者需要做更多的工作。

<!-- 
Untyped exceptions suck when you're trying to build a reliable system.  Operating systems need to be reliable.  Not
knowing that there's a hidden control flow path when you're calling a function is, quite simply, unacceptable.  They
also require heavier weight runtime support to unwind stacks, search for handlers, and so on.  It's also a real pain in
the arse to model exceptional control flow in the compiler.  (If you don't believe me, just read through [this mail
exchange](http://lists.llvm.org/pipermail/llvm-dev/2015-May/085843.html)).  So, lots to hate here. 
-->
当尝试构建可靠的系统时，无类型异常是相当糟糕的，而操作系统需要的就是可靠。 
当你调用一个函数时，不知道会有隐藏的控制流路径是非常不可接受的。
除此之外，它们还需要更笨重的运行时的支持来展开堆栈，搜索处理程序等。 
在编译器中对异常控制流进行建模也是一件真正痛苦的事情（如果你不相信我，只需阅读[这篇邮件交流的内容](http://lists.llvm.org/pipermail/llvm-dev/2015-May/085843.html)）。
因此，我很讨厌这样的特性。

<!-- 
Typed exceptions -- I got used to not saying checked exceptions for fear of hitting Java nerves -- address some of these
shortcomings, but come with their own challenges.  Again, I'll save detailed analysis for my future post. 
-->
我习惯于不会因为害怕遇到Java神经而不检查的异常的有类型异常，解决了其中的一些缺点，但它也遇到了自身的挑战。 
我将在未来的文章中再次进行详细分析。

<!-- 
From a code quality perspective, exceptions can be nice.  First, you can organize code segments so that the "cold"
handlers aren't dirtying your ICACHE on successful pathways.  Second, you don't need to perform any extra work during
the normal calling convention.  There's no wrapping of values -- so no extra register or stack pressure -- and there's
no branching in callers.  There can be some downsides to exceptions, however.  In an untyped model, you must assume
every function can throw, which obviously inhibits your ability to move code around. 
-->
从代码质量的角度上来看，异常是不错的选择。 
首先，可以组织代码段以便“冷”的处理程序不会在成功路径上占用缓存。 
其次，在正常的调用约定期间，不需要执行任何额外的工作。 
没有返回值的封装，因此没有额外的寄存器或栈空间的压力，并且在调用者中无需有分支。
但是，异常也有一些缺点，比如在无类型模型中，必须假设每个函数都可以抛出异常，
这显然抑制代码移动的能力。

<!-- 
Our model ended up being a hybrid of two things: 
-->
我们的模型最终是两者的混合体：

<!-- 
* [Fail-fast](http://joeduffyblog.com/2014/10/13/if-youre-going-to-fail-do-it-fast/) for programming bugs.
* Typed exceptions for dynamically recoverable errors. 
-->
* 对编程bug采取了[快速失败（fail-fast）](http://joeduffyblog.com/2014/10/13/if-youre-going-to-fail-do-it-fast/)的策略；
* 对动态可恢复错误采用了有类型的异常。

<!-- 
I'd say the ratio of fail-fast to typed exceptions usage ended up being 10:1.  Exceptions were generally used for I/O
and things that dealt with user data, like the shell and parsers.  Contracts were the biggest source of fail-fast. 
-->
我可以说，快速失败与有类型异常之间使用的最终比率为10:1， 
异常通常用于I/O和处理用户数据，例如shell和解析器，而合约是快速失败方式最大的来源。

<!-- 
The result was the best possible configuration of the above code quality attributes: 
-->
最终得到的是上述代码质量属性的最佳可能配置：

<!-- 
* No calling convention impact.
* No peanut butter associated with wrapping return values and caller branching.
* All throwing functions were known in the type system, enabling more flexible code motion.
* All throwing functions were known in the type system, giving us novel EH optimizations, like turning try/finally
  blocks into straightline code when the try could not throw. 
-->
* 没有对调用约定带来影响；
* 封装的返回值和调用分支之间没有关联的胶水代码；
* 所有抛出异常的函数在类型系统中都是已知的，从而实现更灵活的代码移动；
* 所有抛出异常的函数在类型系统中都是已知的，为我们提供了精巧的EH优化，
  例如在try不会抛出异常时将try/finally块转换为直接代码。

<!-- 
A nice accident of our model was that we could have compiled it with either return codes or exceptions.  Thanks to this,
we actually did the experiment, to see what the impact was to our system's size and speed.  The exceptions-based system
ended up being roughly 7% smaller and 4% faster on some key benchmarks. 
-->
我们模型一个不错的意外收获是，编译时可以选择使用返回代码返回或异常机制。 
多亏了这一点，实际上我们做了实验来观察这对我们系统的体积和速度有什么影响， 
基于异常的系统最终在某些关键基准测试中体积缩小了约7%，速度提高了4%。

<!-- 
At the end, what we ended up with was the most robust error model I've ever used, and certainly the most performant one. 
-->
最后，我们最终得到的是我使用过的最强大的错误模型，当然也是性能最好的模型。

<!-- 
## Contracts 
-->
## 合约

<!-- 
As implied above, Midori's programming language had first class contracts: 
-->
如上所述，Midori的编程语言有作为一等公民的合约机制：

    void Push(T element)
        requires element != null
        ensures this.Count == old.Count + 1
    {
            ...
    } 


<!-- 
The model was simple: 
-->
其模型很简单：

<!-- 
* By default, all contracts are checked at runtime.
* The compiler was free to prove contracts false, and issue compile-time errors.
* The compiler was free to prove contracts true, and remove these runtime checks. 
-->
* 默认情况下，运行时检查所有合约；
* 编译器可以自由地证明合同是一定有误的，并发出编译时错误；
* 编译器可以自由地证明合约是一定无误的，并删除这些运行时的检查。

<!-- 
We had conditional compilation modes, however I will skip these for now.  Look for an upcoming post on our language. 
-->
我们有条件编译的模式，但是我现在要跳过它们，这部分内容请关注即将发布的关于我们语言的文章。

<!-- 
In the early days, we experimented with contract analyzers like MSR's [Clousot](
http://research.microsoft.com/pubs/138696/Main.pdf), to prove contracts.  For compile-time reasons, however, we had to
abandon this approach.  It turns out compilers are already very good at doing simple constraint solving and propagation.
So eventually we just modeled contracts as facts that the compiler knew about, and let it insert the checks wherever
necessary. 
-->
在早期，我们尝试使用像MSR的[Clousot](http://research.microsoft.com/pubs/138696/Main.pdf)这样的合约分析器来证明合约，
然而，出于编译时的原因，我们不得不放弃这种方法。
事实证明，编译器已经非常擅长于简单的约束求解和传播。 
因此，最终我们只是将合约建模为编译器已知的事实，并让它在必要的时候插入检查代码。

<!-- 
For example, the loop optimizer complete with range information above can already leverage checks like this: 
-->
例如，完成上述范围信息的循环优化器已经可以利用如下的检查：


    void M(int[] array, int index) {
        if (index >= 0 && index < array.Length) {
            int v = array[index];
            ...
        }
    } 


<!-- 
to eliminate the redundant bounds check inside the guarded if statement.  So why not also do the same thing here? 
-->
为了消除在防御性if语句中的冗余边界检查，为什么不在这里做同样的事情呢？


    void M(int[] array, int index)
            requires index >= 0 && index < array.Length {
        int v = array[index];
        ...
    } 

<!-- 
These facts were special, however, when it comes to separate compilation.  A contract is part of a method's signature,
and our system ensured proper [subtyping substitution](https://en.wikipedia.org/wiki/Liskov_substitution_principle),
letting the compiler do more aggressive optimizations at separately compiled boundaries.  And it could do these
optimizations faster because they didn't depend on global analysis. 
-->
然而，当涉及独立的编译时，这些事实是特殊的。 
合约是方法签名的一部分，我们的系统确保了合适的
[子类型替换](https://en.wikipedia.org/wiki/Liskov_substitution_principle)，让编译器在独立编译的边界上进行更激进的优化。 
它可以更快地完成这些优化，因为它们不依赖于全局分析。

<!-- 
## Objects and allocation 
-->
## 对象和分配

<!-- 
In a future post, I'll describe in great detail our war with the garbage collector.  One technique that helped us win,
however, was to aggressively reduce the size and quantity of objects a well-behaving program allocated on the heap.
This helped with overall working set and hence made programs smaller and faster. 
-->
在以后的文章中，我将详细描述我们与垃圾收集器之间的博弈， 
然而，帮助我们取胜的技术是积极地减少行为良好的程序在堆上分配对象的大小和数量。
这有助于优化整体工作集，从而使程序更小更快。

<!-- 
The first technique here was to shrink object sizes. 
-->
第一种技术是减少对象大小。

<!-- 
In C# and most Java VMs, objects have headers.  A standard size is a single word, that is, 4 bytes on 32-bit
architectures and 8 bytes on 64-bit.  This is in addition to the vtable pointer.  It's typically used by the GC to mark
objects and, in .NET, is used for random stuff, like COM interop, locking, memozation of hash codes, and more.  (Even
[the source code calls it the "kitchen sink"](https://github.com/dotnet/coreclr/blob/master/src/vm/syncblk.h#L29).) 
-->
在C#和大多数Java VM中说，所有的对象都有对象头。 
对象头的标准大小是单字（word），即32位体系结构上的4字节和64位上的8字节。
作为vtable指针的扩充，它通常用于标记对象，在.NET中用于随机内容，如COM互操作，加锁，哈希值的记忆化等等
（[甚至在源代码它被为“厨房的水槽”](https://github.com/dotnet/coreclr/blob/master/src/vm/syncblk.h#L27)）。

<!-- 
Well, we ditched both. 
-->
好吧，对于两者，我们最终都抛弃了。

<!-- 
We didn't have COM interop.  There was no unsafe free-threading so there was no locking (and [locking on random objects
is a bad idea anyway](http://joeduffyblog.com/2006/10/26/concurrency-and-the-impact-on-reusable-libraries/)).  Our
`Object` didn't define a `GetHashCode`.  Etc.  This saved a word per object with no discernable loss in the programming
model (actually, to the contrary, it was improved), which is nothing to shake a stick at. 
-->
我们没有COM互操作，没有不安全的自由线程，因此没有加锁操作（任何情况下，[给任意对象上锁](http://joeduffyblog.com/2006/10/26/concurrency-and-the-impact-on-reusable-libraries/)都是一个坏主意）。 
我们的`Object`也没有定义`GetHashCode`等等。
这为每个对象节省了一个单字的空间大小，
并且在编程模型中没有明显的开销（实际上恰恰相反的是，还得到了某些改进），因此也就没什么可说的了。

<!-- 
At that point, the only overhead per object was the vtable pointer.  For structs, of course there wasn't one (unless
they were boxed).  And we did our best to eliminate all of them.  Sadly, due to RTTI, it was difficult to be aggressive.
I think this is another area where I'd go back and entirely upend the C# type system, to follow a more C, C++, or even
maybe Go-like, model.  In the end, however, I think we did get to be fairly competitive with your average C++ program. 
-->
此时，每个对象的唯一开销就是vtable指针。 
对于结构来说，当然是没有vtable指针的（除非它们采用了box的方式）。
我们尽力消除所有这些开销，但可悲的是，由于RTTI方式的使用，很难将优化变得激进。
我认为这是另一个我想回去重新解决，并完全颠覆C#类型系统的领域，使得其能效仿更多的C，C++甚至是类似Go的模型。 
不过最后，我认为我们确实取得了与普通C++程序比拟的竞争力。

<!-- 
There were padding challenges.  Switching the `struct` layout from C#'s current default of `sequential`, to our
preferred default of `auto`, certainly helped.  As did optimizations like the well-known C++ [empty base optimization](
http://en.cppreference.com/w/cpp/language/ebo). 
-->
这里充满了挑战。正如著名的C++[空基优化](http://en.cppreference.com/w/cpp/language/ebo)一样，
将`struct`布局从C#的当前默认的`sequential`切换到我们首选的默认`auto`方式，肯定会有所帮助。

<!-- 
We also did aggressive escape analysis in order to more efficiently allocate objects.  If an object was found to be
stack-confined, it was allocated on the stack instead of the heap.  Our initial implementation of this moved somewhere
in the neighborhood of 10% static allocations from the heap to the stack, and let us be far more aggressive about
pruning back the size of objects, eliminating vtable pointers and entire unused fields.  Given how conservative this
analysis had to be, I was pretty happy with these results. 
-->
我们还进行了激进的逃逸分析，以便更有效地分配对象，如果发现一个对象是能够在栈上分配，则它将被分配在栈而不是堆上。 
我们的初始实现能够将10%左右的静态分配从堆移动到到栈上，
这激励我们更加激进地减少对象的大小，消除vtable指针和整个未使用的字段。 
考虑到这种分析是如此的保守，我对这样的结果非常满意。

<!-- 
We offered a hybrid between C++ references and Rust borrowing if developers wanted to give the compiler a hint while at
the same time semantically enforcing some level of containment.  For example, say I wanted to allocate a little array to
share with a callee, but know for sure the callee does not remember a reference to it.  This was as simple as saying: 
-->
如果开发者想要给编译器提供提示信息，同时在语义上强制执行某种程度的包含，我们也提供了C++引用和Rust借用之间的混合。 
例如，假设我想分配一个小型数组与被调用者进行共享，但确定被调用者不存在对它的引用，其做法也就像下面这样简单：

<!-- 
    void Caller() {
        Callee(new[] { 0, 1, ..., 9 });
    }

    void Callee(int[]& a) {
        ... guaranteed that `a` does not escape ...
    } 
-->
    void Caller() {
        Callee(new[] { 0, 1, ..., 9 });
    }

    void Callee(int[]& a) {
        ... 保证a不会逃逸 ...
    } 

<!-- 
The compiler used the `int[]&` information to stack allocate the array and, often, eliminate the vtable for it
entirely.  Coupled with the sophisticated elimination of bounds checking, this gave us something far closer to C
performance. 
-->
编译器使用`int[]&`信息在栈上分配数组，并且通常完全消除了vtable的生成， 
再加上复杂的边界检查消除算法，这一切给了我们更接近C性能的程序。

<!-- 
Lambdas/delegates in our system were also structs, so did not require heap allocation.  The captured display frame was
subject to all of the above, so frequently we could stack allocate them.  As a result, the following code was heap
allocation-free; in fact, thanks to some early optimizations, if the callee was inlined, it ran as though the actual
lambda body was merely expanded as a sequence of instructions, with no call over head either! 
-->
系统中的Lambda/delegate也是结构，因此无需堆的分配。 
捕获的显示框架受上述所有问题的影响，因此我们经常可以在栈上分配它们，造成的结果是下面的代码是完全无需堆分配的。 
事实上，由于一些早期的优化，如果被调用者被内联优化，
那么它就像通过实际的lambda函数体被扩展为一系列指令一样，而没有产生任何实际的函数调用！

<!-- 
    void Caller() {
        Callee(() => ... do something ... );
    }

    void Callee(Action& callback) {
        callback();
    } 
-->
    void Caller() {
        Callee(() => ... 完成某些任务... );
    }

    void Callee(Action& callback) {
        callback();
    } 

<!-- 
In my opinion, this really was the killer use case for the borrowing system.  Developers avoided lambda-based APIs in
the early days before we had this feature for fear of allocations and inefficiency.  After doing this feature, on the
other hand, a vibrant ecosystem of expressive lambda-based APIs flourished. 
-->
在我看来，这确实是借用系统的杀手级用例。 
由于担心分配和效率低下的问题，开发者在我们拥有此功能之前的早期就避免使用基于lambda的API。 
而另一方面，在完成此功能之后，充满活力的基于lambda的API生态系统又重新蓬勃发展。

<!-- 
# Throughput 
-->
# 吞吐量

<!-- 
All of the above have to do with code quality; that is, the size and speed of the resulting code.  Another important
dimension of compiler performance, however, is *throughput*; that is, how quickly you can compile the code.  Here too
a language like C# comes with some of its own challenges. 
-->
以上所有的内容都与代码质量，也就是说，与生成代码的大小和执行速度有关。 
然而，编译器性能的另一个重要方面是吞吐量，也就是说代码编译的速度能有多快。 
对此，像C#这样的语言也存在一些自身的挑战。

<!-- 
The biggest challenge we encountered has less to do with the inherently safe nature of a language, and more to do with
one very powerful feature: parametric polymorphism.  Or, said less pretentiously, generics. 
-->
我们遇到的最大挑战与语言固有的安全性无关，
更多的是与一个非常强大的功能有关：参数多态，或者不那么自负的说法——泛型。

<!-- 
I already mentioned earlier that generics are just a convenient copy-and-paste mechanism.  And I mentioned some
challenges this poses for code size.  It also poses a problem for throughput, however.  If a `List<T>` instantiation
creates 28 types, each with its own handful of methods, that's just more code for the compiler to deal with.  Separate
compilation helps, however as also noted earlier, generics often flow across module boundaries.  As a result, there's
likely to be a non-trivial impact to compile time.  Indeed, there was. 
-->
我之前已经提到过，泛型只是一种方便的复制粘贴机制。 
并在本文中提到了它对代码大小带来的一些挑战，
然而，其也对编译吞吐量也带来了问题。 
如果`List<T>`的实例创建了28种类型，每种类型都有自己的少数几种方法，那编译器将需要处理更多代码。 
独立编译会有所帮助，但是如前所述，泛型通常是跨越模块边界的方式使用。
因此，其可能会对编译时间产生的影响，而事实上也是如此。

<!-- 
In fact, this is not very different from where most C++ compilers spend the bulk of their time.  In C++, it's templates.
More modern C++ code-bases have similar problems, due to heavy use of templated abstractions, like STL, smart pointers,
and the like.  Many C++ code-bases are still just "C with classes" and suffer this problem less. 
-->
实际上，这与大多数C++编译器花费大量时间的地方没有太大差别。 
在C++中，泛型叫做模板，由于大量使用模板化抽象（如STL，智能指针等），
更现代的C++代码库具有与上述相类似的问题。 
为了避免这个问题带来的影响，许多C++代码库仍然只是“带有类的C”的形式。

<!-- 
As I mentioned earlier, I wish we had banished RTTI.  That would have lessened the generics problem.  But I would guess
generics still would have remained our biggest throughput challenge at the end of the day. 
-->
正如我之前提到的，我希望我们放弃RTTI，这么做会减少泛型的问题。
但我认为即便如此，泛型仍然是我们在编译吞吐量上最大的挑战。

<!-- 
The funny thing -- in a not-so-funny kind of way -- is that you can try to do analysis to prune the set of generics and,
though it is effective, this analysis takes time.  The very thing you're trying to save. 
-->
有一种看起来不是，但实际确实有趣的方式是，你可以尝试进行分析以裁剪一些泛型。
虽然这种方式是有效的，但分析也需要时间，而这就是需要节省的地方。

<!-- 
A metric we got in the habit of tracking was how much slower AOT compiling a program was than simply C# compiling it.
This was a totally unfair comparison, because the C# compiler just needs to lower to MSIL whereas an AOT compler needs
to produce machine code.  It'd have been fairer to compare AOT compiling to JIT compiling.  But no matter, doing a great
job on throughput is especially important for a C# audience.  The expectation of productivity was quite high.  This was
therefore the key metric we felt customers would judge us on, and so we laser-focused on it. 
-->
我们习惯跟踪的一个指标是AOT编译程序的速度比仅仅进行C#编译要慢多少。 
而这是一个完全不公平的比较方式，因为C#编译器只需要生成MSIL，而AOT编译器则需要生成机器代码。
而将AOT编译与JIT编译进行比较是一个比较公平的方式，
但无论如何，在吞吐量方面做得好对于C#的使用者来讲尤其重要，因为对开发生产效率的期望至始至终是相当高的， 
因此，这是我们认为客户对我们评价的关键指标，所以我们也专注于此。

<!-- 
In the early days, the number was ridiculously bad.  I remember it being 40x slower.  After about a year and half with
intense focus we got it down to *3x for debug builds* and *5x for optimized builds*.  I was very happy with this! 
-->
在早期，这个指标是非常糟糕的，我记得是它慢了40倍。 
经过大约一年半的重点优化，我们将*debug版本降低到3倍左右*，而*optimized版本降低到5倍左右*。 
对于这样的成绩我很高兴！

<!-- 
There was no one secret to achieving this.  Mostly it had to do with just making the compiler faster like you would any
program.  Since we built the compiler using Midori's toolchain, however -- and compiled it using itself -- often this
was done by first making Midori better, which then made the the compiler faster.  It was a nice virtuous loop.  We had
real problems with string allocations which informed what to do with strings in our programming model.  We found crazy
generics instantiation closures which forced us to eliminate them and build tools to help find them proactively.  Etc. 
-->
实现这一点没有任何秘密可言。 
大多数情况下，它只需要像任何其他程序一样优化使得将编译器更快。 
因为我们使用Midori的工具链构建了编译器，再使用它编译Midori自身，
所以通常这是通过首先使Midori变得更好，然后再使编译器更快来达到目的的，而这是一个很不错的良性循环。 
我们遇到了字符串分配的实际问题，这些问题告诉我们在编程模型中如何处理字符串，
我们也发现了，疯狂的泛型实例化闭包迫使我们消除并构建工具来主动找到它们等等问题。

<!-- 
# Culture 
-->
# 文化

<!-- 
A final word before wrapping up.  Culture was the most important aspect of what we did.  Without the culture, such an
amazing team wouldn't have self-selected, and wouldn't have relentlessly pursued all of the above achievements.  I'll
devote an entire post to this.  However, in the context of compilers, two things helped: 
-->
结束前的最后一节我想说的是，文化是我们所做的最重要一方面的工作。 
如果没有文化，这样一支出色的团队也就不会进行自我选择，也不会狠狠地追求上述所有这些成绩，
我会花一整篇文章谈论这方面的内容。
但是，回到关于编译器的内容，有两件事是有所帮助的：

<!-- 
1. We measured everything in the lab.  "If it's not in the lab, it's dead to me."
2. We reviewed progress early and often.  Even in areas where no progress was made.  We were habitually self-critical. 
-->
1. 我们在试验环境中测量所有指标。 “如果它不在实验室里，那对我来说它已经死了”。
2. 我们提前和经常性审查进展情况，即使在没有取得进展的地方也是如此，而且我们也习惯性地进行自我批判性思考。

<!-- 
Every sprint, we had a so-called "CQ Review" (where CQ stands for "code quality").  The compiler team prepared for a few
days, by reviewing every benchmark -- ranging from the lowest of microbenchmarks to compiling and booting all of Windows
-- and investigating any changes.  All expected wins were confirmed (we called this "confirming your kill"), any
unexpected regressions were root cause analyzed (and bugs filed), and any wins that didn't materialize were also
analyzed and reported on so that we could learn from it.  We even stared at numbers that didn't change, and asked
ourselves, why didn't they change.  Was it expected?  Do we feel bad about it and, if so, how will we change next
sprint?  We reviewed our competitors' latest compiler drops and monitored their rate of change.  And so on. 
-->
对于每个开发周期（sprint），我们都有一个所谓的“CQ Review”（其中CQ代表“代码质量”）过程。 
通过审查每个基准测试，编译团队准备了好几天的时间，从最低层次的微基准测试到编译和启动所有Windows，并在此过程中分析任何变化。 
所有预期的胜利都得到了确认（我们称之为“确认你的毙敌”），
任何意料之外的退化都进过根本原因的分析（并提交bug），
任何未实现的进展也会被分析和形成报告，以便我们可以从中学习。 
我们甚至盯着那些没有改变的数字，并问自己，为什么他们没有改变？这是预期的吗？ 
我们是否对其感觉不好，如果是这样，我们将如何在下个开发周期中进行改变？
我们审查了竞争对手的最新编译器的进展并监控其变化率等等。

<!-- 
This process was enormously healthy.  Everyone was encouraged to be self-critical.  This was not a "witch hunt"; it was
an opportunity to learn as a team how to do better at achieving our goals. 
-->
这个过程非常的健康，每个人都被鼓励进行自我批判性思考。
这不是所谓的“猎巫”运动，而是一个作为团队学习如何更好地实现目标的机会。

<!-- 
Post-Midori, I have kept this process.  I've been surprised at how contentious this can be with some folks.  They get
threatened and worry their lack of progress makes them look bad.  They use "the numbers aren't changing because that's
not our focus right now" as justification for getting out of the rhythm.  In my experience, so long as the code is
changing, the numbers are changing.  It's best to keep your eye on them lest you get caught with your pants around your
ankles many months later when it suddenly matters most.  The discipline and constant drumbeat are the most important
parts of these reviews, so skipping even just one can be detrimental, and hence was verboten. 
-->
在经历了Midori之后的，我保留了这个过程。 
我对一些成员对此的争议感到惊讶，他们受到威胁并担心缺乏进展会使他们看起来很糟糕。 
他们使用“数字没有变是因为它们现在不是我们关注的焦点”作为摆脱节奏的理由。 
根据我的经验，只要代码发生变化，数字就会发生变化，因此最好密切关注它们，
以免在几个月后当它突然成为最重要的任务时，陷入尴尬的境地。 
处罚和持续的鼓声是这些评论中最重要的部分，因此即使只忽略掉其中一个也可能是有害的，所以也是不被允许的。

<!-- 
This process was as much our secret sauce as anything else was. 
-->
该过程和其他所有东西一样都是我们的秘密武器。

<!-- 
# Wrapping Up 
-->
# 总结

<!-- 
Whew, that was a lot of ground to cover.  I hope at the very least it was interesting, and I hope for the incredible
team who built all of this that I did it at least a fraction of justice.  (I know I didn't.) 
-->
哇噢，有如此多需要被提到的内容。我希望这些内容至少很有意思，
并希望能够建立所有这一切令人难以置信内容的团队，
至少做到一小部分的正义（尽管我知道我没有做到）。

<!-- 
This journey took us over a decade, particularly if you account for the fact that both Bartok and Phoenix had existed
for many years even before Midori formed.  Merely AOT compiling C#, and doing it well, would have netted us many of the
benefits above.  But to truly achieve the magical native-like performance, and indeed even exceed it in certain areas,
required some key "whole system" architectural bets.  I hope that some day we can deliver safety into the world at this
level of performance.  Given the state of security all-up in the industry, mankind seriously needs it. 
-->
这段旅程花了我们十多年的时间，特别是如果当你考虑到Bartok和Phoenix在Midori成立之前已存在多年的事实。 
仅仅通过AOT的方式编译C#并且做得很好，会让我们收获上述许多的好处。 
但要真正获得神奇的原生性能，甚至在某些领域超过它，需要一些关键的“整个系统”结构上的赌注。
我希望有一天我们能够在这个性能水平上为世界提供安全性的保障。 
鉴于业界的安全状况，全世界真的都需要它。

<!-- 
I've now touched on our programming language enough that I need to go deep on it.  Tune in next time! 
-->
到目前为止，我已经简要提及到我们的编程语言，使得我需要进一步对其深入下去。那么，下次再见！