---
title: Midori博客系列翻译（4）——安全的原生代码
date: 2018-12-17 14:46:00
tags: [操作系统, Midori, 翻译, 安全]
categories: 中文
---

<!-- 
In my [first Midori post](http://joeduffyblog.com/2015/11/03/a-tale-of-three-safeties/), I described how safety was the
foundation of everything we did.  I mentioned that we built an operating system out of safe code, and yet stayed
competitive with operating systems like Windows and Linux written in C and C++.  In many ways, system architecture
played a key role, and I will continue discussing how in future posts.  But, at the foundation, an optimizing compiler
that often eeked out native code performance from otherwise "managed", type- and memory-safe code, was one of our most
important weapons.  In this post, I'll describe some key insights and techniques that were essential to our success. -->

在我的[第一篇Midori文章](/2018/10/24/midori/1-a-tale-of-three-safeties/)中，我描述了安全是我们所做的一切的基础。 
我提到我们使用安全代码构建了一个操作系统，
但仍然保持与使用C和C++编写的Windows和Linux等操作系统相比而言的竞争力。 
系统架构在许多方面发挥了关键作用，关于这点我将在未来的帖子中继续讨论。 
但是，在此基础上，一个通常从其他“托管”的类型和内存安全榨取原生代码性能的优化编译器，
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

与C和C++相比，AOT编译托管的[垃圾回收代码](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))提出了一些独特的挑战。 
因此，许多关于AOT的努力与其对应的原生代码相比并没有获得大致的性能。
.NET的NGEN技术就是一个很好的例子，
实际上，.NET中的大部分工作都专门针对于启动时间。这显然是一个关键指标，
但是当你构建一个操作系统及其之上的所有模块时，启动时间几乎只是刚刚开始。

<!-- 
Over the course of 8 years, we were able to significantly narrow the gap between our version of C# and classical C/C++
systems, to the point where basic code quality, in both size of speed dimensions, was seldom the deciding factor when
comparing Midori's performance to existing workloads.  In fact, something counter-intuitive happened.  The ability to
co-design the language, runtime, frameworks, operating system, and the compiler -- making tradeoffs in one area to gain
advantages in other areas -- gave the compiler far more symbolic information than it ever had before about the program's
semantics and, so, I dare say, was able to exceed C and C++ performance in a non-trivial number of situations. 
-->

在8年的时间里，我们显著地缩小我们的C#与经典C/C++系统之间的差距，在速度维度的大小上，基本代码质量在比较Midori的时候很少是决定因素。 对现有工作负载的性能。 
事实上，反直觉的事情发生了，协同设计语言、运行时、框架、操作系统和编译器的能力——使得在一个方面上的折衷从而在其他方面获得优势——为编译器提供了比以往更多的关于程序语义的符号信息。
因此，我敢说，我们的语言能够在非常微不足道的情况下超过C和C++的性能。

