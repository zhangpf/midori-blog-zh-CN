---
title: Midori博客系列翻译（0）——介绍
date: 2018-10-20 14:55:07
tags: [操作系统, Midori, 翻译, 微软]
categories: 中文
---

<!-- 
Enough time has passed that I feel safe blogging about my prior project here at
Microsoft, "Midori."  In the months to come, I'll publish a dozen-or-so articles
covering the most interesting aspects of this project, and my key take-aways. 
-->
已经离开了足够长的时间，因此我觉得在博客中谈论以前在微软的“Midori”项目是安全的。
在接下来的几个月里，我将发表十余篇文章，
以涵盖这个项目最有趣的方面，以及我认为的主要教训。

<!-- 
Midori was a research/incubation project to explore ways of innovating
throughout Microsoft's software stack.  This spanned all aspects, including the
programming language, compilers, OS, its services, applications, and the overall
programming models.  We had a heavy bias towards cloud, concurrency, and safety.
The project included novel "cultural" approaches too, being 100% developers and
very code-focused, looking more like the Microsoft of today and hopefully
tomorrow, than it did the Microsoft of 8 years ago when the project began. 
-->
Midori是一个研究/孵化项目，它的目标是在整个微软软件栈上的探索可能的创新。
其涵盖了包括编程语言、编译器、操作系统及其服务、应用程序和整体编程模型在内的所有方面。
在该项目中，我们侧重于对云计算、并发和安全的考虑。
同时，该项目包含了新颖的“文化”方法——全员开发以及非常专注于代码，
因此它看起来更像是今天微软的样子，以及对微软未来所期望的模样，而不是8年前项目开始时的微软。
<!-- 
I worked on Midori from 2009 until we transitioned the teams to their respective
new homes during 2012-2014.  I led the groups focusing on the developer
experience: language, compilers, core frameworks, concurrency models, and
IDEs/tools.  And I wrote lots of code the whole time. -->
我于2009年开始在Midori项目工作，直到2012至2014年期间，
团队中的各个成员相继离开而去了各自新团队。在这期间，我带领团队专注于面向开发者的体验，
这包括编程语言、编译器、核心框架、并发模型和IDE工具等，同时也写了不少的代码。

<!-- Although we started with C# and .NET, we were forced to radically depart in the
name of security, reliability, and performance.  Now, I am helping to bring many
of those lessons learned back to the shipping products including, perhaps
surprisingly, C++.  Most of my blog entries will focus on the key lessons that
we're now trying to apply back to the products, like asynchrony everywhere,
zero-copy IO, dispelling the false dichotomy between safety and performance,
capability-based security, safe concurrency, establishing a culture of technical
debate, and more. -->
虽然起初我们从C#和.NET技术开始，但在离开时Midori最终走向了对安全性、可靠性和性能的追求。
而现在，我正在帮助将Midori的许多经验教训带回到交付的产品中，这也包括可能会令人惊讶的C++。
因此，我的大多数博客文章都将重点关注于那些我们正尝试用于改善现有其他产品的关键教训，
例如，无处不在的异步、零拷贝I/O、对安全和性能不可调和性的消除、
基于功能的（capability-based）安全、安全并发、建立关于技术的辩论文化等。

<!-- 
I'll be the first to admit, none of us knew how Midori would turn out.  That's
often the case with research.  My biggest regret is that we didn't OSS it from
the start, where the meritocracy of the Internet could judge its pieces
appropriately.  As with all big corporations, decisions around the destiny of
Midori's core technology weren't entirely technology-driven, and sadly, not even
entirely business-driven.  But therein lies some important lessons too.  My
second biggest regret is that we didn't publish more papers.  This blog series
may help to recitify some of this. 
-->
我得首先承认，于初大家都不知道Midori会怎么样发展，因为研究通常就是这样。
而我最大的遗憾是，从一开始就没有将它开源，因为开源可以很好地使互联网的各类优秀开发者对其进行评判。
与所有其他大公司一样，围绕Midori核心技术命运的决策并非完全由技术驱动，
并且可悲的是，甚至不完全由业务所驱动，于此也有一些重要的教训。
我的第二大遗憾是我们没有发表更多关于Midori的论文，但该系列博客可能有助于重新阐述其中的部分内容。

<!-- I shall update this list as new articles are published: -->
在我发布新文章时，也将同时更新此列表：

<!-- 
1. [A Tale of Three Safeties](/2015/11/03/a-tale-of-three-safeties/)
2. [Objects as Secure Capabilities](/2015/11/10/objects-as-secure-capabilities/)
3. [Asynchronous Everything](/2015/11/19/asynchronous-everything/)
4. [Safe Native Code](/2015/12/19/safe-native-code)
5. [The Error Model](/2016/02/07/the-error-model)
6. [Performance Culture](/2016/04/10/performance-culture)
7. [15 Years of Concurrency](/2016/11/30/15-years-of-concurrency/) 
-->

1. [三类安全性的故事](/2018/10/24/midori/1-a-tale-of-three-safeties/)
2. [对象即安全权能](/2018/11/18/midori/2-objects-as-secure-capabilities/)
3. [一切皆异步](/2018/11/25/midori/3-asynchronous-everything/)
4. 安全的原生代码
5. 错误处理模型
6. 关于性能的文化
7. 关于并发的15年

<!-- 
Midori was a fascinating journey, and the most fun I've had in my career
to-date.  I look forward to sharing some of that journey with you.
-->
Midori是一段迷人的旅程，也是我职业生涯中迄今为止最有趣的事情，因此期待与您分享这一旅程。
