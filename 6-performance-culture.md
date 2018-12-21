---
title: Midori博客系列翻译（6）——性能的文化
date: 2018-12-19 15:58:00
tags: [操作系统, Midori, 翻译, 性能]
categories: 中文
---

<!-- 
In this essay, I'll talk about "performance culture."  Performance is one of the key pillars of software engineering,
and is something that's hard to do right, and sometimes even difficult to recognize.  As a famous judge once said, "I
know it when I see it."  I've spoken at length about [performance](/2010/09/06/the-premature-optimization-is-evil-myth/)
and [culture](/2013/02/17/software-leadership-series/) independently before, however the intersection of the two is
where things get interesting.  Teams who do this well have performance ingrained into nearly all aspects of how the
team operates from the start, and are able to proactively deliver loveable customer experiences that crush the
competition.  There's no easy cookie-cutter recipe for achieving a good performance culture, however there are certainly
some best practices you can follow to plant the requisite seeds into your team.  So, let's go! 
-->
在这篇文章中，我将谈论“绩效文化”。绩效是软件工程的关键支柱之一，并且难以做到，有时甚至难以识别。 正如一位着名的法官曾经说过的那样，“当我看到它时，我就知道了。”我之前已经详细地谈过了表演和文化，但两者之间的关系是事情变得有趣。 能够做到这一点的团队的绩效几乎贯穿于团队从一开始就如何运作的各个方面，并且能够主动提供可以摧毁竞争对手的可爱的客户体验。 没有简单的千篇一律的方法来实现良好的绩效文化，但是你可以遵循一些最佳实践来将必要的种子植入你的团队。 那么，我们走吧！

<!-- 
# Introduction 
-->
# 介绍

<!-- 
Why the big focus on performance, anyway? 
-->
无论如何，为什么要关注性能呢？

<!-- 
Partly it's my background.  I've worked on systems, runtimes, compilers, ... things that customers expect to be fast.
It's always much easier to incorporate goals, metrics, and team processes at the outset of such a project, compared to
attempting to recover it later on.  I've also worked on many teams, some that have done amazing at this, some that have
done terribly, and many in between.  The one universal truth is that the differentiating factor is always culture. 
-->
部分是我的背景。 我曾经研究过系统，运行时，编译器......客户期望快速发展的东西。 与尝试稍后恢复它相比，在这样的项目开始时整合目标，指标和团队流程总是容易得多。 我也曾参与很多团队的工作，有些团队在这方面做得很棒，有些团队做得非常糟糕，而且很多团队介于两者之间。 一个普遍的事实是，差异化因素始终是文化。

<!-- 
Partly it's because, no matter the kind of software, performance is almost always worse than our customers would like
it to be.  This is a simple matter of physics: it's impossible to speed up all aspects of a program, given finite time,
and the tradeoffs involved between size, speed, and functionality.  But I firmly believe that on the average teams spend
way less attention to developing a rigorous performance culture.  I've heard the "performance isn't a top priority for
us" statement many times only to later be canceled out by a painful realization that without it the product won't
succeed. 
-->
部分是我的背景。 我曾经研究过系统，运行时，编译器......客户期望快速发展的东西。 与尝试稍后恢复它相比，在这样的项目开始时整合目标，指标和团队流程总是容易得多。 我也曾参与很多团队的工作，有些团队在这方面做得很棒，有些团队做得非常糟糕，而且很多团队介于两者之间。 一个普遍的事实是，差异化因素始终是文化。

<!-- 
And partly it's just been top of mind for all of us in DevDiv, as we focus on .NET core performance, ASP.NET
scalability, integrating performance-motivated features into C# and the libraries, making Visual Studio faster, and
more.  It's particularly top of mind for me, as I've been comparing our experiences to my own in [Midori](
/2015/11/03/blogging-about-midori/) (which heavily inspired this blog post). 
-->
部分它只是DevDiv中我们所有人的头脑，因为我们专注于.NET核心性能，ASP.NET可扩展性，将性能驱动的功能集成到C＃和库中，使Visual Studio更快，等等。 这对我来说尤其重要，因为我一直在将我们的经历与我在Midori中的经历进行比较（这对博客文章有很大启发）。

<!-- 
# Diagnosis and The Cure 
-->
# 诊断和治疗

<!-- 
How can you tell whether your performance culture is on track?  Well, here are some signs that it's not: 
-->
您如何判断您的表现文化是否走上正轨？ 好吧，这里有一些迹象表明它不是：

<!-- 
* Answering the question, "how is the product doing on my key performance metrics," is difficult.
* Performance often regresses and team members either don't know, don't care, or find out too late to act.
* Blame is one of the most common responses to performance problems (either people, infrastructure, or both).
* Performance tests swing wildly, cannot be trusted, and are generally ignored by most of the team.
* Performance is something one, or a few, individuals are meant to keep an eye on, instead of the whole team.
* Performance issues in production are common, and require ugly scrambles to address (and/or cannot be reproduced). 
-->
* 回答这个问题，“产品对我的关键绩效指标的影响如何”很难。
* 绩效经常下降，团队成员要么不知道，要么不关心，要么发现行动太迟。
* 责备是对性能问题（人员，基础设施或两者）最常见的响应之一。
* 性能测试大幅度下降，无法信任，并且通常被大多数团队忽略。
* 表演是一个或几个人应该关注的事情，而不是整个团队。
* 生产中的性能问题是常见的，并且需要难以解决的争用（和/或不能再现）。

<!-- 
These may sound like technical problems.  It may come as a surprise, however, that they are primarily human problems. 
-->
这听起来像技术问题。 然而，令人惊讶的是，它们主要是人类的问题。

<!-- 
The solution isn't easy, especially once your culture is in the weeds.  It's always easier to not dig a hole in the
first place than it is to climb out of one later on.  But the first rule when you're in a hole is to stop digging!  The
cultural transformation must start from the top -- management taking an active role in performance, asking questions,
seeking insights, demanding rigor -- while it simultaneously comes from the bottom -- engineers actively seeking to
understand performance of the code they are writing, ruthlessly taking a zero-tolerance stance on regressions, and
being ever-self-critical and on the lookout for proactive improvements. 
-->
解决方案并不容易，特别是一旦你的文化在杂草中。在第一个地方挖洞并不比以后爬出洞更容易。但是当你陷入困境时，第一条规则就是停止挖掘！文化转型必须从最高层开始 - 管理层在绩效中发挥积极作用，提出问题，寻求洞察力，要求严谨 - 同时从底层开始 - 工程师积极寻求理解他们正在编写的代码的表现，无情地采取对回归的零容忍态度，以及对自我批评和寻求主动改进的态度。

