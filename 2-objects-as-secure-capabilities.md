---
title: Midori博客系列翻译（2）——对象即安全权能
date: 2018-11-18 13:15:00
tags: [操作系统, Midori, 翻译, 安全, 权能]
categories: 中文
---

<!-- 
[Last time](http://joeduffyblog.com/2015/11/03/a-tale-of-three-safeties/), we
saw how Midori built on a foundation of type, memory, and concurrency safety.
This time, we will see how this enabled some novel approaches to security.
Namely, it let our system eliminate [ambient authority and access control](
https://en.wikipedia.org/wiki/Ambient_authority) in favor of [capabilities](
https://en.wikipedia.org/wiki/Capability-based_security) woven into the fabric
of the system and its code.  As with many of our other principles, the
guarantees were delivered "by-construction" via the programming language and its
type system. 
-->
[在上一篇博客中](/2018/10/24/midori/1-a-tale-of-three-safeties/)，
我们已经看到Midori是如何建立在类型，内存和并发安全的基础之上的。 
在本文中，我们将看到它们又是使一些新颖的方法来实现安全变得可能，
也就是说，这些方法让我们的系统消除了[环境权限和访问控制](https://en.wikipedia.org/wiki/Ambient_authority)问题，有利于编织到系统及其代码的结构中的[权能](https://en.wikipedia.org/wiki/Capability-based_security)之上。 
与我们的许多其他原则一样，这种保证是通过编程语言及其类型系统“从构造”处提供的。

<!-- 
# Capabilities 
-->
# 权能

<!-- 
First and foremost: what the heck are capabilities? 
-->
首要的问题是，权能究竟是什么？

<!-- 
In the security systems most of us know and love, i.e. UNIX and Windows,
permission to do something is granted based on identity, typically in the form
of users and groups.  Certain protected objects like files and system calls have
access controls attached to them that restrict which users and groups can use
them.  At runtime, the OS checks that the requested operation is permitted based
on these access controls, using ambient identity like what user is running the
current process. 
-->
在我们大多数人都知道和喜爱的安全系统中（例如UNIX和Windows），
授予做某事的许可是基于身份的，通常以用户（user）和组（group）的形式出现。 
某些受保护的对象（如文件和系统调用）具有附加到这些身份的访问控制方法，这些控制限制了哪些用户和组可以使用该对象。
在运行时，操作系统使用环境标识，如正在运行当前进程的用户，基于这些访问控制来检查是否允许执行所请求的操作。

<!-- 
To illustrate this concept, consider a simple C call to the `open` API:

	void main() {
		int file = open("filename", O_RDONLY, 0);
		// Interact with `file`...
	}
-->
为了说明这个概念，考虑对`open` API进行如下简单的C语言调用：

	void main() {
		int file = open("filename", O_RDONLY, 0);
		// 与`file`进行交互...
	}

<!-- 
Internally, this call is going to look at the identity of the current process,
the access control tables for the given file object, and permit or reject the
call accordingly.  There are various mechanisms for impersonating users, like
`su` and `setuid` on UNIX and `ImpersonateLoggedOnUser` on Windows.  But the
primary point here is that `open` just "knew" how to inspect some global state
to understand the security implications of the action being asked of it.
Another interesting aspect of this is that the `O_RDONLY` flag is passed, asking
for readonly access, which factors into the authorization process too. 
-->
在内部，此调用将查看当前进程的标识，给定的文件对象的访问控制表，以及相应地允许或拒绝调用的标识符。 
有大量的机制用于模仿用户的各种机制，例如UNIX上的`su`和`setuid`操作以及Windows上的`ImpersonateLoggedOnUser`操作。 
但这里的主要问题是`open`仅仅是“知道”如何检查一些全局状态，以了解所请求操作的的安全含义。
另一个有趣的方面是传递了要求进行只读访问的`O_RDONLY`标志，这也会影响对授权过程产生影响。

<!--
Well, what's wrong with this? 
-->
呃，那么这有何问题呢？

<!-- 
It's imprecise.  It relies on ambient state that is invisible to the program.
You can't easily audit to see the security implications of the operation.  You
just need to know how `open` works.  And thanks to its imprecision, it's easy to
get wrong, and going here wrong means security vulnerabilities.  Specifically,
it's easy to trick a program into doing something on behalf of a user that it
was never intended to do.  This is called the ["confused deputy problem"](
http://c2.com/cgi/wiki?ConfusedDeputyProblem).  All you need to do is trick the
shell or program into impersonating a superuser, and you're home free. 
-->
问题在于基于环境权限的访问控制这是不精确的，它依赖于对程序不可见的环境状态，因此无法轻松地对操作存在的安全隐患进行审计。 
你只需要知道`open`是如何工作的，而且正是由于不精确，所以很容易出错，而错误通常导致安全漏洞。 
具体来说，它很容易伪装成用户并欺骗程序做一些从未打算做的事情。 
这被称为[“混淆代理人问题”](http://c2.com/cgi/wiki?ConfusedDeputyProblem)。
你需要做的就是欺骗shell或程序以冒充超级用户，那么就几乎可以做任何特权操作。

<!-- 
[Capability-based security](https://en.wikipedia.org/wiki/Capability-based_security),
on the other hand, isn't reliant on global authority in this same way.  It uses
so-called "unforgeable tokens" to represent the ability to perform privileged
operations.  No matter how the decision gets made -- there is an entirely complex
topic of policy management and granting authority that gets into social and human
behaviors -- if the software isn't meant to perform some operation, it simply
never receives the token necessary to do said operation.  And because tokens are
unforgeable, the program can't even attempt the operation.  In a system like
Midori's, type safety meant that not only could the program not perform the
operation, it would often be caught at compile-time. 
-->
相反地，[基于权能的安全性]((https://en.wikipedia.org/wiki/Capability-based_security)不以同样的方式依赖于全局权限。 
它使用所谓的“不可伪造的令牌”来表示执行特权操作的能力。 
无论决策是如何制定的，都存在一个完全复杂的策略管理主题和关于社会和人类行为的授权行为。
总的来说，如果软件无意执行某些操作，它根本就不会收到执行这些操作所需的令牌。 
并且由于令牌是不可伪造的，程序甚至无法尝试操作。
在像Midori这样的系统中，类型安全也意味着程序不仅不能执行未授权的操作，而且通常会在编译期间捕获这些操作。

<!-- 
Insecure operations rejected at compile-time, how cool is that! 
-->
在编译时拒绝了不安全操作，这有多酷！

<!-- 
The hypothetical `open` API from earlier, as you may have guessed, would look
very different: 

	void main(File file) {
		// Interact with `file`...
	}
-->
正如您可能已经猜到的那样，之前假设的`open` API看起来会非常不同：

	void main(File file) {
		// 和`file`交互...
	}

<!-- 
OK, clearly we're not in Kansas anymore.  This is *extremely* different.  And
I've just passed the buck.  *Someone else* has to show up with a File object?
How do they get one? 
-->
好的，显然我们不再是在基于环境权限的访问控制范畴内了，那么事情将变得*非常*不同。 
我刚刚没有提到的是，这里的*其他部分（调用者）*必须获得一个File对象，那他们又是如何得到的？

<!--
The trite answer is, who cares, that's up to the caller.  But if they *do* show
up with one, they must have been authorized to get it, because object references
in a type safe system are unforgeable.  The matter of policy and authorization
are now pushed to the source where, arguably, they belong.
-->
老套的回答是，没有人会谁在乎，获得的方式完全取决于调用者。
但是如果他们*确实*持有File句柄，那么它们必须被授权访问File，
因为在类型安全的系统中，对象引用是不可伪造的。 
策略和授权的问题现在被推到可以说它们本来就属于的源头处。

<!-- 
I'm over-simplifying a little bit, since this answer likely raised more questions
than it actually answered.  Let's keep digging deeper. 
-->
我想我可能过度简化了一点，因为这个回答可能会产生更多的问题。 
那么让我们继续深入分析。

<!-- 
So, again, let's ask the question: how does one get their hands on a File object? 
-->
那么，让我们再问一个问题：如何获得File对象？

<!-- 
The code above neither knows nor cares whether where it came from.  All it knows
is it is given an object with a File-like API.  It might have been `new`'d up by
the caller.  More likely, it was obtained by consulting a separate entity, like
a Filesystem or a Directory, both of which are also capability objects: 
-->
上面的代码既不知道也不关心File来自何处。 
它只知道给它一个具有类File的API的对象。 
它可能是由调用者通过`new`操作获得， 
更有可能的情况是，它是通过调用一个单独的实体获得的，比如文件系统或目录，而这两个实体也都是权能对象：

	Filesystem fs = ...;
	Directory dir = ... something(fs) ...;
	File file = ... something(dir) ...;
	MyProgram(file);

<!-- 
You might be getting really angry at me now.  Where did `fs` come from?  How did
I get a Directory from `fs`, and how did I get a File from `dir`?  I've just
squished all the interesting topics around, like a water balloon, and answered
nothing at all! 
-->
你现在可能真的会对我生气了。
`fs`又来自哪里？我又如何从`fs`获取Directory对象？我又是如何从`dir`获取File对象的？
我刚刚把所有有趣的话题都挤到一起，就像水球一样，但却什么都没回答！

<!-- 
The reality is that those are all the interesting questions you encounter now
when you try to design a filesystem using capabilities.  You probably don't want
to permit free enumeration of the entire filesystem hierarchy, because if you
get access to a Filesystem object -- or the system's root Directory -- you can
access everything, transitively.  That's the sort of thinking you do when you
begin dealing with capabilities.  You think hard about information encapsulation
and exposure, because all you've got are objects to secure your system.
Probably, you'll have a way that a program requests access to some state
somewhere on the Filesystem, declaratively, and then the "capability oracle"
decides whether to give it to you.  This is the role our application model
played, and is how `main` got its hands on the capabilities a program's manifest
demanded that it needs.  From that point onwards it's just objects.  The key is
that nowhere in the entire system will you find the classical kind of ambient
authority, and so none of these abstractions can "cheat" in their construction. 
-->
现实情况是，当你尝试使用权能设计文件系统时，这些都是你现在将遇到的所有有趣问题。
你应该不希望允许用户自由地在整个文件系统层次结构上进行枚举访问，
因为如果用户可以访问Filesystem对象，或系统的根目录，那么它其实可以以向下传递的方式访问所有内容。
这就是你开始和权能打交道时所做的那种想法。
您认真考虑信息封装和曝光，因为您所拥有的只是保护系统安全的对象。也许，你会有一种方法，一个程序请求在文件系统的某个地方请求访问某个状态，声明，然后“权能母体”决定是否给你。这是我们的应用程序模型所扮演的角色，主要是如何`main`掌握程序清单所需的权能。从那时起，它只是对象。关键是整个系统中没有任何地方可以找到经典的环境权威，因此这些抽象都不能在其构造中“作弊”。

<!-- 
A classic paper, [Protection](
http://research.microsoft.com/en-us/um/people/blampson/08-Protection/Acrobat.pdf),
by Butler Lampson clearly articulates some of the key underlying principles, like
unforgeable tokens.  In a sense, each object in our system is its own "protection
domain."  I also love [Capability Myths Demolished](
http://srl.cs.jhu.edu/pubs/SRL2003-02.pdf)'s way of comparing and contrasting
capabilities with classical security models, if you want more details (or
incorrectly speculate that they might be isomorphic). 
-->
Butler Lampson的一篇经典论文[“Protection”](http://research.microsoft.com/en-us/um/people/blampson/08-Protection/Acrobat.pdf)清楚地阐明了一些设计上关键基本原则，例如不可伪造的令牌。 
从某种意义上说，我们系统中的每个对象都是它自己的“保护域”。
如果想了解更多的细节（或者错误地任务访问控制列表和基于权能的系统是等价的），
那么我也推荐[“Capability Myths Demolished”](http://srl.cs.jhu.edu/pubs/SRL2003-02.pdf)中权能与经典安全模型进行比较和对比的方式。

<!--
Midori was by no means the first to build an operating systems with object
capabilities at its core.  In fact, we drew significant inspiration from
[KeyKOS](http://www.cis.upenn.edu/~KeyKOS/NanoKernel/NanoKernel.html) and its
successors [EROS](https://en.wikipedia.org/wiki/EROS_(microkernel)) and
[Coyotos](http://www.coyotos.org/docs/misc/eros-comparison.html).  These
systems, like Midori, leveraged object-orientation to deliver capabilities.  We
were lucky enough to have some of the original designers of those projects on
the team. 
-->
Midori绝不是第一个以对象权能为核心构建操作系统的系统。
事实上，我们从[KeyKOS](http://www.cis.upenn.edu/~KeyKOS/NanoKernel/NanoKernel.html)及其后继者[EROS](http://t.cn/EVRHoZV)和[Coyotos](http://www.coyotos.org/docs/misc/eros-comparison.html)中获得了重要的灵感。 
这些系统像Midori一样，利用面向对象方式来提供权能，我们很幸运的是能够在团队中拥有这些项目的一些最初设计者。

<!-- Before moving on, a warning's in order: some systems confusingly use the term
"capability" even though aren't true capability systems.  [POSIX defines such a
system](http://c2.com/cgi/wiki?PosixCapabilities) and so [both Linux and Android
inherit it](https://www.kernel.org/pub/linux/libs/security/linux-privs/kernel-2.2/capfaq-0.2.txt).
Although POSIX capabilities are nicer than the typical classical ambient state
and access control mechanisms -- enabling finer-grained controls than usual --
they are closer to them than the true sort of capability we're discussing here.
-->

在继续讨论之前，按顺序发出警告：
即使某些系统不是真正的权能系统，它们也会混淆地使用“capability”这个术语。 
例如，[POSIX定义了这样一个系统](http://c2.com/cgi/wiki?PosixCapabilities)，
因此[Linux和Android都继承使用了它](https://www.kernel.org/pub/linux/libs/security/linux-privs/kernel-2.2/capfaq-0.2.txt)。 
虽然POSIX的“权能”比典型的经典基于环境状态和访问控制机制表现更好，
因为它实现了比这些方式更细粒度的控制，但它们确实比我们在这里讨论的真正权能更接近经典模型。

<!-- # Objects and State -->
# 对象和状态

<!-- 
A nice thing about capabilities just being objects is that you can apply existing
knowledge about object-orientation to the domains of security and authority.
-->
作为对象的权能的一个好处是，
你可以将有关面向对象的现有知识应用于安全和权限领域。

<!-- Since objects represent capabilities, they can be as fine or coarse as you
wish.  You can make new ones through composition, or modify existing ones
through subclassing.  Dependencies are managed just like any dependencies in an
object-oriented system: by encapsulating, sharing, and requesting references to
objects.  You can leverage all sorts of [classic design patterns](
https://en.wikipedia.org/wiki/Design_Patterns) suddenly in the domain of
security.  I do have to admit the simplicity of this idea was jarring to some.
-->
由于对象代表着权能，因此它们可以如你所希望的那样进行细粒度或粗粒度控制。
您可以通过合成方式创建新的权能，或通过继承方式修改现有的权能。 
依赖关系的管理方式与面向对象系统中的任何依赖关系一样：通过封装，共享和请求对象的引用。 
因此你可以在安全领域利用各种[经典的设计模式](https://en.wikipedia.org/wiki/Design_Patterns)。 
但我不得不承认这个想法过于简单，以至于使某些人感到震惊。

<!-- 
One fundamental idea is [revocation](http://c2.com/cgi/wiki?RevokableCapabilities).
An object has a type and our system let you substitute one implementation in place
of another.  That means if you ask me for a Clock, I needn't give you access to
a clock for all time, or even the real one for that matter.  Instead, I can give
you my own subclass of a Clock that delegates to the real one, and rejects your
attempts after some event occurs.  You've got to either trust the source of the
clock, or explicitly safe-guard yourself against it, if you aren't sure. 
-->
一个基本的想法是[撤销（revocation）](http://c2.com/cgi/wiki?RevokableCapabilities)。 
对象是具有类型的，我们的系统允许使用另一个实现来替换现有的实现。 
这意味着如果你向我请求一个Clock对象，
我无需在任何时候都向你授予访问时钟的权限，我甚至都不需向你提供真正的Clock对象。 
相反地，我可以向你提供我自己实现的一个Clock子类，
它作为真正Clock的代理，并可以在某些事件发生后拒绝你的请求。 
因此你必须要么信任时钟源，要么在在不确定的情况下，显式地保护自己免受攻击。

<!-- 
Another concept is state.  In our system, we banned mutable statics,
by-construction at compile-time, in our programming language.  That's right, not
only could a static field only be written to once, but the entire object graph
it referred to was frozen after construction.  It turns out mutable statics are
really just a form of ambient authority, and this approach prevents someone
from, say, caching a Filesystem object in a global static variable, and sharing
it freely, thereby creating something very similar to the classical security
models we are seeking to avoid.  It also had many benefits in the area of safe
concurrency and even gave us performance benefits, because statics simply became
rich constant object graphs that could be frozen and shared across binaries. 
-->
另一个概念是状态。
在我们的系统中，我们通过在编译期间“从构造”的方式，在编程语言中禁掉了可变的静态变量。
这是正确的，不仅静态字段只能被写入一次，而且它所引用的整个对象图在构造之后也将被冻结。 
事实证明，可变静态变量实际上只是环境权限的一种形式，
不可变静态变量可以阻止用户在全局静态变量中缓存Filesystem对象，
并自由地共享它，从而构造出一些和经典安全模型非常类似，而且是Midori极力避免的东西。 
不可变静态变量在安全并发方面也有很多好处，甚至给我们带来了性能优势，
因为这种方式下，静态只是变成了更加丰富的常量对象图，可以在二进制文件中固化和共享。

<!-- 
The total elimination of mutable statics had an improvement to our system's
reliability that is difficult to quantify, and difficult to understate.  This is
one of the biggest things I miss. 
-->
完全消除可变静态变量对Midori系统的可靠性
带来了难以量化和低估的改善，而这也是我最怀念的地方之一。

<!-- 
Recall my mention of Clock above.  This is an extreme example, however, yes,
that's right, there was no global function to read time, like C's `localtime` or
C#'s `DateTime.Now`.  To get the time, you must explicitly request a Clock
capability.  This has the effect of eliminating non-determinism from an entire
class of functions.  A static function that doesn't do IO -- [something we can
ascertain in our type system (think Haskell monads)](
http://research.microsoft.com/apps/pubs/default.aspx?id=170528) -- now becomes
purely functional, memoizable, and even something we can evaluate at
compile-time (a bit like [`constexpr`](
http://en.cppreference.com/w/cpp/language/constexpr) on steroids). 
-->
回想一下上面提到的Clock，这是一个极端的例子。
但没错的是，它没有诸如C的`localtime`或C#的`DateTime.Now`的读取时间的全局函数，
而为了获得时间，你必须显式地请求Clock权能，而这具有消除整个类函数中非确定性的效果。
一个无需IO，即[可以在类型系统中确定（想想Haskell的monad）](http://research.microsoft.com/apps/pubs/default.aspx?id=170528) 
的静态函数，现在变得纯函数化、可记忆化并且甚至可以在编译时进行eval
（这有点类似于[`constexpr` on steroids](http://en.cppreference.com/w/cpp/language/constexpr)）。

<!-- 
I'll be the first to admit, there was a maturity process that developers went
through, as they learned about the design patterns in an object capability
system.  It was common for "big bags" of capabilities to grow over time, and/or
for capabilities to be requested at an inopportune time.  For example, imagine
a Stopwatch API.  It probably needs the Clock.  Do you pass the Clock to every
operation that needs to access the current time, like Start and Stop?  Or do you
construct the Stopwatch with a Clock instance up-front, thereby encapsulating
the Stopwatch's use of the time, making it easier to pass to others (recognizing,
importantly, that this essentially grants the capability to read the time to the
recipient).  Another example, if your abstraction requires 15 distinct
capabilities to get its job done, does its constructor take a flat list of 15
objects?  What an unwieldy, annoying constructor!  Instead, a better approach is
to logically group these capabilities into separate objects, and maybe even use
contextual storage like parents and children to make fetching them easier. 
-->
我将首先承认，这将存在一个逐渐成熟的过程，
是开发者需要面对的，正如他们学习了对象权能系统中的设计模式。 
权能的“大袋子”随着时间的推移而增长，以及在不合时宜的时候请求权能是很常见的。
例如，设想存在一个秒表Stopwatch的API，它可能会需要Clock，
那么你是否需要将Clock传递给需要访问当前时间的每个操作，例如Start和Stop？
或者你是否预先构建了一个带有Clock实例的Stopwatch，
从而将秒表对时间的使用进行封装，使其更容易传递给其他对象
（重要的是，意识到这基本上向接收者赋予了读取时间的权能）。
另一个例子，如果你的抽象需要15个不同的权能才能完成它的工作，
那么它的构造函也需要使用15个参数的列表？
这将是多么笨重和烦人的构造函数！
相反，更好的方法是将这些权能逻辑地分组到单独的对象中，
甚至可以使用父类和子类等上下文存储来简化它们的获取。

<!-- 
The weaknesses of classical object-oriented systems also rear their ugly heads.
Downcasting, for example, means you cannot entirely trust subclassing as a means
of [information hiding](https://en.wikipedia.org/wiki/Information_hiding).  If
you ask for a File, and I supply my own CloudFile that derives from File and adds
its own public cloud-like functions to it, you might sneakily downcast to
CloudFile and do things I didn't intend.  We addressed this with severe
restrictions on casting and by putting the most sensitive capabilities on an
entirely different plan altogether... 
-->
经典的面向对象系统的缺陷也给这种方式带来了弱点。 
例如，向下类型转换（downcasting）意味着你不能完全信任继承作为[信息隐藏](https://en.wikipedia.org/wiki/Information_hiding)的手段。 
如果你请求一个File，而我提供了派生自File的CloudFile类，它向其添加了自己的类似公有云的功能，那么你可能会悄悄地向下转换为CloudFile并执行我不想要的操作。 
我们通过对类型转换的严格限制，以及将最敏感的权能放在另一个完全不同的计划上来解决此问题……

<!-- # Distributed Capabilities -->
# 分布式权能

<!-- 
I'll briefly touch on an area that warrants a lot more coverage in a future
post: our asynchronous programming model.  This model formed the foundation of
how we did concurrent, distributed computing; how we performed IO; and, most
relevant to this discussion, how capabilities could extend their reach across
these critical domains. 
-->
我将简要介绍在未来的帖子中需要进一步涉及的领域：我们的异步编程模型。 
该模型构成了我们如何进行并发和分布式计算的基础，和我们执行IO的方式，
以及与本文最相关的是，权能是如何扩展它们在这些关键域中的覆盖范围的。

<!-- 
In the Filesystem example above, our system often hosted the real object behind
that Filesystem reference in a different process altogether.  That's right,
invoking a method actually dispatched a remote call to another process, which
serviced the call.  So, in practice, most, but not all, capabilities were
asynchronous objects; or, more precisely, unforgeable tokens that permit one to
talk with them, something we called an "eventual" capability.  The Clock was a
counter-example to this.  It was something we called a "prompt" capability:
something that wrapped a system call, rather than a remote call.  But most
security-related capabilities tended to be remote, because most interesting
things that require authority bottom out on some kind of IO.  It's rare you need
authority to simply perform a computation.  In fact, the filesystem, network
stack, device drivers, graphics surfaces, and a whole lot more took the form of
eventual capabilities. 
-->
在上面的Filesystem示例中，
我们的系统通常在不同的进程中托管该Filesystem后面引用的真实对象。 
这种方式下，调用一个方法实际上是将一个远程调用分派到另一个进程上，
而进程为该调用提供相应的服务。
因此，在实践中，大多数（但不是全部的）权能都是异步对象，或者更确切地说，是允许与服务交互的不可伪造的令牌，我们称之为“最终（eventual）权能”。
而Clock却是一个反例，它是我们称之为“提示（prompt）权能”——它包含系统调用而不是远程调用。 
但是大多数与安全相关的权能往往是远程的，因为大多数需要权限的操作通常会最终触及到IO上，而很少需要权限来仅仅执行简单地计算。 
而实际上，文件系统、网络堆栈、设备驱动程序、图形界面以及更多的子系统都采用了最终权能的形式。

<!-- 
This unification of overall security in the OS and how we built distributed, and
highly concurrent, secure systems, was one of our largest, innovative, and most
important accomplishments. 
-->
这种操作系统整体安全性的统一以及我们构建分布式的，高度并发的安全系统的方式，
是我们最显著，最具创新性和最重要的成就之一。

<!-- I should note, like the idea of capabilities in general, similar ideas were
pioneered well before Midori.  Although we didn't use the languages directly,
the ideas from the [Joule](
https://en.wikipedia.org/wiki/Joule_(programming_language)) language and, later,
[E](https://en.wikipedia.org/wiki/E_(programming_language)), laid some very
powerful foundations for us to build upon.  [Mark Miller's 2006 PhD thesis](
http://www.erights.org/talks/thesis/markm-thesis.pdf) is a treasure trove of
insights in this entire area.  We had the privilege of working closely with one
of the brightest minds I've ever worked alongside, who happened to have been a
chief designer of both systems. -->
我应该指出的是，就像通用权能的想法一样，类似的想法在Midori之前就已经存在。
虽然我们没有直接使用，但是来自于[Joule语言](http://t.cn/EV8ovNL)
和后来的[E语言](http://t.cn/EV8S8gf)
的想法为我们提供了非常强大的构建基础。
[Mark Miller在2006年的博士论文](http://www.erights.org/talks/thesis/markm-thesis.pdf)是这整个领域的重要财富。 
我们有幸与我曾合作过的最聪明的人之一密切合作，而他恰好是这两个系统的首席设计师。

<!-- # Wrapping Up -->
# 封装起来

<!-- 
There is so much to say about the benefits of capabilities.  The foundation of
type safety let us make some bold leaps forward.  It led to a very different
system architecture than is commonplace with ambient authority and access
controls.  This system brought secure, distributed computing to the forefront in
a way that I've never witnessed before.  The design patterns that emerged really
embraced object-orientation to its fullest, leveraging all sorts of design
patterns that suddenly seemed more relevant than ever before. 
-->
关于权能的优点还有太多地方可以说的。
总之，类型安全的基础让我们大踏步前进，它产生了一种与环境权限和访问控制相比非常不同的系统架构。
该系统以前所未有的方式将安全的分布式计算带到了最前沿。
出现的设计模式确实充分利用了面向对象，
也充分利用了各种看起来比以往更加重要的设计模式。

<!--
We never did get much real-world exposure on this model.  The user-facing
aspects were under-explored compared to the architectural ones, like policy
management.  For example, I doubt we'd want to ask my mom if she wants to let
the program use a Clock.  Most likely we'd want some capabilities to be granted
automatically (like the Clock), and others to be grouped, through composition,
into related ones.  Capabilities-as-objects thankfully gives us a plethora of
known design patterns for doing this.  We did have a few honey pots, and none
ever got hacked (well, at least, we didn't know if we did), but I cannot attest
for sure about the quantifiable security of the resulting system.  Qualitatively
I can say we felt better having the belts-and-suspenders security at many layers
of the system's construction, but we didn't get a chance to prove it at scale.
-->
我们从未在该模型上进行较多的曝光。
与策略管理等体系结构方面相比，面向用户的层面还未得到充分研究。
例如，我很怀疑我们是否想在程序界面中提示我的母亲，她是否想让程序使用Clock。
最有可能的方式是，我们希望自动授予某些权能（如时钟），并将其他权能通过组合的方式分组为相关的权能，
而作为对象的权能幸运地为我们提供了大量已知的设计模式。
我们的系统中确实有一些蜜罐，却没有一个被黑客攻击（好吧，至少我们尚不知道有被攻击成功过），
但我无法确定最终系统的可量化安全性。
因此我可以定性地说，在系统结构的许多层面上感觉具有更好的冗余安全性，但我们却没有机会大规模地加以证明。

<!-- 
In the next article, we'll dig deeper into the asynchronous model that ran deep
throughout the system.  These days, asynchronous programming is a hot topic,
with `await` showing up in [C#](
https://msdn.microsoft.com/en-us/library/hh156528.aspx), [ECMAScript7](
http://tc39.github.io/ecmascript-asyncawait/), [Python](
https://www.python.org/dev/peps/pep-0492/), [C++](
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4134.pdf), and more.
This plus the fine-grained decomposition into lightweight processes connected by
message passing were able to deliver a highly concurrent, reliable, and
performant system, with asynchrony that was as easy to use as in all of those
languages.  See you next time!
-->
在下一篇博客中，我们将深入研究贯穿于整个系统的异步模型。
异步编程在当前是一个热门话题，例如`await`出现在[C#](https://msdn.microsoft.com/en-us/library/hh156528.aspx)，
[ECMAScript7](http://tc39.github.io/ecmascript-asyncawait/)，
以及[Python](https://www.python.org/dev/peps/pep-0492/)和
[C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4134.pdf)等语言中。
同时加上通过消息传递方式连接的细粒度分解的轻量级进程，
可实现高度并发，可靠且高性能的系统，其异步性与所有这些语言一样易于使用。
下篇博客见！
