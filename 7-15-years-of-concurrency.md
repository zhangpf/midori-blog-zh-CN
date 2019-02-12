---
title: Midori博客系列翻译（7）——关于并发的15年
date: 2018-12-19 15:58:00
tags: [操作系统, Midori, 翻译, 并发]
categories: 中文
---
<!-- 
In [a Tale of Three Safeties](/2015/11/03/a-tale-of-three-safeties/), we discussed three kinds of safety: type, memory,
and concurrency.  In this follow-on article, we will dive deeper into the last, and perhaps the most novel yet
difficult, one.  Concurrency-safety led me to the [Midori](/2015/11/03/blogging-about-midori/) project in the first
place, having spent years on .NET and C++ concurrency models leading up to joining.  We built some great things that
I'm very proud of during this time.  Perhaps more broadly interesting, however, are the reflections on this experience
after a few years away from the project. 
-->
在[三类安全性的故事](/2018/10/24/midori/1-a-tale-of-three-safeties/)中，我们讨论了三种安全性：类型安全，内存安全和并发安全。 
在后续的本文中，我们将深入探讨最后一种安全性——并发安全，这也许是其中最新颖也最难的一种。
对并发安全的追求是使我最初参与到Midori项目的原因之一，因为在此之前的数年时间在.NET和C++并发模型上加入。 
在参与Midori的过程中，我们打造了一些我引以为傲的伟大事物。 
然而，更具有广泛兴趣的是，在离开项目数年时间里，对这段经历的不断反思。

<!-- 
I've tried to write this article about 6 times since earlier this year, and I'm thrilled to finally share it.  I hope
that it's useful to anyone interested in the field, and especially anybody who is actively innovating in this area.
Although the code samples and lessons learned are deeply rooted in C#, .NET, and the Midori project, I have tried to
generalize the ideas so they are easily consumable regardless of programming language.  I hope you enjoy! 
-->
自今年早些时候以来，我已大约6次尝试写这篇文章，并且我很高兴最终能将它分享出去。 
我希望它对任何对该领域感兴趣的人都有用，尤其是那些在这一领域积极创新的人。 
虽然示例代码和经验教训都深深根植于C#，.Net和Midori项目，但我试图概括这些想法，因此无论你使用何种编程语言，都将很容易理解它们。 
我希望你能喜欢！

<!-- 
# Background 
-->
# 背景

