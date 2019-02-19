---
title: Midori博客系列翻译（1）——三类安全性的故事
date: 2018-10-24 13:02:00
tags: [操作系统, Midori, 翻译, 安全]
categories: 中文
---

<!-- 
[Midori](/2015/11/03/blogging-about-midori) was built on a foundation of three
kinds of safety: type, memory, and concurrency-safety.  These safeties eliminated
whole classes of bugs "by-construction" and delivered significant improvements in
areas like reliability, security, and developer productivity.  They also fundamentally
allowed us to depend on the type system in new and powerful ways, to deliver new
abstractions, perform novel compiler optimizations, and more.  As I look back,
the biggest contribution of our project was proof that an entire operating
system and its ecosystem of services, applications, and libraries could indeed
be written in safe code, without loss of performance, and with some quantum
leaps forward in several important dimensions. 
-->

[Midori](/2018/10/20/midori/0-blogging-about-midori/)建立在三类安全性的基础之上，这包括：
类型安全、内存安全和并发安全。 
它们“从构造”上消除了的各类错误，并在可靠性、安全性和开发人员生产力等方面取得了重大改进。 
另外，它们还从根本上允许我们以新颖强大的方式，依赖类型系统来提供新的抽象、执行最新的编译器优化等。 
回顾过去，我们项目的最大贡献就是证明了整个操作系统及其服务、应用程序和库的生态系统确实可以使用安全的代码编写，
同时不会损失性能，并且在数个重要维度上取得了一些重大突破。

<!-- 
First, let us define the three safeties, in foundational order: 
-->
首先，让我们按照基本顺序定义三类安全性：