<!-- 
This essay will describe some ways to ensure this sort of a culture, in addition to some best practices I've found help
to increase its effectiveness once you have one in place.  A lot of it may seem obvious, but believe me, it's pretty
rare to see everything in here working in harmony in practice.  But when it is, wow, what a difference it can make. 
-->
本文将介绍一些确保这种文化的方法，此外还有一些我发现有助于提高效率的最佳实践。很多事情看起来很明显，但相信我，很难看到这里的一切都在实践中协调一致。但是，当它是，哇，它可以做出什么样的不同。

<!-- 
*A quick note on OSS software.  I wrote this essay from the perspective of commercial software development.  As
such, you'll see the word "management" a lot.  Many of the same principles work in OSS too.  So, if you like, anytime
you see "management," mentally transform it into "management or the project's committers."* 
-->
关于OSS软件的快速说明。我从商业软件开发的角度写了这篇文章。因此，你会看到很多“管理”这个词。许多相同的原则也适用于OSS。因此，如果您愿意，只要您看到“管理”，就会将其精神转化为“管理层或项目的提交者”。

<!-- 
# It Starts, and Ends, with Culture 
-->
# 它以文化开始，结束

<!-- 
The key components of a healthy performance culture are: 
-->
健康的表演文化的关键组成部分是：

<!-- 
1. Performance is part of the daily dialogue and "buzz" within the team.  Everybody plays a role.
2. Management must care -- truly, not superficially -- about good performance, and understand what it takes.
3. Engineers take a scientific, data-driven, inquisitive approach to performance.  (Measure, measure, measure!)
4. Robust engineering systems are in place to track goals, past and present performance, and to block regressions. 
-->
1. 表演是团队日常对话和“嗡嗡声”的一部分。 每个人都扮演着一个角色。
2. 管理层必须关心 - 真正的，而不是表面的 - 关于良好的绩效，并了解它需要什么。
3. 工程师采用科学的，数据驱动的，好奇的方法来表现。 （测量，测量，测量！）
4. 有足够的工程系统来跟踪目标，过去和现在的表现，以及阻止回归。

<!-- 
I'll spend a bit of time talking about each of these roughly in turn. 
-->
我会花一点时间粗略地谈论这些中的每一个。

<!-- 
## Dialogue, Buzz, and Communication 
-->
## 对话，嗡嗡声和沟通

<!-- 
The entire team needs to be on the hook for performance. 
-->
整个团队需要保持良好的表现。

<!-- 
In many teams where I've seen this go wrong, a single person is anointed the go-to performance guy or gal.  Now, that's
fine and can help the team scale, can be useful when someone needs to spearhead an investigation, and having a vocal
advocate of performance is great, but it *must not* come at the expense of the rest of the team's involvement. 
-->
在我看过这个出错的很多球队中，一个人被涂成了最佳表现家伙或加仑。 现在，这很好，可以帮助团队扩展，当有人需要带头进行调查时，并且拥有表现的声音很好，但它不能以团队的其他参与为代价。

<!-- 
This can lead to problems similar to those Microsoft use to have with the "test" discipline; engineers learned bad
habits by outsourcing the basic quality of their code, assuming that someone else would catch any problems that arise.
The same risks are present when there's a central performance czar: engineers on the team won't write performance tests,
won't proactively benchmark, won't profile, won't ask questions about the competitive standing of the product, and
generally won't do all the things you need all of the engineers doing to build a healthy performance culture. 
-->
这可能会导致类似于Microsoft使用“测试”规则的问题;工程师通过外包代码的基本质量来学习坏习惯，假设其他人会发现任何出现的问题。当有一个中央性能沙皇时，同样存在风险：团队中的工程师不会编写性能测试，不会主动进行基准测试，不会描述，不会询问有关产品竞争地位的问题，并且通常会赢得不要做所有工程师为建立健康的绩效文化而做的所有事情。

<!-- 
Magical things happen when the whole team is obsessed about performance.  The hallways are abuzz with excitement, as
news of challenges and improvements spreads organically.  "Did you see Martin's hashtable rebalancing change that
reduced process footprint by 30%?"  "Jared just checked in a feature that lets you stack allocate arrays.  I was
thinking of hacking the networking stack this weekend to use it -- care to join in?"  Impromptu whiteboard sessions,
off-the-cuff ideation, group learning.  It's really awesome to see.  The excitement and desire to win propels the
team forward, naturally, and without relying on some heavyweight management "stick." 
-->
当整个团队都对表现着迷时，会发生神奇的事情。随着挑战和改进的新闻有机地传播，走廊充满了兴奋。 “你有没有看到Martin的哈希表重新平衡改变，将流程占用空间减少了30％？”“Jared刚刚检查了一个允许你堆叠分配数组的功能。我本周末想要破解网络堆栈来使用它 - 关心加入？“即兴的白板会议，袖手旁观的想法，小组学习。真是太棒了。胜利的兴奋和渴望推动了团队前进，自然而且不依赖于一些重量级的管理“坚持”。

<!-- 
I hate blame and I hate defensiveness.  My number one rule is "no jerks," so naturally all critiques must be delivered
in the most constructive and respectful way.  I've found a high occurrence of blame, defensiveness, and [intellectual
dishonesty](/2015/11/02/software-leadership-9-on-the-importance-of-intellectual-honesty/) in teams that do poorly on
performance, however.  Like jerks, these are toxic to team culture and must be weeded out aggressively.  It can easily
make or break your ability to develop the right performance culture.  There's nothing wrong with saying we need to do
better on some key metric, especially if you have some good ideas on how to do so! 
-->
我讨厌责备，我讨厌防守。我的头号规则是“没有混蛋”，所以所有的批评都必须以最具建设性和尊重的方式提供。然而，我发现在表现不佳的球队中出现过高的责备，防守和智力不诚实。像混蛋一样，这些对团队文化有害，必须积极淘汰。它可以轻松地决定你发展正确的表演文化的能力。说我们需要在一些关键指标上做得更好没有错，特别是如果你对如何做到这一点有一些好的想法！