<!-- 
Before diving deep, I have to put in a reminder.  The architectural decisions -- like [Async Everywhere](
http://joeduffyblog.com/2015/11/19/asynchronous-everything/) and Zero-Copy IO (coming soon) -- had more to do with us
narrowing the gap at a "whole system" level.  Especially the less GC-hungry way we wrote systems code.  But the
foundation of a highly optimizing compiler, that knew about and took advantage of safety, was essential to our results. 
-->

在深入探究之前，我必须提醒一下。
架构上的决策——如[无处不在的Async](http://joeduffyblog.com/2015/11/19/asynchronous-everything/)和
（即将推出的）Zero-Copy IO——在“整个系统”级别更缩小差距方面与我们更加相关， 
特别是在我们编写缺乏GC的系统代码时。
但是，高度优化的编译器所打下的基础，了解并利用安全性，对我们的结果至关重要。

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
如果我未能指出这个领域中其他与我们一起取得了相当大的进展的工作，这将会是我的失职。
这里的进展包括，[Go语言](https://golang.org/)跨越了系统性能和安全性之间的优雅界限；
另外，[Rust](http://rust-lang.org/)在这方面做的很棒。 
[.NET Native](https://msdn.microsoft.com/en-us/vstudio/dotnetnative.aspx)和相关的[Android Runtime](https://en.wikipedia.org/wiki/Android_Runtime)
项目以更有限的方式为C#和Java带来了AOT编译，并将其作为一种“静默”优化技术，
以避免由JIT引起的移动应用程序延迟。 
最近，我们一直致力于通过[CoreRT项目](https://github.com/dotnet/corert)更广泛地将AOT带入.NET环境中，
通过这项努力，我希望我们能够将下面的一些经验教训带到现实世界环境中。 
由于突破性变化之间的微妙平衡，我们还能走多远还有待观察。
然而，我们花了几年时间，数十人年的工作量，
才能使所有事情和谐地工作，所以这种知识的转移需要时间。


<!-- 
First thing's first.  Let's quickly recap: What's the difference between native and managed code, anyway? 
-->
首先，让我们简要重述一下：本机代码和托管代码之间到底有什么区别？

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
我鄙视错误的二分法“本土和管理”，所以我必须为使用它而道歉。 阅读完这篇文章后，我希望能说服你这是一个连续统一体。 C ++现在比以往任何时候都更安全，同样，C＃表现更好。 有趣的是，这些课程中有多少直接适用于我的团队最近在Safe C ++上所做的工作。

<!-- 
So let's begin by considering what's the same. 
-->
所以让我们先考虑一下是什么。

<!-- 
All the basic [dragon book](https://en.wikipedia.org/wiki/Principles_of_Compiler_Design) topics apply to managed as
much as they do native code. 
-->
所有基本的龙书主题都适用于管理，就像他们使用本机代码一样。

<!-- 
In general, compiling code is a balancing act between, on one hand emitting the most efficient instruction sequences
for the target architecture, to execute the program quickly; and on the other hand emitting the smallest encoding of
instructions for the target architecture, to store the program compactly and effectively use the memory system on the
target device.  Countless knobs exist on your favorite compiler to dial between the two based on your scenario.  On
mobile, you probably want smaller code, whereas on a multimedia workstation, you probably want the fastest. 
-->
通常，编译代码是一方面为目标体系结构发出最有效的指令序列，以快速执行程序之间的平衡行为; 另一方面，为目标体系结构发出最小的指令编码，以便在目标设备上紧凑且有效地使用存储系统来存储程序。 您最喜欢的编译器上存在无数个旋钮，可根据您的场景在两者之间进行拨号。 在移动设备上，您可能需要更小的代码，而在多媒体工作站上，您可能想要最快的代码。

<!-- 
The choice of managed code doesn't change any of this.  You still want the same flexibility.  And the techniques you'd
use to achieve this in a C or C++ compiler are by and large the same as what you use for a safe language. 
-->
托管代码的选择不会改变任何这一点。 你仍然需要相同的灵活性。 在C或C ++编译器中用于实现此目的的技术基本上与用于安全语言的技术相同。
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
你需要一个伟大的内联。 您需要常见的子表达式消除（CSE），常量传播和折叠，强度降低以及出色的循环优化器。 目前，您可能希望使用静态单一赋值形式（SSA）和一些独特的SSA优化（如全局值编号）（尽管在任何地方使用SSA时都需要注意工作集和编译器吞吐量）。 对于对您来说很重要的目标体系结构，您需要专门的机器相关优化器，包括寄存器分配器。 您最终需要一个进行过程间优化的全局分析器，链接时代码生成以跨越传递来扩展那些过程间优化，现代处理器的矢量化器（SSE，NEON，AVX等），以及绝对的配置文件引导优化（ PGO）根据实际情况通知以上所有内容。
<!-- 
Although having a safe language can throw some interesting curveballs your way that are unique and interesting -- which
I'll cover below -- you'll need all of the standard optimizing compiler things. 
-->
虽然拥有一种安全的语言可以以一种独特而有趣的方式抛出一些有趣的曲线球 - 我将在下面介绍 - 你需要所有标准的优化编译器。

<!-- 
I hate to say it, but doing great at all of these things is "table stakes."  Back in the mid-2000s, we had to write
everything by hand.  Thankfully, these days you can get an awesome off-the-shelf optimizing compiler like [LLVM](
http://llvm.org) that has most of these things already battle tested, ready to go, and ready for you to help improve. 
-->
我讨厌这样说，但在所有这些事情上做得很好就是“桌上赌注。”早在2000年代中期，我们不得不手工编写所有内容。 值得庆幸的是，现在您可以获得一个非常棒的现成优化编译器，如LLVM，其中大部分已经经过测试，准备就绪，并随时准备帮助您改进。
<!-- 
## What's different 
-->

<!-- 
But, of course, there are differences.  Many.  This article wouldn't be very interesting otherwise. 
-->
但是，当然，存在差异。 许多。 否则，这篇文章不会很有趣。
<!-- 
The differences are more about what "shapes" you can expect to be different in the code and data structures thrown at
the optimizer.  These shapes come in the form of different instruction sequences, logical operations in the code that
wouldn't exist in the C++ equivalent (like more bounds checking), data structure layout differences (like extra object
headers or interface tables), and, in most cases, a larger quantity of supporting runtime data structures.
-->
差异更多的是关于优化器抛出的代码和数据结构中您可能期望的“形状”。 这些形状以不同的指令序列形式出现，代码中的逻辑操作不存在于C ++等价物中（如更多边界检查），数据结构布局差异（如额外的对象头或接口表），并且在大多数情况下 例如，支持运行时数据结构的数量更多。

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
与C语言中的节俭数据类型相比，对象在大多数托管语言中都有“更多”。（请注意，C ++数据结构并不像您想象的那样节俭，并且可能比您的直觉告诉您更接近C＃ 。）在Java中，每个对象的标题中都有一个vtable指针。 在C＃中，大多数都是这样，尽管结构没有。 GC可以施加额外的布局限制，例如填充和几个单词来进行簿记。 请注意，这些都不是特定于托管语言--C和C ++分配器也可以注入自己的单词，当然，许多C ++对象也带有vtable  - 但是可以公平地说，大多数C和C ++实现往往更经济 在这些方面。 在大多数情况下，出于文化原因而不是硬技术原因。 在堆中添加几千个对象，尤其是当您的系统由许多具有隔离堆的小进程（如Midori）构建时，它会快速累加。

<!-- 
In Java, you've got a lot more virtual dispatch, because methods are virtual by default.  In C#, thankfully, methods
are non-virtual by default.  (We even made classes sealed by default.)  Too much virtual dispatch can totally screw
inlining which is a critical optimization to have for small functions.  In managed languages you tend to have more
small functions for two reasons: 1) properties, and 2) higher level programmers tend to over-use abstraction. 
-->
在Java中，您有更多的虚拟调度，因为默认情况下方法是虚拟的。 在C＃中，幸运的是，默认情况下，方法是非虚拟的。 （我们甚至默认密封了类。）太多的虚拟调度可以完全拧入内联，这是小功能的关键优化。 在托管语言中，您倾向于拥有更多小函数，原因有两个：1）属性，以及2）更高级别的程序员倾向于过度使用抽象。

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
虽然它很少形式地描述，但是有一个“ABI”（应用程序二进制接口）来管理代码和运行时之间的交互。 ABI是橡胶与道路相遇的地方。 它就像调用约定，异常处理，以及最显着的机器代码中的GC清单。 这不是托管代码所特有的！ C ++有一个“运行时”，也有一个ABI。 只是它主要由头，像分配器等库组成，它们更透明地链接到程序而不是传统的C＃和Java虚拟机，其中运行时是不可协商的（在JIT情况下，相当重-handed）。 以这种方式思考对我有帮助，因为C ++的同构突然变得明显。

<!-- 
The real biggie is array bounds checks.  A traditional approach is to check that the index is within the bounds of an
array before accessing it, either for loading or storing.  That's an extra field fetch, compare, and conditional
branch.  [Branch prediction](https://en.wikipedia.org/wiki/Branch_predictor) these days is quite good, however it's just
plain physics that if you do more work, you're going to pay for it.  Interestingly, the work we're doing with C++'s
`array_view<T>` incurs all these same costs. 
-->
真正的大问题是数组边界检查。 传统方法是在访问索引之前检查索引是否在数组的范围内，以便加载或存储。 这是一个额外的字段提取，比较和条件分支。 这些天的分支预测是相当不错的，但是如果你做更多的工作，你只需支付它就可以了。 有趣的是，我们使用C ++的array_view <T>所做的工作会产生所有这些相同的成本。

<!-- 
Related to this, there can be null checks where they didn't exist in C++.  If you perform a method dispatch on a null
object pointer in C++, for example, you end up running the function anyway.  If that function tries to access `this`,
it's bound to [AV](https://en.wikipedia.org/wiki/Segmentation_fault), but in Java and .NET, the compiler is required
(per specification) to explicitly check and throw an exception in these cases, before the call even occurs.  These
little branches can add up too.  We eradicated such checks in favor of C++ semantics in optimized builds. 
-->
与此相关，可以进行空检查，它们在C ++中不存在。 例如，如果在C ++中对空对象指针执行方法调度，则无论如何都会运行该函数。 如果该函数试图访问它，它将绑定到AV，但在Java和.NET中，编译器需要（根据规范）在这些情况下显式检查并抛出异常，甚至在调用发生之前。 这些小分支也可以加起来。 我们根据优化版本中的C ++语义消除了这种检查。

<!-- 
In Midori, we compiled with overflow checking on by default.  This is different from stock C#, where you must explicitly
pass the `/checked` flag for this behavior.  In our experience, the number of surprising overflows that were caught,
and unintended, was well worth the inconvenience and cost.  But it did mean that our compiler needed to get really good
at understanding how to eliminate unnecessary ones. 
-->
在Midori中，我们默认使用溢出检查进行编译。 这与库存C＃不同，您必须为此行为显式传递/ checked标志。 根据我们的经验，被捕获和意外的惊人溢出的数量非常值得给您带来不便和成本。 但这确实意味着我们的编译器需要非常善于理解如何消除不必要的编译器。

<!-- 
Static variables are very expensive in Java and .NET.  Way more than you'd expect.  They are mutable and so cannot be
stored in the readonly segment of an image where they are shared across processes.  And my goodness, the amount of
lazy-initialization checking that gets injected into the resulting source code is beyond belief.  Switching from
`preciseinit` to `beforefieldinit` semantics in .NET helps a little bit, since the checks needn't happen on every
access to a static member -- just accesses to the static variable in question -- but it's still disgusting compared to
a carefully crafted C program with a mixture of constant and intentional global initialization. 
-->
静态变量在Java和.NET中非常昂贵。 超出你期望的方式。 它们是可变的，因此不能存储在图像的只读段中，它们在进程间共享。 而且我的优点是，延迟初始化检查的数量被注入到生成的源代码中是不可信的。 在.NET中从exactinit切换到beforefieldinit语义有一点帮助，因为每次访问静态成员都不需要进行检查 - 只需访问有问题的静态变量 - 但与精心设计的C程序相比，它仍然令人作呕 恒定和有意的全局初始化的混合。

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
最后的主要区域特定于.NET：structs。 虽然结构有助于缓解GC压力，因此对大多数程序来说都是好事，但它们也带有一些微妙的问题。 例如，CLI指定了初始化时的令人惊讶的行为。 即，如果在构造期间发生异常，则struct slot必须保持零初始化。 结果是大多数编译器都会制作防御性副本。 另一个例子是，只要在readonly结构上调用函数，编译器就必须生成防御性副本。 将结构复制到整个地方是很常见的，当你计算周期时，会受到伤害，特别是因为它通常意味着在memcpy中花费的时间。 我们有很多技巧可以解决这个问题，而且很有趣，我很确定何时完成所有工作，我们的代码质量优于C ++，考虑到它的所有RAII，复制构造函数，析构函数等等，处罚。

<!-- 
# Compilation Architecture 
-->
# 编译架构

<!-- 
Our architecture involved three major components: 
-->
我们的架构涉及三个主要组件：

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
* [C＃编译器]（https://github.com/dotnet/roslyn）：执行lexing，解析和语义分析。最终，
   从C＃文本源代码转换为基于[CIL]（https://en.wikipedia.org/wiki/Common_Intermediate_Language）的
   [中间代表（IR）]（https://en.wikipedia.org/wiki/Intermediate_language）。
* [Bartok]（https://en.wikipedia.org/wiki/Bartok_(compiler））：接受所述IR，进行基于MSIL的高级分析，
   转换和优化，最后将此IR降低到更接近更具体的机器
   表示。 例如，当Bartok完成IR时，泛型已经消失。
* [凤凰]（https://en.wikipedia.org/wiki/Phoenix_（compiler_framework））：接受这个降低的IR，然后去镇上
   它。 这就是大多数“金属踏板”优化发生的地方。 输出是机器代码。

<!-- 
The similarities here with Swift's compiler design, particularly [SIL](
http://llvm.org/devmtg/2015-10/slides/GroffLattner-SILHighLevelIR.pdf), are evident.  The .NET Native project also
mirrors this architecture somewhat.  Frankly, most AOT compilers for high level languages do. 
-->
这与Swift的编译器设计（特别是SIL）的相似之处是显而易见的。 .NET Native项目也在某种程度上反映了这种架构。 坦率地说，大多数高级语言的AOT编译器都可以。

<!-- 
In most places, the compiler's internal representation leveraged [static single assignment form (SSA)](
https://en.wikipedia.org/wiki/Static_single_assignment_form).  SSA was preserved until very late in the compilation.
This facilitated and improved the use of many of the classical compiler optimizations mentioned earlier. 
-->
在大多数地方，编译器的内部表示利用了静态单一赋值形式（SSA）。 SSA一直保存到编译的最后阶段。 这促进并改进了前面提到的许多经典编译器优化的使用。

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
* 促进快速原型设计和实验。
* 生成与商业C / C ++编译器相同的高质量机器代码。
* 支持调试优化的机器代码以提高生产率。
* 基于采样和/或检测代码，促进配置文件引导的优化。
* 适合自主：
    - 生成的编译编译器足够快。
    - 它足够快，编译器开发人员喜欢使用它。
    - 当编译器误入歧途时，很容易调试问题。
  
<!-- 
Finally, a brief warning.  We tried lots of stuff.  I can't remember it all.  Both Bartok and Phoenix existed for years
before I even got involved in them.  Bartok was a hotbed of research on managed languages -- ranging from optimizations
to GC to software transactional memory -- and Phoenix was meant to replace the shipping Visual C++ compiler.  So,
anyway, there's no way I can tell the full story.  But I'll do my best. 
-->
最后，简要警告。 我们尝试了很多东西。 我记不起来了。 在我参与其中之前，巴托克和凤凰都存在多年。 Bartok是托管语言研究的温床 - 从优化到GC再到软件事务内存 - 而Phoenix本来是要取代出货的Visual C ++编译器。 所以，无论如何，我无法讲述完整的故事。 但我会尽我所能。

<!-- 
# Optimizations 
-->
# 优化

<!-- 
Let's go deep on a few specific areas of classical compiler optimizations, extended to cover safe code. 
-->
让我们深入研究经典编译器优化的一些特定领域，扩展到涵盖安全代码。

<!-- 
## Bounds check elimination 
-->
## 边界检查消除

<!-- 
C# arrays are bounds checked.  So were ours.  Although it is important to eliminate superfluous bounds checks in regular
C# code, it was even more so in our case, because even the lowest layers of the system used bounds checked arrays.  For
example, where in the bowels of the Windows or Linux kernel you'd see an `int*`, in Midori you'd see an `int[]`. 
-->
检查C#数组的边界。 我们也是。 虽然在常规C＃代码中消除多余的边界检查很重要，但在我们的情况下更是如此，因为即使是使用的系统的最低层也会检查已检查的数组。 例如，在Windows或Linux内核的内容中，您会看到一个`int*`，在Midori中您会看到一个int []。

<!-- 
To see what a bounds check looks like, consider a simple example:
-->
要查看边界检查的外观，请考虑一个简单的示例：

    var a = new int[100];
    for (int i = 0; i < 100; i++) {
        ... a[i] ...;
    }

<!-- 
Here's is an example of the resulting machine code for the inner loop array access, with a bounds check: 
-->
这是内部循环数组访问的结果机器代码的示例，带有边界检查：
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
    ; 首先，将数组长度放入EAX：
    3B15: 8B 41 08        mov         eax,dword ptr [rcx+8]
    ; 如果EDX> = EAX，则访问超出范围; 跳转到错误：
    3B18: 3B D0           cmp         edx,eax
    3B1A: 73 0C           jae         3B28
    ; 否则，访问正常; 计算元素的地址，并分配：
    3B1C: 48 63 C2        movsxd      rax,edx
    3B1F: 8B 44 81 10     mov         dword ptr [rcx+rax*4+10h],r8d
    ; ...
    ; 错误处理程序; 只需调用抛出的运行时助手：
    3B28: E8 03 E5 FF FF  call        2030 
<!-- 
If you're doing this bookkeeping on every loop iteration, you won't get very tight loop code.  And you're certianly not
going to have any hope of vectorizing it.  So, we spent a lot of time and energy trying to eliminate such checks. 
-->
如果你在每次循环迭代中都做这个簿记，那么你就不会得到非常紧密的循环代码。 而且你肯定不会有任何希望进行矢量化。 因此，我们花了很多时间和精力试图消除这种检查。

<!-- 
In the above example, it's obvious to a human that no bounds checking is necessary.  To a compiler, however, the
analysis isn't quite so simple.  It needs to prove all sorts of facts about ranges.  It also needs to know that `a`
isn't aliased and somehow modified during the loop body.  It's surprising how hard this problem quickly becomes. 
-->
在上面的例子中，对于人类来说显然不需要边界检查。 然而，对于编译器来说，分析并不那么简单。 它需要证明关于范围的各种事实。 它还需要知道a在循环体中没有别名并以某种方式进行了修改。 令人惊讶的是这个问题很快变得多么困难。

<!-- 
Our system had multiple layers of bounds check eliminations. 
-->
我们的系统有多层边界检查消除。

<!-- 
First it's important to note that CIL severely constraints an optimizer by being precise in certain areas.  For example,
accessing an array out of bounds throws an `IndexOutOfRangeException`, similar to Java's `ArrayOutOfBoundsException`.
And the CIL specifies that it shall do so at precisely the exception that threw it.  As we will see later on, our
error model was more relaxed.  It was based fail-fast and permitted code motion that led to inevitable failures
happening "sooner" than they would have otherwise.  Without this, our hands would have been tied for much of what I'm
about to discuss. 
-->
首先，重要的是要注意CIL通过在某些区域中精确来严格限制优化器。 
例如，访问数组超出范围会抛出`IndexOutOfRangeException`，类似于Java的ArrayOutOfBoundsException。 并且CIL指定它应该在完全抛出它的例外情况下这样做。 正如我们稍后将看到的，我们的错误模型更加轻松。 它基于失败快速和允许的代码运动，导致不可避免的失败“发生”比其他情况更快。 如果没有这一点，我们的双手将会与我即将讨论的大部分内容联系在一起。

<!-- 
At the highest level, in Bartok, the IR is still relatively close to the program input.  So, some simple patterns could
be matched and eliminated.  Before lowering further, the [ABCD algorithm](
http://www.cs.virginia.edu/kim/courses/cs771/papers/bodik00abcd.pdf) -- a straightforward value range analysis based on
SSA -- then ran to eliminate even more common patterns using a more principled approach than pattern matching.  We were
also able to leverage ABCD in the global analysis phase too, thanks to inter-procedural length and control flow fact
propagation. 
-->
在最高级别，在巴托克，IR仍然相对接近计划输入。 因此，可以匹配和消除一些简单的模式。 在进一步降低之前，ABCD算法 - 基于SSA的直接值范围分析 - 然后使用比模式匹配更原则的方法来消除甚至更常见的模式。 由于程序间长度和控制流事实传播，我们也能够在全局分析阶段利用ABCD。

<!-- 
Next up, the Phoenix Loop Optimizer got its hands on things.  This layer did all sorts of loop optimizations and, most
relevant to this section, range analysis.  For example: 
-->

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
* 循环实现：此分析实际上创建循环。 它识别重复的代码模式，这些模式更理想地表示为循环，并且在有利可图时，将其重写为这样。 这包括展开手动循环，以便矢量化程序可以触及它们，即使它们可能稍后重新展开。
* 循环克隆，展开和版本控制：此分析创建循环副本以用于专业化。 这包括循环展开，创建矢量化循环的体系结构特定版本，等等。
* 感应范围优化：这是我们在本节中最关注的阶段。 除了进行经典的感应变量优化（如加宽）之外，它还使用感应范围分析来删除不必要的检查。 作为该阶段的副产品，边界检查被消除并通过将它们悬挂在环路之外而合并。


<!-- 
This sort of principled analysis was more capable than what was shown earlier.  For example, there are ways to write
the earlier loop that can easily "trick" the more basic techniques discussed earlier: 
-->

这种原则性分析比之前显示的更有能力。 例如，有一些方法可以编写早期的循环，可以轻松“欺骗”前面讨论的更基本的技术：

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
    
    // 技巧#1: 使用长度而不是常量。
    for (int i = 0; i < a.length; i++) {
        a[i] = i;
    }
    
    // 技巧#2: 从1开始计数。
    for (int i = 1; i <= a.length; i++) {
        a[i-1] = i-1;
    }
    
    // 技巧#3: 向后倒数。
    for (int i = a.length - 1; i >= 0; i--) {
        a[i] = i;
    }
    
    // 技巧#4: 根本不要使用for循环。
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
你明白了。 很明显，在某些时候你可以搞定优化器执行任何操作的能力，特别是如果你开始在循环体内进行虚拟调度，其中别名信息会丢失。 显然，当数组长度静态不知道时，事情会变得更加困难，如上面的100例所示。但是，如果你可以证明循环边界和数组之间的关系，那么所有都不会丢失。 大部分分析需要特别了解C＃中的数组长度是不可变的。

<!-- 
At the end of the day, doing a good job at optimizing here is the difference between this: 
-->
在一天结束时，做好优化这里的区别是：
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
    ; 将归纳变量初始化为0：
    3D45: 33 C0           xor         eax,eax
    ; 将界限放入EDX：
    3D58: 8B 51 08        mov         edx,dword ptr [rcx+8]
    ;检查EAX是否仍然在界限内; 如果没有跳：
    3D5B: 3B C2           cmp         eax,edx
    3D5D: 73 13           jae         3D72
    ;计算元素地址并存储到其中：
    3D5F: 48 63 D0        movsxd      rdx,eax
    3D62: 89 44 91 10     mov         dword ptr [rcx+rdx*4+10h],eax
    ; 增加循环归纳变量：
    3D66: FF C0           inc         eax
    ;如果仍然<100，则跳回到循环开始：
    3D68: 83 F8 64        cmp         eax,64h
    3D6B: 7C EB           jl          3D58
    ; ...
    ; 错误例程：
    3D72: E8 B9 E2 FF FF  call        2030 

<!-- 
And the following, completely optimized, bounds check free, loop: 
-->
以下，完全优化，边界检查免费，循环：

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
    ; 将归纳变量初始化为0：
    3D95: 33 C0           xor         eax,eax
    ; 计算元素地址并存储到其中：
    3D97: 48 63 D0        movsxd      rdx,eax
    3D9A: 89 04 91        mov         dword ptr [rcx+rdx*4],eax
    ; 增加循环归纳变量：
    3D9D: FF C0           inc         eax
    ; 如果still<100，则跳回到循环开始：
    3D9F: 83 F8 64        cmp         eax,64h
    3DA2: 7C F3           jl          3D97 
<!-- 
It's amusing that I'm now suffering deja vu as we go through this same exercise with C++'s new `array_view<T>` type.
Sometimes I joke with my ex-Midori colleagues that we're destined to repeat ourselves, slowly and patiently, over the
course of the next 10 years.  I know that sounds arrogant.  But I have this feeling on almost a daily basis. 
-->
有趣的是，当我们使用C ++的新array_view <T>类型进行同样的练习时，我现在正遭遇似曾相识。 有时候我和前Midori的同事开玩笑说，我们注定要在接下来的10年里慢慢地，耐心地重复自己。 我知道这听起来很傲慢。 但我几乎每天都有这种感觉。


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
如前所述，在Midori中，我们默认使用检查算法进行编译（通过C＃的/ checked标志）。 这消除了开发人员没有预料到并因此正确编码的错误类别溢出。 当然，我们保留了显式的已检查和未检查的作用域构造，以在适当时覆盖默认值，但这更可取，因为程序员声明了她的意图。

<!-- 
Anyway, as you might expect, this can reduce code quality too. 
-->
无论如何，正如您所料，这也会降低代码质量。

<!-- 
For comparison, imagine we're adding two variables: 
-->
为了比较，假设我们添加了两个变量：

    int x = ...;
    int y = ...;
    int z = x + y; 


<!-- 
Now imagine `x` is in `ECX` and `y` is in `EDX`.  Here is a standard unchecked add operation: 
-->
现在假设`x`在`ECX`中，`y`在`EDX`中。 这是一个标准的未经检查的添加操作：

    03 C2              add         ecx,edx 


<!-- 
Or, if you want to get fancy, one that uses the `LEA` instruction to also store the result in the `EAX` register using
a single instruction, as many modern compilers might do: 
-->
或者，如果你想得到花哨，那么使用`LEA`指令也可以使用单个指令将结果存储在`EAX`寄存器中，正如许多现代编译器可能会做的那样：

    8D 04 11           lea         eax,[rcx+rdx] 


<!-- 
Well, here's the equivalent code with a bounds check inserted into it: 
-->
好吧，这是插入边界检查的等效代码：

    3A65: 8B C1              mov         eax,ecx
    3A67: 03 C2              add         eax,edx
    3A69: 70 05              jo          3A70
    ; ...
    3A70: E8 B3 E5 FF FF     call        2028 


<!-- 
More of those damn conditional jumps (`JO`) with error handling routines (`CALL 2028`). 
-->
更多那些该死的条件跳转（`JO`）和错误处理例程（`CALL 2028`）。

<!-- 
It turns out a lot of the analysis mentioned earlier that goes into proving bounds checks redundant also apply to
proving that overflow checks are redundant.  It's all about proving facts about ranges.  For example, if you can prove
that some check is [dominated by some earlier check](https://en.wikipedia.org/wiki/Dominator_(graph_theory)), and that
furthermore that earlier check is a superset of the later check, then the later check is unnecessary.  If the opposite
is true -- that is, the earlier check is a subset of the later check, then if the subsequent block postdominates the
earlier one, you might move the stronger check to earlier in the program. 
-->
事实证明，前面提到的很多分析都用于证明边界检查冗余也适用于证明溢出检查是多余的。 这一切都是为了证明范围的事实。 例如，如果您可以证明某些检查由某些早期检查支配，并且此外早期检查是后续检查的超集，那么后面的检查是不必要的。 如果相反的情况也是如此 - 也就是说，较早的检查是后面检查的一个子集，那么如果后续块后置了较早的检查，则可以将更强的检查移到程序的前面。

<!-- 
Another common pattern is that the same, or similar, arithmetic operation happens multiple times near one another:
-->
另一种常见模式是相同或类似的算术运算在彼此附近多次发生：

    int p = r * 32 + 64;
    int q = r * 32 + 64 - 16; 


<!-- 
It is obvious that, if the `p` assignment didn't overflow, then the `q` one won't either. 
-->
很明显，如果`p`赋值没有溢出，那么`q`赋值也不会溢出。

<!-- 
There's another magical phenomenon that happens in real world code a lot.  It's common to have bounds checks and
arithmetic checks in the same neighborhood.  Imagine some code that reads a bunch of values from an array: 
-->
在现实世界的代码中发生了另一种神奇的现象。 在同一邻域中进行边界检查和算术检查是很常见的。 想象一下从数组中读取一堆值的代码：

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
好的C＃数组不能有负界限。 如果编译器知道DATA_SIZE足够小以至于溢出的计算不会绕过0，那么它可以消除范围检查以支持边界检查。

<!-- 
There are many other patterns and special cases you can cover.  But the above demonstrates the power of a really good
range optimizer that is integrated with loops optimization.  It can cover a wide array of scenarios, array bounds and
arithmetic operations included.  It takes a lot of work, but it's worth it in the end. 
-->
您可以涵盖许多其他模式和特殊情况。 但是上面展示了与环路优化集成的非常好的范围优化器的强大功能。 它可以涵盖各种场景，包括数组边界和算术运算。 它需要做很多工作，但最终还是值得的。

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
在大多数情况下，内联与真正的本机代码相同。 同样重要的是。 通常更重要的是，由于C＃开发人员倾向于编写许多小方法（如属性访问器）。 由于本文中的许多主题，获取小代码可能比在C ++中更难 - 更多分支，更多检查等 - 因此，在实践中，大多数托管代码编译器内联比本机代码编译器少得多，或者 至少需要以非常不同的方式进行调整。 这实际上可以决定性能。

<!-- 
There are also areas of habitual bloat.  The way lambdas are encoded in MSIL is unintelligable to a naive backend
compiler, unless it reverse engineers that fact.  For example, we had an optimization that took this code: 
-->
还有一些习惯性膨胀的领域。 lambdas在MSIL中编码的方式对于一个天真的后端编译器是不可理解的，除非它反过来设计这个事实。 例如，我们进行了一次采用此代码的优化：

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
并且，在内联之后，能够将B转变为：

    void B() {
        int x = 43;
        ...
    } 


<!-- 
That `Action` argument to `A` is a lambda and, if you know how the C# compiler encodes lambdas in MSIL, you'll
appreciate how difficult this trick was.  For example, here is the code for B: 
-->
A的Action参数是一个lambda，如果你知道C＃编译器如何在MSIL中编码lambdas，你会明白这个技巧有多难。 例如，这是B的代码：

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
为了获得所需的神奇结果，必须不断传播ldftn，识别委托构造的工作方式（IL_0017），利用该信息内联B并完全消除lambda / delegate，然后再次主要通过常量传播，将算法折叠成常量42 初始化x。 我总是觉得这很“优雅”，这种“堕落”是多重优化的自然组合，并带有不同的顾虑。

<!-- 
As with native code, profile guided optimization made our inlining decisions far more effective. 
-->
与本机代码一样，配置文件引导优化使我们的内联决策更加有效。

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
CLI结构几乎就像C结构一样。 除非他们不是。 CLI强加了一些会产生开销的语义。 这些开销几乎总是表现为过度复制。 更糟糕的是，这些副本通常隐藏在您的程序中。 值得注意的是，由于复制构造函数和析构函数，C ++在这里也存在一些实际问题，通常甚至比我将要描述的更糟糕。

<!-- 
Perhaps the most annoying is that initializing a struct the CLI way requires a defensive copy.  For example, consider
this program, where the initialzer for `S` throws an exception: 
-->
也许最令人讨厌的是，以CLI方式初始化结构需要防御性副本。 例如，考虑这个程序，其中S的初始化器抛出异常：


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
此处的程序行为必须是将值0写入控制台。 实际上，这意味着赋值操作s = new S（42）必须首先在堆栈上创建一个新的S类型槽，然后构造它，然后只在s变量上复制该值。 对于像这样的单int结构，这不是一个大问题。 对于大型结构，这意味着使用memcpy。 在Midori中，我们知道哪些方法可以抛出，哪些方法无法抛出，这要归功于我们的错误模型（更晚些时候），这意味着我们可以在几乎所有情况下避免这种开销。

<!-- 
Another annoying one is the following: 
-->
另一个令人讨厌的是以下内容：


    struct S {
        // ...
        public int Value { get { return this.value; } }
    }
    
    static readonly S s = new S(); 


<!-- 
Every single time we read from `s.Value`: 
-->
每次我们从`s.Value`读取：

    int x = s.Value; 

<!-- 
we are going to get a local copy.  This one's actually visible in the MSIL.  This is without `readonly`: 
-->
我们将获得一份本地副本。 这个实际上在MAIL中可见。 这没有`readonly`：

    ldsflda    valuetype S Program::s
    call       instance int32 S::get_Value() 


<!-- 
And this is with it: 
-->
这就是它：

    ldsfld     valuetype S Program::s
    stloc.0
    ldloca.s   V_0
    call       instance int32 S::get_Value() 


<!-- 
Notice that the compiler elected to use `ldsfld` followed by `lodloca.s`, rather than loading the address directly,
by way of `ldsflda` in the first example.  The resulting machine code is even nastier.  I also can't pass the struct
around by-reference which, as I mention later on, requires copying it and again can be problematic. 
-->
请注意，编译器选择使用ldsfld后跟lodloca.s，而不是通过第一个示例中的ldsflda直接加载地址。 生成的机器代码甚至更糟糕。 我也无法通过引用传递结构，正如我后面提到的那样，它需要复制它并且再次出现问题。

<!-- 
We solved this in Midori because our compiler knew about methods that didn't mutate members.  All statics were immutable
to begin with, so the above `s` wouldn't need defensive copies.  Alternatively, or in addition to this, the struct could
have beem declared as `immutable`, as follows: 
-->
我们在Midori中解决了这个问题，因为我们的编译器知道不会改变成员的方法。 所有的静力学都是一成不变的，所以上面的s不需要防御性的副本。 或者，或者除此之外，struct可以将beem声明为immutable，如下所示：

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
或者因为所有静态值无论如何都是不可变的。 或者，所讨论的属性或方法可以注释为可读，意味着它们不能触发突变，因此不需要防御性副本。

<!-- 
I mentioned by-reference passing.  In C++, developers know to pass large structures by-reference, either using `*` or
`&`, to avoid excessive copying.  We got in the habit of doing the same.  For example, we had `in` parameters, as so: 
-->
我提到了参考传递。 在C ++中，开发人员知道通过引用传递大型结构，使用*或＆，以避免过度复制。 我们养成了做同样的习惯。 例如，我们有参数，如下：
<!-- 
    void M(in ReallyBigStruct s) {
        // Read, but don't assign to, s ...
    } 
-->   
    void M(in ReallyBigStruct s) {
        // 读取，但不要分配 ...
    } 

<!-- 
I'll admit we probably took this to an extreme, to the point where our APIs suffered.  If I could do it all over again,
I'd go back and eliminate the fundamental distinction between `class` and `struct` in C#.  It turns out, pointers aren't
that bad after all, and for systems code you really do want to deeply understand the distinction between "near" (value)
and "far" (pointer).  We did implement what amounted to C++ references in C#, which helped, but not enough.  More on
this in my upcoming deep dive on our programming language. 
-->
我承认我们可能会把这个问题推向极端，直到我们的API受到影响。 如果我能再做一遍，我会回过头来消除C＃中class和struct之间的根本区别。 事实证明，指针毕竟不是那么糟糕，对于系统代码，你真的想深入理解“近”（值）和“远”（指针）之间的区别。 我们确实在C＃中实现了C ++引用的功能，这有所帮助，但还不够。 更多关于我即将深入研究我们的编程语言的内容。

<!-- 
## Code size 
-->
## 代码大小

<!-- 
We pushed hard on code size.  Even more than some C++ compilers I know. 
-->
我们努力推动代码大小。 甚至比我知道的一些C ++编译器还要多。

<!-- 
A generic instantiation is just a fancy copy-and-paste of code with some substitutions.  Quite simply, that means an
explosion of code for the compiler to process, compared to what the developer actually wrote.  [I've covered many of the
performance challenges with generics in the past.](
http://joeduffyblog.com/2011/10/23/on-generics-and-some-of-the-associated-overheads/)  A major problem there is the
transitive closure problem.  .NET's straightforward-looking `List<T>` class actually creates 28 types in its transitive
closure!  And that's not even speaking to all the methods in each type.  Generics are a quick way to explode code size. 
-->
通用实例化只是一些具有一些替换的代码的复制和粘贴。 很简单，这意味着与开发人员实际编写的代码相比，编译器要处理的代码数量激增。 过去，我已经介绍了泛型的许多性能挑战。 一个主要问题是传递闭包问题。 .NET的直观List <T>类实际上在其传递闭包中创建了28种类型！ 而且这甚至都没有说出每种类型的所有方法。 泛型是一种爆炸代码大小的快速方法。


<!-- 
I never forgot the day I refactored our LINQ implementation.  Unlike in .NET, which uses extension methods, we made all
LINQ operations instance methods on the base-most class in our collection type hierarchy.  That meant 100-ish nested
classes, one for each LINQ operation, *for every single collection instantiated*!  Refactoring this was an easy way for
me to save over 100MB of code size across the entire Midori "workstation" operating system image.  Yes, 100MB! 
-->
我永远不会忘记我重构LINQ实现的那一天。 与使用扩展方法的.NET不同，我们在集合类型层次结构中的最基类上创建了所有LINQ操作实例方法。 对于实例化的每个集合，这意味着100-ish嵌套类，每个LINQ操作一个！ 重构这是一种简单的方法，可以在整个Midori“工作站”操作系统映像中保存超过100MB的代码大小。 是的，100MB！

<!-- 
We learned to be more thoughtful about our use of generics.  For example, types nested inside an outer generic are
usually not good ideas.  We also aggressively shared generic instantiations, even more than [what the CLR does](
http://blogs.msdn.com/b/joelpob/archive/2004/11/17/259224.aspx).  Namely, we shared value type generics, where the
GC pointers were at the same locations.  So, for example, given a struct S: 
-->
我们学会了更加周到地使用泛型。 例如，嵌套在外部泛型中的类型通常不是好主意。 我们还积极地共享通用实例，甚至比CLR更多。 也就是说，我们共享了值类型泛型，其中GC指针位于相同的位置。 所以，例如，给定一个结构S：

    struct S {
        int Field;
    } 


<!-- 
we would share the same code representation of `List<int>` with `List<S>`.  And, similarly, given: 
-->
我们将与`List <S>`共享`List <int>`的相同代码表示。 并且，同样地，给出：


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
我们将在`List <S>`和`List <T>`之间共享实例。

<!-- 
You might not realize this, but C# emits IL that ensures `struct`s have `sequential` layout: 
-->
您可能没有意识到这一点，但C＃发出IL，确保`struct`s具有`sequential`布局：


    .class private sequential ansi sealed beforefieldinit S
        extends [mscorlib]System.ValueType
    {
        ...
    } 

<!-- 
As a result, we couldn't share `List<S>` and `List<T>` with some hypothetical `List<U>`: 
-->
结果，我们不能与一些假设的`List <U>`共享`List <S>`和`List <T>`：

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
为此，除了其他原因之外 - 比如让编译器在打包，缓存对齐等方面具有更大的灵活性 - 我们在我们的语言中默认使用自动结构。 实际上，顺序只对你做不安全的代码很重要，在我们的编程模型中，这些代码甚至都不合法。

<!-- 
We did not support reflection in Midori.  In principle, we had plans to do it eventually, as a purely opt-in feature.
In practice, we never needed it.  What we found is that code generation was always a more suitable solution.  We shaved
off at least 30% of the best case C# image size by doing this.  Significantly more if you factor in systems where the
full MSIL is retained, as is usually the case, even for NGen and .NET AOT solutions. 
-->
我们不支持Midori的反思。 原则上，我们最终计划将其作为纯粹的选择加入功能。 在实践中，我们从不需要它。 我们发现代码生成始终是更合适的解决方案。 通过这样做，我们削减了至少30％的最佳案例C＃图像大小。 如果您考虑保留完整MSIL的系统，通常情况下，即使对于NGen和.NET AOT解决方案，也要更多。

<!-- 
In fact, we removed significant pieces of `System.Type` too.  No `Assembly`, no `BaseType`, and yes, even no `FullName`.
The .NET Framework's mscorlib.dll contains about 100KB of just type names.  Sure, names are useful, but our eventing
framework leveraged code generation to produce just those you actually needed to be around at runtime. 
-->
实际上，我们也删除了很多System.Type。 没有程序集，没有BaseType，是的，甚至没有FullName。 .NET Framework的mscorlib.dll包含大约100KB的类型名称。 当然，名称很有用，但我们的事件框架利用代码生成来生成您在运行时实际需要的那些代码。

<!-- 
At some point, we realized 40% of our image sizes were [vtable](https://en.wikipedia.org/wiki/Virtual_method_table)s.
We kept pounding on this one relentlessly, and, after all of that, we still had plenty of headroom for improvements. 
-->
在某些时候，我们意识到我们40％的图像尺寸都是vtable。 
我们一直在坚持不懈地追逐这一点，毕竟，我们仍然有足够的改进空间。

<!-- 
Each vtable consumes image space to hold pointers to the virtual functions used in dispatch, and of course has a runtime
representation.  Each object with a vtable also has a vtable pointer embedded within it.  So, if you care about size
(both image and runtime), you are going to care about vtables. 
-->
每个vtable都使用图像空间来保存指向调度中使用的虚函数的指针，
当然还有一个运行时表示。 具有vtable的每个对象也具有嵌入其中的vtable指针。 
所以，如果你关心大小（图像和运行时），你就会关心vtable。

<!-- 
In C++, you only get a vtable if your type is [polymorphic](http://www.cplusplus.com/doc/tutorial/typecasting/).  In
languages like C# and Java, on the other hand, you get a vtable even if you don't want, need, or use it.  In C#, at
least, you can use a `struct` type to elide them.  I actually love this aspect of Go, where you get a virtual dispatch-
like thing, via interfaces, without needing to pay for vtables on every type; you only pay for what you use, at the
point of coercing something to an interface. 
-->
在C ++中，如果您的类型是多态的，那么您只能得到一个vtable。 另一方面，在C＃和Java等语言中，即使您不想要，也不需要或使用它，您也会获得vtable。 至少在C＃中，您可以使用结构类型来忽略它们。 我真的很喜欢Go的这个方面，你可以通过接口获得类似虚拟调度的东西，而无需为每种类型的vtable付费; 你只需要为某个界面强迫某些东西付费。

<!-- 
Another vtable problem in C# is that all objects inherit three virtuals from `System.Object`: `Equals`, `GetHashCode`,
and `ToString`.  Besides the point that these generally don't do the right thing in the right way anyways -- `Equals`
requires reflection to work on value types, `GetHashCode` is nondeterministic and stamps the object header (or sync-
block; more on that later), and `ToString` doesn't offer formatting and localization controls -- they also bloat every
vtable by three slots.  This may not sound like much, but it's certainly more than C++ which has no such overhead. 
-->
C#中的另一个vtable问题是所有对象都从System.Object继承三个虚拟：Equals，GetHashCode和ToString。 除了这些通常不会以正确的方式做正确的事情之外 -  Equals需要反射来处理值类型，GetHashCode是不确定的并且标记对象头（或同步块;稍后会有更多内容）和ToString 不提供格式化和本地化控制 - 它们也会使每个vtable膨胀三个插槽。 这可能听起来不是很多，但它肯定比没有这样的开销的C ++更多。

<!-- 
The main source of our remaining woes here was the assumption in C#, and frankly most OOP languages like C++ and Java,
that [RTTI](https://en.wikipedia.org/wiki/Run-time_type_information) is always available for downcasts.  This was
particularly painful with generics, for all of the above reasons.  Although we aggressively shared instantiations, we
could never quite fully fold together the type structures for these guys, even though disparate instantiations tended
to be identical, or at least extraordinarily similar.  If I could do it all over agan, I'd banish RTTI.  In 90% of the
cases, type discriminated unions or pattern matching are more appropriate solutions anyway. 
-->
我们剩下的困境的主要来源是C＃中的假设，坦率地说，大多数OOP语言（如C ++和Java），RTTI始终可用于向下转换。 由于上述所有原因，这对于仿制药尤其痛苦。 虽然我们积极地共享实例化，但我们永远无法完全折叠这些人的类型结构，即使不同的实例化往往是相同的，或者至少非常相似。 如果我能全力以赴，我就会放弃RTTI。 在90％的情况下，无论如何，类型区分联合或模式匹配都是更合适的解决方案。

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
我已经提到了配置文件引导优化（PGO）。 在本文中的大部分内容都具有竞争力之后，这是“走到最后一英里”的关键因素。 这使我们的浏览器程序在SunSpider和Octane等基准测试中增加了30-40％。

<!-- 
Most of what went into PGO was similar to classical native profilers, with two big differences. 
-->
进入PGO的大部分内容与传统的原生剖析器类似，有两个很大的不同。

<!-- 
First, we tought PGO about many of the unique optimizations listed throughout this article, such as asynchronous stack
probing, generics instantiations, lambdas, and more.  As with many things, we could have gone on forever here. 
-->
首先，我们向PGO介绍了本文中列出的许多独特优化，例如异步堆栈探测，泛型实例化，lambdas等。 和许多事情一样，我们可以在这里永远继续下去。

<!-- 
Second, we experimented with sample profiling, in addition to the ordinary instrumented profiling.  This is much nicer
from a developer perspective -- they don't need two builds -- and also lets you collect counts from real, live running
systems in the data center.  A good example of what's possible is outlined in [this Google-Wide Profiling (GWP) paper](
http://static.googleusercontent.com/media/research.google.com/en/us/pubs/archive/36575.pdf). 
-->
其次，除了普通的仪器化分析之外，我们还尝试了样本分析。 
从开发人员的角度来看，这样做更好 - 他们不需要两个版本 - 并且还允许您从数据中心的实际运行系统中收集计数。
Google全面分析（GWP）文件中概述了一个很好的例子。


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
上述基础知识都很重要。 但是，一些更具影响力的领域需要与语言，运行时，框架和操作系统本身进行更深层次的架构协同设计和协同演进。 我之前写过关于这种“整个系统”方法的巨大好处。 这有点神奇。

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
Midori是通过垃圾收集的。 这是我们整体模型的安全性和生产率的关键因素。 事实上，在某一点上，我们有11个不同的收藏家，每个收藏家都有自己独特的特征。 （例如，参见这项研究。）我们有一些方法可以解决常见问题，例如长时间停顿。 不过，我会在以后的文章中介绍这些内容。 现在，让我们坚持代码质量的领域。

<!-- 
The first top-level decision is: *conservative* or *precise*?  A conserative collector is easier to wedge into an
existing system, however it can cause troubles in certain areas.  It often needs to scan more of the heap to get the
same job done.  And it can falsely keep objects alive.  We felt both were unacceptable for a systems programming
environment.  It was an easy, quick decision: we sought precision. 
-->
第一个顶级决定是：保守还是精确？ 一个连续的收集器更容易融入现有系统，但它可能会在某些区域引起麻烦。 它通常需要扫描更多的堆才能完成相同的工作。 并且它可以错误地保持对象存活。 我们认为这两者对于系统编程环境都是不可接受的。 这是一个简单，快速的决定：我们追求精确。

<!-- 
Precision costs you something in the code generators, however.  A precise collector needs to get instructions where to
find its root set.  That root set includes field offsets in data structures in the heap, and also places on the stack
or, even in some cases, registers.  It needs to find these so that it doesn't miss an object and erroneously collect it
or fail to adjust a pointer during a relocation, both of which would lead to memory safety problems.  There was no magic
trick to making this efficient other than close integration between runtime and code generator, and being thoughtful. 
-->
但是，精确度会使代码生成器中出现问题。 精确的收集器需要获取指示在哪里找到其根集。 该根集包括堆中数据结构中的字段偏移，并且还放置在堆栈上，或者甚至在某些情况下放置在寄存器中。 它需要找到它们，以便它不会错过一个对象并错误地收集它或者在重定位期间无法调整指针，这两者都会导致内存安全问题。 除了运行时和代码生成器之间的紧密集成以及周到之外，没有任何神奇的技巧可以使其高效。


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
这提出了合作与先发制人的主题，以及GC安全点的概念。以协作模式运行的GC只会在线程达到所谓的“安全点”时进行收集。另一方面，以抢占模式运行的GC可以通过抢占和线程暂停自由地阻止其中的线程，因此它可能会强制收集。一般来说，先发制人需要更多的簿记，因为必须在更多的地方识别根，包括已经泄漏到寄存器中的东西。它还使某些低级代码难以编写，您可能会在操作系统的内核中找到它，因为对象可能会在任意指令之间移动。这很难推理。 （如果您不相信我，请参阅此文件及其在CLR代码库中的相关用法。）因此，我们使用协作模式作为默认值。我们尝试使用编译器插入的自动安全点探针，例如在循环后沿上，但选择代替质量代码。它确实意味着GC“活锁”是可能的，但在实践中我们很少遇到这种情况。


<!-- 
We used a *generational* collector.  This has the advantage of reducing pause times because less of the heap needs to be
inspected upon a given collection.  It does come with one disadvantage from the code generator's perspective, which is
the need to insert write barriers into the code.  If an older generation object ever points back at a younger generation
object, then the collector -- which would have normally preferred to limit its scope to younger generations -- must know
to look at the older ones too.  Otherwise, it might miss something. 
-->
我们使用了世代收藏家。 这具有减少暂停时间的优点，因为在给定集合上需要检查较少的堆。 从代码生成器的角度来看，它确实存在一个缺点，即需要在代码中插入写入障碍。 如果老一代的物体曾经指向一个年轻一代的物体，那么收藏家 - 通常倾向于将其范围限制在年轻一代 - 必须知道也要看旧的物体。 否则，它可能会错过一些东西。

<!-- 
Write barriers show up as extra instructions after certain writes; e.g., note the `call`: 
-->
写入障碍在某些写入后显示为额外指令; 例如，记下`call`：

    48 8D 49 08        lea         rcx,[rcx+8]
    E8 7A E5 FF FF     call        0000064488002028 


<!-- 
That barrier simply updates an entry in the card table, so the GC knows to look at that segment the next time it scans
the heap.  Most of the time this ends up as inlined assembly code, however it depends on the particulars of the
situation.  See [this code](https://github.com/dotnet/coreclr/blob/master/src/vm/amd64/JitHelpers_Fast.asm#L462) for an
example of what this looks like for the CLR on x64. 
-->
该障碍只是更新卡表中的条目，因此GC知道下次扫描堆时查看该段。 
大多数情况下，这最终都是内联汇编代码，但这取决于具体情况。 请参阅此代码以获取x64上CLR的示例。

<!-- 
It's difficult for the compiler to optimize these away because the need for write barriers is "temporal" in nature.  We
did aggressively eliminate them for stack allocated objects, however.  And it's possible to write, or transform code,
into less barrier hungry styles.  For example, consider two ways of writing the same API: 
-->
编译器难以优化这些因为写入障碍的需要本质上是“暂时的”。 
但是，我们积极地为堆栈分配的对象消除它们。 
并且可以将代码编写或转换为更少障碍的风格。 例如，考虑两种编写相同API的方法：

    bool Test(out object o);
    object Test(out bool b); 


<!-- 
In the resulting `Test` method body, you will find a write barrier in the former, but not the latter.  Why?  Because the
former is writing a heap object reference (of type `object`), and the compiler has no idea, when analyzing this method
in isolation, whether that write is to another heap object.  It must be conservative in its analysis and assume the
worst.  The latter, of course, has no such problem, because a `bool` isn't something the GC needs to scan. 
-->
在生成的Test方法体中，您将在前者中找到写屏障，但不会在后者中找到写屏障。 为什么？ 因为前者正在编写堆对象引用（类型为object），并且编译器不知道，在单独分析此方法时，是否该写入是另一个堆对象。 它的分析必须保守，并假设最坏。 当然，后者没有这样的问题，因为bool不是GC需要扫描的东西。

<!-- 
Another aspect of GC that impacts code quality is the optional presence of more heavyweight concurrent read and write
barriers, when using concurrent collection.  A concurrent GC does some collection activities concurrent with the user
program making forward progress.  This is often a good use of multicore processors and it can reduce pause times and
help user code make more forward progress over a given period of time. 
-->
影响代码质量的GC的另一个方面是在使用并发收集时可选地存在更重量级的并发读写障碍。 并发GC与用户程序同时进行一些收集活动，从而取得进展。 这通常可以很好地利用多核处理器，它可以减少暂停时间，并帮助用户代码在给定的时间段内取得更多的进展。

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
构建并发GC存在许多挑战，但其中一个问题是由此产生的障碍成本很高。 Henry Baker的原始并发GC是一个复制GC，并且具有“旧”与“新”空间的概念。 必须检查所有读取和写入，并且必须将针对旧空间的任何操作转发到新空间。 DEC Firefly的后续研究使用硬件内存保护来降低成本，但故障案例仍然非常昂贵。 而且，最糟糕的是，堆的访问时间是不可预测的。 对解决这个问题进行了很多很好的研究，但是我们放弃了复制。

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
相反，我们使用了并发标记扫描压缩收集器。这意味着在正常程序执行期间只需要写入障碍，但是某些代码被克隆，以便在程序在存在对象移动时运行时存在读取障碍。我们的主要GC人员的研究已经发表，所以你可以阅读所有相关内容。 CLR还有一个并发收集器，但它并不是那么好。它使用复制来收集最年轻的一代，对旧版本进行标记扫描，并将标记阶段并行化。遗憾的是有一些条件会导致连续暂停（想象这就像一个大的“锁定”），有时超过10毫秒：1）所有线程必须暂停和扫描，这个操作只受线程数的限制和堆栈的大小; 2）复制最年轻的一代只受那一代的大小限制（幸运的是，在正常配置中，这个很小）; 3）在最坏的情况下，即使是最老的一代，也可能发生压实和碎片整理。


<!-- 
## Separate compilation 
-->
## 单独编译

<!-- 
The basic model to start with is static linking.  In this model, you compile everything into a single executable.  The
benefits of this are obvious: it's simple, easy to comprehend, conceptually straightforward to service, and less work
for the entire compiler toolchain.  Honestly, given the move to Docker containers as the unit of servicing, this model
makes more and more sense by the day.  But at some point, for an entire operating system, you'll want separate
compilation.  Not just because compile times can get quite long when statically linking an entire operating system, but
also because the working set and footprint of the resulting processes will be bloated with significant duplication. 
-->
开始的基本模型是静态链接。 在此模型中，您将所有内容编译为单个可执行文件。 这样做的好处是显而易见的：它简单易懂，概念上服务简单，整个编译器工具链的工作量更少。 老实说，考虑到将Docker容器作为服务单元的转移，这种模式在当天越来越有意义。 但在某些时候，对于整个操作系统，您需要单独编译。 这不仅仅是因为静态链接整个操作系统时编译时间会很长，而且因为生成的进程的工作集和占用空间会大量重复。

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
单独编译面向对象的API很难。 
说实话，很少有人真正开始工作。 
问题包括脆弱的基类问题，这是版本弹性库的真正杀手。 结果，大多数真实系统在组件之间的边界处使用了一个笨拙的“C ABI”。 这就是为什么Windows，例如，历史上使用平面C Win32 API，即使在通过WinRT转向更多面向对象的情况下，也使用了COM。 在某些运行时费用中，ObjectiveC运行时解决了这一挑战。 与计算机科学中的大多数事物一样，几乎所有问题都可以通过额外的间接来解决; 这个也可以。

<!-- 
The design pivot we took in Midori was that whole processes were sealed.  There was no dynamic loading, so nothing that
looked like classical DLLs or SOs.  For those scenarios, we used the [Asynchronous Everything](
http://joeduffyblog.com/2015/11/19/asynchronous-everything/) programming model, which made it easy to dynamically
connect to and use separately compiled and versioned processes. 
-->
我们在Midori中采用的设计枢纽是整个过程都是密封的。 没有动态加载，所以没有看起来像经典的DLL或SO。 对于这些场景，我们使用了Asynchronous Everything编程模型，这使得动态连接和使用单独编译和版本化的过程变得容易。

<!-- 
We did, however, want separately compiled binaries, purely as a developer productivity and code sharing (working set)
play.  Well, I lied.  What we ended up with was incrementally compiled binaries, where a change in a root node triggered
a cascading recompilation of its dependencies.  But for leaf nodes, such as applications, life was beautiful.  Over
time, we got smarter in the toolchain by understanding precisely which sorts of changes could trigger cascading
invaliation of images.  A function that was known to never have been inlined across modules, for example, could have its
implementation -- but not its signature -- changed, without needing to trigger a rebuild.  This is similar to the
distinction between headers and objects in a classical C/C++ compilation model. 
-->
但是，我们确实需要单独编译的二进制文件，纯粹是开发人员的工作效率和代码共享（工作集）。 好吧，我骗了。 我们最终得到的是增量编译的二进制文件，其中根节点的更改触发了其依赖项的级联重新编译。 但对于叶节点，例如应用程序，生活很美。 随着时间的推移，我们通过精确了解哪种类型的更改可以触发图像的级联失效，在工具链中变得更加智能。 例如，一个已知从未在模块中内联的函数可以将其实现 - 但不是其签名 - 更改，而无需触发重建。 这类似于传统C/C++编译模型中标题和对象之间的区别。

<!-- 
Our compilation model was very similar to C++'s, in that there was static and dynamic linking.  The runtime model, of
course, was quite different.  We also had the notion of "library groups," which let us cluster multiple logically
distinct, but related, libraries into a single physical binary.  This let us do more aggressive inter-module
optimizations like inlining, devirtualization, async stack optimizations, and more. 
-->
我们的编译模型与C ++非常相似，因为它有静态和动态链接。 当然，运行时模型是完全不同的。 我们还有“库组”的概念，它允许我们将多个逻辑上不同但相关的库集群到一个物理二进制文件中。 这让我们可以进行更积极的模块间优化，如内联，虚拟化，异步堆栈优化等。

<!-- 
## Parametric polymorphism (a.k.a., generics) 
-->
## 参数多态（a.k.a.，generics）

<!-- 
That brings me to generics.  They throw a wrench into everything. 
-->
这让我想到了泛型（generics）。 他们把一切都搞砸了。

<!-- 
The problem is, unless you implement an [erasure
model](https://docs.oracle.com/javase/tutorial/java/generics/erasure.html) -- which utterly stinks performance-wise due
to boxing allocations, indirections, or both -- there's no way for you to possibly pre-instantiate all possible versions
of the code ahead-of-time.  For example, say you're providing a `List<T>`.  How do you know whether folks using your
library will want a `List<int>`, `List<string>`, or `List<SomeStructYouveNeverHeardOf>`? 
-->
问题是，除非你实现一个擦除模型 - 由于拳击分配，间接或两者都完全在性能方面发臭 - 你没有办法预先实例化代码的所有可能版本。 例如，假设您提供了List <T>。 你怎么知道使用你的库的人是否需要List <int>，List <string>或List <SomeStructYouveNeverHeardOf>？

<!-- 
Solutions abound: 
-->
解决方案比比皆是：

<!-- 
1. Do not specialize.  Erase everything.
2. Specialize only a subset of instantiations, and create an erased instantiation for the rest.
3. Specialize everything.  This gives the best performance, but at some complexity. 
-->
1.不专业。 擦除一切。
2.仅专门化实例化的子集，并为其余实例创建擦除实例化。
3.专注于一切。 这提供了最佳性能，但是有些复杂。

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
Java使用＃1（事实上，擦除被烘焙到语言中）。许多ML编译器使用＃2。 .NET的NGen编译模型是＃2的变体，其中可以简单地专门化的东西是专门的，其他一切都是JIT编译的。 .NET Native还没有这个问题的解决方案，这意味着第三方库，单独的编译和泛型是一个非常大的TBD。和Midori的一切一样，我们选择了最艰难的道路，最具有上升空间，这意味着＃3。其实我有点油腻;我们在团队中有几个ML编译器传说，＃2充满了危险;稍微深入一些关于这有多难（和聪明）的论文。很难先验地知道哪些实例化对程序至关重要。我自己在Longhorn时代试图将C＃代码放入Windows核心的经验也强化了这一点;我们不想要JIT'ting，你可以在这个世界中使用和不能使用的仿制药的规则令人难以置信，最终导致了希腊公式。

<!-- 
Anyway, Midori's approach turned out to be harder than it sounded at first. 
-->
无论如何，Midori的方法结果比起初听起来更难。

<!-- 
Imagine you have a diamond.  Library A exports a `List<T>` type, and libraries B and C both instantiate `List<int>`.  A
program D then consumes both B and C and maybe even passes `List<T>` objects returned from one to the other.  How do we
ensure that the versions of `List<int>` are compatible? 
-->
想象一下你有一颗钻石。 库A导出List <T>类型，库B和C都实例化List <int>。 程序D然后消耗B和C，甚至可以传递从一个返回到另一个的List <T>对象。 我们如何确保List <int>的版本兼容？

<!-- 
We called this problem the *potentially multiply instantiated*, or PMI for short, problem. 
-->
我们称此问题可能是多次实例化，或简称为PMI问题。

<!-- 
The CLR handles this problem by unifying the instantiations at runtime.  All RTTI data structures, vtables, and whatnot,
are built and/or aggressively patched at runtime.  In Midori, on the other hand, we wanted all such data structures to
be in readonly data segments and hence shareable across processes, wherever possible. 
-->
CLR通过在运行时统一实例化来处理此问题。 所有RTTI数据结构，vtable和诸如此类的东西都在运行时构建和/或积极修补。 另一方面，在Midori中，我们希望所有这些数据结构都在只读数据段中，因此尽可能在各个进程之间共享。

<!-- 
Again, everything can be solved with an indirection.  But unlike solution #2 above, solution #3 permits you to stick
indirections only in the rare places where you need them.  And for purposes of this one, that meant RTTI and accessing
static variables of just those generic types that might have been subject to PMI.  First, that affected a vast subset of
code (versus #2 which generally affects even loading of instance fields).  Second, it could be optimized away for
instantiations that were known not to be PMI, by attaching state and operations to the existing generic dictionary that
was gets passed around as a hidden argument already.  And finally, because of all of this, it was pay for play. 
-->
同样，一切都可以通过间接解决。 但与上面的解决方案＃2不同，解决方案＃3允许您仅在您需要它们的稀有地方坚持间接。 就本文而言，这意味着RTTI和访问那些可能受PMI影响的泛型类型的静态变量。 首先，它影响了大量代码（相对于＃2，通常影响甚至加载实例字段）。 其次，它可以通过将状态和操作附加到已经作为隐藏参数传递的现有通用字典来优化远离已知不是PMI的实例化。 最后，因为所有这一切，这是游戏的代价。

<!-- 
But damn was it complex. 
-->
但该死的很复杂。

<!-- 
It's funny, but C++ RTTI for template instantiations actually suffers from many of the same problems.  In fact, the
Microsoft Visual C++ compiler resorts to a `strcmp` of the type names, to resolve diamond issues!  (Thankfully there are
[well-known, more efficient ways to do this](
http://www6.open-std.org/JTC1/SC22/WG21/docs/papers/1992/WG21%201992/X3J16_92-0068%20WG21_N0145.pdf), which we are
actively pursuing for the next release of VC++.) 
-->
这很有趣，但用于模板实例化的C ++ RTTI实际上遇到了许多相同的问题。 实际上，Microsoft Visual C ++编译器会转换为类型名称的strcmp，以解决钻石问题！ （值得庆幸的是，有一些众所周知的，更有效的方法可以做到这一点，我们正在积极寻求下一版VC ++。）

<!-- 
## Virtual dispatch 
-->
## 虚拟指派

<!-- 
Although I felt differently when first switching from Java to C#, Midori made me love that C# made methods non-virtual
by default.  I'm sure we would have had to change this otherwise.  In fact, we went even further and made classes
`sealed` by default, requiring that you explicitly mark them `virtual` if you wanted to facilitate subclasses. 
-->
虽然我第一次从Java切换到C＃时感觉不同，但是Midori让我喜欢C＃默认情况下非虚拟方法。 我相信我们不得不改变这一点。 事实上，我们更进一步，默认情况下使类密封，如果你想促进子类，则要求你明确标记它们。

<!-- 
Aggressive devirtualization, however, was key to good performance.  Each virtual means an indirection.  And more
impactfully, a lost opportunity to inline (which for small functions is essential).  We of course did global
intra-module analysis to devirtualize, but also extended this across modules, using whole program compilation, when
multiple binaries were grouped together into a library group. 
-->
然而，积极的虚拟化是良好绩效的关键。 每个虚拟意味着间接。 更有影响力的是，失去了内联的机会（对于小功能来说是必不可少的）。 我们当然进行了全局模块内分析以进行虚拟化，但是当多个二进制文件组合成一个库组时，还使用整个程序编译在模块之间进行扩展。

<!-- 
Although our defaults were right, m experience with C# developers is that they go a little hog-wild with virtuals and
overly abstract code.  I think the ecosystem of APIs that exploded around highly polymorphic abstractions, like LINQ and
Reactive Extensions, encouraged this and instilled a bit of bad behavior ("gratuitous over-abstraction").  I guess you
could make similar arguments about highly templated code in C++.  As you can guess, there wasn't very much of it in the
lowest levels of our codebase -- where every allocation and instruction mattered -- but in higher level code, especially
in applications that tended to be dominated by high-latency asynchronous operations, the overheads were acceptable and
productivity benefits high.  A strong culture around identifying and trimming excessive fat helped to ensure features
like this were used appropriately, via code reviews, benchmarks, and aggressive static analysis checking. 
-->
虽然我们的默认设置是正确的，但是对于C＃开发人员来说，他们对虚拟和过于抽象的代码有点兴趣。 我认为围绕高度多态抽象（如LINQ和Reactive Extensions）爆炸的API生态系统鼓励了这种情况并灌输了一些不良行为（“无偿的过度抽象”）。 我想你可以在C ++中对高度模板化的代码做出类似的论证。 正如您所猜测的那样，在我们的代码库的最低层 - 其中每个分配和指令都很重要 - 并没有很多 - 但在更高级别的代码中，特别是在那些倾向于由高延迟异步操作主导的应用程序中， 开销是可以接受的，生产效率很高。 通过代码审查，基准测试和积极的静态分析检查，围绕识别和修剪过多脂肪的强大文化有助于确保适当使用此类功能。

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
.NET Framework中只有一些设计不良，效率低下的模式。 IEnumerator <T>只需要两个接口调度来提取下一个项目！ 将其与C ++迭代器进行比较，C ++迭代器可以编译指针增量加解除引用。 其中许多问题可以通过更好的库设计来解决。 （我们的枚举最终设计甚至根本不会侵入接口。）

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
再加上调用C＃接口很棘手。 现有系统不像C ++那样使用指针调整，因此通常接口调度需要表搜索。 首先是到达vtable的间接级别，然后是另一个级别来查找有问题的接口的接口表。 有些系统尝试对单态调用进行callsite缓存; 也就是说，缓存最新的调用，希望同一个对象类一次又一次地通过该调用点。 这需要可变的存根，更不用说一个非常复杂的thunk系统和诸如此类的东西。 在Midori，我们从未违反过W ^ X; 我们避免了可变的运行时数据结构，因为它们在工作集方面抑制了共享，但也分摊了TLB和数据缓存压力。

<!-- 
Our solution took advantage of the memory ordering model earlier.  We used so-called "fat" interface pointers.  A fat
interface pointer was two words: the first, a pointer to the object itself; the second, a pointer to the interface
vtable for that object.  This made conversion to interfaces slightly slower -- because the interface vtable lookup had
to happen -- but for cases where you are invoking it one or more times, it came out a wash or ahead.  Usually,
significantly.  Go does something like this, but it's slightly different for two reasons.  First, they generate the
interface tables on the fly, because interfaces are duck typed.  Second, fat interface pointers are subject to tearing
and hence can violate memory safety in Go, unlike Midori thanks to our strong concurrency model. 
-->
我们的解决方案早先利用了内存排序模型。 我们使用了所谓的“胖”接口指针。 胖接口指针是两个单词：第一个是指向对象本身的指针; 第二个，指向该对象的接口vtable的指针。 这使得转换到接口的速度稍慢 - 因为接口vtable查找必须发生 - 但是对于你一次或多次调用它的情况，它出现了一个清洗或提前。 通常，显着。 Go做了类似的事情，但由于两个原因，它略有不同。 首先，它们动态生成接口表，因为接口是鸭子类型。 其次，由于我们强大的并发模型，胖接口指针会受到撕裂，因此可能会违反Go中的内存安全性，这与Midori不同。

<!-- 
The finally challenge in this category was *generic virtual methods*, or GVMs.  To cut to the chase, we banned them.
Even if you NGen an image in .NET, all it takes is a call to the LINQ query `a.Where(...).Select(...)`, and you're
pulling in the JIT compiler.  Even in .NET Native, there is considerable runtime data structure creation, lazily, when
this happens.  In short, there is no known way to AOT compile GVMs in a way that is efficient at runtime.  So, we didn't
even bother offering them.  This was a slightly annoying limitation on the programming model but I'd have done it all
over again thanks to the efficiencies that it bought us.  It really is surprising how many GVMs are lurking in .NET. 
-->
此类别中的最终挑战是通用虚拟方法或GVM。 为了减少追捕，我们禁止他们。 即使您在.NET中使用图像，只需调用LINQ查询a.Where（...）。选择（...），然后您就可以使用JIT编译器了。 即使在.NET Native中，当发生这种情况时，懒洋洋地创建了相当多的运行时数据结构。 简而言之，AOT编译GVM没有已知的方法在运行时有效。 所以，我们甚至没有提供它们。 这对编程模型来说是一个有点恼人的限制，但由于它为我们带来了效率，我已经重新做了一遍。 令人惊讶的是，有多少GVM潜伏在.NET中。

<!-- 
## Statics 
-->
## 静态值

<!-- 
I was astonished the day I learned that 10% of our code size was spent on static initialization checks. 
-->
当我了解到10％的代码大小用于静态初始化检查时，我感到很惊讶。

<!-- 
Many people probably don't realize that the [CLI specification](
http://www.ecma-international.org/publications/standards/Ecma-335.htm) offers two static initialization modes.  There
is the default mode and `beforefieldinit`.  The default mode is the same as Java's.  And it's horrible. The static
initializer will be run just prior to accessing any static field on that type, any static method on that type, any
instance or virtual method on that type (if it's a value type), or any constructor on that type.  The "when" part
doesn't matter as much as what it takes to make this happen; *all* of those places now need to be guarded with explicit
lazy initialization checks in the resulting machine code! 
-->
许多人可能没有意识到CLI规范提供了两种静态初始化模式。 
有默认模式和beforefieldinit。 
默认模式与Java相同。 这太可怕了。 
静态初始化程序将在访问该类型上的任何静态字段，该类型上的任何静态方法，该类型上的任何实例或虚方法（如果它是值类型）或该类型上的任何构造函数之前运行。 “何时”部分与实现这一目标所需要的一样重要; 现在，所有这些地方都需要在生成的机器代码中进行显式延迟初始化检查！

<!-- 
The `beforefieldinit` relaxation is weaker.  It guarantees the initializer will run sometime before actually accessing
a static field on that type.  This gives the compiler a lot of leeway in deciding on this placement.  Thankfully the
C# compiler will pick `beforefieldinit` automatically for you should you stick to using field initializers only.  Most
people don't realize the incredible cost of choosing instead to use a static constructor, however, especially for value
types where suddenly all method calls now incur initialization guards.  It's just the difference between: 
-->
beforefieldinit放松较弱。 它保证初始化程序将在实际访问该类型的静态字段之前运行一段时间。 这为编译器在决定此位置时提供了很大的余地。 值得庆幸的是，如果你只坚持使用字段初始化器，C＃编译器会自动选择beforefieldinit。 然而，大多数人并没有意识到选择使用静态构造函数的不可思议的成本，特别是对于突然所有方法调用现在都会产生初始化保护的值类型。 这只是区别：

    struct S {
        static int Field = 42;
    } 


<!-- 
and: 
-->
以及：

    struct S {
        static int Field;
        static S() {
            Field = 42;
        }
    } 


<!-- 
Now imagine the struct has a property: 
-->
现在想象结构有一个属性：

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
如果S没有静态初始化程序，或者使用beforefieldinit（在上面的字段初始化程序示例中由C＃自动注入），这是Property的机器代码：

<!-- 
    ; The struct is one word; move its value into EAX, and return it:
    8B C2                mov         eax,edx
    C3                   ret 
-->
    ; 结构是一个词; 将其值移入EAX，并将其返回：
    8B C2                mov         eax,edx
    C3                   ret 

<!-- 
And here's what happens if you add a class constructor: 
-->
如果添加类构造函数，会发生以下情况：

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
    ; 大到足以得到一个框架：
    56                   push        rsi
    48 83 EC 20          sub         rsp,20h
    ; 将字段加载到ESI：
    8B F2                mov         esi,edx
    ; 加载cctor的初始化状态：
    48 8D 0D 02 D6 FF FF lea         rcx,[1560h]
    48 8B 09             mov         rcx,qword ptr [rcx]
    BA 03 00 00 00       mov         edx,3
    ; 调用条件初始化助手：
    E8 DD E0 FF FF       call        2048
    ; 将字段从ESI移动到EAX，然后返回：
    8B C6                mov         eax,esi
    48 83 C4 20          add         rsp,20h
    5E                   pop         rsi 

<!-- 
On every property access! 
-->
在每个属性访问！

<!-- 
Of course, all static members still incur these checks, even if `beforefieldinit` is applied. 
-->
当然，即使应用了`beforefieldinit`，所有静态成员仍会产生这些检查。

<!-- 
Although C++ doesn't suffer this same problem, it does have mind-bending [initialization ordering semantics](
http://en.cppreference.com/w/cpp/language/initialization).  And, like C# statics, C++11 introduced thread-safe
initialization, by way of the ["magic statics" feature](
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2660.htm). 
-->
尽管C ++没有遇到同样的问题，但它确实具有令人费解的初始化排序语义。 
而且，就像C＃静态一样，C ++ 11通过“魔法静态”功能引入了线程安全初始化。

<!-- 
We virtually eliminated this entire mess in Midori. 
-->
我们几乎消除了Midori的整个混乱局面。

<!-- 
I mentioned offhandedly earlier that Midori had no mutable statics.  More accurately, we extended the notion of `const`
to cover any kind of object.  This meant that static values were evaluated at compile-time, written to the readonly
segment of the resulting binary image, and shared across all processes.  More importantly for code quality, all runtime
initialization checks were removed, and all static accesses simply replaced with a constant address. 
-->
我之前提到过，Midori没有可变的静力学。 更准确地说，我们扩展了const的概念以涵盖任何类型的对象。 这意味着静态值在编译时进行评估，写入生成的二进制映像的只读段，并在所有进程中共享。 更重要的是，对于代码质量，删除了所有运行时初始化检查，并且所有静态访问仅用常量地址替换。

<!-- 
There were still mutable statics at the core of the system -- in the kernel, for example -- but these did not make their
way up into user code.  And because they were few and far between, we did not rely on the classical C#-style lazy
initialization checks for them.  They were manually initialized on system startup. 
-->
在系统的核心仍然存在可变的静态 - 例如在内核中 - 但这些并没有进入用户代码。 因为它们很少，所以我们不依赖于它们的经典C＃式延迟初始化检查。 它们是在系统启动时手动初始化的。

<!-- 
As I said earlier, a 10% reduction in code size, and lots of speed improvements.  It's hard to know exactly how much
saved this was than a standard C# program because by the time we made the change, developers were well aware of the
problems and liberally applied our `[BeforeFieldInit]` attribute all over their types, to avoid some of the overheads.
So the 10% number is actually a lower bound on the savings we realized throughout this journey. 
-->
正如我之前所说，代码大小减少了10％，并且速度提升了很多。 很难确切知道这比标准C＃程序节省了多少，因为当我们进行更改时，开发人员很清楚这些问题，并且在他们的类型中自由地应用了我们的[BeforeFieldInit]属性，以避免一些开销。 因此，10％的数字实际上是我们在整个旅程中实现的节省的下限。

<!-- 
## Async model 
-->
## Async模型

<!-- 
I already wrote a lot about [our async model](http://joeduffyblog.com/2015/11/19/asynchronous-everything/).  I won't
rehash all of that here.  I will reiterate one point: the compiler was key to making linked stacks work. 
-->
我已经写了很多关于异步模型的文章。 我不会在这里重复所有这些。 我将重申一点：编译器是使链接栈工作的关键。

<!-- 
In a linked stacks model, the compiler needs to insert probes into the code that check for available stack space.  In
the event there isn't enough to perform some operation -- make a function call, dynamically allocate on the stack, etc.
-- the compiler needs to arrange for a new link to get appended, and to switch to it.  Mostly this amounts to some
range checking, a conditional call to a runtime function, and patching up `RSP`.  A probe looked something like: 
-->
在链接堆栈模型中，编译器需要将探针插入到检查可用堆栈空间的代码中。 如果没有足够的时间来执行某些操作 - 进行函数调用，在堆栈上动态分配等 - 编译器需要安排新的链接以进行追加，并切换到它。 大多数情况下，这相当于一些范围检查，对运行时函数的条件调用以及修补RSP。 探针看起来像：

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
    ; 检查堆栈空间量：
        lea     rax, [rsp-250h]
        cmp     rax, qword ptr gs:[0]
        ja      prolog
    ; 如果堆栈不足，请链接新段：
        mov     eax, 10029h
        call    ?g_LinkNewStackTrampoline
    prolog:
    ; 真正的代码在这里...

<!-- 
Needless to say, you want to probe as little as possible, for two reasons.  First, they incur runtime expense.  Second,
they chew up code size.  There are a few techniques we used to eliminate probes. 
-->
不用说，出于两个原因，您希望尽可能少地进行探测。 首先，它们会产生运行时费用。 其次，他们咀嚼代码大小。 我们使用一些技术来消除探针。

<!-- 
The compiler of course knew how to compute stack usage of functions.  As a result, it could be smart about the amount of
memory to probe for.  We incorporated this knowledge into our global analyzer.  We could coalesce checks after doing
code motion and inlining.  We hoisted checks out of loops.  For the most part, we optimized for eliminating checks,
sometimes at the expense of using a little more stack. 
-->
编译器当然知道如何计算函数的堆栈使用。 
因此，探测的内存量可能很明智。 
我们将这些知识融入我们的全球分析仪。 
我们可以在执行代码运动和内联后合并检查。 
我们提出了循环检查。 
在大多数情况下，我们优化了消除检查，
有时以牺牲使用更多堆栈为代价。

<!-- 
The most effective technique we used to eliminate probes was to run synchronous code on a classical stack, and to teach
our compiler to elide probes altogether for them.  This took advantage of our understanding of async in the type system.
Switching between the classical stack and back again again amounted to twiddling `RSP`: 
-->
我们用来消除探测器的最有效的技术是在经典堆栈上运行同步代码，并教我们的编译器为它们完全省略探测器。 这利用了我们对类型系统中异步的理解。 经典堆栈之间的切换再次回到了RSP：

<!-- 
    ; Switch to the classical stack:
    move    rsp, qword ptr gs:[10h]
    sub     rsp, 20h

    ; Do some work (like interop w/ native C/C++ code)...

    ; Now switch back:
    lea     rsp, [rbp-50h] 
-->
    ; 切换到经典堆栈：
    move    rsp, qword ptr gs:[10h]
    sub     rsp, 20h

    ; 做一些工作（如interop w / native C / C ++代码）...

    ; 现在切换回来：
    lea     rsp, [rbp-50h] 

<!-- 
I know Go abandoned linked stacks because of these switches.  At first they were pretty bad for us, however after about
a man year or two of effort, the switching time faded away into the sub-0.5% noise. 
-->
我知道由于这些开关，Go放弃了链接堆栈。 起初它们对我们来说非常糟糕，但经过大约一年或两年的努力，切换时间逐渐消失到低于0.5％的噪音。

<!-- 
## Memory ordering model 
-->
## 内存顺序模型

<!-- 
Midori's stance on [safe concurrency](http://joeduffyblog.com/2015/11/03/a-tale-of-three-safeties/) had truly one
amazing benefit: you get a [sequentially consistent](https://en.wikipedia.org/wiki/Sequential_consistency) memory
ordering model *for free*.  You may wish to read that again.  Free! 
-->
Midori对安全并发的立场确实有一个惊人的好处：您可以免费获得顺序一致的内存排序模型。 您可能希望再次阅读。 自由！

<!-- 
Why is this so?  First, Midori's [process model](http://joeduffyblog.com/2015/11/19/asynchronous-everything/) ensured
single-threaded execution by default.  Second, any fine-grained parallelism inside of a process was governed by a finite
number of APIs, all of which were race-free.  The lack of races meant we could inject a fence at fork and join points,
selectively, without a developer needing to care or know. 
-->
为什么会这样？ 首先，Midori的流程模型默认确保单线程执行。 
其次，进程内部的任何细粒度并行都由有限数量的API控制，所有这些都是无竞争的。 
缺少比赛意味着我们可以选择性地在分叉处注入围栏并加入点，而无需开发人员需要关心或了解。

<!-- 
Obviously this had incredible benefits to developer productivity.  The fact that Midori programmers never got bitten by
memory reordering problems was certainly one of my proudest outcomes of the project. 
-->
显然，这对开发人员的工作效率有着不可思议的好处。 
Midori程序员从未被记忆重新排序问题所困扰，这无疑是该项目最值得骄傲的结果之一。

<!-- 
But it also meant the compiler was free to make [more aggressive code motion optimizations](
https://www.cs.princeton.edu/courses/archive/fall10/cos597C/docs/memory-models.pdf), without any sacrifices to this
highly productive programming model.  In other words, we got the best of both worlds. 
-->
但这也意味着编译器可以自由地进行更积极的代码运动优化，而不会牺牲这种高效的编程模型。 换句话说，我们两全其美。

<!-- 
A select few kernel developers had to think about the memory ordering model of the underlying machine.  These were the
people implementing the async model itself.  For that, we eliminated C#'s notion of `volatile` -- which is [utterly
broken anyway](http://joeduffyblog.com/2010/12/04/sayonara-volatile/) -- in favor of something more like C++
[`atomic`s](http://en.cppreference.com/w/cpp/atomic).  That model is quite nice for two reasons.  First, what kind of
fence you need is explicit for every read and write, where it actually matters.  (ences affect the uses of a variable,
not its declaration.  Second, the explicit model tells the compiler more information about what optimizations can or
cannot take place, again at a specific uses, where it matters most. 
-->
少数内核开发人员不得不考虑底层机器的内存排序模型。 这些是实现异步模型本身的人。 为此，我们消除了C＃的易变性概念 - 无论如何都完全被破坏 - 支持更像C ++原子的东西。 这个模型非常好，有两个原因。 首先，你需要什么样的围栏是明确的每一次读写，它实际上是重要的。 （这会影响变量的使用，而不是它的声明。其次，显式模型告诉编译器有关哪些优化可以或不可以进行的更多信息，再次在特定用途中，它最重要。

<!-- 
## Error model 
-->
## 错误模型

<!-- 
Our error model journey was a long one and will be the topic of a future post.  In a nutshell, however, we experimented
with two ends of the spectrum -- exceptions and return codes -- and lots of points in betweeen. 
-->
我们的错误模型之旅很长，并且将成为未来帖子的主题。 然而，简而言之，我们试验了频谱的两个端点 - 异常和返回代码 - 以及其中的许多点。

<!-- 
Here is what we found from a code quality perspective. 
-->
以下是我们从代码质量角度发现的内容。

<!-- 
Return codes are nice because the type system tells you an error can happen.  A developer is thus forced to deal with
them (provided they don't ignore return values).  Return codes are also simple, and require far less "runtime magic"
than exceptions or related mechanisms like setjmp/longjmp.  So, lots to like here. 
-->
返回代码很好，因为类型系统会告诉您可能发生错误。 因此开发人员被迫处理它们（前提是它们不忽略返回值）。 返回代码也很简单，并且需要比异常或相关机制（如setjmp / longjmp）少得多的“运行时魔术”。 所以，很喜欢这里。

<!-- 
From a code quality persective, however, return codes suck.  They force you to execute instructions in hot paths that
wouldn't have otherwise been executed, including when errors aren't even happening.  You need to return a value from
your function -- occupying register and/or stack space -- and callers need to perform branches to check the results.
Granted, we hope that these are predicted correctly, but the reality is, you're just doing more work. 
-->
然而，从代码质量的持久性来看，返回代码很糟糕。 它们会强制您在热路径中执行本来不会执行的指令，包括甚至不会发生错误。 您需要从函数返回一个值 - 占用寄存器和/或堆栈空间 - 调用者需要执行分支来检查结果。 当然，我们希望这些是正确预测的，但事实是，你只是在做更多的工作。

<!-- 
Untyped exceptions suck when you're trying to build a reliable system.  Operating systems need to be reliable.  Not
knowing that there's a hidden control flow path when you're calling a function is, quite simply, unacceptable.  They
also require heavier weight runtime support to unwind stacks, search for handlers, and so on.  It's also a real pain in
the arse to model exceptional control flow in the compiler.  (If you don't believe me, just read through [this mail
exchange](http://lists.llvm.org/pipermail/llvm-dev/2015-May/085843.html)).  So, lots to hate here. 
-->
当您尝试构建可靠的系统时，无类型的异常会很糟糕。 操作系统需要可靠。 当你调用一个函数时，不知道隐藏的控制流路径是非常简单的，是不可接受的。 它们还需要更大的权重运行时支持来展开堆栈，搜索处理程序等等。 在编译器中对异常控制流进行建模也是一个真正的痛苦。 （如果你不相信我，只需阅读这个邮件交换）。 所以，很多人讨厌这里。

<!-- 
Typed exceptions -- I got used to not saying checked exceptions for fear of hitting Java nerves -- address some of these
shortcomings, but come with their own challenges.  Again, I'll save detailed analysis for my future post. 
-->
类型化的异常 - 我习惯于不会因为害怕遇到Java神经而不检查已检查的异常 - 解决其中的一些缺点，但遇到了他们自己的挑战。 再次，我将保存我未来帖子的详细分析。

<!-- 
From a code quality perspective, exceptions can be nice.  First, you can organize code segments so that the "cold"
handlers aren't dirtying your ICACHE on successful pathways.  Second, you don't need to perform any extra work during
the normal calling convention.  There's no wrapping of values -- so no extra register or stack pressure -- and there's
no branching in callers.  There can be some downsides to exceptions, however.  In an untyped model, you must assume
every function can throw, which obviously inhibits your ability to move code around. 
-->
从代码质量的角度来看，异常可能很好。 首先，您可以组织代码段，以便“冷”处理程序不会在成功路径上弄脏您的ICACHE。 其次，在正常调用约定期间，您不需要执行任何额外的工作。 没有值的包装 - 因此没有额外的寄存器或堆栈压力 - 并且在调用者中没有分支。 但是，异常可能会有一些缺点。 在非类型化模型中，您必须假设每个函数都可以抛出，这显然会抑制您移动代码的能力。

<!-- 
Our model ended up being a hybrid of two things: 
-->
我们的模型最终成为两件事的混合体：

<!-- 
* [Fail-fast](http://joeduffyblog.com/2014/10/13/if-youre-going-to-fail-do-it-fast/) for programming bugs.
* Typed exceptions for dynamically recoverable errors. 
-->
* 编程错误的速度快；
* 动态可恢复错误的类型异常。

<!-- 
I'd say the ratio of fail-fast to typed exceptions usage ended up being 10:1.  Exceptions were generally used for I/O
and things that dealt with user data, like the shell and parsers.  Contracts were the biggest source of fail-fast. 
-->
我会说，故障快速与类型异常使用的比率最终为10：1。 
异常通常用于I / O和处理用户数据的事情，如shell和解析器。 合同是快速失败的最大来源。

<!-- 
The result was the best possible configuration of the above code quality attributes: 
-->
结果是上述代码质量属性的最佳配置：
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
A nice accident of our model was that we could have compiled it with either return codes or exceptions.  Thanks to this,
we actually did the experiment, to see what the impact was to our system's size and speed.  The exceptions-based system
ended up being roughly 7% smaller and 4% faster on some key benchmarks. 
-->
我们模型的一个不错的意外是我们可以使用返回代码或异常编译它。 多亏了这一点，我们实际上做了实验，看看对我们系统的大小和速度有什么影响。 基于例外的系统最终在某些关键基准测试中缩小了约7％，速度提高了4％。

<!-- 
At the end, what we ended up with was the most robust error model I've ever used, and certainly the most performant one. 
-->
最后，我们最终得到的是我曾经使用过的最强大的错误模型，当然也是性能最好的模型。

<!-- 
## Contracts 
-->
## 合约

<!-- 
As implied above, Midori's programming language had first class contracts: 
-->
如上所述，Midori的编程语言有一流的合约：

    void Push(T element)
        requires element != null
        ensures this.Count == old.Count + 1
    {
            ...
    } 


<!-- 
The model was simple: 
-->
模型很简单：

<!-- 
* By default, all contracts are checked at runtime.
* The compiler was free to prove contracts false, and issue compile-time errors.
* The compiler was free to prove contracts true, and remove these runtime checks. 
-->
* 默认情况下，在运行时检查所有合同。
* 编译器可以自由地证明合同是错误的，并发出编译时错误。
* 编译器可以自由地证明契约是真的，并删除这些运行时检查。

<!-- 
We had conditional compilation modes, however I will skip these for now.  Look for an upcoming post on our language. 
-->
我们有条件编译模式，但是我现在将跳过它们。 寻找关于我们语言的即将发布的帖子。

<!-- 
In the early days, we experimented with contract analyzers like MSR's [Clousot](
http://research.microsoft.com/pubs/138696/Main.pdf), to prove contracts.  For compile-time reasons, however, we had to
abandon this approach.  It turns out compilers are already very good at doing simple constraint solving and propagation.
So eventually we just modeled contracts as facts that the compiler knew about, and let it insert the checks wherever
necessary. 
-->
在早期，我们尝试使用像MSR的Clousot这样的合同分析器来证明合同。 然而，出于编译时的原因，我们不得不放弃这种方法。 事实证明，编译器已经非常擅长简单的约束求解和传播。 因此，最终我们只是将合同建模为编译器知道的事实，并让它在必要时插入检查。

<!-- 
For example, the loop optimizer complete with range information above can already leverage checks like this: 
-->
例如，完成上述范围信息的循环优化器已经可以利用这样的检查：


    void M(int[] array, int index) {
        if (index >= 0 && index < array.Length) {
            int v = array[index];
            ...
        }
    } 


<!-- 
to eliminate the redundant bounds check inside the guarded if statement.  So why not also do the same thing here? 
-->
消除在guarded if语句中的冗余边界检查。 那么为什么不在这里做同样的事情呢？


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
然而，当涉及单独的编译时，这些事实是特殊的。 契约是方法签名的一部分，我们的系统确保了正确的子类型替换，让编译器在单独编译的边界上进行更积极的优化。 它可以更快地完成这些优化，因为它们不依赖于全局分析。

<!-- 
## Objects and allocation 
-->
## 对象和分配

<!-- 
In a future post, I'll describe in great detail our war with the garbage collector.  One technique that helped us win,
however, was to aggressively reduce the size and quantity of objects a well-behaving program allocated on the heap.
This helped with overall working set and hence made programs smaller and faster. 
-->
在以后的文章中，我将详细描述我们与垃圾收集器的战争。 
然而，一种帮助我们获胜的技术是积极地减少在堆上分配的良好行为程序的对象的大小和数量。
这有助于整体工作集，从而使程序更小，更快。

<!-- 
The first technique here was to shrink object sizes. 
-->
这里的第一种技术是缩小对象大小。

<!-- 
In C# and most Java VMs, objects have headers.  A standard size is a single word, that is, 4 bytes on 32-bit
architectures and 8 bytes on 64-bit.  This is in addition to the vtable pointer.  It's typically used by the GC to mark
objects and, in .NET, is used for random stuff, like COM interop, locking, memozation of hash codes, and more.  (Even
[the source code calls it the "kitchen sink"](https://github.com/dotnet/coreclr/blob/master/src/vm/syncblk.h#L29).) 
-->
在C＃和大多数Java VM中，对象都有标头。 
标准大小是单个字，即32位体系结构上的4个字节和64位上的8个字节。 
这是vtable指针的补充。 它通常用于标记对象，在.NET中用于随机内容，如COM互操作，锁定，哈希码的记忆等等。 （甚至源代码称它为“厨房水槽”。）

<!-- 
Well, we ditched both. 
-->
好吧，我们两个都抛弃了。

<!-- 
We didn't have COM interop.  There was no unsafe free-threading so there was no locking (and [locking on random objects
is a bad idea anyway](http://joeduffyblog.com/2006/10/26/concurrency-and-the-impact-on-reusable-libraries/)).  Our
`Object` didn't define a `GetHashCode`.  Etc.  This saved a word per object with no discernable loss in the programming
model (actually, to the contrary, it was improved), which is nothing to shake a stick at. 
-->
我们没有COM互操作。 没有不安全的自由线程，所以没有锁定（无论如何锁定随机对象是一个坏主意）。 我们的对象没有定义GetHashCode。 等等。这为每个对象保存了一个单词，在编程模型中没有明显的损失（实际上，恰恰相反，它得到了改进），这没有什么可以动摇的。

<!-- 
At that point, the only overhead per object was the vtable pointer.  For structs, of course there wasn't one (unless
they were boxed).  And we did our best to eliminate all of them.  Sadly, due to RTTI, it was difficult to be aggressive.
I think this is another area where I'd go back and entirely upend the C# type system, to follow a more C, C++, or even
maybe Go-like, model.  In the end, however, I think we did get to be fairly competitive with your average C++ program. 
-->
此时，每个对象的唯一开销是vtable指针。 对于结构，当然没有结构（除非它们是盒装的）。 我们尽力消除所有这些。 可悲的是，由于RTTI，很难进取。 我认为这是另一个我会回去并完全颠覆C＃类型系统的领域，以跟随更多的C，C ++甚至是类似Go的模型。 但是，最后，我认为我们确实与您的普通C ++程序相当竞争。

<!-- 
There were padding challenges.  Switching the `struct` layout from C#'s current default of `sequential`, to our
preferred default of `auto`, certainly helped.  As did optimizations like the well-known C++ [empty base optimization](
http://en.cppreference.com/w/cpp/language/ebo). 
-->
有填充挑战。 将结构布局从C＃的当前默认顺序切换到我们首选的默认auto，肯定有帮助。 正如着名的C ++空基优化一样优化。

<!-- 
We also did aggressive escape analysis in order to more efficiently allocate objects.  If an object was found to be
stack-confined, it was allocated on the stack instead of the heap.  Our initial implementation of this moved somewhere
in the neighborhood of 10% static allocations from the heap to the stack, and let us be far more aggressive about
pruning back the size of objects, eliminating vtable pointers and entire unused fields.  Given how conservative this
analysis had to be, I was pretty happy with these results. 
-->
我们还进行了积极的逃逸分析，以便更有效地分配对象。 如果发现一个对象是堆栈限制的，则它被分配在堆栈而不是堆上。 我们的初始实现在从堆到堆栈的10％静态分配附近移动，让我们更加积极地修剪对象的大小，消除vtable指针和整个未使用的字段。 考虑到这种分析是多么保守，我对这些结果非常满意。

<!-- 
We offered a hybrid between C++ references and Rust borrowing if developers wanted to give the compiler a hint while at
the same time semantically enforcing some level of containment.  For example, say I wanted to allocate a little array to
share with a callee, but know for sure the callee does not remember a reference to it.  This was as simple as saying: 
-->
如果开发人员想要给编译器一个提示，同时在语义上强制执行某种程度的包含，我们提供了C ++引用和Rust借用之间的混合。 例如，假设我想分配一个小数组与被调用者共享，但确定被调用者不记得对它的引用。 这就像说：

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
        ... 保证`a`不会逃脱 ...
    } 

<!-- 
The compiler used the `int[]&` information to stack allocate the array and, often, eliminate the vtable for it
entirely.  Coupled with the sophisticated elimination of bounds checking, this gave us something far closer to C
performance. 
-->
编译器使用int []和信息来堆栈分配数组，并且通常完全消除vtable。 再加上边界检查的复杂消除，这给了我们更接近C性能的东西。

<!-- 
Lambdas/delegates in our system were also structs, so did not require heap allocation.  The captured display frame was
subject to all of the above, so frequently we could stack allocate them.  As a result, the following code was heap
allocation-free; in fact, thanks to some early optimizations, if the callee was inlined, it ran as though the actual
lambda body was merely expanded as a sequence of instructions, with no call over head either! 
-->
我们系统中的Lambdas / delegates也是结构体，因此不需要堆分配。 捕获的显示框架受上述所有操作的影响，因此我们经常可以堆叠分配它们。 结果，以下代码是堆分配免费的; 事实上，由于一些早期的优化，如果被调用者被内联，它就好像实际的lambda体只是作为一系列指令而扩展，没有任何调用！

<!-- 
    void Caller() {
        Callee(() => ... do something ... );
    }

    void Callee(Action& callback) {
        callback();
    } 
-->
    void Caller() {
        Callee(() => ... 完成某些事情 ... );
    }

    void Callee(Action& callback) {
        callback();
    } 

<!-- 
In my opinion, this really was the killer use case for the borrowing system.  Developers avoided lambda-based APIs in
the early days before we had this feature for fear of allocations and inefficiency.  After doing this feature, on the
other hand, a vibrant ecosystem of expressive lambda-based APIs flourished. 
-->
在我看来，这确实是借款系统的杀手级用例。 由于担心分配和效率低下，开发人员在我们拥有此功能之前的早期就避免使用基于lambda的API。 另一方面，在完成此功能之后，充满活力的基于lambda的API的生态系统蓬勃发展。

<!-- 
# Throughput 
-->
# 吞吐量

<!-- 
All of the above have to do with code quality; that is, the size and speed of the resulting code.  Another important
dimension of compiler performance, however, is *throughput*; that is, how quickly you can compile the code.  Here too
a language like C# comes with some of its own challenges. 
-->
以上所有都与代码质量有关; 也就是说，结果代码的大小和速度。 然而，编译器性能的另一个重要方面是吞吐量; 也就是说，编译代码的速度有多快。 像C＃这样的语言也带来了一些自己的挑战。

<!-- 
The biggest challenge we encountered has less to do with the inherently safe nature of a language, and more to do with
one very powerful feature: parametric polymorphism.  Or, said less pretentiously, generics. 
-->
我们遇到的最大挑战与语言固有的安全性无关，更多的是与一个非常强大的功能有关：参数多态。 或者说，不那么自命不凡，仿制药。

<!-- 
I already mentioned earlier that generics are just a convenient copy-and-paste mechanism.  And I mentioned some
challenges this poses for code size.  It also poses a problem for throughput, however.  If a `List<T>` instantiation
creates 28 types, each with its own handful of methods, that's just more code for the compiler to deal with.  Separate
compilation helps, however as also noted earlier, generics often flow across module boundaries.  As a result, there's
likely to be a non-trivial impact to compile time.  Indeed, there was. 
-->
我之前已经提到过，泛型只是一种方便的复制粘贴机制。 我提到了这对代码大小带来的一些挑战。 然而，它也对吞吐量造成问题。 如果List <T>实例化创建了28种类型，每种类型都有自己的少数几种方法，那么这只是编译器需要处理的更多代码。 单独的编译有所帮助，但是如前所述，泛型通常跨越模块边界。 因此，编译时间可能会产生非平凡的影响。 的确，有。

<!-- 
In fact, this is not very different from where most C++ compilers spend the bulk of their time.  In C++, it's templates.
More modern C++ code-bases have similar problems, due to heavy use of templated abstractions, like STL, smart pointers,
and the like.  Many C++ code-bases are still just "C with classes" and suffer this problem less. 
-->
实际上，这与大多数C ++编译器花费大量时间的情况没有太大差别。 在C ++中，它是模板。 由于大量使用模板化抽象（如STL，智能指针等），更现代的C ++代码库具有类似的问题。 许多C ++代码库仍然只是“带有类的C”并且减少了这个问题。

<!-- 
As I mentioned earlier, I wish we had banished RTTI.  That would have lessened the generics problem.  But I would guess
generics still would have remained our biggest throughput challenge at the end of the day. 
-->
正如我之前提到的，我希望我们放弃了RTTI。 这会减少仿制药问题。 但我认为仿制品仍然是我们在一天结束时最大的吞吐量挑战。

<!-- 
The funny thing -- in a not-so-funny kind of way -- is that you can try to do analysis to prune the set of generics and,
though it is effective, this analysis takes time.  The very thing you're trying to save. 
-->
有趣的是 - 以一种不那么有趣的方式 - 是你可以尝试进行分析以修剪一组泛型，虽然它是有效的，但这种分析需要时间。 你要保存的东西。

<!-- 
A metric we got in the habit of tracking was how much slower AOT compiling a program was than simply C# compiling it.
This was a totally unfair comparison, because the C# compiler just needs to lower to MSIL whereas an AOT compler needs
to produce machine code.  It'd have been fairer to compare AOT compiling to JIT compiling.  But no matter, doing a great
job on throughput is especially important for a C# audience.  The expectation of productivity was quite high.  This was
therefore the key metric we felt customers would judge us on, and so we laser-focused on it. 
-->
我们习惯跟踪的一个指标是AOT编译程序的速度比C＃编译简单得多。 这是一个完全不公平的比较，因为C＃编译器只需要降低到MSIL，而AOT编译器需要生成机器代码。 将AOT编译与JIT编译进行比较是比较公平的。 但无论如何，在吞吐量方面做得很好对于C＃受众尤其重要。 对生产力的期望很高。 因此，这是我们认为客户会评价我们的关键指标，因此我们专注于它。

<!-- 
In the early days, the number was ridiculously bad.  I remember it being 40x slower.  After about a year and half with
intense focus we got it down to *3x for debug builds* and *5x for optimized builds*.  I was very happy with this! 
-->
在早期，这个数字非常糟糕。 我记得它慢了40倍。 经过大约一年半的重点关注，我们将调试版本降低到3倍，优化版本降低5倍。 我很高兴这个！

<!-- 
There was no one secret to achieving this.  Mostly it had to do with just making the compiler faster like you would any
program.  Since we built the compiler using Midori's toolchain, however -- and compiled it using itself -- often this
was done by first making Midori better, which then made the the compiler faster.  It was a nice virtuous loop.  We had
real problems with string allocations which informed what to do with strings in our programming model.  We found crazy
generics instantiation closures which forced us to eliminate them and build tools to help find them proactively.  Etc. 
-->
实现这一点没有任何秘密。 大多数情况下，它只需要像任何程序一样使编译器更快。 因为我们使用Midori的工具链构建了编译器 - 然后使用它自己编译 - 通常这是通过首先使Midori更好，然后使编译器更快来完成的。 这是一个很好的良性循环。 我们遇到了字符串分配的实际问题，这些问题告诉我们在编程模型中如何处理字符串。 我们发现疯狂的泛型实例化闭包迫使我们消除它们并构建工具来帮助主动找到它们。 等等。

<!-- 
# Culture 
-->
# 文化

<!-- 
A final word before wrapping up.  Culture was the most important aspect of what we did.  Without the culture, such an
amazing team wouldn't have self-selected, and wouldn't have relentlessly pursued all of the above achievements.  I'll
devote an entire post to this.  However, in the context of compilers, two things helped: 
-->
结束前的最后一句话。 文化是我们所做的最重要的方面。 如果没有文化，这样一支出色的团队就不会有自我选择，也不会无情地追求上述所有成就。 我会在这里投入一整个帖子。 但是，在编译器的上下文中，有两件事有所帮助：

<!-- 
1. We measured everything in the lab.  "If it's not in the lab, it's dead to me."
2. We reviewed progress early and often.  Even in areas where no progress was made.  We were habitually self-critical. 
-->
1. 我们在实验室测量了一切。 “如果它不在实验室里，那对我来说已经死了。”
2. 我们提前和经常审查进展情况。 即使在没有取得进展的地区也是如此。 我们习惯性地自我批评。
3. 
<!-- 
Every sprint, we had a so-called "CQ Review" (where CQ stands for "code quality").  The compiler team prepared for a few
days, by reviewing every benchmark -- ranging from the lowest of microbenchmarks to compiling and booting all of Windows
-- and investigating any changes.  All expected wins were confirmed (we called this "confirming your kill"), any
unexpected regressions were root cause analyzed (and bugs filed), and any wins that didn't materialize were also
analyzed and reported on so that we could learn from it.  We even stared at numbers that didn't change, and asked
ourselves, why didn't they change.  Was it expected?  Do we feel bad about it and, if so, how will we change next
sprint?  We reviewed our competitors' latest compiler drops and monitored their rate of change.  And so on. 
-->
每个sprint，我们都有一个所谓的“CQ Review”（其中CQ代表“代码质量”）。 编译团队准备了几天，通过审查每个基准测试 - 从最低的微基准测试到编译和启动所有Windows  - 并调查任何变化。 所有预期的胜利都得到了确认（我们称之为“确认你的杀人”），任何意外的回归都是根本原因分析（并提交了错误），任何未实现的胜利也会被分析和报告，以便我们可以从中学习。 我们甚至盯着那些没有改变的数字，并问自己，为什么他们没有改变。 这是预期的吗？ 我们是否感觉不好，如果是这样，我们将如何改变下一个冲刺？ 我们审查了竞争对手的最新编译器下降并监控其变化率。 等等。

<!-- 
This process was enormously healthy.  Everyone was encouraged to be self-critical.  This was not a "witch hunt"; it was
an opportunity to learn as a team how to do better at achieving our goals. 
-->
这个过程非常健康。 每个人都被鼓励自我批评。 这不是“猎巫”; 这是一个学习团队如何更好地实现目标的机会。

<!-- 
Post-Midori, I have kept this process.  I've been surprised at how contentious this can be with some folks.  They get
threatened and worry their lack of progress makes them look bad.  They use "the numbers aren't changing because that's
not our focus right now" as justification for getting out of the rhythm.  In my experience, so long as the code is
changing, the numbers are changing.  It's best to keep your eye on them lest you get caught with your pants around your
ankles many months later when it suddenly matters most.  The discipline and constant drumbeat are the most important
parts of these reviews, so skipping even just one can be detrimental, and hence was verboten. 
-->
后Midori，我保持这个过程。 我对一些人的争议感到惊讶。 他们受到威胁并担心他们缺乏进展会使他们看起来很糟糕。 他们使用“数字不变，因为现在不是我们关注的焦点”作为摆脱节奏的理由。 根据我的经验，只要代码发生变化，数字就会发生变化。 最好是密切注意它们，以免在几个月后突然最重要的时候，你的裤子被你的脚踝夹住。 纪律和持续的鼓声是这些评论中最重要的部分，因此即使只跳过一个也可能是有害的，因此是禁止的。

<!-- 
This process was as much our secret sauce as anything else was. 
-->
这个过程和其他任何东西一样都是我们的秘密调料。

<!-- 
# Wrapping Up 
-->
# 总结

<!-- 
Whew, that was a lot of ground to cover.  I hope at the very least it was interesting, and I hope for the incredible
team who built all of this that I did it at least a fraction of justice.  (I know I didn't.) 
-->
哇，这是很多要覆盖的地方。 我希望至少它很有趣，我希望建立所有这一切的令人难以置信的团队至少可以做到这一点。 （我知道我没有。）

<!-- 
This journey took us over a decade, particularly if you account for the fact that both Bartok and Phoenix had existed
for many years even before Midori formed.  Merely AOT compiling C#, and doing it well, would have netted us many of the
benefits above.  But to truly achieve the magical native-like performance, and indeed even exceed it in certain areas,
required some key "whole system" architectural bets.  I hope that some day we can deliver safety into the world at this
level of performance.  Given the state of security all-up in the industry, mankind seriously needs it. 
-->
这段旅程花了我们十多年的时间，特别是如果你考虑到巴托克和凤凰在Midori成立之前已存在多年的事实。 仅仅AOT编译C＃，并且做得很好，会让我们获得上述许多好处。 但要真正实现神奇的原生性能，甚至在某些领域甚至超过它，需要一些关键的“整个系统”建筑投注。 我希望有一天我们能够在这个性能水平上为世界提供安全保障。 鉴于该行业的安全状况，人类非常需要它。

<!-- 
I've now touched on our programming language enough that I need to go deep on it.  Tune in next time! 
-->
我现在已经触及了我们的编程语言，我需要深入研究它。 下次调整！