<!-- 
* [Memory Safety](https://en.wikipedia.org/wiki/Memory_safety) prohibits access
  to invalid regions of memory.  Numerous flaws arise when memory safety is
  violated, including buffer overflow, use after free, and double frees. 
  Generally speaking, violation of memory safety is a critical error that can
  lead to exploits such as code injection. 
-->
* [内存安全](https://en.wikipedia.org/wiki/Memory_safety)禁止访问无效的内存区域。
  当破坏内存安全性时会产生多种缺陷，这包括缓冲区溢出，释放后使用（use after free，UAF）和双重释放（double free）等。
  一般来讲，违反内存安全性是严重的错误，可能导致代码注入等漏洞。

<!-- 
* [Type Safety](https://en.wikipedia.org/wiki/Type_safety) prohibits use of
  memory that is at odds with the type allocated within that memory.  Numerous
  flaws arise when type safety is violated, including type confusion, casting
  errors, and uninitialized variables.  Although generally less severe than
  memory safety violations, type safety violations can lead to exploits,
  particularly when exposing pathways to memory safety holes. 
-->
* [类型安全](https://en.wikipedia.org/wiki/Type_safety)禁止以与内存分配时类型不一致的类型方式访问该内存。
  当违背类型安全时也会导致多种缺陷，这包括类型混淆，数据类型转换错误和未初始化的变量等。 
  虽然通常不如内存安全严重，但类型安全被破坏也可能导致漏洞，特别是当间接导致内存安全漏洞时。

<!-- 
* [Concurrency Safety](https://en.wikipedia.org/wiki/Thread_safety) prohibits
  unsafe concurrent use of shared memory.  These concurrency hazards are widely
  known in the form of [data races](
  https://en.wikipedia.org/wiki/Race_condition), or read-write, write-read, and
  write-write hazards.   Generally speaking, if concurrency safety is violated,
  it can frequently lead to type, and therefore memory, safety being violated.
  These exploits are often quite subtle -- like tearing memory -- and we often
  said that concurrency vulnerabilities are the "next frontier" of exploitable
  security holes. 
-->
* [并发安全](https://en.wikipedia.org/wiki/Thread_safety)禁止以不安全的方式并发使用共享内存。
  这些并发冒险以[数据竞争](https://en.wikipedia.org/wiki/Race_condition)，或读后写、写后读和写后写冒险的形式广为人知。
  一般来说，如果违背了并发安全性，常常会导致类型安全，进而内存安全被破坏。 
  并发安全的漏洞通常非常微妙，例如内存撕裂（memory tearing）等，因此我们认为并发性漏洞是可利用安全漏洞的“下一个热点领域”。

<!-- 
Many approaches exist to establish one or more of these safeties, and/or
safeguard against violations. 
-->
存在多种方法来构建上述三种安全性中的一个或多个，以及防止被破坏的安全手段。

<!-- 
[Software fault isolation](http://www.cs.cmu.edu/~srini/15-829/readings/sfi.pdf)
establishes memory safety as a backstop against the most severe exploits.  This
comes at some runtime cost, although [proof carrying code](
https://en.wikipedia.org/wiki/Proof-carrying_code) can lessen it.  These
techniques don't deliver all the added benefits of type and concurrency safety. 
-->

[软件故障隔离（Software fault isolation）](http://www.cs.cmu.edu/~srini/15-829/readings/sfi.pdf)
构建内存安全，作为对抗最严重攻击的措施，但这种方式也带来了一些运行时开销。
尽管[携带证明的代码](https://en.wikipedia.org/wiki/Proof-carrying_code)可以减少开销，但这些技术也无法提供类型安全和并发安全。

<!-- 
Language-based safety, on the other hand, is done through a system of type
system rules and local checks (as opposed to global) that, inductively, ensure
certain operations do not occur, plus optional runtime checks (like array bounds
checking in the absence of a more capable [dependent type system](
https://en.wikipedia.org/wiki/Dependent_type)).  The benefits of this approach
are often a more productive approach to stopping safety holes because a developer
finds them while writing his or her code, rather than at runtime.  But if you can
trick the type system into permitting an illegal operation, you're screwed,
because there is no backstop to prevent hackers from violating memory safety in
order to running arbitrary code, for example. 
-->
另一方面，基于语言的安全，则是通过类型系统规则和局部检查（与全局相反的）体系来完成，
通过推导的方式，该体系可确保不发生违反安全性的操作，再加上可选的运行时检查（例如在缺乏更强大的[依赖类型系统](https://en.wikipedia.org/wiki/Dependent_type)时的数组边界检查）。 
这种方法的好处是它采取一种通常更有效的方法来阻止安全漏洞，
因为开发人员不是在软件运行时，而在编写代码时便可发现这些漏洞。 
但是，如果你采取欺骗的方式使得类型系统允许非法得操作，你就完蛋了，
因为没有后备措施可以阻止黑客们违反内存安全性以运行任意代码。

<!-- 
Multiple techniques are frequently used in conjunction with another, something
called "defense in depth," in order to deliver the best of all of these
techniques. 
-->
多种技术经常被结合一起使用，以获得这些技术的所有优点，这也被称为“深度防御”。

<!-- 
Good examples of the runtime approach to safety include [Google's C++ sanitizers](
https://github.com/google/sanitizers) and [Microsoft's "/guard" feature](
http://blogs.msdn.com/b/vcblog/archive/2014/12/08/visual-studio-2015-preview-work-in-progress-security-feature.aspx).
Good examples of the language approach include C#, Java, most functional
languages, Go, etc.  We can see some cracks already, however, since C# has the
`unsafe` keyword which permits unsafe regions that violate safety. 
-->
采用运行时安全的优秀案例包括[Google的C++ sanitizer](https://github.com/google/sanitizers)
和[微软的“/guard”功能](http://blogs.msdn.com/b/vcblog/archive/2014/12/08/visual-studio-2015-preview-work-in-progress-security-feature.aspx)。 
而采用基于语言的安全的不错的例子则包括C#、Java、大多数函数式语言和Go等。
但是，我们也看到这种方式的不完备，因为C#有“unsafe”关键字允许违反安全性的不安全区域的存在。

<!-- 
So, anyway, how do you build an operating system, whose central purpose is to
control hardware resources, buffers, services and applications running in
parallel, and so on, all of which are pretty damn unsafe things, using a safe
programming environment?  Great question. 
-->
那么，到底应该如何构建一个操作系统，其核心目的是，
使用安全的编程环境来控制并行运行的硬件资源、缓冲区、服务和应用程序等所有可能造成不安全后果的例程？ 
这是一个不错的问题。

<!-- The answer is surprisingly simple: layers. -->
答案非常简单：分层。

<!-- 
There was of course _some_ unsafe code in the system.  Each unsafe component was
responsible for "encapsulating" its unsafety.  This is easier said than done,
and was certainly the hardest part of the system to get right.  Which is why
this so-called [trusted computing base](
https://en.wikipedia.org/wiki/Trusted_computing_base) (TCB) always remained as
small as we could make it.  Nothing above the OS kernel and runtime was meant to
employ unsafe code, and very little above the microkernel did.  Yes, our OS
scheduler and memory manager was written in safe code.  And all application-
level and library code was most certainly 100% safe, like our entire web browser. 
-->
当然，系统中会有*一些*不安全的代码，而每个不安全组件都需要负责“安全封装”它自身的不安全性。 
这说起来容易做起来难，而且肯定是系统中最难实现的部分。 
这就是为什么所谓的[可信计算基（TCB）](https://en.wikipedia.org/wiki/Trusted_computing_base)
要始终保持尽可能小的原因。
因此，不安全代码不应存在于操作系统内核和运行时之上的任何部件中，而应存在于微内核之上的极少部分。 
没错，Midori的操作系统调度程序和内存管理器皆是用由安全代码编写而成的，
并且所有应用级和库代码也肯定是100%安全，就像我们的Web浏览器一样安全。

<!-- 
One interesting aspect of relying on type safety was that [your compiler](
https://en.wikipedia.org/wiki/Bartok_(compiler)) becomes part of your TCB.
Although our compiler was written in safe code, it emitted instructions for the
processor to execute.  The risk here can be remedied slightly by techniques like
proof-carrying code and [typed assembly language](
https://en.wikipedia.org/wiki/Typed_assembly_language) (TAL).  Added runtime
checks, a la software fault isolation, can also lessen some of this risk. 
-->
有趣的是，依靠类型安全的方法中，[编译器](http://t.cn/EVXZ53S)将成为TCB的一部分，因为虽然编译器是用安全代码编写的，但它仍需输出指令供处理器直接执行。 
但这里产生风险可以通过携带证明的代码和[类型汇编语言（TAL）](https://en.wikipedia.org/wiki/Typed_assembly_language)等技术稍作补救，
另外，添加运行时检查和软件故障隔离等方法，也可以减少部分风险。

<!-- 
A nice consequence of our approach was that the system was built upon itself.
This was a key principle we took to an extreme.  I covered it a bit [in a prior
article](http://joeduffyblog.com/2014/09/10/software-leadership-7-codevelopment-is-a-powerful-thing/).
But when you've got an OS kernel, filesystem, networking stack, device drivers,
UI and graphics stack, web browser, web server, multimedia stack, ..., and even
the compiler itself, all written in your safe programming model, you can be
pretty sure it will work for mostly anything you can throw at it. 
-->
我们的分层方法的一个很好的后果是系统构建在自己的基础之上，这将我们的关键原则发挥到了极致，而在我[前面的一篇文章](http://joeduffyblog.com/2014/09/10/software-leadership-7-codevelopment-is-a-powerful-thing/)中对此也进行了一些介绍。 
当你的操作系统内核、文件系统、网络栈、设备驱动程序、用户界面、图形堆栈、网页浏览器、网络服务器和多媒体堆栈……甚至编译器本身都是用你的设计的安全编程模型编写而成，那么可以非常肯定这种安全模型适用于你系统中的大部分的部件。

<!-- 
You may be wondering what all this safety cost.  Simply put, there are things
you can't do without pointer arithmetic, data races, and the like.  Much of
what we did went into minimizing these added costs.  And I'm happy to say, in the
end, we did end up with a competitive system.  Building the system on itself was
key to keeping us honest.  It turns out architectural decisions like no blocking
IO, lightweight processes, fine grained concurrency, asynchronous message
passing, and more, far outweighed the "minor" costs incurred by requiring safety
up and down the stack. 
-->
所以，你可能考虑所有这些安全性带来的开销有多大。
简单的说，系统中总会存在那些如果没有指针运算和数据竞争等不安全操作就无法完成的例程。
而我们所做的大部分工作都是为了尽量减少这些增加的开销。 
我可以很高兴地告诉你，我们最终得到了一个具有竞争力的系统，
在自身的基础上构建系统是保持这种竞争力的关键。
事实证明，诸如无阻塞I/O、轻量级进程、细粒度并发、异步消息传递等架构决策带来的好处，
远远超过了需要在全部堆栈保持安全性所带来的“较小”的开销。

<!-- 
For example, we did have certain types that were just buckets of bits.  But these
were just [PODs](https://en.wikipedia.org/wiki/Passive_data_structure).  This
allowed us to parse bits out of byte buffers -- and casting to and fro between
different wholly differnt "types" -- efficiently and without loss of safety.
We had a first class slicing type that permit us to form safe, checked windows
over buffers, and unify the way we accessed all memory in the system
([the slice type](https://github.com/joeduffy/slice.net) we're adding to .NET
was inspired by this). 
-->
例如，我们确实有某些类型只是存放数据的桶结构，但这些只是[被动数据结构（POD）](https://en.wikipedia.org/wiki/Passive_data_structure)。 
它们的存在使我们能够以高效且不会损失安全性的方式，从字节缓冲区中解析数据，
以及在完全不同的类型之间来回转换。 
例如，我们有作为一等公民的切片（slice）类型，它允许在缓冲区上形成安全和校检的访问窗口，
并形成统一访问所有系统内存的安全方式
（我们正在添加至.NET的[切片类型](https://github.com/joeduffy/slice.net)的灵感也来源于此）。

<!-- 
You might also wonder about the [RTTI](
https://en.wikipedia.org/wiki/Run-time_type_information) overheads required to
support type safety.  Well, thanks to PODs, and proper support for [
discriminated unions](https://en.wikipedia.org/wiki/Tagged_union), we didn't
need to cast things all that much.  And anywhere we did, the compiler optimized
the hell out of the structures.  The net result wasn't much more than what a
typical C++ program has just to support virtual dispatch (never mind casting). 
-->
你可能还会考虑支持类型安全所需的[运行时类型信息（RTTI）](https://en.wikipedia.org/wiki/Run-time_type_information)的开销有多大。 
好吧，多亏了POD，以及对[可辨识联合](https://en.wikipedia.org/wiki/Tagged_union)合适的支持，
使得我们无需进行太多的类型转换。
而即使在我们进行类型任何地方，编译器都对结构进行了优化，
因此最终结果并不比只支持虚拟调度（virtual dispatch）的典型C++程序要差（所以不要担心类型转换和它的开销）。

<!-- A general theme that ran throughout this journey is that compiler technology has
advanced tremeodusly in the past 20 years.  In most cases, safety overheads can
be optimized very aggressively.  That's not to say they drop to zero, but we were
able to get them within the noise for most interesting programs.  And --
surprisingly -- we found plenty of cases where safety _enabled_ new, novel
optimization techniques!  For example, having immutability in the type system
permit us to share pages more aggressively across multiple heaps and programs;
teaching the optimizer about [contracts](
https://en.wikipedia.org/wiki/Design_by_contract) let us more aggressively hoist
type safety checks; and so on. -->
贯穿于整个过程的通用主题是，编译器技术在过去20年中已经发展得非常好。 
在大多数情况下，安全性带来的额外开销可在很大程度上被优化掉，虽然这并不表示开销可以降到零，
但在大多数程序中，我们能够让其控制在可接受的范围内。 
并且，*令人惊讶的是*，我们发现了大量由于安全性所导致的新奇的优化方法！
例如，类型系统中的不变性使得我们可以在多个堆和程序之间采用更积极的方式共享页面，以及采用[合约](https://en.wikipedia.org/wiki/Design_by_contract)的方式使优化器更激进地提升类型安全检查等。

<!--
Another controversial area was concurrency safety.  Especially given that the
start of the project overlapped with the heady multicore days of the late 2000s.
What, no parallelism, you ask? 
-->
而另一个有争议的方面是并发安全，特别是考虑到Midori项目开始时正好与2000年代后期令人兴奋的多核的发展相重叠。 
你说什么，Midori没有并行性？

<!-- 
Note that I didn't say we banned concurrency altogether, just that we banned
_unsafe_ concurrency.  First, most concurrency in the system was expressed using
message passing between lightweight [software isolated processes](
http://research.microsoft.com/apps/pubs/default.aspx?id=71996).  Second, within
a process, we formalized the rules of safe shared memory parallelism, enforced
through type system and programming model rules.  The net result was that you
couldn't write a shared memory data race. 
-->
需要注意的使，我未说我们完全禁止并发，只是我们禁止了*不安全的*并发。 
首先，系统中的大多数并发采用在轻量级的[软件隔离进程](http://research.microsoft.com/apps/pubs/default.aspx?id=71996)之间消息传递的方式进行表达。 
其次，在一个进程中，我们通过强制类型系统和编程模型规则的方式来形式化安全共享内存的并行规则，
其最终结果很自然地就禁止了编写具有共享内存数据冒险的代码。

<!-- 
They key insight driving the formalism here was that no two "threads" sharing an
address space were permitted to see the same object as mutable at the same time.
Many could read from the same memory at once, and one could write, but multiple
could not write at once.  A few details were discussed in [our OOPSLA paper](
http://research.microsoft.com/apps/pubs/default.aspx?id=170528), and Rust
achieved a similar outcome [and documented it nicely](
http://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html).  It worked well
enough for many uses of fine-grained parallelism, like our multimedia stack. 
-->
这里推动形式化的关键内因是，不允许共享地址空间的两个“线程”同时看到同一个对象是可变的，也就是说
多个线程可同时读取同一内存中，或者仅单个线程可写入，但不允许多线程同时写入。 
[我们的OOPSLA论文](http://research.microsoft.com/apps/pubs/default.aspx?id=170528)讨论了一些细节，Rust也取得了类似的成果[并进行了很好地描述](https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html)。 
在多种细粒度并行的场景中，例如Midori的多媒体栈中，它都工作的很好。

<!-- 
Since Midori, I've been working to bring some of our key lessons about how to
achieve simultaneous safety and performance to both .NET and C++.  Perhaps the
most visible artifact are the [safety profiles](
https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md#S-profile)
we recently launched as part of the C++ Core Guidelines effort.  I expect more
to show up in C# 7 and the C# AOT work we're doing right now as we take .NET
cross-platform.  Midori was greenfield, whereas these environments require
delicate compromises, which has been fun, but slowed down some of the transfer
of these ideas into production.  I'm happy to finally start seeing some of it
bearing fruit. 
-->
在Midori以后，我一直致力于提供对于.Net以及C++而言如何同时实现安全性和性能的重要经验。
其中最显著的产品是我们最近作为C++核心指南的一部分推出的[安全配置（safety profiles）](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md#S-profile)。 
因为我们将.Net变成了跨平台的项目，所以我希望能在C# 7和我们正在进行的C# AOT的项目中展示更多内容。
与Midori宛如真空的安全环境不同，其他这些环境（.Net和C++）则需要进行微妙的妥协，
这虽然很有趣，但也减缓我们将这些想法实现到产品中的步伐，
但我也很高兴开始看到在Midori的安全性上的工作终于接出了果实。

<!-- 
The combination of memory, type, and concurrency safety gave us a powerful
foundation to stand on.  Most of all, it delivered a heightened level of
developer productivity and let us move fast.  The extremely costly buffer
overflows, data races, deadlocks, and so on, simply did not happen.
Someday all operating systems will be written this way. 
-->
内存安全、类型安全和并发安全的结合为我们的开发提供了强大的基础。 
最重要的是，它提高了开发人员的工作效率，使我们能够快速演进，因为
导致严重后果的缓冲区溢出、数据冒险和死锁等安全漏洞在Midori中根本就不会发生。
因此，（我相信）总有一天，所有的操作系统都会采用这种方式编写。

<!--
In the next article in this series, we'll look at how this foundational safety
let us deliver a [capability-based security model](
https://en.wikipedia.org/wiki/Capability-based_security) that was first class in
the programming model and type system, and brought the same "by-construction"
solution to eliminating [ambient authority](
https://en.wikipedia.org/wiki/Ambient_authority) and enabling the [
principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege)
everywhere, by default, in a big way.  See you next time. 
-->
在本系列的下一篇文章中，我们将看到这种基本的安全性是如何提供了在编程模型和类型系统中担任一等公民的[基于权能的安全模型（capability-based security model）](https://en.wikipedia.org/wiki/Capability-based_security)的，
以及“从构造”上消除[环境权限（ambient authority）](https://en.wikipedia.org/wiki/Ambient_authority)和默认在所有地方启用[最小权限原则](https://en.wikipedia.org/wiki/Principle_of_least_privilege)。
我们下次再见！