<!-- 
In addition to the ad-hoc communication, there of course needs to be structured communication also.  I'll describe some
techniques later on.  But getting a core group of people in a room regularly to discuss the past, present, and future of
performance for a particular area of the product is essential.  Although the organic conversations are powerful,
everyone gets busy, and it's important to schedule time as a reminder to keep pushing ahead. 
-->
除了ad-hoc通信之外，当然还需要结构化通信。我稍后会介绍一些技巧。但是，定期在一个房间内安排一个核心人群来讨论产品特定区域的过去，现在和未来的表现是至关重要的。尽管有机会话非常强大，但每个人都很忙碌，重要的是要安排时间作为继续推进的提醒。

<!-- 
## Management: More Carrots, Fewer Sticks 
-->
## 管理：更多的胡萝卜，更少的棍棒
<!-- 
In every team with a poor performance culture, it's management's fault.  Period.  End of conversation. 
-->
在每个表现不佳的团队中，都是管理层的错。期。谈话结束。

<!-- 
Engineers can and must make a difference, of course, but if the person at the top and everybody in between aren't deeply
involved, budgeting for the necessary time, and rewarding the work and the superstars, the right culture won't take
hold.  A single engineer alone can't possibly infuse enough of this culture into an entire team, and most certainly not
if the entire effort is running upstream against the management team. 
-->
当然，工程师可以而且必须有所作为，但如果顶层人员和中间人都没有深入参与，预算必要时间，奖励工作和超级明星，那么正确的文化就不会成功。单凭一名工程师就不可能将这种文化注入整个团队，而且如果整个工作都是针对管理团队向上游运行，那么肯定不会。

<!-- 
It's painful to watch managers who don't appreciate performance culture.  They'll often get caught by surprise and won't
realize why -- or worse, think that this is just how engineering works.  ("We can't predict where performance will
matter in advance!")  Customers will complain that the product doesn't perform as expected in key areas and, realizing
it's too late for preventative measures, a manager whose team has a poor performance culture will start blaming things.
Guess what?  The blame game culture spreads like wildfire, the engineers start doing it too, and accountability goes out
the window.  Blame doesn't solve anything.  Blaming is what jerks do. 
-->
观看不欣赏表演文化的经理人是痛苦的。他们经常会感到意外，并且不会意识到为什么 - 或者更糟糕的是，认为这就是工程的工作原理。 （“我们无法提前预测性能的重要性！”）客户会抱怨产品在关键领域没有达到预期的效果，并且意识到预防措施为时已晚，经理的团队表现不佳会开始责怪事情。你猜怎么着？责备游戏文化像野火一样传播，工程师也开始这样做，而问责制就在窗外。责备没有解决任何问题。责备是混蛋所做的。

<!-- 
Notice I said management must be "*deeply* involved": this isn't some superficial level of involvement.  Sure, charts
with greens, reds, and trendlines probably need to be floating around, and regular reviews are important.  I suppose you
could say that these are pointy-haired manager things.  (Believe me, however they do help.)  A manager must go deeper
than this, however, proactively and regularly reviewing the state of performance across the product, alongside the other
basic quality metrics and progress on features.  It's a core tenet of the way the team does its work.  It must be
treated as such.  A manager must wonder about the competitive landscape and ask the team good, insightful questions
that get them thinking. 
-->
注意我说管理层必须“深入参与”：这不是一些表面的参与。当然，绿色，红色和趋势线的图表可能需要浮动，定期评论很重要。我想你可以说这些都是尖头发的经理事。 （相信我，但他们确实提供了帮助。）然而，经理必须深入研究，然后主动并定期审查整个产品的绩效状况，以及其他基本质量指标和功能进展。这是团队工作方式的核心原则。它必须这样对待。经理必须对竞争格局感到疑惑，并向团队询问让他们思考的好的，富有洞察力的问题。

<!-- 
Performance doesn't come for free.  It costs the team by forcing them to slow down at times, to spend energy on things
other than cranking out features, and hence requires some amount of intelligent tradeoff.  How much really depends on
the scenario.  Managers need to coach the team to spend the right ratio of time.  Those who assume it will come for free
usually end up spending 2-5x the amount it would have taken, just at an inopportune time later on (e.g., during the
endgame of shipping the product, in production when trying to scale up from 1,000 customers to 100,000, etc). 
-->
表演不是免费的。通过迫使他们有时放慢速度来为团队付出代价，将能量花在除了启动功能之外的事情上，因此需要进行一些智能权衡。多少真的取决于场景。经理需要指导团队花费适当的时间比例。那些认为它将免费获得的人通常最终会花费2-5倍的金额，只是在稍后的不合时宜的时间（例如，在运输产品的最后阶段，在尝试从1,000个客户扩大规模时的生产中）到100,000，等）。

<!-- 
A mentor of mine used to say "You get from your teams what you reward."  It's especially true with performance and the
engineering systems surrounding them.  Consider two managers: 
-->
我的一位导师曾经说过“你从你的团队中得到你所奖励的东西。”对于性能和围绕它们的工程系统尤其如此。考虑两位经理：

<!-- 
* *Manager A* gives lip service to performance culture.  She, however, packs every sprint schedule with a steady stream
  of features -- "we've got to crush competitor Z and *must* reach feature parity!" -- with no time for breaks
  in-between.  She spends all-hands team meetings praising new features, demos aplenty, and even gives out a reward to
  an engineer at each one for "the most groundbreaking feature."  As a result, her team cranks out features at an
  impressive clip, delivers fresh demos to the board every single time, and gives the sales team plenty of ammo to
  pursue new leads.  There aren't performance gates and engineers generally don't bother to think much about it.

* *Manager B* takes a more balanced approach.  She believes that given the competitive landscape, and the need to
  impress customers and board members with whizbang demos, new features need to keep coming.  But she is also wary of
  building up too much debt in areas like performance, reliability, and quality for areas she expects to stick.  So she
  intentionally puts her foot on the brake and pushes the team just as hard on these areas as she does features.  She
  demands good engineering systems and live flighting of new features with performance telemetry built-in, for example.
  This requires that she hold board members and product managers at bay, which is definitely unpopular and difficult.
  In addition to a reward for "the most groundbreaking feature" award at each all-hands, she shows charts of performance
  progress and delivers a "performance ninja" award too, to the engineer who delivered the most impactful performance
  improvement.  Note that engineering systems improvements also qualify! 
-->
* 经理A为绩效文化提供口头服务。然而，她为每个冲刺计划提供了源源不断的功能 - “我们必须粉碎竞争对手Z并且必须达到功能平价！” - 没有时间中断。她花了很多时间参加团队会议，赞扬新功能，演示丰富，甚至为每个工程师提供“最具开创性功能”的奖励。结果，她的团队以一个令人印象深刻的剪辑创造出功能，提供新鲜感每次都向董事会演示，并为销售团队提供大量弹药以追求新的潜在客户。没有性能门户，工程师通常也懒得去思考它。

* 经理B采取更加平衡的方法。她认为，鉴于竞争格局，以及需要通过whizbang演示给客户和董事会成员留下深刻印象，新功能需要不断涌现。但她也担心在性能，可靠性和质量等方面为她希望坚持的领域积累太多债务。因此，她故意将自己的脚放在制动器上，并在这些区域上尽可能地推动团队。例如，她需要良好的工程系统以及内置性能遥测的新功能。这要求她让董事会成员和产品经理处于困境，这绝对不受欢迎和困难。除了在每个人手中获得“最具突破性功能”奖项之外，她还展示了性能进步图表，并为提供最具影响力的性能改进的工程师颁发了“性能忍者”奖。请注意，工程系统改进也符合条件！

<!-- 
Which manager do you think is going to ship a quality product, on time, that customers are in love with?  My money is
on Manager B.  Sometimes you've got to [slow down to speed up](
/2013/04/12/software-leadership-4-slow-down-to-speed-up/). 
-->
您认为哪位经理会按时发送客户喜爱的优质产品？我的钱是经理B的。有时候你需要放慢脚步才能加快速度。