<!-- 
For most of the 2000s, my job was figuring out how to get concurrency into the hands of developers, starting out as a
relatively niche job on [the CLR team](https://en.wikipedia.org/wiki/Common_Language_Runtime) at Microsoft. 
-->
在2000年代的大部分时间里，我的工作是弄清楚如何
将并发性提供给开发人员，并最初作为微软[CLR团队](https://en.wikipedia.org/wiki/Common_Language_Runtime)的一个相对小众的工作而进行开展。

<!-- 
## Niche Beginnings 
-->
## 起点
<!-- 
Back then, this largely entailed building better versions of the classic threading, locking, and synchronization
primitives, along with attempts to solidify best practices.  For example, we introduced a thread-pool to .NET 1.1,
and used that experience to improve the scalability of the Windows kernel, its scheduler, and its own thread-pool.  We
had this crazy 128-processor [NUMA](https://en.wikipedia.org/wiki/Non-uniform_memory_access) machine that kept us busy
with all sorts of esoteric performance challenges.  We developed rules for [how to do concurrency right](
http://joeduffyblog.com/2006/10/26/concurrency-and-the-impact-on-reusable-libraries/) -- lock leveling, and so on -- and
experimented with [static analysis](
https://www.microsoft.com/en-us/research/wp-content/uploads/2008/08/tr-2008-108.pdf).  I even [wrote a book](
https://www.amazon.com/Concurrent-Programming-Windows-Joe-Duffy/dp/032143482X) about it. 
-->
在当时，主要涉及的工作包括构建更好版本的经典线程、加锁和同步原语，以及巩固最佳实践的尝试。
例如，我们向.Net 1.1版本中引入了线程池，并利用这种经验来提高Windows内核、调度器和线程池的可伸缩性。 
我们所拥有的那台疯狂的128核的[NUMA](https://en.wikipedia.org/wiki/Non-uniform_memory_access)机器，使得我们忙于各种深奥的性能挑战中。 
我们制定了[如何正确进行并发](http://joeduffyblog.com/2006/10/26/concurrency-and-the-impact-on-reusable-libraries/)的规则——加锁平衡等等，并进行了[静态分析](https://www.microsoft.com/en-us/research/wp-content/uploads/2008/08/tr-2008-108.pdf)的实验。 
为此，我甚至写了一本[关于并发的书](
https://www.amazon.com/Concurrent-Programming-Windows-Joe-Duffy/dp/032143482X)。

<!-- 
Why concurrency in the first place? 
-->
为什么要把并发放到首位？

<!-- 
In short, it was enormously challenging, technically-speaking, and therefore boatloads of fun. 
-->
简而言之，从技术角度讲，它具有极大的挑战性，因此非常有趣。

<!-- 
I had always been a languages wonk.  So, I was naturally fascinated by the decades of deep work in academia, including
programming language and runtime symbiosis (especially [Cilk](https://en.wikipedia.org/wiki/Cilk) and [NESL](
https://en.wikipedia.org/wiki/NESL)), advanced type systems, and even specialized parallel hardware architectures
(especially radical ones like [the Connection Machine](https://en.wikipedia.org/wiki/Connection_Machine), and [MIMD](
https://en.wikipedia.org/wiki/MIMD) supercomputers, that innovated beyond our trustworthy pal, [von Neumann](
https://en.wikipedia.org/wiki/Von_Neumann_architecture)). 
-->
我一直是编程语言爱好者。因此，我自然而然地着迷于学术界数十年的深入研究，
这包括编程语言、运行时共生（特别是[Cilk](https://en.wikipedia.org/wiki/Cilk)和[NESL](https://en.wikipedia.org/wiki/NESL)）
、高级类型系统、甚至是专门的并行硬件架构（尤其是像[Connection Machine](https://en.wikipedia.org/wiki/Connection_Machine)和[MIMD](https://en.wikipedia.org/wiki/MIMD)超级计算机这样的激进体系结构），它们甚至突破了我们值得信赖的[冯·诺依曼体系结构](https://en.wikipedia.org/wiki/Von_Neumann_architecture)。

<!-- 
Although some very large customers actually ran [symmetric multiprocessor (SMP)](
https://en.wikipedia.org/wiki/Von_Neumann_architecture) servers -- yes, we actually used to call them that -- I
wouldn't say that concurrency was a very popular area to specialize in.  And certainly any mention of those cool
"researchy" sources would have gotten an odd glance from my peers and managers.  Nevertheless, I kept at it. 
-->
虽然一些非常大的客户实际上使用的是[对称多处理器（SMP）](https://en.wikipedia.org/wiki/Von_Neumann_architecture) 服务器 - 是的，我们实际上习惯称之为 - 我不会说并发性是一个非常受欢迎的专业领域。当然，任何提及那些很酷的“研究”消息来源可能会让我的同行和经理们一目了然。尽管如此，我仍然坚持下去。

<!-- 
Despite having fun, I wouldn't say the work we did during this period was immensely impactful to the casual observer.
We raised the abstractions a little bit -- so that developers could schedule logical work items, think about higher
levels of synchronization, and so on -- but nothing game-changing.  Nonetheless, this period was instrumental to laying
the foundation, both technically and socially, for what was to come later on, although I didn't know it at the time. 
-->
尽管这很有趣，但我不会说在此期间我们所做的工作对于不经意的观察者来说是非常有影响力的。我们稍微提出了抽象 - 这样开发人员可以安排逻辑工作项，考虑更高级别的同步等等 - 但没有改变游戏规则。尽管如此，这个时期对于在技术上和社会上为后来的事情奠定基础是有帮助的，尽管我当时并不知道。

<!-- 
## No More Free Lunch; Enter Multicore 
-->
## 没有免费的午餐——进入多核领域

<!-- 
Then something big happened. 
-->
然后发生了一件大事。

<!-- 
In 2004, Intel and AMD approached us about [Moore's Law](https://en.wikipedia.org/wiki/Moore's_law), notably its
[imminent demise](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.87.8775&rep=rep1&type=pdf).  [Power wall
challenges](https://www.quora.com/Why-havent-CPU-clock-speeds-increased-in-the-last-5-years) would [seriously curtail
the ever-increasing year-over-year clock speed improvements](
http://www.economist.com/technology-quarterly/2016-03-12/after-moores-law) that the industry had grown accustomed to. 
-->
2004年，英特尔和AMD向我们介绍了[摩尔定律](https://en.wikipedia.org/wiki/Moore's_law)，
特别是它[即将消亡](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.87.8775&rep=rep1&type=pdf)的事实。
[能量墙的挑战](https://www.quora.com/Why-havent-CPU-clock-speeds-increased-in-the-last-5-years)将[严重限制该行业已经习惯的逐年增加时钟速度的改进方式](
http://www.economist.com/technology-quarterly/2016-03-12/after-moores-law) 。

<!-- 
Suddenly management cared a whole lot more about concurrency.  Herb Sutter's 2005 ["Free Lunch is Over" article](
http://www.gotw.ca/publications/concurrency-ddj.htm) captured the fever pitch.  If we couldn't enable developers to
write massively parallel software -- something that was historically very difficult and unlikely to happen without
significantly lower barriers to entry -- both Microsoft and Intel's businesses, and mutually beneficial business models,
were in trouble.  If the hardware didn't get faster in the usual ways, software couldn't automatically get better, and
people would have less reason to buy new hardware and software.  An end to the [Wintel era](
https://en.wikipedia.org/wiki/Wintel) and [Andy and Bill's Law](
http://www.forbes.com/2005/04/19/cz_rk_0419karlgaard.html), *"What Andy giveth, Bill taketh away"*. 
-->
因此，突然之间管理层更加关注于并发性。 
Herb Sutter在2005年[“免费午餐已结束”的文章](http://www.gotw.ca/publications/concurrency-ddj.htm)捕捉到了这种发烧的声音。
如果我们无法使开发人员能够编写大规模并行软件（由于历史原因，该任务非常困难，并且不会在没有明显降低进入门槛的情况下发生），
那么微软和英特尔的业务以及其互利的商业模式都将遇到麻烦。
如果硬件没有以通常的方式变得更快，那么软件就不会自动变得更好，
因此人们没有理由再购买新的硬件和软件。
这标志着[Wintel时代](https://en.wikipedia.org/wiki/Wintel)和[安迪-比尔法定律](
http://www.forbes.com/2005/04/19/cz_rk_0419karlgaard.html)（*“凡是安迪提供的（计算资源），比尔全都带走”*）的结束。

<!-- 
Or, so the thinking went. 
-->
也就是说，这产生了思考。

<!-- 
This is when the term ["multicore"](https://en.wikipedia.org/wiki/Multi-core_processor) broke into the mainstream, and
we began envisioning a world with 1,024 core processors and even more forward-looking ["manycore" architectures](
https://en.wikipedia.org/wiki/Manycore_processor) that took a page from [DSP](
https://en.wikipedia.org/wiki/Digital_signal_processor)s, mixing general purpose cores with specialized ones that
could offload heavy-duty functions like encryption, compression, and the like. 
-->
这就是[“多核”](https://en.wikipedia.org/wiki/Multi-core_processor)这个术语进入主流的时候。
我们开始设想一个拥有1,024个处理器核心和更具前瞻性的，从[DSP](https://en.wikipedia.org/wiki/Digital_signal_processor)中获得启发的[“众核”架构](https://en.wikipedia.org/wiki/Manycore_processor)领域。
这个领域通用处理核心与专用核心混合使用，以迁移计算密集型任务，例如加密，压缩等。

<!-- 
As an aside, with 10 years of hindsight, things didn't unfold exactly as we thought they would.  We don't run PCs with
1,024 traditional cores, although [our GPUs have blown way past that number](
http://www.geforce.com/hardware/10series/titan-x-pascal), and we do see more heterogeneity than ever before, especially
in the data center where [FPGA](https://en.wikipedia.org/wiki/Field-programmable_gate_array)s are now [offloading
critical tasks like encryption and compression](https://www.wired.com/2016/09/microsoft-bets-future-chip-reprogram-fly/). 
-->
顺便说一下，经过10年的回顾，事情并没有像我们想象的那样完全展开。
我们未产生能够运行1,024个传统内核的PC，
尽管[GPU中核心的数量已经超过了这个数字](http://www.geforce.com/hardware/10series/titan-x-pascal)，并且确实获得了比以往更多的异构性。
特别是在数据中心中，[FPGA](https://en.wikipedia.org/wiki/Field-programmable_gate_array)正在[迁移加密和压缩等关键任务](https://www.wired.com/2016/09/microsoft-bets-future-chip-reprogram-fly/)的计算量。

<!-- 
The real big miss, in my opinion, was mobile.  This was precisely when the thinking around power curves, density, and
heterogeneity should have told us that mobile was imminent, and in a big way.  Instead of looking to beefier PCs, we
should have been looking to PCs in our pockets.  Instead, the natural instinct was to cling to the past and "save" the
PC business.  This is a classical [innovator's dilemma](https://en.wikipedia.org/wiki/The_Innovator's_Dilemma) although
it sure didn't seem like one at the time.  And of course PCs didn't die overnight, so the innovation here was not
wasted, it just feels imbalanced against the backdrop of history.  Anyway, I digress. 
-->
在我看来，真正的重大缺失是在移动领域。
这恰恰是围绕功率曲线、密度和异质性的思考，应该告诉我们移动即将来临，并在很大程度上来临。
我们应更多地使用自己口袋中的PC设备，而不是寻求更强大的电脑。
相反，一种天生的本能就是坚持过去并“拯救”传统PC业务，这是经典的[创新者的困境](https://en.wikipedia.org/wiki/The_Innovator's_Dilemma)，即使在当时看起来这并不是。
当然，PC也并没有一夜之间消失，因此创新并没有浪费，只是在历史背景下感觉不平衡。
不管怎样，我有点偏题了。

<!-- 
## Making Concurrency Easier 
-->
## 使并发变得更容易

<!-- 
As a concurrency geek, this was the moment I was waiting for.  Almost overnight, finding sponsors for all this
innovative work I had been dreaming about got a lot easier, because it now had a real, and very urgent, business need. 
-->
作为一个并发的极客，这就是我期待的那一刻。 
几乎在一夜之间，找到我所有梦寐以求的创新工作的赞助变得更加容易，
因为它现在变成一个真实且非常紧急的业务需求。

<!-- 
In short, we needed to: 
-->
简而言之，我们需要：

<!-- 
* Make it easier to write parallel code.
* Make it easier to avoid concurrency pitfalls.
* Make both of these things happen almost "by accident." 
-->
* 使得编写并行代码变得更容易。
* 更容易避免并发陷阱。
* 让前面两者的发生变得几乎成为“偶然”。

<!-- 
We already had threads, thread-pools, locks, and basic events.  Where to go from here? 
-->
我们已经有了线程、线程池、锁和基本的事件，那么然后呢？

<!-- 
Three specific projects were hatched around this point and got an infusion of interest and staffing. 
-->
围绕这一点，我们设计了三个具体项目，引起了相关人员的兴趣，并组建了团队。

<!-- 
### Software Transactional Memory 
-->
## 软件事务内存

<!-- 
Ironically, we began with safety first.  This foreshadows the later story, because in general, safety took a serious
backseat until I was able to pick it back up in the context of Midori. 
-->
具有讽刺意味的是，我们首先以安全开始。这预示着后来的故事，因为总的来说，安全性是一个严重的后座，直到我能够在Midori的背景下重新找回它。

<!-- 
Developers already had several mechanisms for introducing concurrency, and yet struggled to write correct code.  So we
sought out those higher level abstractions that could enable correctness as if by accident. 
-->
开发人员已经有几种引入并发的机制，但仍努力编写正确的代码。 
因此，我们找到了那些能够正确实现具有正确性的更高级别的抽象。


<!-- 
Enter [software transactional memory](https://en.wikipedia.org/wiki/Transactional_memory) (STM).  An outpouring of
promising research had been coming out in the years since [Herlihy and Moss's seminal 1993 paper](
https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-895-theory-of-parallel-systems-sma-5509-fall-2003/readings/herlihy_mo93.pdf)
and, although it wasn't a panacea, a number of us became enamored with its ability to raise the abstraction level. 
-->
回到[软件事务内存](https://en.wikipedia.org/wiki/Transactional_memory)（STM）上来，自Herlihy和Moss于1993年发表的开创性论文以来，已经出现了大量有前景的研究，虽然它不是灵丹妙药，但我们中的许多人都迷恋于其提高抽象水平的能力。

<!-- 
STM let you write things like this, and get automatic safety: 
-->
STM让你写这样的东西，并自动获得安全性：

    void Transfer(Account from, Account to, int amt) {
        atomic {
            from.Withdraw(amt);
            to.Deposit(amt);
        }
    } 


<!-- 
Look ma, no locks! 
-->
瞧瞧，这里没有上锁！

<!-- 
STM could handle all of the decisions transparently like figuring out how coarse- or fine-grained synchronization to
use, the contention policies around that synchronization, deadlock detection and prevention, and guarantee that you
didn't forget to lock when accessing a shared data structure.  All behind a tantalizingly simple keyword, `atomic`. 
-->
STM可以透明地处理所有决策，例如，确定如何使用粗粒度或细粒度同步，围绕该同步的竞争策略、死锁检测和避免，并保证在访问共享数据结构时不会忘记加锁，而完成所有这些都是一个吸引人得简单关键字——`atomic`。

<!-- 
STM also came with simple, more declarative, coordination mechanisms, like [orElse](
https://hackage.haskell.org/package/stm-2.4.4.1/docs/Control-Monad-STM.html#v:orElse).  So, although the focus was on
eliminating the need to manually manage locking, it also helped evolve synchronization between threads. 
-->
STM还具有简单的，更具声明性的协调机制，
如[`orElse`](https://hackage.haskell.org/package/stm-2.4.4.1/docs/Control-Monad-STM.html#v:orElse)。
因此，虽然其初衷是消除手动加锁的负担，但它同时也有助于发展线程之间的同步。

<!-- 
Unfortunately, after a few years of prototyping deep runtime, OS, and even hardware support, we abandoned the efforts.
My brief summary is that it's more important to encourage good concurrency architecture than it is to make poor ones
"just work", although I have written more details [here](/2010/01/03/a-brief-retrospective-on-transactional-memory/) and
[here](/2010/05/16/more-thoughts-on-transactional-memory/).  It was this higher level architecture that we should focus
on solving first and foremost and, after the dust settled, see what gaps remained.  It wasn't even clear that STM would
be the correct tool for the job once we got to that point.  (In hindsight, I do think it's one of the very many
reasonable tools in the toolbelt, although with more distributed application architectures on the rise, it's [a
dangerous thing to give to people](http://wiki.c2.com/?DistributedTransactionsAreEvil).) 
-->
不幸的是，经过几年对运行时，操作系统甚至硬件支持的深度原型设计，我们放弃了这种努力。
对此，我的简要总结是，鼓励良好的并发体系结构比使在较差的硬件上“恰好够用”更重要，尽管我已经在这里和这里写了更多细节。正是这种更高层次的架构我们应该首先关注解决问题，并在尘埃落定之后，看看还有什么差距。一旦我们达到这一点，甚至还不清楚STM是否是正确的工具。 （事后看来，我确实认为它是工具带中许多合理的工具之一，尽管随着更多分布式应用程序架构的增加，给人们带来危险的事情。）

<!-- 
Our STM efforts weren't a complete failure, however.  It was during this time that I began experimenting with type
systems for safe concurrency.  Moreover, bits and pieces ended up incorporated into Intel's Haswell processor as the
[Transactional Synchronization Extensions (TSX)](https://en.wikipedia.org/wiki/Transactional_Synchronization_Extensions)
suite of instructions, delivering the capability to leverage [speculative lock elision](
http://citeseer.ist.psu.edu/viewdoc/download;jsessionid=496F867855F76185B4C1EA3195D42F8C?doi=10.1.1.136.1312&rep=rep1&type=pdf)
for ultra-cheap synchronization and locking operations.  And again, I worked with some amazing people during this time. 
-->
然而，我们在STM上的努力并非彻底失败。
正是在这段时间里，我开始尝试使用类型系统来实现安全并发​​。
此外，最终的零碎件作为[事务同步扩展（TSX）](https://en.wikipedia.org/wiki/Transactional_Synchronization_Extensions)指令集合并入英特尔的Haswell处理器，
以提供了利用[推断性锁省略](http://citeseer.ist.psu.edu/viewdoc/download;jsessionid=496F867855F76185B4C1EA3195D42F8C?doi=10.1.1.136.1312&rep=rep1&type=pdf)
实现超低成本同步加锁操作的能力。
并且，在这段时间里，我和一些非常厉害的人一起工作过。

<!-- 
### Parallel Language Integrated Query (PLINQ) 
-->
### 并行语言集成查询（PLINQ）

<!-- 
Alongside STM, I'd been prototyping a "skunkworks" data parallel framework, on nights and weekends, to leverage our
recent work in [Language Integrated Query (LINQ)](https://en.wikipedia.org/wiki/Language_Integrated_Query). 
-->
除了STM之外，为了利用我们最近在[语言集成查询（LINQ）](https://en.wikipedia.org/wiki/Language_Integrated_Query)
方面的成果，我还在晚上和周末对“skunkworks”数据并行框架进行原型设计。

<!-- 
The idea behind parallel LINQ (PLINQ) was to steal a page from three well-researched areas: 
-->
并行LINQ（PLINQ）背后的想法分别从三个得到很好研究的领域中获得：


<!-- 
1. [Parallel databases](https://en.wikipedia.org/wiki/Parallel_database), which already [parallelized SQL queries on
   users' behalves](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.21.2197&rep=rep1&type=pdf) without them
   needing to know about it, often with impressive results.

2. Declarative and functional languages, which often used [list comprehensions](
   https://en.wikipedia.org/wiki/List_comprehension) to express higher-level language operations that could be
   aggressively optimized, including parallelism.  For this, I deepened my obsession with [Haskell](
   https://wiki.haskell.org/GHC/Data_Parallel_Haskell), and was inspired by [APL](
   https://en.wikipedia.org/wiki/APL_(programming_language)).

3. Data parallelism, which had quite a [lengthy history in academia](https://en.wikipedia.org/wiki/Data_parallelism)
   and even some more mainstream incarnations, most notably [OpenMP](https://en.wikipedia.org/wiki/OpenMP). 
-->
1. [并行数据库](https://en.wikipedia.org/wiki/Parallel_database)：已经在[针对用户行为的SQL查询并行化](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.21.2197&rep=rep1&type=pdf)而无需了解细节方面，产生了令人印象深刻的结果；
2. 声明性和函数式语言：通常使用[列表推导](   https://en.wikipedia.org/wiki/List_comprehension)的方式，
来表达包括并行性在内的，可以积极优化的高级语言操作。
为此，我加深了对[Haskell](https://wiki.haskell.org/GHC/Data_Parallel_Haskell)的执念，
并从[APL](https://en.wikipedia.org/wiki/APL_(programming_language))中获得了启发。
3. [数据并行](https://en.wikipedia.org/wiki/Data_parallelism)：
在学术界甚至更为主流的化身中有相当长的历史，
其中最著名的是[OpenMP](https://en.wikipedia.org/wiki/OpenMP)。

<!-- 
The idea was pretty straightforward.  Take existing LINQ queries, which already featured operations like maps, filters,
and aggregations -- things that were classically parallelizable in both languages and databases -- and auto-parallelize
them.  Well, it couldn't be implicit, because of side-effects.  But all it took was a little `AsParallel` to enable: 
-->
这样的想法很简单。采用现有的LINQ查询，这些查询已经具有地图，过滤器和聚合等操作 - 在语言和数据库中可以经典地并行化的东西 - 并自动并行化它们。嗯，由于副作用，它不能隐含。但所有这一切都需要一点`AsParallel`才能实现：

<!-- 
    // Sequential:
    var q = (from x in xs
             where p(x)
             select f(x)).Sum();
    
    // Parallel:
    var q = (from x in xs.AsParallel()
             where p(x)
             select f(x)).Sum(); 
-->
    // 顺序：
    var q = (from x in xs
             where p(x)
             select f(x)).Sum();
    
    // 并行：
    var q = (from x in xs.AsParallel()
             where p(x)
             select f(x)).Sum(); 
            
<!-- 
This demonstrates one of the great things about data parallelism.  It can scale with the size of your inputs: either
data quantity, expense of the operations against that data, or both.  And when expressed in a sufficiently high-level
language, like LINQ, a developer needn't worry about scheduling, picking the right number of tasks, or synchronization. 
-->
这展示了数据并行性的一大优点。
它可以根据输入的大小进行扩展：数据量，针对该数据的操作费用，或两者。
当使用足够高级的语言（如LINQ）表示时，
开发人员无需担心计划，选择正确数量的任务或同步。

<!-- 
This is essentially [MapReduce](https://en.wikipedia.org/wiki/MapReduce), on a single machine, across many processors.
Indeed, we later collaborated with MSR on a project called [DryadLINQ](
https://www.microsoft.com/en-us/research/project/dryadlinq/) which not only ran such queries across many processors, but
also distributed them across many machines too.  (Eventually we went even finer-grained with SIMD and GPGPU
implementations.)  This eventually led to Microsoft's own internal equivalent to Google's MapReduce, [Cosmos](
https://www.quora.com/Distributed-Systems-What-is-Microsofts-Cosmos), a system that powers a lot of big data innovation
at Microsoft still to this date. 
-->
它在本质上是单个机器上跨越多个处理器的`MapReduce`(https://en.wikipedia.org/wiki/MapReduce)模型。
实际上，我们后来与MSR就一个名为[DryadLINQ](https://www.microsoft.com/en-us/research/project/dryadlinq/)的项目进行了合作，
该项目不仅可以在单个机器上的多个处理器上并行查询，
而且还可以通过分布的方式在多个机器上并行（最终我们使用SIMD和GPGPU实现了更细粒度的控制）。
这最终使得微软在自己的内部实现了等同于谷歌的MapReduce和[Cosmos](https://www.quora.com/Distributed-Systems-What-is-Microsofts-Cosmos)的系统，该系统至今仍在微软驱动着许多的大数据创新。

<!-- 
Developing PLINQ was a fond time in my career and a real turning point.  I collaborated and built relationships with
some amazing people.  BillG wrote a full-page review of the idea, concluding with "We will have to put more resources
specifically on this work."  Such strong words of encouragement didn't hurt with securing funding to deliver on the
idea.  It also attracted the attention of some incredible people.  For example, [Jim Gray](
https://en.wikipedia.org/wiki/Jim_Gray_(computer_scientist)) took notice, and I got to experience his notorious
generosity 1st hand, just two months before his tragic disappearance. 
-->

开发PLINQ是我职业生涯中的一段美好时光，并且也是一个真正的转折点。
我与一些了不起的人合作并建立了关系。 
Bill Gates写了一篇关于该想法的整版评论，并以“我们将专门为这项工作投入更多资源”作为结语。
这样的鼓励措辞并没有因实现这一想法需进行秘密资助，其强烈程度而有所保留。
另外，该项目也引起了一些特别的大佬的注意。
例如，[Jim Gray](https://en.wikipedia.org/wiki/Jim_Gray_(computer_scientist))注意到了该项目，在他失踪悲剧发生前两个月，我得到了他广为人知的慷慨相助。

<!-- 
Needless to say, this was an exciting time! 
-->
不用说，这是一个激动人心的时刻！

<!-- 
### Interlude: Forming the PFX Team 
-->
### 插曲：成立PFX团队

<!-- 
Around this time, I decided to broaden the scope of our efforts beyond just data parallelism, tackling task parallelism
and other concurrency abstractions.  So I went around pitching the idea of forming a new team.
-->
大约就在这个时候，我决定扩大我们的工作范围，
而不仅仅只是数据并行，而应该包括任务处理并行和其他并发抽象。
所以我开始提出组建新团队的想法。

<!-- 
Much to my surprise, a new parallel computing group was being created in the Developer Division in response to the
changing technology landscape, and they wanted to sponsor these projects.  It was an opportunity to roll everything up
under a nice top-level business theme, unify recruiting efforts, and take things even further, eventually branching out
into C++, GPGPUs, and more. 
-->
令我感到意外的是，为了应对不断变化的技术环境，
开发部门正在组建一个新的并行计算组，他们希望赞助这些项目。
这是一个在漂亮的顶层业务主题下推动一切的机会，我们能够统一招聘工作，并采取进一步的行动，达到最终扩展到C++，GPGPU等方面的目的。

<!-- 
So, obviously, I said yes. 
-->
显然，我可以说我做到了。

<!-- 
I named the team ["PFX"](https://en.wikipedia.org/wiki/Parallel_Extensions), initially short for "parallel frameworks",
although by the time we shipped marketing working its magic on us, renaming it to "Parallel Extensions to .NET."  This
team's initial deliverable encompassed PLINQ, task parallelism, and a new effort, Coordination Data Structures (CDS),
meant to handle advanced synchronization efforts, like [barrier-style synchronization](
https://en.wikipedia.org/wiki/Barrier_(computer_science)), [concurrent and lock-free collections](
https://github.com/dotnet/corefx/tree/master/src/System.Collections.Concurrent/src/System/Collections/Concurrent)
derived from [many great research papers](http://cs.rochester.edu/u/scott/papers/1995_TR600.pdf), and more. 
-->
我将团队命名为[“PFX”](https://en.wikipedia.org/wiki/Parallel_Extensions)，
最初是“parallel frameworks”的缩写，尽管当我们为了
发挥营销的优势而将其重命名为“Parallel Extensions to .NET”。
该团队的初始可交付的成果包括PLINQ，任务并行，
以及一项新工作——Coordination Data Structures（CDS），
其旨在处理高级同步工作，例如[屏障同步](https://en.wikipedia.org/wiki/Barrier_(computer_science))，
由许多优秀研究论文提出的[并发和无锁集合](https://github.com/dotnet/corefx/tree/master/src/System.Collections.Concurrent/src/System/Collections/Concurrent)等。

<!--  
### Task Parallel Library
-->
### 任务并行库

<!-- 
This brings me to task parallelism. 
-->
它（*任务并行库*）给我们带来了任务并行性。

<!-- 
As part of PLINQ, we needed to create our own concept of parallel "tasks."  And we needed a sophisticated scheduler that
could scale automatically given the machine's available resources.  Most existing schedulers were thread-pool like, in
that they required that a task run on a separate thread, even if doing so was not profitable.  And the mapping of tasks
to threads was fairly rudimentary, although [we did make improvements to that over the years](
http://www.sigmetrics.org/conferences/sigmetrics/2009/workshops/papers_hotmetrics/session2_2.pdf). 
-->
作为PLINQ的一部分，我们需要创建自己的并行“task”概念。
为此，我们需要一个复杂的调度程序，
其可以根据机器的可用资源进行自动扩展。
大多数现有的调度程序都是采用类似线程池的方式，
因为它们要求在单独的线程上运行任务，
即使这样做也没有获取相应的好处。
并且任务到线程的映射是相当简陋的，
虽然[多年来我们确实对其进行了改进](http://www.sigmetrics.org/conferences/sigmetrics/2009/workshops/papers_hotmetrics/session2_2.pdf)。

<!-- 
Given my love of Cilk, and the need to schedule lots of potentially-recursive fine-grained tasks, choosing a
[work stealing scheduler](https://en.wikipedia.org/wiki/Work_stealing) for our scheduling architecture was a no-brainer. 
-->
考虑到我对Cilk的喜爱，以及需要调度大量潜在的递归细粒度任务，
为我们的调度架构选择[work-stealing调度器](https://en.wikipedia.org/wiki/Work_stealing)不失为一个明智的选择。

<!-- 
At first, our eyes were locked squarely on PLINQ, and so we didn't pay as much attention to the abstractions.  Then MSR
began exploring what standalone a task parallel library would look like.  This was a perfect partnership opportunity and
so we started building something together.  The `Task<T>` abstraction was born, we rewrote PLINQ to use it, and created
a suite of [`Parallel` APIs](https://msdn.microsoft.com/en-us/library/system.threading.tasks.parallel(v=vs.110).aspx)
for common patterns such as fork/join and parallel `for` and `foreach` loops. 
-->
起初，我们的焦点完全放在了PLINQ，导致对抽象并没有太多的关注。
而后，MSR开始探索独立的任务并行库的形式。
这是一个完美的合作机会，因此我们开始一起构建一些东西。
在这样的条件下，`Task<T>`抽象诞生了，
为了使用它我们重写了PLINQ，
并为fork/join以及并行`for`和`foreach`循环等常见模式创建了一套
[`Parallel`的API](https://msdn.microsoft.com/en-us/library/system.threading.tasks.parallel(v=vs.110).aspx)，


<!-- 
Before shipping, we replaced the guts of the thread-pool with our new shiny work-stealing scheduler, delivering unified
resource management within a process, so that multiple schedulers wouldn't fight with one another.  To this day, [the
code is almost identical](
https://github.com/dotnet/coreclr/blob/1a47d11a6a721a9bed1009d2930de4614b9f6d46/src/mscorlib/src/System/Threading/ThreadPool.cs#L133)
to my early implementation in support of PLINQ (with many bug fixes and improvements, of course). 
-->
在发布之前，我们用全新的work-stealing调度器替换了线程池，
并在进程内部提供了统一的资源管理，使得多个调度程序就不会相互竞争。
到目前为止，为了支持PLINQ，[我的现在的代码与早期实现几乎完全相同](
https://github.com/dotnet/coreclr/blob/1a47d11a6a721a9bed1009d2930de4614b9f6d46/src/mscorlib/src/System/Threading/ThreadPool.cs#L133)（当然还有许多错误修复和改进）。

<!-- 
We really obsessed over the usability of a relatively small number of APIs for a long time.  Although we made mistakes,
I'm glad in hindsight that we did so.  We had a hunch that `Task<T>` was going to be core to everything we did in the
parallelism space but none of us predicted the widespread usage for asynchronous programming that really popularized it
years later.  Now-a-days, this stuff powers `async` and `await` and I can't imagine life without `Task<T>`. 
-->
在很长的一段时间里，我们确实很关注于相对少量API的可用性。
虽然我们犯了错误，但事后我很高兴我们这样做了。
我们预感到`Task<T>`将成为我们在并行领域中所做的一切的核心，
但是我们都没有预测到异步编程的广泛使用，
这些编程在几年后真正普及了它。
现在，这些东西已经支持了`async`和`await`，
我已经无法想象没有`Task<T>`的世界的样子。

<!-- 
### A Shout-Out: Inspiration From Java 
-->
### 呐喊：来自Java的灵感

<!-- 
I would be remiss if I didn't mention Java, and the influence it had on my own thinking. 
-->
如果我没有提及到Java，以及它对我自身思考的影响，那对我来讲是不负责任的。

<!-- 
Leading up to this, our neighbors in the Java community had also begun to do some innovative work, led by Doug Lea, and
inspired by many of the same academic sources.  Doug's 1999 book, [Concurrent Programming in Java](
http://gee.cs.oswego.edu/dl/cpj/index.html), helped to popularize these ideas in the mainstream and eventually led to
the incorporation of [JSR 166](https://jcp.org/en/jsr/detail?id=166) into the JDK 5.0.  Java's memory model was also
formalized as [JSR 133](https://jcp.org/en/jsr/detail?id=133) around this same time, a critical underpinning for the
lock-free data structures that would be required to scale to large numbers of processors. 
-->
因为在此之前，我们在Java社区的邻居们也开始做一些，
由Doug Lea领导并受到许多相同学术资源启发的创新工作。 
Doug在1999年的著作《[Java中的并发编程](http://gee.cs.oswego.edu/dl/cpj/index.html)》
帮助将这些想法推广成为主流，并最终将JSR-166纳入JDK的5.0版本中。 
Java的内存模型也在同一时期被正式化为JSR-133，
并且成为扩展到多处理器所需的无锁数据结构的关键基础。

<!-- 
This was the first mainstream attempt I saw to raise the abstraction level beyond threads, locks, and events, to
something more approachable: concurrent collections, [fork/join](http://gee.cs.oswego.edu/dl/papers/fj.pdf), and more.
It also brought the industry closer to some of the beautiful concurrent programming languages in academia.  These
efforts were a huge influence on us.  I especially admired how academia and industry partnered closely to bring decades'
worth of knowledge to the table, and explicitly [sought to emulate](
http://www.cs.washington.edu/events/colloquia/search/details?id=768) this approach in the years to come. 
-->
这是我看到的，首次将线程、锁和事件之外的抽象级别提升到更易使用的主流尝试：
并发集合，[fork/join](http://gee.cs.oswego.edu/dl/papers/fj.pdf)等等，
它还使产业界更接近于学术界的一些优秀并发编程语言，
这些努力对我们产生了巨大影响。
我特别钦佩于学术界和产业界是如何密切合作的，
将数十年的研究转化为成果，并在未来几年明确地[努力模仿](http://www.cs.washington.edu/events/colloquia/search/details?id=768)这种方法。

<!-- 
Needless to say, given the similarities between .NET and Java, and level of competition, we were inspired. 
-->
总之毋庸置疑的是，鉴于.Net和Java之间的相似性以及相互竞争的程度，我们从Java受到启发。

<!-- 
## O Safety, Where Art Thou? 
-->
## 安全，艺术在哪里？

<!-- 
There was one big problem with all of this.  It was all unsafe.  We had been almost exclusively focused on mechanisms
for introducing concurrency, but not any of the safeguards that would ensure that using them was safe. 
-->
所有这一切都有一个大问题，那就是这一切都是不安全的。
我们几乎全部专注于引入并发机制，
但没有任何保证安全使用它们的保护措施。

<!-- 
This was for good reason: it's hard.  Damn hard.  Especially across the many diverse kinds of concurrency available to
developers.  But thankfully, academia had decades of experience in this area also, although it was arguably even more
"esoteric" to mainstream developers than the parallelism research.  I began wallowing in it night and day. 
-->
对此，我们是有充分理由：安全很难，非常非常难，
特别是对于那些跨越开发人员可用的各种各样的并发性。
但值得庆幸的是，学术界在这一领域也有数十年的研究经验，
尽管对于主流开发者而言，它可能比并行性的研究更为“深奥”。
对此，我也开始昼夜不停地徘徊。

<!-- 
The turning point for me was another BillG ThinkWeek paper I wrote, *Taming Side Effects*, in 2008.  In it, I described
a new type system that, little did I know at the time, would form the basis of my work for the next 5 years.  It wasn't
quite right, and was too tied up in my experiences with STM, but it was a decent start. 
-->
对我来说，转折点出现在我在2008年写的另一篇Bill Gates的
ThinkWeek文章——*Taming Side Effects（驯服副作用）*。
在那篇文章中，我描述了一种新型的系统，
却不知道那将成为我下一个五年所做工作的基础。
尽管里面有些地方不是很正确，
并且与我在STM中的经历过分地捆绑在一起，但这却是一个不错的开始。

<!-- 
Bill again concluded with a "We need to do this."  So I got to work! 
-->
比尔再次以“我们需要这样做”作为总结，
因此我开始了这方面的工作！

<!-- 
# Hello, Midori 
-->
# 你好，Midori

<!-- 
But there was still a huge problem.  I couldn't possibly imagine doing this work incrementally in the context of the
existing languages and runtimes.  I wasn't looking for a warm-and-cozy approximation of safety, but rather something
where, if your program compiled, you could know it was free of data races.  It needed to be bulletproof. 
-->
但仍然存在一个巨大的问题。
我无法想象在现有语言和运行时的上下文中逐步完成这项工作。
我并不是在寻找温暖和舒适的安全近似，而是在某些地方，如果您的程序编译，您可以知道它没有数据竞争。它需要防弹。

<!-- 
Well, actually, I tried.  I prototyped a variant of the system using C# custom attributes and static analysis, but
quickly concluded that the problems ran deep in the language and had to be integrated into the type system for any of
the ideas to work.  And for them to be even remotely usable.  Although we had some fun incubation projects at the time
(like [Axum](https://en.wikipedia.org/wiki/Axum_(programming_language))), given the scope of the vision, and for a
mixture of cultural and technical reasons, I knew this work needed a new home. 
-->
嗯，实际上，我试过了。我使用C＃自定义属性和静态分析对系统的变体进行了原型设计，但很快得出结论，问题深入到语言中，并且必须集成到类型系统中才能使任何想法发挥作用。而且它们甚至可以远程使用。虽然我们当时有一些有趣的孵化项目（如阿克苏姆），考虑到愿景的范围，并且由于文化和技术原因的混合，我知道这项工作需要一个新家。

<!-- 
So I joined Midori. 
-->
所以我加入了Midori。

<!-- 
## An Architecture, and An Idea 
-->
## 建筑和想法

<!-- 
A number of concurrency gurus were also on the Midori team, and I had been talking to them about all of this for a
couple years leading up to me joining.  At a top-level, we knew the existing foundation was the wrong one to bet on.
Shared-memory multithreading really isn't the future, we thought, and notably absent from all of my prior work was
fixing this problem.  The Midori team was set up exactly to tackle grand challenges and make big bets. 
-->
Midori团队也有很多并发大师，在我加入之前的几年里，我一直在和他们谈论所有这些。在顶级，我们知道现有的基础是错误的赌注。我们认为，共享内存多线程确实不是未来，而且我以前的所有工作中都缺少修复这个问题。 Midori团队的成立正是为了应对重大挑战并做出巨大努力。

<!-- 
So, we made some:
-->
所以，我们做了一些：


<!--
* Isolation is paramount, and we will embrace it wherever possible.

* Message passing will connect many such isolated parties through strongly typed RPC interfaces.

* Namely, inside of a process, there exists a single message loop, and, by default, no extra parallelism.

* A "promises-like" programming model will be first class so that:
    - Synchronous blocking is disallowed.
    - All asynchronous activity in the system is explicit.
    - Sophisticated coordination patterns are possible without resorting to locks and events. 
-->
* 隔离是至关重要的，我们将尽可能地接受隔离。
* 消息传递将通过强类型RPC接口连接许多这样的隔离方。
* 即，在进程内部，存在单个消息循环，并且默认情况下不存在额外的并行性。
* “承诺式”编程模型将是第一类，因此：
  - 不允许同步阻塞。
  - 系统中的所有异步活动都是显式的。
  - 无需借助锁和事件即可实现复杂的协调模式。

<!-- 
To reach these conclusions we were heavily inspired by [Hoare CSPs](
https://en.wikipedia.org/wiki/Communicating_sequential_processes), Gul Agha's and Carl Hewitt's work on [Actors](
https://en.wikipedia.org/wiki/Actor_model), [E](https://en.wikipedia.org/wiki/E_(programming_language)), [π](
https://en.wikipedia.org/wiki/%CE%A0-calculus), [Erlang](https://en.wikipedia.org/wiki/Erlang_(programming_language)),
and our own collective experiences building concurrent, distributed, and various RPC-based systems over the years. 
-->
为了得出这些结论，我们深受Hoare CSP，Gul Agha和Carl Hewitt关于Actors，E，π，Erlang的工作以及我们多年来构建并发，分布式和各种基于RPC的系统的集体经验的启发。

<!-- 
I didn't say this before, however message passing was notably absent in my work on PFX.  There were multiple reasons.
First, there were many competing efforts, and none of them "felt" right.  For instance, the [Concurrency and
Coordination Runtime (CCR)](https://en.wikipedia.org/wiki/Concurrency_and_Coordination_Runtime) was very complex (but
had many satisfied customers); the Axum language was, well, a new language; MSR's [Cω](
http://research.microsoft.com/en-us/um/cambridge/projects/comega/) was powerful, but required language changes which
some were hesitant to pursue (though the derivative library-only work, [Joins](
http://research.microsoft.com/en-us/um/people/crusso/joins/), held some promise); and so on.  Second, it didn't help
that everyone seemed to have a different idea on what the fundamental concepts should be. 
-->
我之前没有这么说过，但是在我的PFX工作中，消息传递显然不存在。原因有很多。首先，有许多相互竞争的努力，其中没有一个“感觉”正确。例如，并发和协调运行时（CCR）非常复杂（但有许多满意的客户）;阿克苏姆语是一种新语言; MSR的Cω是强大的，但需要语言变化，有些人犹豫不决（尽管衍生的图书馆工作，加入，有一些承诺）;等等。其次，每个人似乎都对基本概念应该有不同的看法并没有帮助。

<!-- 
But it really came down to isolation.  Windows processes are too heavyweight for the fine-grained isolation we thought
necessary to deliver safe, ubiquitous and easy message passing.  And no sub-process isolation technology on Windows was
really up for the task: [COM apartments](https://en.wikipedia.org/wiki/Component_Object_Model#Threading),
CLR AppDomains, ... many flawed attempts instantly come to mind; frankly, I did not want to die on that hill.
-->
但它真的归结为孤立。对于我们认为有必要提供安全，无处不在且容易传递消息的细粒度隔离，Windows进程过于重量级。 Windows上没有任何子进程隔离技术可以完成任务：COM公寓，CLR AppDomains，......许多有缺陷的尝试立即浮现在脑海中;坦率地说，我不想死在那座山上。

<!-- 
(Since then, I should note, there have been some nice efforts, like [Orleans](https://github.com/dotnet/orleans) --
built in part by some ex-Midori members -- [TPL Dataflow](
https://msdn.microsoft.com/en-us/library/hh228603(v=vs.110).aspx), and [Akka.NET](http://getakka.net/).  If you want
to do actors and/or message passing in .NET today, I recommend checking them out.) 
-->
（从那时起，我应该注意到，有一些很好的努力，比如奥尔良 - 部分是由一些前Midori成员构建的 -  TPL Dataflow和Akka.NET。如果你想在.NET中做演员和/或消息传递今天，我建议你把它们拿出去。）

<!-- 
Midori, on the other hand, embraced numerous levels of isolation, beginning with processes themselves, which were
even cheaper than Windows threads thanks to software isolation.  Even coarser-grained isolation was available in the
form of domains, adding added belts-and-suspenders hardware protection for hosting untrusted or logically separate code.
In the early days, we certainly wanted to go finer-grained too -- inspired by [E's concept of "vats"](
http://www.erights.org/elib/concurrency/vat.html), the abstraction we already began with for process message pumps --
but weren't sure how to do it safely.  So we waited on this.  But this gave us precisely what we needed for a robust,
performant, and safe message passing foundation. 
-->
另一方面，Midori采用了多种级别的隔离，从进程本身开始，由于软件隔离，它甚至比Windows线程便宜。甚至更粗粒度的隔离也以域的形式提供，为托管不可信或逻辑上独立的代码添加了附加的腰带和吊带硬件保护。在早期，我们当然也想要更细粒度 - 受到E的“大桶”概念的启发，我们已经开始对流程消息泵进行抽象 - 但不确定如何安全地进行。所以我们等了。但这恰恰为我们提供了一个强大，高效，安全的信息传递基础所需要的东西。

<!-- 
Important to the discussion of this architecture is the notion of [shared nothing](
https://en.wikipedia.org/wiki/Shared_nothing_architecture), something Midori leveraged as a core operating principle.
Shared nothing architectures are great for reliability, eliminating single points of failure, however they are great
for concurrency safety too.  If you don't share anything, there is no opportunity for race conditions!  (This is a bit
of a lie, and generally insufficient, as we shall see.) 
-->
对这种架构的讨论很重要的是无共享的概念，Midori将其作为核心操作原理。无共享架构非常适合可靠性，消除了单点故障，但它们对于并发安全性也非常有用。如果你不分享任何东西，就没有竞争条件的机会！ （这有点谎言，而且通常不够，正如我们将要看到的那样。）

<!-- 
It's interesting to note that we were noodling on this around the same time Node.js was under development.  The core
idea of an asynchronous, non-blocking, single process-wide event loop, is remarkably similar.  Perhaps something tasty
was in the water during 2007-2009.  In truth, many of these traits are common to [event-loop concurrency](
https://en.wikipedia.org/wiki/Event_loop). 
-->
值得注意的是，我们在Node.js正在开发的同时对此进行了研究。异步，非阻塞，单进程范围事件循环的核心思想非常相似。 2007-2009期间，水中也许有些美味可口。事实上，许多这些特征在事件循环并发中很常见。

<!-- 
This formed the canvas on top of which the entire concurrency model was painted.  I've already discussed this in the
[asynchronous everything](http://joeduffyblog.com/2015/11/19/asynchronous-everything/) article.  But there was more... 
-->
这形成了画布，其上绘制了整个并发模型。我已经在异步所有文章中讨论了这个问题。但还有更多......

<!-- 
## Why Not Stop Here? 
-->
## 为什么不停在这里？

<!-- 
It's a reasonable question.  A very robust system could be built with nothing more than the above, and I should say,
throughout multiple years of pounding away at the system, the above foundation stood the test of time and underwent
far fewer changes than what came next (syntax aside).  There is a simplicity to leaving it at this that I admire.  In
fact, with perfect hindsight, I believe stopping here would have been a reasonable story for "V1." 
-->
这是一个合理的问题。一个非常强大的系统可以构建，只有上述内容，我应该说，在多年的冲击系统中，上面的基础经受了时间的考验，并且经历了比下一个更少的更改（语法除外） 。我很钦佩，这样做很简单。事实上，凭​​借完美的后见之明，我相信在这里停下来对于“V1”来说是一个合理的故事。

<!-- 
However, a number of things kept us pushing for more: 
-->
然而，许多事情让我们更加努力：

<!-- 
* There was no sub-process parallelism.  Notably absent were task and data parallelism.  This was painful for a guy who
  had just come from building .NET's task and PLINQ programming models.  We had plenty of places that had latent
  parallelism just waiting to be unlocked, like image decoding, the multimedia pipeline, FRP rendering stack, browser,
  eventually speech recognition, and more.  One of Midori's top-level goals was to tackle the concurrency monster and,
  although a lot of parallelism came for "free" thanks to processes, the absence of task and data parallelism hurt. 
-->
没有子流程并行性。值得注意的是缺少任务和数据并行性。对于刚刚建立.NET任务和PLINQ编程模型的人来说，这是痛苦的。我们有很多地方潜伏并行等待解锁，如图像解码，多媒体管道，FRP渲染堆栈，浏览器，最终语音识别等等。 Midori的顶级目标之一是解决并发性怪物问题，尽管由于流程而导致许多并行性“免费”，但缺乏任务和数据并行性会受到影响。

<!-- 
* All messages between processes required RPC data marshaling, so rich objects could not be shared.  One solution to the
  absence of task parallelism could have been to model everything as processes.  Need a task?  Spawn a process.  In
  Midori, they were cheap enough for this to work.  Doing that, however, entailed marshaling data.  Not only could that
  be an expensive operation, not all types were marshalable, severely limiting parallelizable operations. 
-->
进程之间的所有消息都需要RPC数据封送，因此无法共享丰富的对象。没有任务并行性的一个解决方案可能是将所有内容建模为流程。需要一个任务？产生一个过程。在Midori，它们足够便宜，可以工作。但是，这样做需要编组数据。这不仅是一种昂贵的操作，并非所有类型都是可编组的，严重限制了可并行化的操作。

<!-- 
* In fact, an existing ["exchange heap"](http://read.seas.harvard.edu/cs261/2011/singularity.html) was developed for
  buffers, loosely based on the concept of linearity.  To avoid marshaling large buffers, we already had a system for
  exchanging them between processes without copying as part of the   RPC protocol.  This idea seemed useful enough to
  generalize and offer for higher-level data structures. 
-->
事实上，现在的“交换堆”是为缓冲区开发的，基于线性概念。为了避免编组大缓冲区，我们已经有了一个系统，用于在进程之间交换它们而不作为RPC协议的一部分进行复制。这个想法似乎足以概括和提供更高级别的数据结构。

<!-- 
* Even intra-process race conditions existed, due to multiple asynchronous activities in-flight and interleaving,
  despite the lack of data races thanks to the single message loop model described above.  A benefit   of the `await`
  model is that interleaving are at least visible and auditable in the source code; but they could still trigger
  concurrency errors.  We saw opportunities for the language and frameworks to help developers get this correct. 
-->
由于上面描述的单个消息循环模型，由于缺少数据竞争，由于飞行和交错中的多个异步活动，甚至存在进程内竞争条件。 await模型的一个好处是交错在源代码中至少是可见的和可审计的;但它们仍然可能触发并发错误。我们看到语言和框架的机会，以帮助开发人员正确。

<!-- 
* Finally, we also had a vague desire to have more immutability in the system.  Doing so could help with concurrency
  safety, of course, but we felt the language should also help developers get existing commonplace patterns
  correct-by-construction.  We also saw performance optimization opportunities if the compiler could trust immutability. 
-->
最后，我们还希望在系统中拥有更多的不变性。当然，这样做有助于提高并发安全性，但我们认为该语言还应该帮助开发人员正确地构建现有的普通模式。如果编译器可以信任不变性，我们也看到了性能优化机会。

<!-- 
We went back to academia and the ThinkWeek paper in search of inspiration.  These approaches, if combined in a tasteful
way, seemed like they could give us the tools necessary to deliver not only safe task and data parallelism, but also
finer-grained isolation, immutability, and tools to possibly address some of the intra-process race conditions. 
-->
我们回到学术界和ThinkWeek论文寻找灵感。这些方法如果以一种有品位的方式结合起来，似乎可以为我们提供必要的工具，不仅可以提供安全的任务和数据并行性，还可以提供更细粒度的隔离，不变性和可能解决某些进程内竞争的工具条件。

<!-- 
So, we forked the C# compiler, and went to town. 
-->
所以，我们分叉了C＃编译器，然后去了城镇。

<!-- 
## The Model 
-->
## 模型

<!-- 
In this section, I will rearrange the story to be a bit out of order.  (How appropriate.)  I'll first describe the
system we ended up with, after many years of work, in "tutorial style" rather than starting with the slightly messier
history of how we ended up there.  I hope this gives a more concise appreciation of the system.  I will then afterwards
give the complete historical account, including the dozens of systems that came before which influenced us greatly. 
-->
在本节中，我将重新安排故事有点乱。 （多么合适。）我将首先用“教程风格”来描述我们最终得到的系统，而不是从我们如何在那里结束的稍微混乱的历史开始。我希望这会对系统有一个更简洁的认识。然后我将提供完整的历史记录，包括之前影响我们的数十个系统。

<!-- 
We started with C#'s type system and added two key concepts: permission and ownership. 
-->
我们从C＃的类型系统开始，并添加了两个关键概念：权限和所有权。

<!-- 
### Permission 
-->
### 允许

<!-- 
The first key concept was *permission*. 
-->
第一个关键概念是许可。

<!-- 
Any reference could have one and it governed what you could do with the referent object: 
-->
任何引用都可以有一个，它控制你可以用引用对象做什么：

<!-- 
* `mutable`: The target object (graph) can be mutated through the usual ways.
* `readonly`: The target object (graph) can be read from but cannot be mutated.
* `immutable`: The target object (graph) can be read from and will *never* be mutated. 
-->
* mutable：目标对象（图形）可以通过常规方式进行变异。
* readonly：可以读取目标对象（图形）但不能进行变异。
* immutable：可以读取目标对象（图形），并且永远不会发生变异。

<!-- 
A [subtyping relationship](https://en.wikipedia.org/wiki/Subtyping) meant you could implicitly convert either
`mutable` or `immutable` to `readonly`.  In other words, `mutable <: readonly` and `immutable <: readonly`. 
-->
子类型关系意味着您可以隐式地将可变或不可变转换为只读。换句话说，mutable <：readonly和immutable <：readonly。

<!-- 
For example: 
-->
例如：

<!-- 
    Foo m = new Foo(); // mutable by default.
    
    immutable Foo i = new Foo(); // cannot ever be mutated.
    i.Field++; // error: cannot mutate an immutable object.
    
    readonly Foo r1 = m; // ok; cannot be mutated by this reference.
    r1.Field++; // error: cannot mutate a readonly object.
    
    readonly Foo r2 = i; // ok; still cannot be mutated by this reference.
    r2.Field++; // error: cannot mutate a readonly object. 
-->
    Foo m = new Foo(); // 默认是可变的。
    
    immutable Foo i = new Foo(); // 永远不会发生变异。
    i.Field++; // 错误：无法改变不可变对象。
    
    readonly Foo r1 = m; // 好; 不能被这个引用变异。
    r1.Field++; // 错误：无法改变只读对象。
    
    readonly Foo r2 = i; // 好; 仍然不能通过这个引用变异。
    r2.Field++; // 错误：无法改变只读对象。

<!-- 
These are guarantees, enforced by the compiler and subject to [verification](
https://en.wikipedia.org/wiki/Typed_assembly_language). 
-->
这些是由编译器强制执行的保证，并受[验证]的约束（
https://en.wikipedia.org/wiki/Typed_assembly_language）。

<!-- 
The default, if unstated, was `immutable` for primitive types like `int`, `string`, etc., and `mutable` for all others.
This preserved existing C# semantics in almost all scenarios.  (That is, C# compiled as-is had no change in meaning.)
This was contentious but actually a pretty cool aspect of the system.  It was contentious because the principle of least
authority would lead you to choose `readonly` as the default.  It was cool because you could take any C# code and start
incrementally sprinkling in permissions where they delivered value.  If we had decided to break from C# more radically
-- something in hindsight we should have done -- then breaking with compatibility and choosing the safer default would
have been the right choice; but given our stated goals of C# compatibility, I think we made the right call. 
-->
如果没有说明，默认值对于诸如int，string等原始类型是不可变的，对于所有其他类型是可变的。 这几乎在所有场景中都保留了现有的C＃语义。 （也就是说，按原样编译的C＃没有意义上的变化。）这是有争议的，但实际上是系统的一个非常酷的方面。 这是有争议的，因为最小权限原则会导致您选择只读作为默认值。 它很酷，因为你可以使用任何C＃代码并开始逐步增加权限，在这些代码中它们可以传递价值。 如果我们决定更彻底地摆脱C＃ - 事后我们应该做的事情 - 然后打破兼容性并选择更安全的默认值将是正确的选择; 但鉴于我们声明的C＃兼容性目标，我认为我们做出了正确的决定。


<!-- 
These permissions could also appear on methods, to indicate how the `this` parameter got used:
-->
这些权限也可以出现在方法上，以指示如何使用此参数：

    class List<T> {
        void Add(T e);
        int IndexOf(T e) readonly;
        T this[int i] { readonly get; set; }
    } 

<!-- 
A caller would need a sufficient permission in order to invoke a method: 
-->
调用者需要足够的权限才能调用方法：

<!-- 
    readonly List<Foo> foos = ...;
    foos[0] = new Foo(); // error: cannot mutate a readonly object. 
-->
    readonly List<Foo> foos = ...;
    foos[0] = new Foo(); // 错误：无法改变只读对象。

<!-- 
A similar thing could be stated using delegate types and lambdas.  For example: 
-->
可以使用委托类型和lambdas来声明类似的事情。 例如：

    delegate void PureFunc<T>() immutable; 

<!-- 
This meant that a lambda conforming to the `PureFunc` interface could only close over `immutable` state. 
-->
这意味着符合`PureFunc`接口的lambda只能关闭`immutable`状态。

<!-- 
Notice how powerful this has suddenly become!  This `PureFunc` is precisely what we would want for a parallel task.  As
we will see shortly, these simple concepts alone are enough to enable many of those PFX abstractions to become safe. 
-->
请注意这突然变得多么强大！ 这个PureFunc正是我们想要的并行任务。 我们很快就会看到，仅仅这些简单的概念足以使许多PFX抽象变得安全。

<!-- 
By default, permissions are "deep", in that they apply transitively, to the entire object graph.  This interacts with
generics in the obvious way, however, so that you could, for example, have combinations of deep and shallow permissions: 
-->
默认情况下，权限是“深度”，因为它们可传递地应用于整个对象图。 但是，这会以明显的方式与泛型交互，因此您可以使用深层和浅层权限的组合：

<!-- 
    readonly List<Foo> foos = ...;             // a readonly list of mutable Foos.
    readonly List<readonly Foo> foos = ...;    // a readonly list of readonly Foos.
    immutable List<Foo> foos = ...;            // an immutable list of mutable Foos.
    immutable List<immutable Foo> foos = ...;  // an immutable list of immutable Foos.
    // and so on... 
-->
    readonly List<Foo> foos = ...;             // 一份可变食物的简单清单。
    readonly List<readonly Foo> foos = ...;    // readonly Foos的只读列表。
    immutable List<Foo> foos = ...;            // 可变食物的不可变列表。
    immutable List<immutable Foo> foos = ...;  // 不可变的Foos的不可变列表。
    // 等等 ... 
<!-- 
Despite this working, and appearing obvious, man was this a difficult thing to get right! 
-->
尽管这很有效，而且看起来显而易见，但是对于正确的人来说，这是一件困难的事情！

<!-- 
For power users, we also had a way to write generic types that parameterized over permissions.  This was definitely
required deep in the bowels of highly generic code, but otherwise could be ignored by 90% of the system's users: 
-->
对于高级用户，我们还有一种方法可以编写通过权限参数化的泛型类型。 这在高度通用的代码中绝对是必需的，但在90％的系统用户中可能会被忽略：

<!-- 
    delegate void PermFunc<T, U, V, permission P>(P T, P U, P V);

    // Used elsewhere; expands to `void(immutable Foo, immutable Bar, immutable Baz)`:
    PermFunc<Foo, Bar, Baz, immutable> func = ...; 
-->
    delegate void PermFunc<T, U, V, permission P>(P T, P U, P V);

    // 用于别处; 扩展为`void(immutable Foo, immutable Bar, immutable Baz)`：
    PermFunc<Foo, Bar, Baz, immutable> func = ...; 

<!-- 
I should also note that, for convenience, you could mark a type as `immutable` to indicate "all instances of this type
are immutable."  This was actually one of the most popular features of all of this.  At the end of the day, I'd estimate
that 1/4-1/3 of all types in the system were marked as immutable: 
-->
我还应该注意，为方便起见，您可以将类型标记为不可变，以指示“此类型的所有实例都是不可变的。”这实际上是所有这些中最受欢迎的功能之一。 在一天结束时，我估计系统中所有类型的1 / 4-1 / 3被标记为不可变：

    immutable class Foo {...}
    immutable struct Bar {...} 

<!-- 
There is an interesting twist.  As we'll see below, `readonly` used to be called `readable`, and was entirely distinct.
But after we left Midori and were hard at work trying to ready these concepts for inclusion in C#, we decided to try and
unify them.  So that's what I am presenting here.  The only hitch is that `readonly` would be given a slightly different
meaning.  On a field, `readonly` today means "the value cannot be changed"; in the case of a pointer, therefore, the
`readonly` of today did not impact the referent object graph.  In this new model, it would.  Given that we anticipated
an opt-in flag, `--strict-mutability`, this would be acceptable, and would require `readonly mutable`, a slight wart, to
get the old behavior.  This wasn't a deal-breaker to me -- especially given that a very common bug in C# is developers
assuming that `readonly` is deep (which now it would be), and obvious similarities to `const` come to mind. 
-->
有一个有趣的转折。 我们将在下面看到，readonly曾经被称为可读，并且完全不同。 但是在我们离开Midori并且努力工作以试图将这些概念包含在C＃中之后，我们决定尝试将它们统一起来。 这就是我在这里呈现的内容。 唯一的障碍是，readonly会有一个略微不同的含义。 在一个领域，今天只读意味着“价值无法改变”; 因此，在指针的情况下，今天的readonly不影响指示对象图。 在这个新模型中，它会。 鉴于我们预期了一个选择加入标志--strict-mutability，这是可以接受的，并且需要readonly mutable，一个轻微的疣，以获得旧的行为。 这对我来说不是一个交易破坏者 - 特别是考虑到C＃中一个非常常见的错误是开发人员认为readonly很深（现在它会是这样），并且会想到与const的明显相似之处。

<!-- 
### Ownership 
-->
### 所有权

<!-- 
The second key concept was *ownership*. 
-->
第二个关键概念是所有权。

<!-- 
A reference could be given an ownership annotation, just as it could be given a permission: 
-->
可以为引用提供所有权注释，就像它可以获得权限一样：

<!-- 
* `isolated`: The target object (graph) forms an unaliased transitive closure of state.

 For example: 
-->

隔离：目标对象（图形）形成一个非混淆的传递状态闭包。
例如：


    isolated List<int> builder = new List<int>(); 


<!-- 
Unlike permissions, which indicate what operations are legal on a given reference, ownership annotations told us
important aliasing properties about the given object graphs.  An isolated graph has a single "in-reference" to the root
object in the object graph, and no "out-references" (except for immutable object references, which are permitted). 
-->
与权限不同，权限指示哪些操作在给定引用上是合法的，所有权注释告诉我们关于给定对象图的重要别名属性。 隔离图对象图中的根对象有一个“引用”，没有“out-references”（允许的不可变对象引用除外）。


<!-- 
A visual aid might help to conceptualize this: 
-->
视觉辅助可能有助于概念化：

![Isolation Bubbles](/assets/img/2016-11-30-15-years-of-concurrency.isolated-bubble.jpg)
<!-- 
Given an isolated object, we can mutate it in-place: 
-->
给定一个孤立的对象，我们可以就地改变它：

    for (int i = 0; i < 42; i++) {
        builder.Add(i);
    } 

<!-- 
And/or destroy the original reference and transfer ownership to a new one: 
-->
和/或销毁原始参考并将所有权转让给新参考：


    isolated List<int> builder2 = consume(builder); 

<!-- 
The compiler from here on would mark `builder` as uninitialized, though if it is stored in the heap multiple possible
aliases might lead to it, so this analysis could never be bulletproof.  In such cases, the original reference would be
`null`ed out to avoid safety gotchas.  (This was one of many examples of making compromises in order to integrate more
naturally into the existing C# type system.) 
-->
此处的编译器会将构建器标记为未初始化，但如果它存储在堆中，则多个可能的别名可能会导致它，因此此分析永远不会是防弹的。 在这种情况下，原始参考将被删除以避免安全陷阱。 （这是为了更好地集成到现有的C＃类型系统中而做出妥协的众多例子之一。）

<!-- 
It's also possible to destroy the isolated-ness, and just get back an ordinary `List<int>`: 
-->
也可以破坏隔离，然后返回一个普通的List <int>：

    List<int> built = consume(builder); 

<!-- 
This enabled a form of linearity that was useful for safe concurrency -- so objects could be handed off safely,
subsuming the special case of the exchange heap for buffers -- and also enabled patterns like builders that laid the
groundwork for strong immutability. 
-->
这启用了一种对安全并发有用的线性形式 - 因此对象可以安全地传递，包含缓冲区交换堆的特殊情况 - 并且还启用了类似构建器的模式，为强大的不变性奠定了基础。

<!-- 
To see why this matters for immutability, notice that we skipped over exactly how an immutable object gets created.
For it to be safe, the type system needs to prove that no other `mutable` reference to that object (graph) exists at a
given time, and will not exist forever.  Thankfully that's precisely what `isolated` can do for us! 
-->
要了解为什么这对于不变性很重要，请注意我们完全跳过了如何创建不可变对象。 为了安全起见，类型系统需要证明在给定时间不存在对该对象（图形）的其他可变引用，并且不会永远存在。 值得庆幸的是，这正是隔离可以为我们做的！

    immutable List<int> frozen = consume(builder); 


<!-- 
Or, more concisely, you're apt to see things like:
-->
或者，更简洁地说，您很容易看到以下内容：

    immutable List<int> frozen = new List<int>(new[] { 0, ..., 9 }); 

<!-- 
In a sense, we have turned our isolation bubble (as shown earlier) entirely green: 
-->
从某种意义上说，我们将隔离泡沫（如前所示）完全变为绿色：

![Immutability from Isolation Bubbles](/assets/img/2016-11-30-15-years-of-concurrency.immutable-bubble.jpg)
<!-- 
Behind the scenes, the thing powering the type system here is `isolated` and ownership analysis.  We will see more of
the formalisms at work in a moment, however there is a simple view of this: all inputs to the `List<int>`'s constructor
are `isolated` -- namely, in this case, the array produced by `new[]` -- and therefore the resulting `List<int>` is too. 
-->
在幕后，这里为类型系统供电的东西是孤立的和所有权分析。 我们将在稍后看到更多的形式，但是有一个简单的视图：List <int>的构造函数的所有输入都是隔离的 - 即，在这种情况下，new []生成的数组 -  因此产生的List <int>也是如此。

<!-- 
In fact, any expression consuming only `isolated` and/or `immutable` inputs and evaluating to a `readonly` type was
implicitly upgradeable to `immutable`; and, a similar expression, evaluating to a `mutable` type, was upgradeable to
`isolated`. This meant that making new `isolated` and `immutable` things was straightforward using ordinary expressions. 
-->
实际上，任何只消耗孤立和/或不可变输入并且只读取只读类型的表达式都可以隐式升级为不可变; 并且，评估为可变类型的类似表达式可升级为隔离。 这意味着使用普通表达式创建新的孤立和不可变的东西是直截了当的。

<!-- 
The safety of this also depends on the elimination of ambient authority and leaky construction. 
-->
这种安全性还取决于消除环境权限和泄漏结构。

<!-- 
### No Ambient Authority 
-->
### 没有环境权威

<!-- 
A principle in Midori was the elimination of [ambient authority](https://en.wikipedia.org/wiki/Ambient_authority).
This enabled [capability-based security](/2015/11/10/objects-as-secure-capabilities/), however in a subtle way was also
necessary for immutability and the safe concurrency abstractions that are to come. 
-->
Midori的一个原则是消除环境权威。 这启用了基于功能的安全性，但是以一种微妙的方式对于不可变性和即将到来的安全并发抽象也是必需的。

<!-- 
To see why, let's take our `PureFunc` example from earlier.  This gives us a way to reason locally about the state
captured by a lambda.  In fact, a desired property was that functions accepting only `immutable` inputs would result in
[referential transparency](https://en.wikipedia.org/wiki/Referential_transparency), unlocking a number of [novel
compiler optimizations](http://joeduffyblog.com/2015/12/19/safe-native-code/) and making it easier to reason about code. 
-->
为了了解原因，让我们从之前的PureFunc示例中获取。 这为我们提供了一种在本地推理lambda捕获的状态的方法。 事实上，一个理想的属性是，仅接受不可变输入的函数将导致引用透明性，解锁许多新颖的编译器优化并使得更容易推理代码。

<!-- 
However, if mutable statics still exist, the invocation of that `PureFunc` may not actually be pure!

For example: 
-->
但是，如果仍然存在可变静态，那么PureFunc的调用可能实际上并不纯粹！

例如：


    static int x = 42;

    PureFunc<int> func = () => x++; 

<!-- 
From the type system's point of view, this `PureFunc` has captured no state, and so it obeys the immutable capture
requirement.  (It may be tempting to say that we can "see" the `x++`, and therefore can reject the lambda, however of
course this `x++` might happen buried deep down in a series of virtual calls, where it is invisible to us.) 
-->
从类型系统的角度来看，这个PureFunc没有捕获任何状态，因此它遵循不可变的捕获要求。 （可能很有可能说我们可以“看到”x ++，因此可以拒绝lambda，但是当然这个x ++可能会发生在一系列虚拟调用中，这些虚拟调用对我们来说是不可见的。）

<!-- 
All side-effects need to be exposed to the type system.  Over the years, we explored extra annotations to say "this
function has mutable access to static variables"; however, the `mutable` permission is already our way of doing that,
and felt more consistent with the overall stance on ambient authority that Midori took. 
-->
所有副作用都需要暴露于类型系统。多年来，我们探索了额外的注释来说“这个函数具有对静态变量的可变访问”;然而，可变许可已经是我们这样做的方式，并且感觉更符合Midori所采取的环境权威的整体立场。

<!-- 
As a result, we eliminated all ambient side-effectful operations, leveraging capability objects instead.  This obviously
covered I/O operations -- all of which were asynchronous RPC in our system -- but also even -- somewhat radically --
meant that even just getting the current time, or generating a random number, required a capability object.  This let
us model side-effects in a way the type-system could see, in addition to reaping the other benefits of capabilities. 
-->
因此，我们消除了所有环境副作用操作，而是利用功能对象。这显然涵盖了I / O操作 - 所有这些操作都是我们系统中的异步RPC  - 但是甚至 - 有点根本 - 意味着即使只是获取当前时间或生成随机数，也需要一个能力对象。这让我们以类型系统可以看到的方式模拟副作用，此外还可以获得功能的其他好处。

<!-- 
This meant that all statics must be immutable.  This essentially brought C#'s `const` keyword to all statics: 
-->
这意味着所有静态都必须是不可变的。这基本上将C＃的const关键字带到所有静态：

    const Map<string, int> lookupTable = new Map<string, int>(...); 


<!-- 
In C#, `const` is limited to primitive constants, like `int`s, `bool`s, and `string`s.  Our system expanded this same
capability to arbitrary types, like lists, maps, ..., anything really. 
-->
在C＃中，const仅限于原始常量，如int，bools和字符串。我们的系统将相同的功能扩展到任意类型，例如列表，地图，......，真的。

<!-- 
Here's where it gets interesting.  Just like C#'s current notion of `const`, our compiler evaluated all such objects at
compile-time and froze them into the readonly segment of the resulting binary image.  Thanks to the type system's
guarantee that immutability meant immutability, there was no risk of runtime failures as a result of doing so. 
-->
这是有趣的地方。就像C＃目前的const概念一样，我们的编译器在编译时评估了所有这些对象，并将它们冻结到生成的二进制图像的只读段中。由于类型系统保证不变性意味着不变性，因此不存在运行时故障的风险。

<!-- 
Freezing had two fascinating performance consequences.  First, we could share pages across multiple processes, cutting
down on overall memory usage and TLB pressure.  (For instance, lookup tables held in maps were automatically shared
across all programs using that binary.)  Second, we were able to eliminate all class constructor accesses, replacing
them with constant offsets, [leading to more than a 10% reduction in code size across the entire OS along with
associated speed improvements](http://joeduffyblog.com/2015/12/19/safe-native-code/), particularly at startup time. 
-->
冻结有两个令人着迷的性能后果。首先，我们可以跨多个进程共享页面，减少总体内存使用量和TLB压力。 （例如，使用该二进制文件在所有程序中自动共享地图中保存的查找表。）其次，我们能够消除所有类构造函数访问，用常量偏移替换它们，导致代码大小减少10％以上整个操作系统以及相关的速度改进，特别是在启动时。

<!-- 
Mutable statics sure are expensive! 
-->
可变静力肯定是昂贵的！

<!-- 
### No Leaky Construction 
-->
### 没有漏水的建筑

<!-- 
This brings us to the second "hole" that we need to patch up: leaky constructors. 
-->
这带来了我们需要修补的第二个“漏洞”：漏洞的构造函数。

<!-- 
A leaky constructor is any constructor that shares `this` before construction has finished.  Even if it does so at the
"very end" of its own constructor, due to inheritance and constructor chaining, this isn't even guaranteed to be safe. 
-->
漏洞构造函数是在构造完成之前共享它的任何构造函数。即使它在自己的构造函数的“最终”处这样做，由于继承和构造函数链接，这甚至不能保证是安全的。

<!-- 
So, why are leaky constructors dangerous?  Mainly because they expose other parties to [partially constructed objects](
http://joeduffyblog.com/2010/06/27/on-partiallyconstructed-objects/).  Not only are such objects' invariants suspect,
particularly in the face of construction failure, however they pose a risk to immutability too. 
-->
那么，泄漏的构造函数为何危险？主要是因为他们将其他方暴露给部分构建的对象。这些物体的不变量不仅是可疑的，特别是在施工失败的情况下，它们也会对不变性造成风险。

<!-- 
In our particular case, how are we to know that after creating a new supposedly-immutable object, someone isn't
secretively holding on to a mutable reference?  In that case, tagging the object with `immutable` is a type hole. 
-->
在我们的特定情况下，我们如何知道在创建一个新的所谓不可变对象后，有人不会秘密地持有一个可变引用？在这种情况下，使用immutable标记对象是一个类型孔。

<!-- 
We banned leaky constructors altogether.  The secret?  A special permission, `init`, that meant the target object is
undergoing initialization and does not obey the usual rules.  For example, it meant fields weren't yet guaranteed to be
assigned to, non-nullability hadn't yet been established, and that the reference could *not* convert to the so-called
"top" permission, `readonly`.   Any constructor got this permission by default and you couldn't override it.  We also
automatically used `init` in select areas where it made the language work more seamlessly, like in object initializers. 
-->
我们完全禁止泄漏的构造函数。秘密？特殊权限init，意味着目标对象正在进行初始化，并且不遵守通常的规则。例如，它意味着尚未保证分配字段，尚未建立非可空性，并且该引用无法转换为所谓的“顶级”权限，只读。默认情况下，任何构造函数都获得此权限，您无法覆盖它。我们还在选择区域中自动使用init，使语言更加无缝地工作，就像在对象初始化器中一样。

<!-- 
This had one unfortunate consequence: by default, you couldn't invoke other instance methods from inside a constructor.
(To be honest, this was actually a plus in my opinion, since it meant you couldn't suffer from partially constructed
objects, couldn't accidentally [invoke virtuals from a constructor](
https://www.securecoding.cert.org/confluence/display/cplusplus/OOP50-CPP.+Do+not+invoke+virtual+functions+from+constructors+or+destructors),
and so on.)  In most cases, this was trivially worked around.  However, for those cases where you really needed to call
an instance method from a constructor, we let you mark methods as `init` and they would take on that permission. 
-->
这有一个不幸的后果：默认情况下，您无法从构造函数中调用其他实例方法。 （老实说，在我看来，这实际上是一个加分，因为它意味着你不会受到部分构造的对象的影响，不会意外地从构造函数中调用虚拟对象，等等。）在大多数情况下，这是非常简单的工作周围。但是，对于那些真正需要从构造函数调用实例方法的情况，我们允许您将方法标记为init，并且它们将接受该权限。

<!-- 
### Formalisms and Permission Lattices 
-->
### 形式主义和许可格式

<!-- 
Although the above makes intuitive sense, there was a formal type system behind the scenes. 
-->
虽然上面有直观的意义，但幕后有一个正式的类型系统。

<!--  
Being central to the entire system, we partnered with MSR to prove the soundness of this approach, especially
`isolated`, and published the paper in [OOPSLA'12](http://dl.acm.org/citation.cfm?id=2384619) (also available as a free
[MSR tech report](http://research-srv.microsoft.com/pubs/170528/msr-tr-2012-79.pdf)).  Although the paper came out a
couple years before this final model solidified, most of the critical ideas were taking shape and well underway by then.
-->
作为整个系统的核心，我们与MSR合作证明了这种方法的可靠性，特别是孤立的，并在OOPSLA'12中发表了论文（也可作为免费的MSR技术报告提供）。尽管该论文在最终模型凝固之前几年出现，但大多数关键思想正在形成并且正在进行中。

<!-- 
For a simple mental model, however, I always thought about things in terms of subtyping and substitution. 
-->
然而，对于一个简单的心智模型，我总是在分类和替换方面考虑事物。

<!-- 
In fact, once modeled this way, most implications to the type system "fall out" naturally.  `readonly` was the "top
permission", and both `mutable` and `immutable` convert to it implicitly.  The conversion to `immutable` was a delicate
one, requiring `isolated` state, to guarantee that it obeyed the immutability requirements.  From there, all of the
usual implications follow, including [substitution](https://en.wikipedia.org/wiki/Liskov_substitution_principle),
[variance](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)), and their various impact to
conversions, overrides, and subtypes. 
-->
实际上，一旦以这种方式建模，对类型系统的大多数影响自然会“脱落”。 readonly是“顶级权限”，并且mutable和immutable都隐式转换为它。转换为不可变的是一个微妙的，需要隔离状态，以保证它遵守不变性要求。从那里开始，所有常见的含义都会随之而来，包括替换，差异以及它们对转换，覆盖和子类型的各种影响。

<!-- 
This formed a two-dimensional lattice wherein one dimension was "type" in the classical sense
and the other "permission", such that all types could convert to `readonly Object`.  This diagram illustrates: 
-->
这形成了二维点阵，其中一个维度在经典意义上是“类型”而另一个是“许可”，使得所有类型都可以转换为只读对象。该图表说明：

![Permission Lattice](/assets/img/2016-11-30-15-years-of-concurrency.lattice.jpg)

<!-- 
The system could obviously be used without any knowledge of these formalisms.  However, I had lived through
enough sufficiently [scary, yet subtle, security problems over the years due to type system gotchas](
https://www.microsoft.com/en-us/research/wp-content/uploads/2007/01/appsem-tcs.pdf), so going the extra mile and doing
the formalism not only helped us understand our own system better, but also helped us sleep better at night. 
-->
显然可以在不了解这些形式的情况下使用该系统。 然而，由于类型系统陷阱，我多年来经历了足够多的可怕但又微妙的安全问题，所以加倍努力并做正式主义不仅帮助我们更好地理解我们自己的系统，而且还帮助我们更好地睡在 晚。

<!-- 
### How This Enabled Safe Concurrency 
-->
### 如何实现安全并发

<!-- 
New type system in hand, we can now go back and revisit those PFX abstractions, and make them all safe. 
-->
手中有新型系统，我们现在可以回过头来重新审视那些PFX抽象，并使它们都安全。

<!-- 
The essential property we must establish is that, when an activity has `mutable` rights to a given object, then that
object must not be simultaneously accessible to any other activities.  Notice that I am using the term "activity"
deliberately.  For now, imagine this maps directly to "task", although we shall return to this subtlety momentarily.
Also note that I've said "object"; that too is a gross simplification, since for certain data structures like arrays,
simply ensuring that activities do not have `mutable` rights to overlapping regions is sufficient. 
-->
我们必须建立的基本属性是，当一个活动对给定对象具有可变权限时，该对象不能同时被任何其他活动访问。请注意，我故意使用“活动”一词。现在，想象一下这个地图直接映射到“任务”，虽然我们将暂时回到这个微妙的地方。还要注意我说的“对象”;这也是一种粗略的简化，因为对于像数组这样的某些数据结构，只需确保活动不具有重叠区域的可变权限就足够了。

<!-- 
Beyond what this disallows, it actually allows for some interesting patterns.  For instance, any number of concurrent
activities may share `readonly` access to the same object.  (This is a bit like a reader/writer lock, just without any
locks or runtime overheads.)  Remember that we can convert `mutable` to `readonly`, which means that, given an activity
with `mutable` access, we can use fork/join parallelism that captures an object with `readonly` permissions, provided
the mutator is temporally paused for the duration of this fork/join operation. 
-->
除了这种不允许之外，它实际上允许一些有趣的模式。例如，任何数量的并发活动可以共享对同一对象的只读访问。 （这有点像读取器/写入器锁，只是没有任何锁或运行时开销。）请记住，我们可以将mutable转换为readonly，这意味着，给定一个具有可变访问权限的活动，我们可以使用fork / join并行性捕获具有只读权限的对象，前提是mutator在此fork / join操作的持续时间内暂时暂停。

<!-- 
Or, in code: 
-->
或者，在代码中：

    int[] arr = ...;
    int[] results = await Parallel.Fork(
        () => await arr.Reduce((x, y) => x+y),
        () => await arr.Reduce((x, y) => x*y)
    ); 


<!-- 
This code computes the sum and product of an array, in parallel, by merely reading from it.  It is data race-free. 
-->
此代码仅通过读取就可以并行计算数组的总和和乘积。 它是无数据竞争的。

<!-- 
How so?  This example `Fork` API uses permissions to enforce the required safety: 
-->
怎么会这样？ 此示例Fork API使用权限来强制执行所需的安全性：

    public static async T[] Fork<T>(params ForkFunc<T>[] funcs);
    
    public async delegate T ForkFunc<T>() readonly; 

<!-- 
Let's break this apart.  `Fork` simply takes an array of `ForkFunc`s.  Since `Fork` is `static`, we needn't worry about
it capturing state dangerously.  But `ForkFunc` is a delegate and can be satisfied by instance methods and lambdas,
both of which can close over state.  By marking the `this` position as `readonly`, we limit the captures to `readonly`;
so, although the lambdas can capture `arr` in our above example, they cannot mutate it.  And that's it. 
-->
让我们分开吧。 Fork只需要一个ForkFuncs数组。由于Fork是静态的，我们不必担心它会危险地捕获状态。但是ForkFunc是一个委托，可以通过实例方法和lambda来满足，这两种方法都可以关闭状态。通过将此位置标记为只读，我们将捕获限制为只读;所以，虽然lambdas可以在上面的例子中捕获arr，但是它们不能改变它。就是这样。

<!-- 
Notice too that the nested function `Reduce` can also be run in parallel, thanks to `ForkFunc`!  Obviously, all of
the familiar `Parallel.For`, `Parallel.ForEach`, and friends, can enjoy similar treatment, with similar safety. 
-->
还要注意，嵌套函数Reduce也可以并行运行，这要归功于ForkFunc！显然，所有熟悉的Parallel.For，Parallel.ForEach和朋友都可以享受类似的安全性。

<!-- 
It turns out most fork/join patterns, where we can guarantee the mutator is paused, work this way.  All of PLINQ, for
example, can be represented this way, with complete data-race freedom.  This was the use-case I always had in mind. 
-->
事实证明，大多数fork / join模式，我们可以保证mutator暂停，以这种方式工作。例如，所有PLINQ都可以用这种方式表示，具有完全的数据竞争自由度。这是我一直想到的用例。

<!-- 
In fact, we can now introduce [automatic parallelism](https://en.wikipedia.org/wiki/Automatic_parallelization)!  There
are a few ways to go about this.  One way is to never offer LINQ operators that aren't protected by `readonly`
annotations.  This was always my preferred approach, given the absurdity of having query operators performing mutations.
But other approaches were possible.  One is to offer overloads -- one set of `mutable` operators, one set of `readonly`
operators -- and the compiler's overload resolution would then pick the one with the least permission that type-checked. 
-->
实际上，我们现在可以引入自动并行！有几种方法可以解决这个问题。一种方法是永远不要提供不受readonly注释保护的LINQ运算符。考虑到查询运算符执行突变的荒谬性，这始终是我的首选方法。但其他方法也是可能的。一种是提供重载 - 一组可变操作符，一组只读操作符 - 然后编译器的重载决策将选择具有类型检查权限最少的那个。

<!-- 
As mentioned earlier, tasks are even simpler than this: 
-->
如前所述，任务甚至比这简单：

    public static Task<T> Run<T>(PureFunc<T> func); 


<!-- 
This accepts our friend from earlier, `PureFunc`, that is guaranteed to be referentially transparent.  Since tasks do
not have structured lifetime like our fork/join and data parallel friends above, we cannot permit capture of even
`readonly` state.  Remember, the trick that made the above examples work is that the mutator was temporarily paused,
something we cannot guarantee here with unstructured task parallelism. 
-->
这接受了我们之前的朋友，PureFunc，保证是引用透明的。 由于任务没有像上面的fork / join和数据并行朋友那样的结构化生命周期，我们不能允许捕获甚至只读状态。 请记住，使上述示例有效的技巧是mutator暂时暂停，这是我们无法保证的非结构化任务并行性。

<!-- 
So, what if a task needs to party on mutable state? 
-->
那么，如果任务需要在可变状态下聚会怎么办？

<!-- 
For that, we have `isolated`!  There are various ways to encode this, however, we had a way to mark delegates to
indicate they could capture `isolated` state too (which had the side-effect of making the delegate itself `isolated`): 
-->
为此，我们已经孤立了！ 有各种方法对此进行编码，但是，我们有一种方法来标记委托，以表明它们也可以捕获隔离状态（这会产生使委托本身隔离的副作用）：

    public static Task<T> Run<T>(TaskFunc<T> func);
    
    public async delegate T TaskFunc<T>() immutable isolated; 


<!-- 
Now we can linearly hand off entire object graphs to a task, either permanently or temporarily:
-->
现在我们可以永久或暂时将整个对象图线性地移交给任务：

<!-- 
    isolated int[] data = ...;
    Task<int> t = Task.Run([consume data]() => {
        // in here, we own `data`.
    }); 
-->
    isolated int[] data = ...;
    Task<int> t = Task.Run([consume data]() => {
        // 在这里，我们拥有`data`。
    }); 

<!-- 
Notice that we leveraged lambda capture lists to make linear captures of objects straightforward.  There's an [active
proposal](https://github.com/dotnet/roslyn/issues/117) to consider adding a feature like this to a future C#, however
without many of the Midori features, it remains to be seen whether the feature stands on its own. 
-->
请注意，我们利用lambda捕获列表直接捕获对象的线性。有一个积极的建议考虑将这样的功能添加到未来的C＃中，但是没有很多Midori功能，这个功能是否独立存在还有待观察。

<!-- 
Because of the rules around `isolation` production, `mutable` objects produced by tasks could become `isolated`, and
`readonly` object could be frozen to become `immutable`.  This was tremendously powerful from a composition standpoint. 
-->
由于围绕隔离生成的规则，任务产生的可变对象可能变得孤立，而只读对象可能被冻结成不可变的。从构图的角度来看，这是非常强大的。

<!-- 
Eventually, we created higher level frameworks to help with data partitioning, non-uniform data parallel access to
array-like structures, and more.  All of it free from data races, deadlocks, and the associated concurrency hazards. 
-->
最终，我们创建了更高级别的框架来帮助进行数据分区，对数组结构进行非均匀数据并行访问等等。所有这些都没有数据争用，死锁和相关的并发危险。

<!-- 
Although we designed what running subsets of this on a GPU would look like, I would be lying through my teeth if I
claimed we had it entirely figured out.  All that I can say is understanding the [side-effects and ownership of memory
are very important concepts](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#shared-memory) when programming
GPUs, and we had hoped the above building blocks would help create a more elegant and unified programming model. 
-->
虽然我们设计了GPU上运行这个子集的样子，但如果我声称我们完全弄明白了，我会撒谎。我所能说的就是在编程GPU时理解副作用和内存所有权是非常重要的概念，我们希望上述构建块有助于创建更优雅和统一的编程模型。

<!-- 
The final major programming model enhancement this enabled was fine-grained "actors", a sort of mini-process inside of a
process.  I mentioned the vat concept earlier, but that we didn't know how to make it safe.  Finally we had found the
missing clue: a vat was really just an `isolated` bubble of state.  Now that we had this concept in the type system, we
could permit "marshaling" of `immutable` and `isolated` objects as part of the message passing protocol without
marshaling of any sort -- they could be shared safely by-reference! 
-->
最终的主要编程模型增强功能是细粒度的“演员”，这是一个过程中的一种迷你过程。我之前提到了增值税概念，但我们不知道如何使其安全。最后我们找到了缺失的线索：一个大桶实际上只是一个孤立的状态泡沫。现在我们在类型系统中有了这个概念，我们可以允许不可变和隔离对象的“编组”作为消息传递协议的一部分，而不需要任何类型的编组 - 它们可以通过引用安全地共享！

<!-- 
I would say that the major weakness of this system was also its major benefit.  The sheer permutations of concepts could
be overwhelming.  Most of them composed nicely, however the poor developers creating the underlying "safe concurrency"
abstractions -- myself included -- almost lost our sanity in doing so.  There is probably some generics-like unification
between permissions and ownership that could help here, however the "funniness" of linearity is hard to quarantine. 
-->
我想说这个系统的主要弱点也是它的主要好处。概念的绝对排列可能是压倒性的。他们中的大多数组成很好，但是那些创建潜在的“安全并发”抽象的可怜的开发人员 - 包括我自己 - 在这样做时几乎失去了理智。权限和所有权之间可能存在一些类似于泛型的统一，这可能对此有所帮助，但线性度的“有趣”难以隔离。

<!-- 
Amazingly, it all worked!  All those cases I mentioned earlier -- image decoders, the multimedia stack, the browser,
etc. -- could now use safe intra-process parallelism in addition to being constructed out of many parallel processes.
Even more interestingly, our one production workload -- taking Speech Recognition traffic for Bing.com -- actually saw
significant reductions in latency and improvements in throughput as a result.  In fact, Cortana's [DNN](
https://en.wikipedia.org/wiki/Deep_learning)-based speech recognition algorithms, which delivered a considerable boost
to accuracy, could have never reached their latency targets were it not for this overall parallelism model. 
-->
令人惊讶的是，一切都奏效了！我之前提到的所有情况 - 图像解码器，多媒体堆栈，浏览器等 - 除了由许多并行进程构建之外，现在还可以使用安全的进程内并行性。更有趣的是，我们的一个生产工作负载 - 为Bing.com提供语音识别流量 - 实际上显着减少了延迟并提高了吞吐量。事实上，Cortana基于DNN的语音识别算法可以提供相当大的精确度提升，如果不是这种整体并行模型，它可能永远不会达到他们的延迟目标。

<!-- 
### Sequential Consistency and Tear-Free Code 
-->
### 顺序一致性和无泪代码

<!-- 
There was another unanticipated consequence of safe concurrency that I quite liked: [sequential consistency (SC)](
https://en.wikipedia.org/wiki/Sequential_consistency). 
-->
我非常喜欢安全并发的另一个意想不到的后果：顺序一致性（SC）。

<!-- 
For free. 
-->
免费。

<!-- 
After all those years trying to achieve a sane memory model, and ultimately [realizing that most of the popular
techniques were fundamentally flawed](http://joeduffyblog.com/2010/12/04/sayonara-volatile/), we had cracked the nut.
All developers got SC without the price of barriers everywhere.  Given that we had been running on ARM processors where
a barrier cost you 160 cycles, this gave us not only a usability edge, but also a performance one.  This also gave our
optimizing compiler much more leeway on code motion, because it could now freely order what used to be possibly-side-
effectful operations visible to multiple threads. 
-->
经过这么多年努力实现一个理智的记忆模型，并最终意识到大多数流行的技术从根本上是有缺陷的，我们已经破解了坚果。所有开发商都在没有任何障碍价格的情鉴于我们一直在ARM处理器上运行，其中屏障成本为160个周期，这不仅为我们提供了可用性优势，而且还提供了性能优势。这也为我们的优化编译器提供了更多的代码运动余地，因为它现在可以自由地订购多线程可见的可能有效的操作。


<!-- 
To see how we got SC for free, consider how the overall system was layered. 
-->
要了解我们如何免费获得SC，请考虑整个系统是如何分层的。

<!-- 
At the bottom of all of the above safe concurrency abstractions, there was indeed `unsafe` code.  This code was
responsible for obeying the semantic contract of safe concurrency by decorating APIs with the right permissions and
ownership, even if the implementation physically violated them.  But it is important to note: this is the only code in
the system -- plus the 1st party kernel code -- that had to deal with concurrency at the threads, locks, events, and
lock-free level of abstraction.  Everything else built atop the higher-level abstractions, where barriers had already
been placed into the instruction stream at all the right places, thanks to the infrastructure. 
-->
在所有上述安全并发抽象的底部，确实存在不安全的代码。此代码负责通过使用正确的权限和所有权来装饰API来遵守安全并发的语义协定，即使实现实际违反了它们也是如此。但重要的是要注意：这是系统中唯一的代码 - 加上第一方内核代码 - 必须处理线程，锁，事件和无锁抽象级别的并发。由于基础设施的原因，所有其他东西都建立在更高层次的抽象之上，其中障碍已经被放置在所有正确位置的指令流中。

<!-- 
This had another consequence: no [struct tearing](
http://joeduffyblog.com/2006/02/07/threadsafety-torn-reads-and-the-like/) was visible in the 3rd party programming
model.  Everything was "atomic", again for free. 
-->
这有另一个结果：第三方编程模型中没有可见的结构撕裂。一切都是“原子的”，再次免费。

<!-- 
This allowed us to use multi-word slice and interface representations, just like Go does, but [without the type-safety-
threatening races](http://research.swtch.com/gorace).  It turns out, the risk of struct tearing is one of major factors
preventing us from having a great Go-like slice type to C# and .NET.  In Midori, slices were safe, efficient, and
everywhere. 
-->
这允许我们使用多字切片和界面表示，就像Go一样，但没有类型安全威胁的比赛。事实证明，结构撕裂的风险是阻止我们对C＃和.NET有一个很好的Go-like切片类型的主要因素之一。在Midori，切片安全，高效，无处不在。

<!-- 
### Message Passing Races 
-->

### 消息传递竞争

<!-- 
Message passing helps tremendously when building correct, reliable concurrent systems, however it is not a panacea.  I
had mentioned shared nothing earlier on.  It's a dirty little secret, however, even if you don't have shared memory, but
agents can communicate with one another, you still have shared state encoded in the messaging between those agents, and
the opportunity for race conditions due to the generally unpredictable order of arrival of these messages. 
-->
在构建正确，可靠的并发系统时，消息传递有很大帮助，但它不是灵丹妙药。我曾提到过，之前没有提到过。这是一个肮脏的小秘密，但是，即使你没有共享内存，但代理可以相互通信，你仍然在这些代理之间的消息传递中编码共享状态，并且由于通常不可预测的竞争条件的机会这些消息的到达顺序。

<!-- 
This is [understood](http://erlang.org/workshop/2004/cronqvist.pdf), although [perhaps not very widely](
https://www.it.uu.se/research/group/hipe/dialyzer/publications/races.pdf).  The most worrisome outcome from these kind
of races is [time of check time of use (TOCTOU)](https://en.wikipedia.org/wiki/Time_of_check_to_time_of_use), one of the
more common kinds of races that can lead to security vulnerabilities.  (Midori's type- and memory-safety of course helps
to avoid this particular symptom, however reliability problems are very real also.) 
-->
这是可以理解的，尽管可能不是很广泛。这类比赛最令人担忧的结果是检查使用时间（TOCTOU）的时间，这是一种可能导致安全漏洞的更常见种类。 （Midori的类型和记忆安全当然有助于避免这种特殊症状，但是可靠性问题也是非常真实的。）

<!-- 
Although people used to hate it when I compared this situation to COM STAs, for those familiar with them, an analogy is
apt.  If you need to block a thread inside of a COM STA, you must decide: Do I pump the message loop, or do I not pump
the message loop?  If you choose to pump the message loop, you can suffer [reentrancy](
https://en.wikipedia.org/wiki/Reentrancy_(computing)), and that reentrancy might be witness to broken invariants, or
even mutate state out from underneath the blocking call, much to its dismay after it reawakens.  If you choose not to
pump the message loop, you can suffer deadlock, as calls pile up, possibly ones that are required to unblock the thread. 
-->
虽然当我将这种情况与COM STA比较时，人们常常讨厌它，但对于那些熟悉它们的人来说，类比是恰当的。如果您需要阻止COM STA内部的线程，您必须决定：我是否抽取消息循环，还是不抽取消息循环？如果你选择抽取消息循环，你可能会遭受重入，并且这种重入可能见证了不变的不变量，甚至可以从阻塞调用下面变异状态，这让它在重新唤醒后感到沮丧。如果你选择不抽取消息循环，你可能会遇到死锁，因为调用堆积起来，可能需要解锁线程。

<!-- 
In Midori's system, we did not give this choice to the developer.  Instead, every `await` was an opportunity to pump the
underlying message loop.  Just as with a COM STA, these pumps possibly dispatched work that might interact with shared
state.  Note that this is not *parallelism*, mind you, since process event loops did not permit parallelism, however
there is possibly a lot of concurrency going on here, and it can definitely screw you: 
-->
在Midori的系统中，我们没有给开发人员这个选择。相反，每次等待都是一个抽取潜在消息循环的机会。与COM STA一样，这些泵可能会分派可能与共享状态交互的工作。请注意，这不是并行性，因为进程事件循环不允许并行，但是这里可能存在很多并发性，它肯定会让你感到困惑：

    async bool IsRed(AsyncColor c) {
        return (await c.R > 0 && await c.G == 0 && await c.B == 0);
    } 


<!-- 
This rather simple (and silly) function checks to see if an `AsyncColor` is "red"; to do so, it reads the `R`, `G`,
and `B` properties.  For whatever reason, they are asynchronous, so we must `await` between accesses.  If `AsyncColor`
is a mutable object, well, guess what -- these values might change after we've read them, opening up a possible TOCTOU
bug.  For instance, imagine a caller's surprise when `IsRed` may have lied to it: 
-->
这个相当简单（和愚蠢）的函数检查AsyncColor是否为“红色”; 为此，它会读取R，G和B属性。 无论出于何种原因，它们都是异步的，因此我们必须在访问之间等待。 如果AsyncColor是一个可变对象，那么，猜猜看 - 在我们阅读它们之后这些值可能会改变，从而打开一个可能的TOCTOU错误。 例如，想象一下，当IsRed可能对它撒谎时，来电者会感到惊讶：

    AsyncColor c = ...;
    await IsRed(c);
    assert(await c.R > 0); 


<!-- 
That assertion can very well fire.  Even this callsite has a TOCTOU bug of its own, since `c.R` might be `>0` at the end
of `IsRed`'s return, but not after the `assert` expression's own `await` has completed. 
-->
这种说法非常好。 即使这个调用点也有自己的TOCTOU错误，因为在IsRed返回结束时c.R可能> 0，但是在断言表达式自己的等待完成之后不会。

<!-- 
All of this should be familiar territory for concurrency experts.  But we sought to eliminate these headaches. 
-->
对于并发专家来说，所有这些都应该是熟悉的领域。 但我们试图消除这些令人头疼的问题。

<!--  
This area of the system was still under active development towards the end of our project, however we had sketched out a
very promising approach.  It was to essentially apply similar permission annotations to asynchronous activity -- hence
my choice of the term "activity" earlier -- as we did parallel tasks.  Although this seriously limited an asynchronous
activity's state purview, combined with a reader/writer-lock like idea, meant that we could use permissions affixed to
asynchronous interfaces to automatically ensure state and asynchronous operations were dispatched safely.
-->
在我们的项目结束时，该系统的这一领域仍处于积极发展阶段，但我们已经勾画出一种非常有前景的方法。 它本质上是将类似的权限注释应用于异步活动 - 因此我之前选择了“活动”一词 - 就像我们执行并行任务一样。 虽然这严重限制了异步活动的状态权限，并结合读取/写入锁定的想法，意味着我们可以使用附加到异步接口的权限来自动确保状态和异步操作被安全地分派。

<!-- 
### Evolution 
-->
### 演化 

<!-- 
Before moving on, a brief word on the evolution of the system.  As I mentioned earlier, I presented the system in its
final form.  In reality, we went through five major phases of evolution.  I won't bore you with exhaustive details on
each one, although I will note the major mistakes and lessons learned in each phase. 
-->
在继续之前，简要介绍一下系统的演变。正如我之前提到的，我以最终形式呈现了系统。实际上，我们经历了五个主要的进化阶段。尽管我会注意到每个阶段的重大错误和经验教训，但我不会在每个问题上详尽无遗地详述您。

<!-- 
In the first phase, I tried to build the system entirely out of annotations that were "outside" of the type system.  As
I've already said, that failed spectacularly.  At this point, I hope you can appreciate how deeply integrated into the
compiler and its type system these concepts need to be for them to work and for the result to be usable. 
-->
在第一阶段，我尝试完全使用类型系统“外部”的注释来构建系统。正如我已经说过的那样，它失败了。在这一点上，我希望您能理解这些概念需要与编译器及其类型系统深入集成才能使它们工作并使结果可用。

<!-- 
Next, I tried a variant of this with just `readonly`.  Except I called it `readable` (a name that would stick until the
very tail end of the project), and it was always deep.  There was no `immutable` and there was no `isolated`.  The
concept of `mutable` was called `writable`, although I was delusional, and thought you'd never need to state it.  I was
very confused about the role generics played here, and ended up coding myself up into a corner trying to make it work. 
-->
接下来，我只用readonly尝试了这个变种。除了我称之为可读（这个名称会一直延伸到项目的最后端），而且它总是很深。没有不可改变的，也没有孤立的。可变的概念被称为可写，虽然我是妄想，并认为你永远不需要陈述它。我对这里的泛型角色感到非常困惑，并最终将自己编码到一个角落试图让它发挥作用。

<!-- 
After that, I recognized at least that `readable` and `writable` were related to one another, and recognized the
subtyping relationship of (`writable <: readable`).  And, largely based on conversations with colleagues in MSR, I
decided to toss out everything I had done on generics and redo it.  It was at that time I recognized that each generic
type variable, despite looking like a naked type, actually carried *both* a permission and a type.  That helped. 
-->
在那之后，我至少认识到可读和可写相互关联，并认识到（可写<：可读）的子类型关系。而且，主要基于与MSR同事的对话，我决定抛弃我在仿制药上所做的一切并重做它。正是在那个时候，我认识到每个泛型类型变量，尽管看起来像裸体类型，但实际上同时携带了权限和类型。这有帮助。

<!-- 
I then came up with `immutable`, however it wasn't what you see today.  Instead, it had the slightly confusing meaning
of being a "view" over just the immutable subset of data in the target object graph.  (This was at first limited to only
`readonly` fields (in the classical C# sense) that were of a primitive type.)  If you tried reading a non-immutable part
from this view, you'd get a compiler error.  Bizarrely, this meant you could have an `immutable List<T>` that wasn't
actually immutable.  In hindsight, this was pretty wonky, but it got us thinking about and discussing immutability. 
-->
然后我提出了不可改变的问题，但这不是你今天所看到的。相反，它只是对目标对象图中的不可变数据子集的“视图”有一点混淆的含义。 （这首先仅限于只有原始类型的只读字段（在经典的C＃意义上）。）如果您尝试从此视图中读取非不可变部分，则会出现编译器错误。奇怪的是，这意味着你可以拥有一个不可变的List <T>。事后看来，这非常不稳定，但它让我们思考和讨论不变性。

<!-- 
Somewhere in here, we recognized the need for generic parameterization over permissions, and so we added that.
Unfortunately, I originally picked the `%` character to indicate that a generic type was a permission, which was quite
odd; e.g., `G<%P>` versus `G<T>`.  We renamed this to `permission`; e.g., `G<permission P>` versus `G<T>`. 
-->
在这里的某个地方，我们认识到需要对权限进行泛型参数化，因此我们添加了它。不幸的是，我最初选择％字符表示泛型类型是一种权限，这很奇怪;例如，G <％P>对G T。我们将此更名为权限;例如，G <许可P>与G <T>。

<!-- 
There was one problem.  Generic permissions were needed in way more places than we expected, like most property getters.
We experimented with various "shortcuts" in an attempt to avoid developers needing to know about generic permissions.
This hatched the `readable+` annotation, which was a shortcut for "flow the `this` parameter's permission."  This
concept never really left the system, although (as we will see shortly), we fixed generics and eventually this concept
became much easier to swallow, syntax-wise (especially with smart defaults like auto-properties). 
-->
有一个问题。与大多数财产获取者一样，需要更多地方的通用权限。我们试验了各种“快捷方式”，试图避免开发人员需要了解通用权限。这标志着可读+注释，这是“流动这个参数的权限。”这个概念从未真正离开系统，虽然（我们很快就会看到），我们修复了泛型，最终这个概念变得更容易吞下，语法-wise（特别是智能默认值，如自动属性）。

<!-- 
We lived with this system for some time and this was the first version deployed at-scale into Midori. 
-->
我们和这个系统一起生活了一段时间，这是第一个大规模部署到Midori的版本。

<!-- 
And then a huge breakthrough happened: we discovered the concepts necessary for `isolated` and, as a result, an
`immutable` annotation that truly meant that an object (graph) was immutable. 
-->
然后发生了巨大的突破：我们发现了隔离所必需的概念，因此，一个真正意味着对象（图形）不可变的不可变注释。

<!-- 
I can't claim credit for this one.  That was the beauty of getting to this stage: after developing and prototyping the
initial ideas, and then deploying them at-scale, we suddenly had our best and brightest obsessing over the design of
this thing, because it was right under their noses.  This was getting an initial idea out in front of "customers"
early-and-often at its finest, and, despite some growing pains, worked precisely as designed. 
-->
我不能声称这个。这就是进入这个阶段的美妙之处：在对最初的想法进行开发和原型设计，然后大规模部署它们之后，我们突然对这件事的设计产生了最好和最聪明的感觉，因为它正好在他们的鼻子底下。这是在早期和最常见的“顾客”面前得到最初的想法，尽管有一些成长的痛苦，但仍然按照设计精确地工作。

<!-- 
We then wallowed in the system for another year and 1/2 and, frankly, I think lost our way a little bit.  It turns out
deepness was a great default, but sometimes wasn't what you wanted.  `List<T>` is a perfect example; sometimes you want
the `List` to be `readonly` but the elements to be `mutable`.  In the above examples, we took this capability for
granted, but it wasn't always the case.  The outer `readonly` would infect the inner `T`s. 
-->
然后，我们在系统中沉溺了一年又一半，坦率地说，我认为我们失去了一点点。事实证明，深度是一个很好的默认，但有时不是你想要的。 List <T>是一个很好的例子;有时你希望List只是readonly但元素是可变的。在上面的例子中，我们认为这种能力是理所当然的，但并非总是如此。外部只读会感染内部Ts。

<!-- 
Our initial whack at this was to come up with shallow variants of all the permissions.  This yielded keywords that
became a never-ending source of jokes in our hallways: `shreadable`, `shimmutable`, and -- our favorite -- `shisolated`
(which sounds like a German swear word when said aloud).  Our original justification for such nonsense was that in C#,
the signed and unsigned versions of some types used abbreviations (`sbyte`, `uint`, etc.), and `shallow` sure would make
them quite lengthy, so we were therefore justified in our shortening into a `sh` prefix.  How wrong we were. 
-->
我们对此的初步看法是提出所有权限的浅层变体。这产生了关键词，在我们的走廊里变成了一个永无止境的笑话来源：可怕的，可吸收的，以及 - 我们最喜欢的 -  shisolated（当大声说出时听起来像德国人发誓）。我们对这种无意义的原始理由是，在C＃中，某些类型的有符号和无符号版本使用缩写（sbyte，uint等），而浅的确定会使它们非常冗长，因此我们因此缩短为sh字首。我们有多糟糕。

<!-- 
From there, we ditched the special permissions and recognized that objects had "layers", and that outer and inner layers
might have differing permissions.  This was the right idea, but like most ideas of this nature, we let the system get
inordinately more complex, before recognizing the inner beauty and collapsing it back down to its essence. 
-->
从那里，我们抛弃了特殊权限，并认识到对象具有“层”，外层和内层可能具有不同的权限。这是正确的想法，但是像大多数这种性质的想法一样，在认识到内在美并将其折叠回其本质之前，我们让系统变得非常复杂。

<!-- 
At the tail end of our project, we were working to integrate our ideas back into C# and .NET proper.  That's when I was
adamant that we unify the concept of `readable` with `readonly`, leading to several keyword renames.  Ironically,
despite me having left .NET to pursue this project several years earlier, I was the most optimistic out of anybody that
this could be done tastefully.  Sadly, it turned out I was wrong, and the project barely got off the ground before
getting axed, however the introductory overview above is my best approximation of what it would have looked like. 
-->
在我们项目的最后，我们正在努力将我们的想法重新整合到C＃和.NET中。那时我坚持认为我们用readonly统一了可读性的概念，导致了几个关键字重命名。具有讽刺意味的是，尽管我几年前离开.NET去追求这个项目，但我最乐观地认为这可以做得很有品味。可悲的是，事实证明我错了，项目在被砍掉之前几乎没有起步，但上面的介绍性概述是我最接近它的样子。

<!-- 
## Inspirations 
-->
## 启示

<!-- 
Now that we have seen the system in its final state, let's now trace the roots back to those systems that were
particularly inspiring for us.  In a picture: 
-->
现在我们已经看到系统处于最终状态，现在让我们回顾那些对我们特别鼓舞人心的系统。 在图片中：


![Influences](/assets/img/2016-11-30-15-years-of-concurrency.influences.jpg)
<!-- 
I'll have to be brief here, since there is so much ground to cover, although there will be many pointers to follow up
papers should you want to dive deeper.  In fact, I read something like 5-10 papers per week throughout the years I was
working on all of this stuff, as evidenced by the gigantic tower of papers still sitting in my office: 
-->
我必须在这里简要介绍一下，因为有很多内容可以覆盖，尽管如果你想深入研究，会有很多关于后续论文的指示。 事实上，我每年都会阅读5-10篇论文，这些年来我正在处理所有这些事情，正如我办公室里仍然坐着的巨大文件塔所证明的那样：

![Concurrency Paper Stack](/assets/img/2016-11-30-15-years-of-concurrency.papers.jpg)

<!-- 
### const 
-->
### 常量

<!-- 
The similarities with `const` should, by now, be quite evident.  Although people generally have a love/hate relationship
with it, I've always found that being [`const` correct](https://isocpp.org/wiki/faq/const-correctness) is worth the
effort for anything bigger than a hobby project.  (I know plenty of people who would disagree with me.) 
-->
到目前为止，与const的相似之处应该是非常明显的。虽然人们通常与它有爱/恨的关系，但我总是发现，对于任何比业余爱好项目更大的东西来说，保持正确是值得的。 （我知道很多人不同意我的看法。）

<!-- 
That said, `const` is best known for its unsoundness, thanks to the pervasive use of `const_cast`.  This is commonly
used at the seams of libraries with different views on `const` correctness, although it's also often used to cheat;
this is often for laziness, but also due to some compositional short-comings.  The lack of parameterization over
`const`, for example, forces one to duplicate code; faced with that, many developers would rather just cast it away. 
-->
也就是说，由于const_cast的普遍使用，const因其不健全而闻名。这通常用于库的接缝，对const的正确性有不同的看法，虽然它也经常被用来作弊;这通常是为了懒惰，也是由于一些构成上的缺点。例如，缺少对const的参数化会迫使人们重复代码;面对这一点，许多开发人员宁愿把它抛弃。

<!-- 
`const` is also not deep in the same way that our permissions were, which was required to enable the safe concurrency,
isolation, and immutability patterns which motivated the system.  Although many of the same robustness benefits that
`const` correctness delivers were brought about by our permissions system, that wasn't its original primary motivation. 
-->
const也不像我们的权限那样深，这是启用安全并发，隔离和不变性模式所需要的，这些模式激励了系统。尽管我们的权限系统带来了许多与const正确性相同的健壮性，但这并不是它最初的主要动机。

<!-- 
### Alias Analysis 
-->
### 别名分析

<!-- 
Although it's used more as a compiler analysis technique than it is in type systems, [alias analysis](
https://en.wikipedia.org/wiki/Pointer_aliasing) is obviously a close cousin to all the work we did here.  Although the
relationship is distant, we looked closely at many uses of aliasing annotations in C(++) code, including
`__declspec(noalias)` in Visual C++ and `restrict` (`__restrict`, `__restrict__`, etc.) in GCC and standard C.  In fact,
some of our ideas around `isolated` eventually assisted the compiler in performing better alias analysis. 
-->
虽然它更多地用作编译器分析技术而不是类型系统，但别名分析显然是我们在这里所做的所有工作的近亲。虽然这种关系很遥远，但我们仔细研究了C（++）代码中别名注释的许多用法，包括Visual C ++中的__declspec（noalias）和GCC和标准C中的限制（__限制，__ restrict__等）。事实上，我们围绕隔离的一些想法最终帮助编译器执行更好的别名分析。

<!-- 
### Linear Types 
-->
### 线性类型

<!-- 
Phillip Wadler's 1990 ["Linear types can change the world!"](
http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.55.5439&rep=rep1&type=pdf) was immensely influential for me in
the early days.  I remember a huge lightbulb going off when I first read this paper.  Linear types are inspired by the
[linear logic of J.-Y. Girard](https://en.wikipedia.org/wiki/Linear_logic), and it is easy to get lost in the
mathematics for hours. 
-->
Phillip Wadler 1990年的“线性类型可以改变世界！”在早期对我有极大的影响力。我记得当我第一次读这篇论文时，一个巨大的灯泡熄灭了。线性类型的灵感来自J.-Y的线性逻辑。吉拉德，很容易迷失在数学中数小时。

<!-- 
In a nutshell, a linear type lets you prove that a variable is used exactly once.  This is similar to `isolated`,
however due to the aliasing properties of an imperative language like C# (especially for heap structures with possible
cycles between them), the simple and elegant model of strict linearity is hard to make work. 
-->
简而言之，线性类型可以让您证明变量只使用了一次。这与隔离类似，但是由于C＃等命令式语言的别名属性（特别是对于它们之间可能存在循环的堆结构），严格线性的简单而优雅的模型很难实现。

<!-- 
Linear types themselves also aren't very commonly seen in the wild, and are mostly useful for their mathematical and
proof properties.  If you go looking, [you will find examples](https://ghc.haskell.org/trac/ghc/wiki/LinearTypes),
however.  More than real syntax in real languages, linear types have been hugely influential on subsequent innovations
in type systems that also impacted us, such as affine and uniqueness types. 
-->
线性类型本身在野外也不常见，并且主要用于它们的数学和证明属性。如果你去看，你会发现例子。线性类型不仅仅是真实语言中的真实语法，而且对影响我们的类型系统的后续创新产生了极大的影响，例如仿射和唯一性类型。

<!-- 
### Haskell Monads 
-->
### Haskell Monads

<!-- 
In the early days, I was pretty obsessed with [Haskell](https://en.wikipedia.org/wiki/Haskell_(programming_language)),
to put it mildly. 
-->
在早期，我非常沉迷于Haskell，温和地说。

<!-- 
I often describe the above system that we built as the inverse of the [Haskell state monad](
https://wiki.haskell.org/State_Monad).  In Haskell, what you had was a purely functional language, with [sugar to make
certain aspects look imperative](https://en.wikibooks.org/wiki/Haskell/do_notation).  If you wanted side-effects, you
needed to enter the beautiful world of [monads](https://en.wikibooks.org/wiki/Haskell/Understanding_monads/State).  In
particular, for simple memory side-effects, the state monad let you have traditional mutable data structures, but in a
way that the type system very much understood and could restrict for safety where appropriate.
-->
我经常将我们构建的上述系统描述为Haskell状态monad的逆。在Haskell中，你所拥有的是一种纯粹的功能性语言，用糖来使某些方面看起来势在必行。如果你想要副作用，你需要进入美丽的monad世界。特别是，对于简单的内存副作用，状态monad让你拥有传统的可变数据结构，但是在某种程度上类型系统非常理解并且可能在适当的时候限制安全性。

<!-- 
Well, the system we built was sort of the opposite: you were in an imperative language, and had a way of marking certain
aspects of the program as being purely functional.  I am pretty sure I read the classic ["State in Haskell" paper](
http://research.microsoft.com/en-us/um/people/simonpj/Papers/state-lasc.pdf) at least a dozen times over the years.  In
fact, as soon as I recognized the similarities, I compared notes with Simon Peyton-Jones, who was immensely gracious and
helpful in working through some very difficult type system design challenges. 
-->
好吧，我们构建的系统恰恰相反：你使用命令式语言，并且有一种方法可以将程序的某些方面标记为纯粹的功能。我很确定这些年来至少十几次读过经典的“Haskell中的状态”论文。事实上，一旦我认识到这些相似之处，我就会与Simon Peyton-Jones进行比较，他非常亲切，有助于解决一些非常困难的类型系统设计挑战。

<!-- 
### Effect Types
-->
### 效果类型

<!-- 
[Effect typing](http://web.cs.ucla.edu/~palsberg/tba/papers/nielson-nielson-csd99.pdf), primarily in the ML community,
was also influential in the early days.  An effect type propagates information at compile-time describing the dynamic
effect(s) executing said code is expected to bring about.  This can be useful for checking many properties. 
-->
主要在ML社区中的效果打字在早期也具有影响力。效果类型在编译时传播信息，描述预期执行所述代码所带来的动态效果。这对于检查许多属性非常有用。

<!-- 
For example, I always thought of `await` and `throws` annotations as special kinds of effects that indicate a method
might block or throw an exception, respectively.  Thanks to the additive and subtractive nature of effect types, they
propagate naturally, and are even amenable to parametric polymorphism. 
-->
例如，我总是想到等待并将注释作为特殊类型的效果分别表示方法可能会阻塞或抛出异常。由于效应类型的加性和减性性质，它们自然地传播，甚至适合参数多态性。

<!-- 
It turns out that permissions can be seen as a kind of effect, particularly when annotating an instance method.  In a
sense, a `mutable` instance method, when invoked, has the "effect" of mutating the receiving object.  This realization
was instrumental in pushing me towards leveraging subtyping for modeling the relationship between permissions. 
-->
事实证明，权限可以被视为一种效果，特别是在注释实例方法时。从某种意义上说，可变实例方法在被调用时具有改变接收对象的“效果”。这种实现有助于我利用子类型来建模权限之间的关系。


<!-- 
Related to this, the various ownership systems over the years were also top-of-mind, particularly given Midori's
heritage with Singularity, which used the Spec# language.  This language featured [ownership annotations](
https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/specsharp-tim.pdf). 
-->
与此相关的是，多年来的各种所有权制度也是最重要的，特别是考虑到Midori使用Spec＃语言的Singularity传统。该语言包含所有权注释。

<!-- 
### Regions 
-->
### 地区

<!-- 
[Regions](https://en.wikipedia.org/wiki/Region-based_memory_management), despite classically being used mostly for
deterministic and efficient memory management, were incredibly interesting towards the days of figuring out `isolated`. 
-->
尽管经典地主要用于确定性和有效的内存管理，但这些区域在确定孤立的时代是非常有趣的。

<!-- 
They aren't identical for several reasons, however. 
-->
然而，由于几个原因，它们并不相同。

<!-- 
The first reason is that isolated object graphs in our system weren't as strictly partitioned as regions, due to
immutable in- and out- references.  Regions are traditionally used to collect memory efficiently and hence dangling
references like this wouldn't be permitted (and the reachability analysis to detect them would basically devolve into
garbage collection). 
-->
第一个原因是由于不可变的输入和输出参考，我们系统中的孤立对象图不像区域那样严格划分。区域传统上用于有效地收集内存，因此不允许这样的悬挂引用（并且用于检测它们的可达性分析将基本上转换为垃圾收集）。

<!-- 
The second reason is that we wanted to avoid the syntactic burden of having regions in the language.  A good example of
this in action is [Deterministic Parallel Java](
https://pdfs.semanticscholar.org/de3d/6c78392c86802af835d0337758605e160bf9.pdf), which requires explicit region
annotations on objects using a very generics-like syntax (e.g., `Foo<region R>`).  Some amount of this can be hidden
from the developer through more sophisticated compiler analysis -- much like [Cyclone](
https://en.wikipedia.org/wiki/Cyclone_(programming_language)) did -- however, we worried that in some very common cases,
regions would rear their ugly heads and then the developer would be left confused and dismayed.
-->
第二个原因是我们想避免在语言中使用区域的语法负担。实际操作中的一个很好的例子是确定性并行Java，它需要使用类似于泛型的语法（例如，Foo <region R>）在对象上进行显式区域注释。通过更复杂的编译器分析可以向开发人员隐藏一些这样的内容 - 就像Cyclone所做的那样 - 但是，我们担心在一些非常常见的情况下，区域会使他们丑陋的头部发挥作用，然后开发人员会感到困惑和沮丧。


<!-- 
All that said, given our challenges with garbage collection, in addition to our sub-process actor model, we often
pondered whether some beautiful unification of `isolated` object graphs and regions awaited our discovery. 
-->
所有这一切，鉴于我们的垃圾收集挑战，除了我们的子流程演员模型，我们经常思考是否有一些美丽的孤立对象图和区域的统一等待我们的发现。

<!-- 
### Separation Logic 
-->
### 分离逻辑

<!-- 
Particularly in the search for formalisms to prove the soundness of the system we built, [separation logic](
https://en.wikipedia.org/wiki/Separation_logic) turned out to be instrumental, especially the [concurrent form](
http://www.cs.cmu.edu/~brookes/papers/seplogicrevisedfinal.pdf).  This is a formal technique for proving the
disjointness of different parts of the heap, which is very much what our system is doing with the safe concurrency
abstractions built atop the `isolated` primitive.  In particular, our OOPSLA paper used a novel proof technique,
[Views](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/views.pdf), which can be constructed from
separation algebras.  Caution: this is getting into some very deep mathematical territory; several colleagues far
smarter than I am were the go-to guys on all of this.  But, it certainly helped all of us sleep better at night. 
-->
特别是在寻求证明我们构建的系统健全性的形式主义时，分离逻辑被证明是有用的，特别是并发形式。这是一种证明堆的不同部分不相交的正式技术，这正是我们的系统在隔离原语之上构建的安全并发抽象所做的事情。特别是，我们的OOPSLA论文使用了一种新颖的证明技术，即视图，它可以用分离代数构造。警告：这是进入一些非常深刻的数学领域;几位比我聪明得多的同事都是这些人的首选。但是，它确实帮助我们所有人在晚上睡得更好。

<!-- 
### Uniqueness Types 
-->
### 唯一性类型

<!-- 
[Uniqueness types](https://en.wikipedia.org/wiki/Uniqueness_type) are a more recent invention, derived from some of the
early linear type systems which so fascinated me early on.  For a period of time, we actually had a `unique` keyword in
the language.  Eventually we folded that back into the concept of `isolated` (it was essentially a "shallow"
`isolated`).  But there is no denying that all of this was heavily inspired by what we saw with uniqueness types,
especially in languages like [Clean](https://en.wikipedia.org/wiki/Clean_(programming_language)), the [experimental work
to bring uniqueness to Scala](http://lampwww.epfl.ch/~phaller/doc/capabilities-uniqueness2.pdf), and, now, Rust. 
-->
唯一性类型是最近的一项发明，源于一些早期的线性类型系统，这些系统在早期让我很着迷。在一段时间内，我们实际上在该语言中有一个唯一的关键字。最终我们把它折回到了孤立的概念中（它本质上是一个“浅层的”孤立的）。但不可否认的是，所有这一切都深受我们所看到的独特类型的启发，特别是在像Clean这样的语言中，Scala带来了独特性的实验性工作，现在还有Rust。

<!-- 
### Model Checking 
-->
### 模型检查

<!--
Finally, I would be remiss if I didn't at least mention [model checking](https://en.wikipedia.org/wiki/Model_checking).
It's easy to confuse this with static analysis, however, model checking is far more powerful and complete, in that it
goes beyond heuristics and therefore statistics.  [MSR's Zing](
https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/zing-tool.pdf) and, although we used it to verify
the correctness of certain aspects of our implementation, I don't think we sufficiently considered how model checking
might impact the way safety was attained.  This was top-of-mind as we faced intra-process interleaving race conditions.
Especially as we look to the future with more distributed-style concurrency than intra-process parallelism, where state
machine verification is critical, many key ideas in here are relevant.
-->
最后，如果我至少没有提到模型检查，那将是我的疏忽。很容易将它与静态分析混淆，但是，模型检查功能更强大，更完整，因为它超出了启发式，因此超出了统计数据。 MSR的Zing虽然我们用它来验证我们实施的某些方面的正确性，但我认为我们没有充分考虑模型检查如何影响安全性的实现方式。当我们遇到进程间交错竞争条件时，这是最重要的。特别是当我们展望更多分布式并发而不是进程内并行性时，状态机验证至关重要，这里的许多关键思想都是相关的。

<!-- 
### Other Languages
-->
### 其他语言

<!-- 
This story spans many years.  During those years, we saw several other languages tackling similar challenges, sometimes
in similar ways.  Because of the complex timeline, it's hard to trace every single influence to a given point in time,
however it's fair to say that four specific languages had a noteworthy influence on us. 
-->
这个故事跨越多年。在那些年里，我们看到其他几种语言处理类似的挑战，有时以类似的方式。由于复杂的时间表，很难追踪到给定时间点的每一个影响，但可以说四种特定语言对我们有着值得注意的影响。

<!-- 
(Note that there are [dozens of influential concurrent and parallel languages](
https://en.wikipedia.org/wiki/List_of_concurrent_and_parallel_programming_languages) that inspired our work.  I'm sure I
haven't read everything there is to read -- there's always more to learn -- however I did my best to survey the field.
I will focus here on the most mainstream and relevant to people writing production code in the year 2016.) 
-->
（请注意，有许多有影响力的并发和并行语言激发了我们的工作。我确信我还没有阅读所有要阅读的内容 - 总是需要学习更多内容 - 但是我尽力调查这个领域。我会重点关注2016年编写生产代码的最主流和相关人员。）

<!-- 
#### (Modern) C++ 
-->
#### （现代）C ++

<!-- 
I already mentioned `const` and its influence. 
-->
我已经提到了const及其影响力。

<!-- 
It is also interesting to note the similarities between `isolated` and C++11's [`std::unique_ptr`](
https://en.wikipedia.org/wiki/Smart_pointer#unique_ptr).  Although born in different times, and in very different
worlds, they both clearly deliver a similar perspective on ownership.  Noted difference include deepness -- C++'s
approach is "deep" insofar as you leverage RAII faithfully in your data structures -- and motivations -- C++'s
motivation being primarily memory management, and neither safe concurrency nor immutability.
-->
注意隔离和C ++ 11的std :: unique_ptr之间的相似性也很有趣。虽然出生在不同的时代，但在不同的世界中，他们都清楚地表达了对所有权的类似观点。注意差异包括深度 - 在你的数据结构中忠实地利用RAII时，C ++的方法是“深度的” - 和动机 -  C ++的动机主要是内存管理，既不安全的并发性，也不是不可变性。

<!-- 
The concept of [`constexpr`](
https://en.wikipedia.org/wiki/C%2B%2B11#constexpr_.E2.80.93_Generalized_constant_expressions) has obvious similarities
to both `isolated` and `immutable`, particularly the compile-time evaluation and freezing of the results.  The continued
evolution of `constexpr` in C++13 and C++17 is taking the basic building blocks to new frontiers that I had always
wanted to do with our system, but never had time, like arbitrary compile-time evaluation of expressions, and
freezing/memoization of the results. 
-->
constexpr的概念与孤立和不可变的明显相似，特别是编译时评估和冻结结果。 constixpr在C ++ 13和C ++ 17中的不断发展，正在将基本构建块带到我一直想用我们的系统做的新边界，但从来没有时间，比如表达式的任意编译时评估，以及冻结/记忆结果。


<!-- 
Thankfully, because I was leading the C++ group at Microsoft for some time after Midori, I was able to bring many of our
lessons learned to the discussion, and I like to think it has had a positive impact on evolving C++ even further. 
-->
值得庆幸的是，因为在Midori之后我在Microsoft领导C ++小组一段时间，我能够将我们的许多经验教训带到讨论中，并且我认为它对进一步发展C ++产生了积极影响。

<!-- 
#### D 
-->
#### D

<!-- 
The system we came up with has obvious comparisons to D's take on [`const` and `immutable`](
https://dlang.org/spec/const3.html); just as D's `const` is a view over mutable or immutable data, so too is our
`readonly`.  And just as D added deepness to the concept of `const`, so did we in our permissions model generally.
This is perhaps the closest analogy in any existing systems.  I am frankly surprised it doesn't get used an order of
magnitude more than it does, although Andrei, one of its chief developers, [has some thoughts on that topic](
https://www.quora.com/Which-language-has-the-brightest-future-in-replacement-of-C-between-D-Go-and-Rust-And-Why).
-->
我们提出的系统与D对const和不可变的对比有明显的比较;就像D的const是对可变或不可变数据的看法一样，我们的readonly也是如此。正如D为const的概念增加了深度，我们在权限模型中也是如此。这可能是任何现有系统中最接近的类比。我坦率地感到惊讶的是它没有比它更多地使用一个数量级，尽管其主要开发人员之一安德烈对该主题有一些想法。

<!-- 
#### Go 
-->
#### Go

<!-- 
Although I personally love programming in Go, it didn't have as much influence on our system as you might expect.  Go
lists concurrency as one of its primary features.  Although concurrency is easy to generate thanks to [`go`routines](
https://gobyexample.com/goroutines), and best practices encourage wonderful things like ["Share Memory by
Communicating"](https://blog.golang.org/share-memory-by-communicating), the basic set of primitives doesn't go much
beyond the threads, thread-pools, locks, and events that I mention us beginning with in the early days of this journey. 
-->
虽然我个人喜欢在Go中编程，但它对我们的系统的影响并没有你想象的那么大。 Go将并发列为其主要功能之一。虽然由于goroutines很容易生成并发性，并且最佳实践鼓励诸如“通过通信共享内存”之类的奇妙内容，但基本的原语集并没有超出我提到的线程，线程池，锁和事件的范围。从这个旅程的早期开始。

<!-- 
On one hand, I see that Go has brought its usual approach to bear here; namely, eschewing needless complexity, and
exposing just the bare essentials.  I compare this to the system we built, with its handful of keywords and associated
concept count, and admire the simplicity of Go's approach.  It even has nice built-in deadlock detection.  And yet, on
the other hand, when I find myself debugging classical data races, and [torn structs or interfaces](
https://blog.golang.org/share-memory-by-communicating), I clamor for more.  I have remarked before that simply running
with [`GOMAXPROCS=1`](https://golang.org/pkg/runtime/#GOMAXPROCS), coupled with a simple [RPC system](
http://www.grpc.io/) -- ideally integrated in such a way where you needn't step outside of Go's native type system --
can get you close to the simple "no intra-process parallelism" Midori model that we began with.  And perhaps the best
sweet spot of all. 
-->
一方面，我看到Go已经带来了通常的做法;也就是说，避免不必要的复杂性，并暴露出基本要素。我将它与我们构建的系统进行比较，其中包含一些关键字和相关的概念计数，并且钦佩Go的方法的简单性。它甚至还有很好的内置死锁检测功能。然而，另一方面，当我发现自己调试经典数据竞赛，破坏结构或界面时，我吵着要更多。我之前已经说过，只需运行GOMAXPROCS = 1，再加上一个简单的RPC系统 - 理想地集成在一起，你不需要走出Go的原生类型系统 - 可以让你接近简单的“无内部进程”并行性“我们开始的Midori模型。也许是最好的甜蜜点。

<!-- 
#### Rust 
-->
#### Rust

<!-- 
Out of the bunch, [Rust](https://www.rust-lang.org/en-US/) has impressed me the most.  They have delivered on much of
what we set out to deliver with Midori, but actually shipped it (whereas we did not).  My hat goes off to that team,
seriously, because I know first hand what hard, hard, hard work this level of type system hacking is.
-->
在这一堆中，Rust给我留下了最深刻的印象。他们已经完成了我们计划与Midori交付的大部分内容，但实际上已发货（而我们没有）。我认真对待那个团队，因为我知道这种级别的系统黑客攻击是一项艰难，艰苦，艰苦的工作。

<!-- 
I haven't yet described our "borrowed references" system, or the idea of auto-destructible types, however when you add
those into the mix, the underlying type system concepts are remarkably similar.  Rust is slightly less opinionated on
the overall architecture of your system than Midori was, which means it is easier to adopt piecemeal, however the
application of these concepts to traditional concurrency mechanisms like locks is actually fascinating to see. 
-->
我还没有描述我们的“借用引用”系统，或者自动可破坏类型的概念，但是当你将它们添加到混合中时，底层类型系统概念非常相似。 Rust对系统整体架构的看法略逊于Midori，这意味着它更容易采用零碎方式，但是将这些概念应用于传统的并发机制（如锁）实际上是非常吸引人的。

<!-- 
[This article gives a great whirlwind tour](https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html) of safe
concurrency in Rust.  Poking through to some of the references reveals APIs designed with similar principles in mind.
For example, [`simple_parallel`](http://huonw.github.io/simple_parallel/simple_parallel/) looks a whole lot like the
PFX `Parallel` API described earlier with safety annotations applied to it.  I trust their system more than ours,
because they have shipped and had thousands of eyes and real-world experience applied to it. 
-->
本文为Rust提供了一个安全并发的旋风之旅。深入研究一些参考文献，揭示了在设计时考虑了类似原则的API。例如，simple_parallel看起来很像前面描述的PFX Parallel API，并且应用了安全注释。我相信他们的系统比我们的系统更多，因为他们已经发货，并且有数千只眼睛和现实世界的经验应用于它。

<!-- 
# Epilogue and Conclusion 
-->
# 结语和结论

<!-- 
Although I've glossed over many details, I hope you enjoyed the journey, and that the basic ideas were clear.  And, most
importantly, that you learned something new.  If you want to understand anything in greater detail, please see [our
OOPSLA paper](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/msr-tr-2012-79.pdf), or just ask. 
-->
虽然我已经掩盖了许多细节，但我希望你喜欢这个旅程，并且基本的想法很明确。而且，最重要的是，你学到了新东西。如果您想更详细地了解任何内容，请参阅我们的OOPSLA论文，或者只是询问。

<!-- 
It's been a couple years since I've been away from this.  As most of you know, Midori happened before the OSS
renaissance at Microsoft, and so it never saw the light of day.  In that time, I've pondered what lessons we learned on
this journey, and whether any of it is relevant beyond the hallways of our old building 34.  I believe it is, otherwise
I'd not have taken the time to write up this article. 
-->
距离我已经有几年了。正如大多数人所知，Midori发生在微软的OSS复兴之前，因此它从未见过光明。在那段时间里，我思考了我们在这次旅程中学到了什么，以及是否有任何相关课程超出了我们旧楼34的走廊。我相信它是，否则我没有花时间写这篇文章文章。

<!-- 
I'm thrilled that the world has adopted tasks in a big way, although it was for a different reason than we expected
(asynchrony and not parallelism).  In many ways this was inevitable, however I have to think that doing tasks a
half-decade ahead of the curve at least had a minor influence, including the `async` and `await` ideas built atop it. 
-->
我很高兴世界已经大量采用了任务，尽管这是出于与我们预期不同的原因（异步而非并行）。在许多方面，这是不可避免的，但是我不得不认为，在曲线前五年做任务至少会产生一些影响，包括异步和等待在它上面构建的想法。

<!-- 
Data parallelism has taken off...sort of.  Far fewer people leverage CPUs in the way we imagined, but that's for good
reason: GPUs are architected for extremely wide SIMD operations over floating points, which is essentially the killer
scenario for this sort of parallelism.  It doesn't cover all of the cases, but man does it scream. 
-->
数据并行性已经起飞......以我们想象的方式利用CPU的人数要少得多，但这是有充分理由的：GPU的架构是针对浮点的极宽SIMD操作，这实际上是这种并行性的杀手级场景。它没有涵盖所有的情况，但是男人会尖叫。

<!-- 
Safe concurrency is still critically important, yet lacking, and the world still needs it.  I think we collectively
underestimated how long it would take for the industry to move to type- and memory-safe programming models.  Despite the
increasing popularity of safe systems languages like Go and Rust, it pains me to say it, but I still believe we are a
decade away from our fundamental technology stacks -- like the operating systems themselves -- being safe to the core.
But our industry desperately needs this to happen, given that [buffer errors remain the #1 attack type](
https://nvd.nist.gov/visualizations/cwe-over-time) for critical security vulnerabilities in our software. 
-->
安全并发仍然至关重要，但缺乏，世界仍然需要它。我认为我们总体上低估了行业转向类型和内存安全编程模型需要多长时间。尽管像Go和Rust这样的安全系统语言越来越受欢迎，但我很难说，但我仍然相信我们距离我们的基础技术堆栈还有十年的时间 - 就像操作系统本身一样 - 对核心安全。但鉴于缓冲区错误仍然是我们软件中关键安全漏洞的首要攻击类型，我们的行业迫切需要这样做。

<!-- 
I do think that concurrency-safety will be our next frontier after type- and memory-safety have arrived.  TOCTOU, and
race conditions generally, are an underexploited yet readily attackable vector.  (Thankfully, just as writing correct
concurrent code is hard, so too is provoking a latent concurrency error through the delicate orchestration of race
conditions).  As more systems become concurrent (distributed) systems this will become an increasing problem for us.
It's not clear the exact formulation of techniques I demonstrated above is the answer -- in fact, given our focus on
parallelism over asynchrony, surely it is not -- however we will need *some* answer.  It's just too damn hard to build
robust, secure, and safe concurrent programs, still, to this day, 15 years later. 
-->
在类型和内存安全问题到来之后，我确实认为并发安全将是我们的下一个前沿。通常情况下，TOCTOU和竞争条件是一个未被充分利用但容易攻击的矢量。 （值得庆幸的是，正如编写正确的并发代码很难，因此通过精确的竞争条件编排也会引发潜在的并发错误）。随着越来越多的系统成为并发（分布式）系统，这将成为我们日益严重的问题。目前尚不清楚我上面演示的技术的确切形式是答案 - 事实上，鉴于我们关注的是异步并行性，当然不是 - 但是我们需要一些答案。到目前为止，15年之后，构建强大，安全，安全的并发程序真是太难了。

<!-- 
In particular, I'm still conflicted about whether all those type system extensions were warranted.  Certainly
immutability helped with things far beyond safe concurrency.  And so did the side-effect annotations, as they commonly
helped to root out bugs caused by unintended side-effects.  The future for our industry is a massively distributed one,
however, where you want simple individual components composed into a larger fabric.  In this world, individual nodes are
less "precious", and arguably the correctness of the overall orchestration will become far more important.  I do think
this points to a more Go-like approach, with a focus on the RPC mechanisms connecting disparate pieces. 
-->
特别是，我仍然对所有这些类型的系统扩展是否有必要存在冲突。当然，不变性有助于远远超出安全并发性的事情。副作用注释也是如此，因为它们通常有助于根除由意外副作用引起的错误。然而，我们行业的未来是一个大规模分布的行业，您需要将简单的单个组件组合成更大的结构。在这个世界上，个别节点不那么“珍贵”，可以说整体编排的正确性将变得更加重要。我认为这指向更像Go的方法，重点是连接不同部分的RPC机制。

<!-- 
The model of leveraging decades of prior research was fascinating and I'm so happy we took this approach.  I literally
tried not to invent anything new.  I used to joke that our job was to sift through decades of research and attempt to
combine them in new and novel ways.  Although it sounds less glamorous, the reality is that this is how a lot of our
industry's innovation takes place; very seldom does it happen by inventing new ideas out of thin air. 
-->
利用几十年前的研究模型非常吸引人，我很高兴我们采用这种方法。我确实试图不发明任何新东西。我曾经开玩笑说，我们的工作是筛选数十年的研究，并尝试以新的和新颖的方式将它们结合起来。虽然听起来不那么迷人，但事实是，这就是我们行业创新的重要方式;通过凭空发明新想法很少发生这种情况。

<!-- 
Anyway, there you have it.  Next up in the series, we will talk about Battling the GC. 
-->
无论如何，你有它。接下来我们将讨论对抗GC的问题。