<!-- 
Microsoft is undergoing two interesting transitions recently that are related to this point: on one hand, the
elimination of "test" as a discipline mentioned earlier; and, on the other hand, a renewed focus on engineering systems.
It's been a bumpy ride.  Surprisingly, one of the biggest hurdles to get over wasn't with the individual engineers at
all -- it was the managers!  "Development managers" in the old model got used to focusing on features, features,
features, and left most of the engineering systems work to contractors, and most of the quality work to testers.  As a
result, they were ill-prepared to recognize and reward the kind of work that is essential to building a great
performance culture.  The result?  You guessed it: a total lack of performance culture.  But, more subtly, you also
ended up with "leadership holes"; until recently, there were virtually no high-ranking engineers working on the
mission-critical engineering systems that make the entire team more productive and capable.  Who wants to make a career
out of the mere grunt work assigned to contractors and underappreciated by management?  Again, you get what you reward. 
-->
微软最近正在经历两个与这一点相关的有趣转变：一方面，消除了“测试”作为前面提到的一门学科;另一方面，重新关注工程系统。这是一个坎坷的旅程。令人惊讶的是，克服困难的最大障碍之一不是个人工程师 - 这是管理者！旧模型中的“开发经理”习惯于专注于功能，特性和功能，并将大部分工程系统工作留给承包商，并且大多数质量工作都是针对测试人员的。结果，他们没有准备好承认和奖励那些对建立卓越绩效文化至关重要的工作。结果？你猜对了：完全缺乏表现文化。但是，更巧妙的是，你最终还是出现了“领导漏洞”;直到最近，几乎没有高级工程师致力于关键任务工程系统，使整个团队更高效，更有能力。谁想通过分配给承包商的那些繁琐的工作来创造事业并且被管理层低估？再一次，你获得了你的奖励。

<!-- 
There's a catch-22 with early prototyping where you don't know if the code is going to survive at all, and so the
temptation is to spend zero time on performance.  If you're hacking away towards a minimum viable product (MVP), and
you're a startup burning cash, it's understandable.  But I strongly advise against this.  Architecture matters, and some
poorly made architectural decisions made at the outset can lay the foundation for an entire skyscraper of
ill-performing code atop.  It's better to make performance part of the feasibility study and early exploration. 
-->
有一个早期原型制作的捕获22，你不知道代码是否能够存活下来，所以诱惑就是在性能上花费零时间。如果你正在寻找最低可行产品（MVP），并且你是一个鼓励现金的创业公司，这是可以理解的。但我强烈反对这一点。建筑很重要，一开始就做出的一些制作不当的建筑决策可以为整个不良代码的摩天大楼奠定基础。最好将性能作为可行性研究和早期探索的一部分。

<!-- 
Finally, to tie up everything above, as a manager of large teams, I think it's important to get together regularly
-- every other sprint or two -- to review performance progress with the management team.  This is in addition to the
more fine-grained engineer, lead, and architect level pow-wows that happen continuously.  There's a bit of a "stick"
aspect of such a review, but it's more about celebrating the team's self-driven accomplishments, and keeping it on
management's radar.  Such reviews should be driven from the lab and manually generated numbers should be outlawed. 
-->
最后，作为大型团队的经理，为了统一上述所有内容，我认为定期召开会议 - 每隔一两个冲刺 - 来审核管理团队的绩效进展非常重要。除此之外，还有更细粒度的工程师，领导者和建筑师级别的战俘不断发生。这种评论有一点“坚持”的方面，但它更多的是关于庆祝团队的自我驱动成就，并将其保持在管理层的雷达上。这些评论应该来自实验室，手动生成的数字应该是非法的。

<!-- 
Which brings me to ... 
-->
这让我...

<!-- 
# Process and Infrastructure 
-->
# 流程和基础设施

<!-- 
"Process and infrastructure" -- how boring! 
-->
“流程和基础设施” - 多么无聊！

<!-- 
Good infrastructure is a must.  A team lacking the above cultural traits won't even stop to invest in infrastructure;
they will simply live with what should be an infuriating lack of rigor.  And good process must ensure effective use
of this infrastructure.  Here is the bare minimum in my book: 
-->
良好的基础设施是必须的。 缺乏上述文化特征的团队甚至不会停止投资基础设施; 他们只会生活在一个令人愤怒的缺乏严谨的环境中。 良好的流程必须确保有效使用此基础架构。 这是我书中的最低要求：

<!-- 
* All commits must pass a set of gated performance tests beforehand.
* Any commits that slip past this and regress performance are reverted without question.  I call this the zero
  tolerance rule.
* Continuous performance telemetry is reported from the lab, flighting, and live environments.
* This implies that performance tests and infrastructure have a few important characteristics:
    - They aren't noisy.
    - They measure the "right" thing.
    - They can be run in a "reasonable" amount of time. 
-->
* 所有提交必须事先通过一组门控性能测试。
* 任何滑过这个并且回归表现的提交都会毫无疑问地得到回复。 我称之为零容忍规则。
* 实验室，飞行和现场环境报告了连续性能遥测。
* 这意味着性能测试和基础架构具有一些重要特征：
  - 他们不吵。
  - 他们测量“正确”的东西。
  - 它们可以在“合理”的时间内运行。

<!-- 
I have this saying: "If it's not automated, it's dead to me." 
-->
我有这样一句话：“如果它不是自动化的，它对我来说已经死了。”

<!-- 
This highlights the importance of good infrastructure and avoids the dreaded "it worked fine on my computer" that
everybody, I'm sure, has encountered: a test run on some random machine -- under who knows what circumstances -- is
quoted to declare success on a benchmark... only to find out some time later that the results didn't hold.  Why is this? 
-->
这凸显了良好基础设施的重要性，并避免了可怕的“它在我的计算机上工作得很好”，我确信每个人都遇到过：在某个随机机器上运行测试 - 在谁知道什么情况下 - 被引用来宣布成功一个基准......只是在一段时间后发现结果没有成功。为什么是这样？

<!-- 
There are countless possibilities.  Perhaps a noisy process interfered, like AntiVirus, search indexing, or the
application of operating system updates.  Maybe the developer accidentally left music playing in the background on
their multimedia player.  Maybe the BIOS wasn't properly adjusted to disable dynamic clock scaling.  Perhaps it was
due to an innocent data-entry error when copy-and-pasting the numbers into a spreadhseet.  Or maybe the numbers for two
comparison benchmarks came from two, incomparable machine configurations.  I've seen all of these happen in practice. 
-->
有无数的可能性。也许是一个嘈杂的过程干扰，如AntiVirus，搜索索引或操作系统更新的应用程序。也许开发人员不小心把音乐留在他们的多媒体播放器的后台播放。可能BIOS没有正确调整以禁用动态时钟缩放。也许这是由于在将数字复制并粘贴到spreadhseet时出现无辜的数据输入错误。或者两个比较基准的数字可能来自两个无与伦比的机器配置。我已经看到所有这些都在实践中发生。

<!-- 
In any manual human activity, mistakes happen.  These days, I literally refuse to look at or trust any number that
didn't come from the lab.  The solution is to automate everything and focus your energy on making the automation
infrastructure as damned good as possible.  Pay some of your best people to make this rock solid for the rest of the
team.  Encourage everybody on the team to fix broken windows, and take a proactive approach to improving the
infrastructure.  And reward it heartily.  You might have to go a little slower, but it'll be worth it, trust me. 
-->
在任何手动人类活动中，都会发生错误。这些天，我真的拒绝查看或信任任何未来自实验室的数字。解决方案是实现一切自动化，并将精力集中在尽可能使自动化基础设施变得更好。支付一些最优秀的人才，为团队的其他成员打造坚实的基础。鼓励团队中的每个人修复损坏的窗户，并采取积极主动的方法来改善基础设施。并衷心地奖励它。你可能要慢一点，但它值得，相信我。

<!-- 
## Test Rings 
-->
## 测试戒指

<!-- 
I glossed over a fair bit above when I said "all commits must pass a set of performance tests," and then went on to
talk about how a checkin might "slip past" said tests.  How is this possible? 
-->
当我说“所有提交都必须通过一系列性能测试”时，我对上面的内容进行了一些修改，然后继续谈论签到可能会“超过”所述测试。 这怎么可能？

<!-- 
The reality is that it's usually not possible to run all tests and find all problems before a commit goes in, at least
not within a reasonable amount of time.  A good performance engineering system should balance the productivity of speedy
codeflow with the productivity and assurance of regression prevention. 
-->
实际情况是，通常不可能在提交之前运行所有测试并找到所有问题，至少在合理的时间内不会。 一个好的性能工程系统应该平衡快速代码流的生产力与回归预防的生产力和保证。

<!-- 
A decent approach for this is to organize tests into so-called "rings": 
-->
一个合适的方法是将测试组织成所谓的“环”：

<!-- 
* An inner ring containing tests that all developers on the team measure before each commit.
* An inner ring containing tests that developers on your particular sub-team measure before each commit.
* An inner ring containing tests that developers run at their discretion before each commit.
* Any number of successive rings outside of this:
    - Gates for each code-flow point between branches.
    - Post-commit testing -- nightly, weekly, etc. -- based on time/resource constraints.
    - Pre-release verification.
    - Post-release telemetry and monitoring. 
-->
* 一个内环，包含团队中所有开发人员在每次提交之前测量的测试。
* 一个内环，包含特定子团队的开发人员在每次提交之前测量的测试。
* 一个内环，包含开发人员在每次提交之前自行决定运行的测试。
* 除此之外的任意数量的连续响铃：
  - 分支之间的每个代码流点的门。
  - 提交后测试 - 每晚，每周等 - 基于时间/资源限制。
  - 预发布验证。
  - 释放后的遥测和监测。

<!-- 
As you can see, there's a bit of flexibility in how this gets structured in practice.  I wish I could lie and say that
it's a science, however it's an art that requires intelligently trading off many factors.  This is a constant source of
debate and one that the management team should be actively involved in. 
-->
正如您所看到的，在实践中如何构建它有一定的灵活性。我希望我能说谎，并说这是一门科学，但这是一门艺术需要智能地交易许多因素。这是一个不断争论的焦点，也是管理团队应该积极参与的辩论。

<!-- 
A small team might settle on one standard set of benchmarks across the whole team.  A larger team might need to split
inner ring tests along branching lines.  And no matter the size, we would expect the master/main branch to enforce the
most important performance metrics for the whole team, ensuring no code ever flows in that damages a core scenario. 
-->
一个小团队可能会在整个团队中建立一套标准的基准。一个更大的团队可能需要沿着分支线分割内环测试。无论规模大小，我们都希望主/主分支能够为整个团队强制执行最重要的性能指标，确保没有任何代码流入，从而损害核心场景。

<!-- 
In some cases, we might leave running certain pre-commit tests to the developer's discretion.  (Note, this does not mean
running pre-commit tests altogether is optional -- only a particular set of them!)  This might be the case if, for
example, the test covered a lesser-used component and we know the nightly tests would catch any post-commit regression.
In general, when you have a strong performance culture, it's okay to trust judgement calls sometimes.  Trust but verify. 
-->
在某些情况下，我们可能会将某些预先提交的测试留给开发人员自行决定。 （注意，这并不意味着运行预提交测试完全是可选的 - 只有一组特定的！）例如，如果测试覆盖了较少使用的组件并且我们知道夜间测试会捕获，则可能是这种情况任何提交后回归。一般来说，当你拥有强大的表现文化时，有时候可以信任判断。信任但要验证。

<!-- 
Let's take a few concrete examples.  Performance tests often range from micro to macro in size.  These typically range
from easier to harder to pinpoint the source of a regression, respectively.  (Micro measures just one thing, and so
fluctuations tend to be easier to understand, whereas macro measures an entire system, where fluctuations tend to take a
bit of elbow grease to track down.)  A web server team might include a range of micro and macro tests in the innermost
pre-commit suite of tests: number of bytes allocated per requests (micro), request response time (micro), ... perhaps
a half dozen other micro-to-midpoint benchmarks ..., and [TechEmpower](https://www.techempower.com/benchmarks/) (macro),
let's say.  Thanks to lab resources, test parallelism, and the awesomeness of GitHub webhooks, let's say these all
complete in 15 minutes, nicely integrated into your pull request and code review processes.  Not too bad.  But this
clearly isn't perfect coverage.  Maybe every night, TechEmpower is run for 4 hours, to measure performance over a longer
period of time to identify leaks.  It's possible a developer could pass the pre-commit tests, and then fail this longer
test, of course.  Hence, the team lets developers run this test on-demand, so a good doobie can avoid getting egg on his
or her face.  But alas, mistakes happen, and again there isn't a culture of blame or witchhunting; it is what it is. 
-->
我们来看几个具体的例子。性能测试的范围通常从微观到宏观。这些通常分别从更容易到更难以精确定位回归的来源。 （微观测量只是一件事，所以波动往往更容易理解，而宏测量整个系统，其中波动往往需要一些肘部油脂来追踪。）网络服务器团队可能包括一系列的微观和最里面的预提交测试套件中的宏测试：每个请求分配的字节数（微），请求响应时间（微），......可能还有其他六个微到中点基准......，以及TechEmpower（宏），让我们看看说。感谢实验室资源，测试并行性以及GitHub webhooks的精彩，让我们说这些都在15分钟内完成，很好地集成到您的拉取请求和代码审查流程中。还不错。但这显然不是完美的报道。也许每天晚上，TechEmpower运行4个小时，以便在更长的时间内测量性能以识别泄漏。开发人员可以通过预提交测试，然后失败这个更长的测试。因此，团队允许开发人员按需运行此测试，因此一个好的doobie可以避免在他或她的脸上吃蛋。但是，唉，错误发生了，再也没有一种责备或魔法的文化;就是这样。

<!-- 
This leads me to back to the zero tolerance rule. 
-->
这导致我回到零容忍规则。

<!-- 
Barring exceptional circumstances, regressions should be backed out immediately.  In teams where I've seen this succeed,
there were no questions asked, and no IOUs.  As soon as you relax this stance, the culture begins to break down.  Layers
of regressions pile on top of one another and you end up ceding ground permanently, no matter the best intentions of the
team.  The commit should be undone, the developer should identify the root cause, remedy it, ideally write a new test if
appropriate, and then go through all the hoops again to submit the checkin, this time ensuring good performance. 
-->
除非出现特殊情况，否则应立即撤销回归。在我看到这个成功的团队中，没有问题，也没有欠条。一旦你放松这种立场，文化就会开始崩溃。无论团队的最佳意图是什么，回归层都堆积在一起，你最终永久地放弃了地面。提交应该撤消，开发人员应该确定根本原因，补救它，理想情况下，如果合适的话，编写一个新的测试，然后再次通过所有环节提交签入，这次确保良好的性能。

<!-- 
## Measurement, Metrics, and Statistics 
-->
## 测量，度量和统计

<!-- 
Decent engineers intuit.  Good engineers measure.  Great engineers do both. 
-->
体面的工程师直觉。 好的工程师测量。 伟大的工程师做到了。

<!-- 
Measure what, though? 
-->
虽然衡量什么？

<!-- 
I put metrics into two distinct categories: 
-->
我将指标分为两个不同的类别：

<!-- 
* *Consumption metrics.*  These directly measure the resources consumed by running a test.
* *Observational metrics.*  These measure the outcome of running a test, *observationally*, using metrics "outside" of
  the system. 
-->
* *消费指标*。 这些直接测量运行测试所消耗的资源。
* *观察指标*。 它们使用系统“外部”的度量标准来衡量运行测试的结果。

<!-- 
Examples of consumption metrics are hardware performance counters, such as instructions retired, data cache misses,
instruction cache misses, TLB misses, and/or context switches.  Software performance counters are also good candidates,
like number of I/Os, memory allocated (and collected), interrupts, and/or number of syscalls.  Examples of observational
metrics include elapsed time and cost of running the test as billed by your cloud provider.  Both are clearly important
for different reasons. 
-->
消耗度量的示例是硬件性能计数器，诸如退出的指令，数据高速缓存未命中，指令高速缓存未命中，TLB未命中和/或上下文切换。软件性能计数器也是很好的候选者，例如I / O数量，分配（和收集）的内存，中断和/或系统调用的数量。观察指标的示例包括由云提供商计费的运行时间和运行测试的成本。由于各种原因，两者显然都很重要。

<!-- 
Seeing a team measure time and time alone literally brings me to tears.  It's a good measure of what an end-user will
see -- and therefore makes a good high-level test -- however it is seriously lacking in the insights it can give you.
And if there's no visibility into variance, it can be borderline useless. 
-->
看到一个团队测量时间和时间本身就让我流泪。它可以很好地衡量最终用户将会看到什么 - 因此可以进行良好的高级测试 - 但是它可以为您提供的见解严重缺乏。如果没有对方差的可见性，那么它可能是无用的。

<!-- 
Consumption metrics are obviously much more helpful to an engineer trying to understand why something changed.  In our
above web server team example, imagine request response time regressed by 30%.  All the test report tells us is the
time.  It's true, a developer can then try to reproduce the scenario locally, and manually narrow down the cause,
however can be tedious, takes time, and is likely imperfect due to differences in lab versus local hardware.  What if,
instead, both instructions retired and memory allocated were reported alongside the regression in time?  From this, it
could be easy to see that suddenly 256KB of memory was being allocated per request that wasn't there before.  Being
aware of recent commits, this could make it easy for an engineer to quickly pinpoint and back out the culprit in a
timely manner before additional commits pile on top, further obscuring the problem.  It's like printf debugging. 
-->
对于试图理解为什么会发生变化的工程师而言，消费指标显然更有帮助。在我们上面的Web服务器团队示例中，假设请求响应时间回退了30％。所有测试报告告诉我们的是时间。确实，开发人员可以尝试在本地重现场景，并手动缩小原因范围，但是由于实验室与本地硬件的差异，可能会很繁琐，需要时间，而且可能不完美。相反，如果退出指令和分配内存的报告与时间回归一起报告怎么办？从这一点可以很容易地看出，每个请求突然分配了256KB的内存，这是以前没有的。意识到最近的提交，这可以使工程师在额外的提交堆积在最顶层之前很容易地及时快速查明并退回罪魁祸首，进一步模糊了问题。这就像printf调试一样。

<!-- 
Speaking of printf debugging, telemetry is essential for long-running tests.  Even low-tech approaches like printfing
the current set of metrics every so often (e.g., every 15 seconds), can help track down where something went into the
weeds simply by inspecting a database or logfile.  Imagine trying to figure out where the 4-hour web server test went
off the rails at around the 3 1/2 hour mark.  This can can be utterly maddening without continuous telemetry!  Of
course, it's also a good idea to go beyond this.  The product should have a built-in way to collect this telemtry out
in the wild, and correlate it back to key metrics.  [StatsD](https://github.com/etsy/statsd) is a fantastic option. 
-->
说到printf调试，遥测对于长时间运行的测试至关重要。即使是低技术的方法，例如每隔一段时间（例如，每15秒）打印一组当前指标，也可以通过检查数据库或日志文件来帮助追踪某些东西进入杂草的位置。想象一下，试图找出4小时网络服务器测试在3个半小时左右的轨道上脱轨的地方。如果没有连续的遥测，这可能会令人头疼！当然，超越这个也是一个好主意。该产品应具有内置的方式来在野外收集此遥测，并将其与关键指标相关联。 StatsD是一个很棒的选择。

<!-- 
Finally, it's important to measure these metrics as scientifically as possible.  That includes tracking [standard
deviation](https://en.wikipedia.org/wiki/Standard_deviation), [coefficient of variation (CV)](
https://en.wikipedia.org/wiki/Coefficient_of_variation), and [geomean](https://en.wikipedia.org/wiki/Geometric_mean),
and using these to ensure tests don't vary wildly from one run to the next.  (Hint: commits that tank CV should be
blocked, just as those that tank the core metric itself.)  Having a statistics wonk on your team is also a good idea! 
-->
最后，尽可能科学地衡量这些指标非常重要。这包括跟踪标准偏差，变异系数（CV）和几何，并使用这些来确保测试不会从一次运行到下一次运行。 （提示：承诺坦克CV应该被阻止，就像那些能够控制核心指标本身一样。）对你的团队进行统计数据也是一个好主意！

<!-- 
## Goals and Baselines 
-->
## 目标和基线
<!-- 
Little of the above matters if you lack goals and baselines.  For each benchmark/metric pair, I recommend recognizing
four distinct concepts in your infrastructure and processes: 
-->
如果你缺乏目标和基线，上述事项很少。 对于每个基准/度量对，我建议您在基础架构和流程中识别四个不同的概念：

<!-- 
* *Current*: the current performance (which can span multiple metrics).
* *Baseline*: the threshold the product must stay above/below, otherwise tests fail.
* *Sprint Goal*: where the team must get to before the current sprint ends.
* *Ship Goal*: where the team must get to in order to ship a competitive feature/scenario. 
-->
* *当前*：当前性能（可以跨越多个指标）。
* *基线*：产品必须保持高于/低于阈值，否则测试失败。
* *冲刺目标*：在当前冲刺结束之前团队必须到达的位置。
* *发货目标*：团队必须到达的地方才能发布竞争性功能/方案。

<!-- 
Assume a metric where higher is better (like throughput); then it's usually the case that Ship Goal &gt;= Sprint Goal
&gt;= Current &gt;= Baseline.  As wins and losses happen, continual adjustments should be made. -->
假设一个指标，其中更高的更好（如吞吐量）;那么通常情况下，船舶目标> =冲刺目标> =当前> =基线。随着输赢，应该不断进行调整。

<!-- 
For example, a "baseline ratcheting" process is necessary to lock in improvements.  A reasonable approach is to ratchet
the baseline automatically to within some percentage of the current performance, ideally based on standard deviation
and/or CV.  Another approach is to require that developers do it manually, so that all ratcheting is intentional and
accounted for.  And interestingly, you may find it helpful to ratchet in the other direction too.  That is, block
commits that *improve* performance dramatically and yet do not ratchet the baseline.  This forces engineers to stop and
think about whether performance changes were intentional -- even the good ones!  A.k.a., "confirm your kill." 
-->
例如，需要“基线棘轮”过程来锁定改进。一种合理的方法是将基线自动调整到当前性能的某个百分比内，理想情况是基于标准偏差和/或CV。另一种方法是要求开发人员手动执行此操作，以便所有棘轮操作都是有意识的并加以考虑。有趣的是，你可能会发现在另一个方向上也是有帮助的。也就是说，块提交可以显着提高性能，但不会提高基线。这迫使工程师停下来思考性能变化是否是有意的 - 即使是好的！ A.k.a.，“确认你的杀人。”

<!-- 
It's of course common that sprint goals remain stable from one sprint to the next.  All numbers can't be improving all
the time.  But this system also helps to ensure that the team doesn't backslide on prior achievements. 
-->
从一个sprint到下一个sprint，sprint目标保持稳定当然很常见。所有数字都无法一直在改善。但是这个系统也有助于确保团队不会倒退以前的成就。

<!-- 
I've found it useful to organize sprint goals behind themes.  Make this sprint about "server performance."  Or "shake
out excessive allocations."  Or something else that gives the team a sense of cohesion, shared purpose, and adds a
little fun into the mix.  As managers, we often forget how important fun is.  It turns out performance can be the
greatest fun of all; it's hugely measurable -- which engineers love -- and, speaking for myself, it's a hell of a time
to pull out the scythe and start whacking away!  It can even be a time to learn as a team, and to even try out some fun,
new algorithmic techniques, like [bloom filters](https://en.wikipedia.org/wiki/Bloom_filter). 
-->
我发现组织主题背后的sprint目标很有用。制作关于“服务器性能”的这个冲刺。或者“摆脱过多的分配。”或者让团队有一种凝聚力，共同目标，并为混合添加一点乐趣的其他东西。作为管理者，我们经常忘记乐趣的重要性。事实证明，性能可以是所有人最大的乐趣;这是非常可测量的 - 工程师喜欢 - 而且，对我自己来说，这是一个时间来拔出镰刀并开始打瞌睡！它甚至可以是一个团队学习的时间，甚至可以尝试一些有趣的，新的算法技术，比如布隆过滤器。

<!-- 
Not every performance test needs this level of rigor.  Any that are important enough to automatically run pre-commit
most certainly demand it.  And probably those that are run daily or monthly.  But managing all these goals and baselines
and whatnot can get really cumbersome when there are too many of them.  This is a real risk especially if you're
tracking multiple metrics for each of your benchmarks. 
-->
并非每项性能测试都需要这种严格程度。任何足够重要的自动运行预提交肯定都需要它。也许那些每天或每月运行。但管理所有这些目标和基线，以及当它们太多时，可能会变得非常麻烦。这是一个真正的风险，特别是如果您要跟踪每个基准的多个指标。

<!-- 
This is where the idea of ["key performance indicators" (KPIs)](https://en.wikipedia.org/wiki/Performance_indicator)
becomes very important.  These are the performance metrics important enough to track at a management level, to the whole
team how healthy the overall product is at any given time.  In my past team who built an operating system and its
components, this included things like process startup time, web server throughput, browser performance on standard
industry benchmarks, and number of frames dropped in our realtime audio/video client, including multiple metrics apiece
plus the abovementioned statistics metrics.  These were of course in the regularly running pre- and post-commit test
suites, but rolling them up in one place, and tracking against the goals, was a hugely focusing exercise. 
-->
这就是“关键绩效指标”（KPI）的概念变得非常重要的地方。这些绩效指标非常重要，足以在管理层跟踪整个团队在任何给定时间内整体产品的健康程度。在我过去建立操作系统及其组件的团队中，这包括流程启动时间，Web服务器吞吐量，标准行业基准测试中的浏览器性能以及实时音频/视频客户端中丢弃的帧数，包括多个指标加上上述统计指标。这些当然是在定期运行的预先提交和提交后的测试套件中，但是将它们放在一个地方并跟踪目标，是一个非常集中的练习。

<!-- 
# In Summary 
-->
# 综上所述

<!-- 
This post just scratches the surface of how to do good performance engineering, but I hope you walk away with at least
one thing: doing performance well is all about having a great performance culture. 
-->
这篇文章只是简单介绍了如何进行良好的性能工程，但我希望你能带走至少一件事：做好表现就是要有一个很好的表现文化。

<!-- 
This culture needs to be infused throughout the entire organization, from management down to engineers, and everybody in
between.  It needs to be transparent, respectful, aggressive, data-driven, self-critical, and relentlessly ambitious.
Great infrastructure and supporting processes are a must, and management needs to appreciate and reward these, just as
they would feature work (and frequently even more).  Only then will the self-reinforcing flywheel get going. 
-->
这种文化需要贯穿整个组织，从管理层到工程师，以及介于两者之间的每个人。它需要透明，尊重，积极，数据驱动，自我批评和雄心勃勃。伟大的基础设施和支持流程是必须的，管理层需要欣赏和奖励这些，就像他们将工作（甚至更多）一样。只有这样，自增强飞轮才会开始运转。

<!-- 
Setting goals, communicating regularly, and obsessively tracking goals and customer-facing metrics is paramount. 
-->
设定目标，定期沟通，以及痴迷地追踪目标和面向客户的指标是至关重要的。

<!-- It's not easy to do everything I've written in this article.  It's honestly very difficult to remember to slow down and
be disciplined in these areas, and it's easy to trick yourself into thinking running as fast as possible and worrying
about performance later is the right call.  Well, I'm sorry to tell you, sometimes it is.  You've got to use your
intuition and your gut, however, in my experience, we tend to undervalue performance considerably compared to features. 
-->
要完成我在本文中所写的所有内容并不容易。老实说，很难记住在这些领域放慢速度并保持纪律，而且很容易欺骗自己思考尽可能快地运行，而后来担心性能是正确的。好吧，我很遗憾地告诉你，有时它是。你必须使用你的直觉和直觉，但是，根据我的经验，与功能相比，我们倾向于低估性能。

<!-- 
If you're a manager, your team will thank you for instilling a culture like this, and you'll be rewarded by shipping
better performing software on schedule.  If you're an engineer, I guarantee you'll spend far less time scrambling, more
time being proactive, and more time having fun, in a team obsessed over customer performance.  I'd love to hear what you
think and your own experiences establishing a performance culture. 
-->
如果您是经理，您的团队会感谢您灌输这样的文化，并且您将获得按计划运送性能更佳的软件的奖励。如果你是一名工程师，我保证你会花费更少的时间来争抢，更多的时间积极主动，更多的时间玩得开心，在一个痴迷于客户绩效的团队中。我很想知道你的想法以及你自己建立表演文化的经历。
