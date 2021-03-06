---
title: Midori博客系列翻译（6）——性能文化
date: 2019-3-13 12:30:00
tags: [操作系统, Midori, 翻译, 性能, 软件工程]
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
在本文中，我将聊聊“性能文化”。
性能是软件工程的关键支柱之一，并且很难做正确，有时甚至都难以识别，
正如一位著名的法官曾经说过的那样，“我一眼就能看出来（I know it when I see it）”。
我之前已详细地聊到了[性能]((http://joeduffyblog.com/2010/09/06/the-premature-optimization-is-evil-myth/)
和[文化](http://joeduffyblog.com/2013/02/17/software-leadership-series/)，
但两者之间的交互才会变得更有趣。
能够做好这一点的团队几乎从一开始就将性能贯穿于团队运作的各个方面，
并且能够主动提供可以摧毁竞争对手的良好客户体验。 
虽然没有千篇一律的简单方法来获得良好的性能文化，
但是你依然可以遵循一些最佳实践来将必要条件的种子植入到团队中。
因此，让我们一起出发吧！

<!-- 
# Introduction 
-->
# 介绍

<!-- 
Why the big focus on performance, anyway? 
-->
那么到底为什么要关注于性能呢？

<!-- 
Partly it's my background.  I've worked on systems, runtimes, compilers, ... things that customers expect to be fast.
It's always much easier to incorporate goals, metrics, and team processes at the outset of such a project, compared to
attempting to recover it later on.  I've also worked on many teams, some that have done amazing at this, some that have
done terribly, and many in between.  The one universal truth is that the differentiating factor is always culture. 
-->
其中部分原因是由于我的背景。
我曾经研究过系统、运行时、编译器……
这些都是客户期望运行得足够快的软件。 
在这样的项目中，与在事后进行弥补相比，
从一开始时就整合目标、指标和团队的流程总是容易得多。
我也曾参与过很多团队的工作，有些团队在这方面做得很棒，
有些团队却做得非常糟糕，而很多团队则介于两者之间。
一个普遍的事实是，造成这之间差异的因素始终是文化。

<!-- 
Partly it's because, no matter the kind of software, performance is almost always worse than our customers would like
it to be.  This is a simple matter of physics: it's impossible to speed up all aspects of a program, given finite time,
and the tradeoffs involved between size, speed, and functionality.  But I firmly believe that on the average teams spend
way less attention to developing a rigorous performance culture.  I've heard the "performance isn't a top priority for
us" statement many times only to later be canceled out by a painful realization that without it the product won't
succeed. 
-->
另外部分原因是，无论哪种软件，
性能几乎总是比我们的客户所期望的要更差。
这是一个简单的物理问题：
在给定有限时间以及在大小速度和功能之间进行折衷的情况下，
不可能在所有方面提高程序的运行速度。
但我坚信，就平均而言，团队普遍花费较少的注意力来发展严谨的性能文化。
我已经很多次听说过这样的言论——“性能不是我们的首要任务”，
而后来却又痛苦地意识到它的重要性。
没有性能，产品就不会成功。

<!-- 
And partly it's just been top of mind for all of us in DevDiv, as we focus on .NET core performance, ASP.NET
scalability, integrating performance-motivated features into C# and the libraries, making Visual Studio faster, and
more.  It's particularly top of mind for me, as I've been comparing our experiences to my own in [Midori](
/2015/11/03/blogging-about-midori/) (which heavily inspired this blog post). 
-->
另外部分原因是，对于DevDiv团队而言，
由于我们专注于.Net的核心性能、ASP.Net的扩展性以及将性能驱动的功能集成到C#和库中，
使Visual Studio变得更快等等，使得性能是我们所有人摆在首位的指标。
这对我来说尤其重要，所以我一直在将我们的经历与我在[Midori](/2018/10/20/midori/0-blogging-about-midori/)
中的经历进行比较（本文也受此很大的启发）。

<!-- 
# Diagnosis and The Cure 
-->
# 诊断和治疗

<!-- 
How can you tell whether your performance culture is on track?  Well, here are some signs that it's not: 
-->
如何判断团队性能文化是否走上正轨？ 
好吧，如果存在下面一些迹象，则表明尚未达到目标：

<!-- 
* Answering the question, "how is the product doing on my key performance metrics," is difficult.
* Performance often regresses and team members either don't know, don't care, or find out too late to act.
* Blame is one of the most common responses to performance problems (either people, infrastructure, or both).
* Performance tests swing wildly, cannot be trusted, and are generally ignored by most of the team.
* Performance is something one, or a few, individuals are meant to keep an eye on, instead of the whole team.
* Performance issues in production are common, and require ugly scrambles to address (and/or cannot be reproduced). 
-->
* 回答“产品在我的关键性能指标上发挥如何”这个问题很难；
* 性能经常退化，团队成员要么不知道，要么不关心，要么发现太晚而难以行动；
* 责备是对性能问题（人员，基础设施或两者皆有）的最常见反应之一；
* 性能测试大幅度摇摆不定，无法得到确认，并且通常被大多数团队成员所忽略；
* 性能是一个或几个人应该关注的事情，而不是整个团队；
* 产品常常出现性能问题，并且需要仓促行动才能解决（和/或不能重现问题）。

<!-- 
These may sound like technical problems.  It may come as a surprise, however, that they are primarily human problems. 
-->
这些听起来像是技术问题，但出乎意料之外的是，它们主要是人的问题。

<!-- 
The solution isn't easy, especially once your culture is in the weeds.  It's always easier to not dig a hole in the
first place than it is to climb out of one later on.  But the first rule when you're in a hole is to stop digging!  The
cultural transformation must start from the top -- management taking an active role in performance, asking questions,
seeking insights, demanding rigor -- while it simultaneously comes from the bottom -- engineers actively seeking to
understand performance of the code they are writing, ruthlessly taking a zero-tolerance stance on regressions, and
being ever-self-critical and on the lookout for proactive improvements. 
-->
对此的解决方法并不容易，特别是一旦当性能文化处于十分糟糕的时候，
从一开始就不挖坑始终比之后从坑里爬出来要更容易。
可是当你陷入困境时，首要的原则就是停止挖坑！
文化的转型必须同时从最高层开始——管理层需要在性能的提升中发挥积极的作用，提出问题，寻找方法，严谨要求
——以及从下层开始，工程师积极寻求对他们正在编写的代码的性能理解，
对性能退化采取无情的零容忍态度，并进行自我批判和寻求主动改进的积极性。

<!-- 
This essay will describe some ways to ensure this sort of a culture, in addition to some best practices I've found help
to increase its effectiveness once you have one in place.  A lot of it may seem obvious, but believe me, it's pretty
rare to see everything in here working in harmony in practice.  But when it is, wow, what a difference it can make. 
-->
本文将介绍一些建立这种文化的方法，
此外还有一些我发现的有助于提高效率的最佳实践。
很多事情看起来很明显，但请相信我，在实践中很难看到所有一切都协调一致的进行。
但是，如果一旦这样的文化得以建立，那它将产生巨大的影响力。

<!-- 
*A quick note on OSS software.  I wrote this essay from the perspective of commercial software development.  As
such, you'll see the word "management" a lot.  Many of the same principles work in OSS too.  So, if you like, anytime
you see "management," mentally transform it into "management or the project's committers."* 
-->
* 对于OSS（开源）软件的简单说明——我是从商业软件开发的角度写下这篇文章，
  因此你会看到很多“管理层”这个词，不过这里的许多相同原则也同样适用于OSS。
  因此，如果你愿意，只要你看到“管理层”这个词，请在脑海中转化为“管理层或项目的提交者”。*

<!-- 
# It Starts, and Ends, with Culture 
-->
# 以文化开始，以文化结束

<!-- 
The key components of a healthy performance culture are: 
-->
健康的性能文化的关键组成部分包括：

<!-- 
1. Performance is part of the daily dialogue and "buzz" within the team.  Everybody plays a role.
2. Management must care -- truly, not superficially -- about good performance, and understand what it takes.
3. Engineers take a scientific, data-driven, inquisitive approach to performance.  (Measure, measure, measure!)
4. Robust engineering systems are in place to track goals, past and present performance, and to block regressions. 
-->
1. 性能是团队日常交流和闲聊的一部分，每个人都扮演着其中一个角色；
2. 管理层必须（真正地，而不是表面上地）关心于优良的性能，并了解获得它们需要什么；
3. 工程师采用科学的，数据驱动的，刨根问底的方法来提升性能（测量，测量，还是测量！）；
4. 有健壮的工程系统来追踪目标以及它过去和现在的性能，并阻止其发生退化。

<!-- 
I'll spend a bit of time talking about each of these roughly in turn. 
-->
我会花一点时间大体地谈论它们中的每一个。

<!-- 
## Dialogue, Buzz, and Communication 
-->
## 对话，闲聊和沟通

<!-- 
The entire team needs to be on the hook for performance. 
-->
整个团队需要保持对性能的追求。

<!-- 
In many teams where I've seen this go wrong, a single person is anointed the go-to performance guy or gal.  Now, that's
fine and can help the team scale, can be useful when someone needs to spearhead an investigation, and having a vocal
advocate of performance is great, but it *must not* come at the expense of the rest of the team's involvement. 
-->
在我见过的把事情搞砸的团队中，一个成员被指定为性能小伙/姑娘。
这是不错的方法，它可以帮助团队进行扩展，
当有人带头进行调查时也很有用，并也在团队中存在主张性能的声音。
但这种方式*不能*以团队的其他成员为代价。

<!-- 
This can lead to problems similar to those Microsoft use to have with the "test" discipline; engineers learned bad
habits by outsourcing the basic quality of their code, assuming that someone else would catch any problems that arise.
The same risks are present when there's a central performance czar: engineers on the team won't write performance tests,
won't proactively benchmark, won't profile, won't ask questions about the competitive standing of the product, and
generally won't do all the things you need all of the engineers doing to build a healthy performance culture. 
-->
这可能会导致类似于过去Microsoft对待“测试”的原则问题：
工程师通过外包代码的基本质量，并假设其他人（外包人员）会发现任何可能的问题，并养成了坏习惯。
当存在中心化的性能独裁者时，也会有同样的风险存在：
团队中的工程师不会编写性能测试，不会主动进行基准测试，不会对性能进行profile，
不会询问有关产品当前所处竞争地位的问题，
并且通常不会做任何一个工程师为建立健康性能文化而做的所有事情。

<!-- 
Magical things happen when the whole team is obsessed about performance.  The hallways are abuzz with excitement, as
news of challenges and improvements spreads organically.  "Did you see Martin's hashtable rebalancing change that
reduced process footprint by 30%?"  "Jared just checked in a feature that lets you stack allocate arrays.  I was
thinking of hacking the networking stack this weekend to use it -- care to join in?"  Impromptu whiteboard sessions,
off-the-cuff ideation, group learning.  It's really awesome to see.  The excitement and desire to win propels the
team forward, naturally, and without relying on some heavyweight management "stick." 
-->
当整个团队都对性能着迷时，会发生神奇的事情。
随着挑战和改进的消息在团队里有机地传播，走廊充满了兴奋的气氛。 
“你有没有看到Martin的哈希表重平衡改变后，将进程占用空间减少了30%？”
“Jared刚刚加入了一个允许堆叠分配数组功能，
我本周末想在网络堆栈上使用它，你有兴趣加入吗？”
即兴的白板交流，袖手旁观的想法，以小组的方式学习等等，看到这些真的是太棒了，
对胜利的兴奋和渴望推动着团队前进，便自然且不依赖于一些重量级管理层的“驱使”。

<!-- 
I hate blame and I hate defensiveness.  My number one rule is "no jerks," so naturally all critiques must be delivered
in the most constructive and respectful way.  I've found a high occurrence of blame, defensiveness, and [intellectual
dishonesty](/2015/11/02/software-leadership-9-on-the-importance-of-intellectual-honesty/) in teams that do poorly on
performance, however.  Like jerks, these are toxic to team culture and must be weeded out aggressively.  It can easily
make or break your ability to develop the right performance culture.  There's nothing wrong with saying we need to do
better on some key metric, especially if you have some good ideas on how to do so! 
-->
我讨厌抱怨和防备，因此我的首要原则就是：“（团队里）没有笨蛋”，
所以所有的批评都必须以最具建设性和尊重的方式提出来。
然而，我发现在性能状况不佳的团队中往往出现过多的抱怨、防备和[机智的不诚实](http://joeduffyblog.com/2015/11/02/software-leadership-9-on-the-importance-of-intellectual-honesty/)，
这就像笨蛋一样，对团队文化是有害的，因此必须积极将这种方式淘汰掉，
它可以轻松地决定你发展正确性能文化的能力。
我们需要在一些关键指标上做得更好这一点上是没有错的，
特别是如果你对如何做到这一点上有一些好的想法！

<!-- 
In addition to the ad-hoc communication, there of course needs to be structured communication also.  I'll describe some
techniques later on.  But getting a core group of people in a room regularly to discuss the past, present, and future of
performance for a particular area of the product is essential.  Although the organic conversations are powerful,
everyone gets busy, and it's important to schedule time as a reminder to keep pushing ahead. 
-->
除了ad-hoc的交谈方式之外，当然还需要结构化的交流，我稍后会介绍一些技巧。
但是，定期在一个房间内安排一组核心人群来讨论产品特定区域的过去，现在和未来的性能至关重要。
尽管有机的讨论非常有用，但每个人都很忙碌，所以重要的是要安排时间作为继续推进的提醒。

<!-- 
## Management: More Carrots, Fewer Sticks 
-->
## 管理层：更多的胡萝卜和更少的大棒
<!-- 
In every team with a poor performance culture, it's management's fault.  Period.  End of conversation. 
-->
在每个具有糟糕性能文化的团队中，都是因为管理层出了问题。

<!-- 
Engineers can and must make a difference, of course, but if the person at the top and everybody in between aren't deeply
involved, budgeting for the necessary time, and rewarding the work and the superstars, the right culture won't take
hold.  A single engineer alone can't possibly infuse enough of this culture into an entire team, and most certainly not
if the entire effort is running upstream against the management team. 
-->
当然，工程师可以而且必须有所作为，
但如果顶层的管理者和中间层都没有深入参与，对必要时间进行预算，
对工作和超级明星进行奖励，那么正确的文化就不会成功建立起来。
单凭一名工程师不可能将这种文化注入到整个团队，
而且如果整个工作都是与管理团队相抵触的话，那么肯定更不会成功。

<!-- 
It's painful to watch managers who don't appreciate performance culture.  They'll often get caught by surprise and won't
realize why -- or worse, think that this is just how engineering works.  ("We can't predict where performance will
matter in advance!")  Customers will complain that the product doesn't perform as expected in key areas and, realizing
it's too late for preventative measures, a manager whose team has a poor performance culture will start blaming things.
Guess what?  The blame game culture spreads like wildfire, the engineers start doing it too, and accountability goes out
the window.  Blame doesn't solve anything.  Blaming is what jerks do. 
-->
看到一个不会欣赏性能文化的管理者是痛苦的。
他们经常会对此感到意外，并且不会意识到为什么会是这样，
或者更糟糕地认为这就是工程的工作原理（“我们无法提前料到性能的重要性！”）。
客户会抱怨产品在关键领域没有达到预期的效果，并且在意识到预防措施为时已晚时，
具有糟糕性能文化的团队管理者便开始抱怨。
你猜怎么着？这种抱怨的文化就像野火一样传播，
使得工程师们也开始这样做，而真正的责任追究制却被抛之脑后。
抱怨不能解决任何问题，是笨蛋才所做的事情。

<!-- 
Notice I said management must be "*deeply* involved": this isn't some superficial level of involvement.  Sure, charts
with greens, reds, and trendlines probably need to be floating around, and regular reviews are important.  I suppose you
could say that these are pointy-haired manager things.  (Believe me, however they do help.)  A manager must go deeper
than this, however, proactively and regularly reviewing the state of performance across the product, alongside the other
basic quality metrics and progress on features.  It's a core tenet of the way the team does its work.  It must be
treated as such.  A manager must wonder about the competitive landscape and ask the team good, insightful questions
that get them thinking. 
-->
注意我说管理层必须“*深入*参与”的意思时，不能只是表面的参与而已。
当然，绿色、红色加上趋势线的图表可能需要出现在周围，并且定期的评审也很重要，
我想你可以说这些都是尖头发的经理所从事的事情（相信我，这确实是有帮助的）。
然而，经理必须比这些更深入，主动并定期审查整个产品的性能状况，
以及其他基本质量指标和功能的进展。这是团队工作方式的核心原则，并且必须这样对待。
经理必须思考竞争的格局，并向团队询问引发他们思考，且优秀而富有洞察力的问题。

<!-- 
Performance doesn't come for free.  It costs the team by forcing them to slow down at times, to spend energy on things
other than cranking out features, and hence requires some amount of intelligent tradeoff.  How much really depends on
the scenario.  Managers need to coach the team to spend the right ratio of time.  Those who assume it will come for free
usually end up spending 2-5x the amount it would have taken, just at an inopportune time later on (e.g., during the
endgame of shipping the product, in production when trying to scale up from 1,000 customers to 100,000, etc). 
-->
性能不是免费的，它有时通过迫使团队放慢速度为代价，将精力花在除了完成功能之外的事情上，因此需要进行一些合理的权衡。
在性能上花费多少精力真的取决于场景，而经理需要指导团队花费适当比例的时间。
那些认为会免费获得性能的人通常在稍后不凑巧的时间点上
（例如，在发布产品的最后阶段，以及在生产中尝试从1000扩大到100000个客户规模时等），最终会花费2-5倍的精力。

<!-- 
A mentor of mine used to say "You get from your teams what you reward."  It's especially true with performance and the
engineering systems surrounding them.  Consider two managers: 
-->
我的一位导师曾经说过，“你从你的团队中获得你的奖励”，
对于性能和围绕它们的工程系统来说尤其如此。考虑以下两种经理：

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
* 经理A为性能文化提出口头上的鼓励。然而，她为每个迭代周期提出了固定数量的功能计划，
  比如，“我们必须粉碎竞争对手Z并且*必须*实现相同的功能！”，却没有中途休息的时间。
  她花了很多时间参加团队会议，赞扬新功能和演示大量的demo，甚至为工程师颁发“最具突破性功能”的奖励。
  结果，她的团队以一个令人印象深刻的剪辑实现出功能，并每次都向董事会演示新出炉的demo，
  为销售团队提供大量资源以获取新的潜在客户。但没有为性能提供门路，工程师们通常也懒得去思考性能。

* 经理B采取更加平衡的方法。她认为，鉴于竞争的格局和通过杰出的演示给客户和董事会成员留下深刻印象的必要性，
  新功能是需要不断涌现的。但她也担心在性能，可靠性和质量等方面在她所希望坚持的领域积累太多债务。
  因此，她会故意放慢追求新功能的步伐，并尽可能地推动团队在这些方面上努力。
  例如，她需要良好的工程系统以及内置性能遥测的新功能，
  这会让董事会成员和产品经理处于困境，而这绝对是不受欢迎和有难度的。
  除了颁发“最具突破性功能”奖项之外，她还要展示了性能的进展图表，
  并为提供最具影响力性能改进的工程师颁发了“性能忍者”奖。
  请注意，工程系统的改进也是符合“性能忍者”奖颁发条件的！

<!-- 
Which manager do you think is going to ship a quality product, on time, that customers are in love with?  My money is
on Manager B.  Sometimes you've got to [slow down to speed up](
/2013/04/12/software-leadership-4-slow-down-to-speed-up/). 
-->
你认为哪位经理会按时交付给客户所喜爱的优质产品呢？我会投给经理B。
有时候[为了加快速度而必须放慢脚步](http://joeduffyblog.com/2013/04/12/software-leadership-4-slow-down-to-speed-up/)。

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
微软最近正在经历两个与这一点相关的有趣转变：
一方面，消除了以“测试”作为前面提到的原则；另一方面，又重新关注于工程系统。
这是一个坎坷的旅程，但令人惊讶的是，
克服困难的最大障碍之一不是工程师个人，而是管理者！
旧模型中的“开发经理”习惯专注于功能，功能，还是功能，
并将大部分工程系统任务交给承包商，以及将大多数软件质量工作都交给了测试人员。
结果，他们没有准备好承认和奖励那些对建立卓越性能文化至关重要的工作。
结果如何？你猜对了：完全缺乏性能文化，但更微妙的是，最终还是出现了“领导漏洞”。
直到最近，几乎没有高级工程师致力于任务关键的工程系统，并使整个团队更高效和有能力。
谁想通过分配给承包商的那些繁琐的工作来创造自己的事业，而不能得到管理层的重视？
再一次地，你获得了应得的回报。

<!-- 
There's a catch-22 with early prototyping where you don't know if the code is going to survive at all, and so the
temptation is to spend zero time on performance.  If you're hacking away towards a minimum viable product (MVP), and
you're a startup burning cash, it's understandable.  But I strongly advise against this.  Architecture matters, and some
poorly made architectural decisions made at the outset can lay the foundation for an entire skyscraper of
ill-performing code atop.  It's better to make performance part of the feasibility study and early exploration. 
-->
有一个关于项目早期原型的“22条军规”，它是这么说的：
如果你不知道项目是否能够存活下来，那么不要在性能上花费任何时间。
如果你正在实现最低可行产品（MVP），并且处于一个非常烧钱的创业公司中，那这种做法是可以理解的。
但我强烈反对这一点，因为架构真的很重要，
一开始就做出的一些不当的架构决策可能为整个低质量代码的摩天大楼奠定了基础。
因此，最好将性能作为可行性研究和早期探索的一部分。

<!-- 
Finally, to tie up everything above, as a manager of large teams, I think it's important to get together regularly
-- every other sprint or two -- to review performance progress with the management team.  This is in addition to the
more fine-grained engineer, lead, and architect level pow-wows that happen continuously.  There's a bit of a "stick"
aspect of such a review, but it's more about celebrating the team's self-driven accomplishments, and keeping it on
management's radar.  Such reviews should be driven from the lab and manually generated numbers should be outlawed. 
-->
最后，作为大型团队的经理，为了统一上述所有内容，
我认为定期，比如每隔一两个迭代周期，召开会议来审核管理团队的性能进展是非常重要的。
除此之外，还有更细粒度的工程师，leader和架构师级别的碰头要不断的进行。
这种审查有一点“坚持”的味道，但它更多的是关于庆祝团队的自我驱动成就，并将其保持在管理层的雷达上。
这些审查应该由实验室所驱动，而手动生成的数字是不合规的。

<!-- 
Which brings me to ... 
-->
这为我带来了……

<!-- 
# Process and Infrastructure 
-->
# 流程和基础设施

<!-- 
"Process and infrastructure" -- how boring! 
-->
“流程和基础设施”——多么无聊的事情！

<!-- 
Good infrastructure is a must.  A team lacking the above cultural traits won't even stop to invest in infrastructure;
they will simply live with what should be an infuriating lack of rigor.  And good process must ensure effective use
of this infrastructure.  Here is the bare minimum in my book: 
-->
良好的基础设施是必须的。 
缺乏上述文化特征的团队甚至不会停下来对基础设施进行投资，所以他们只会处在一个令人恼怒并缺乏严谨的环境中，
而良好的流程能确保有效使用此基础设施。
下面是我在本书中的最低要求：

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
* 所有的代码提交必须先通过一组门控性能测试；
* 任何惊险通过测试并且性能退化的提交都将毫无疑问地被撤回，我称其为零容忍规则；
* 连续性能遥测通过实验室，飞行和在线环境中上报；
* 这意味着性能测试和基础架构具有一些重要特征：
  - 它们不会让人觉得吵闹；
  - 能测量“正确”的东西；
  - 可以在“合理”的时间内运行完成。

<!-- 
I have this saying: "If it's not automated, it's dead to me." 
-->
有这么一句话：“如果它不是自动化的，那么对我来说它已经死了”。

<!-- 
This highlights the importance of good infrastructure and avoids the dreaded "it worked fine on my computer" that
everybody, I'm sure, has encountered: a test run on some random machine -- under who knows what circumstances -- is
quoted to declare success on a benchmark... only to find out some time later that the results didn't hold.  Why is this? 
-->
这凸显了良好基础设施的重要性，并避免了可怕的“它明明在我的机器上运行的很好”的情况出现，
我可以保证每个人都遇到过：在某个谁也不知道环境配置的随机机器上运行通过测试，
然后便宣称在某个基准测试上获得了成功，而在一段时间后发现并没有这种预期的结果。
那么为什么会遇到这种情况？

<!-- 
There are countless possibilities.  Perhaps a noisy process interfered, like AntiVirus, search indexing, or the
application of operating system updates.  Maybe the developer accidentally left music playing in the background on
their multimedia player.  Maybe the BIOS wasn't properly adjusted to disable dynamic clock scaling.  Perhaps it was
due to an innocent data-entry error when copy-and-pasting the numbers into a spreadhseet.  Or maybe the numbers for two
comparison benchmarks came from two, incomparable machine configurations.  I've seen all of these happen in practice. 
-->
这里存在无数种可能性：也许是一个嘈杂进程的干扰，
比如AntiVirus或搜索索引进程，或操作系统的更新程序，
或者是开发者不小心把音乐留在他们的多媒体播放器的后台进行播放，
也可能是BIOS没有正确调整到禁用动态时钟缩放，
或由于在将数字复制并粘贴到电子表格时出现了数据输入错误，
更或者是两个进行比较的基准数字来自两个根本无法比较的机器配置上。
我见证过以上所有这些都在现实世界中发生过。

<!-- 
In any manual human activity, mistakes happen.  These days, I literally refuse to look at or trust any number that
didn't come from the lab.  The solution is to automate everything and focus your energy on making the automation
infrastructure as damned good as possible.  Pay some of your best people to make this rock solid for the rest of the
team.  Encourage everybody on the team to fix broken windows, and take a proactive approach to improving the
infrastructure.  And reward it heartily.  You might have to go a little slower, but it'll be worth it, trust me. 
-->
在任何有人参与的活动中，都会发生错误。
现在这些天，我真的拒绝查看或信任任何不是来自实验室的数字，
而解决方案是实现所有一切的自动化，
并将精力集中在尽可能使自动化基础设施变得更好上面，
通过部署最优秀的人才在这上面，为团队的其他成员打造坚实的基础。
鼓励团队中的每个人修复“破损的窗户”，
并采取积极主动的方法来改善基础设施，以及衷心地奖励这些做法。
这样做可能会稍微前进的慢一点，但请相信我，这样做是值得的。

<!-- 
## Test Rings 
-->
## 测试环

<!-- 
I glossed over a fair bit above when I said "all commits must pass a set of performance tests," and then went on to
talk about how a checkin might "slip past" said tests.  How is this possible? 
-->
当我说“所有代码提交都必须通过一系列性能测试”时，我是很认真的，
然后我又继续说到代码提交可能会“侥幸通过”所述测试。那么为什么会这样？

<!-- 
The reality is that it's usually not possible to run all tests and find all problems before a commit goes in, at least
not within a reasonable amount of time.  A good performance engineering system should balance the productivity of speedy
codeflow with the productivity and assurance of regression prevention. 
-->
实际的情况是，通常不可能在提交之前运行所有测试并找到所有问题，至少在合理的时间内不会达到。
一个好的性能工程系统应该要在快速代码流的生产力与回归预防的保证之间做出平衡。

<!-- 
A decent approach for this is to organize tests into so-called "rings": 
-->
一种合适的方法是将测试组织成所谓的“环（ring）”：

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
* 一个内环，包含团队中所有开发者在每次提交之前要测量的测试；
* 一个内环，包含特定子团队的开发者在每次提交之前要测量的测试；
* 一个内环，包含开发者在每次提交之前自行决定要运行的测试；
* 除此之外的任意数量的连续环，这包括：
  - 分支之间的每个代码流门控测试；
  - 每晚（nightly）每周（weekly）或基于时间/资源的限制等条件的，提交后（post-commit）的测试；
  - 发布前（pre-release）的验证；
  - 发布后（post-release）的遥测和监测。

<!-- 
As you can see, there's a bit of flexibility in how this gets structured in practice.  I wish I could lie and say that
it's a science, however it's an art that requires intelligently trading off many factors.  This is a constant source of
debate and one that the management team should be actively involved in. 
-->
正如你所看到的，在实践中如何构建它有一定的灵活性。
我多么希望我能谎称这是一门科学，但这确实是一门需要智慧地折衷许多因素的艺术。
这是一个不断争论的焦点，也是管理团队应该积极参与辩论的话题。

<!-- 
A small team might settle on one standard set of benchmarks across the whole team.  A larger team might need to split
inner ring tests along branching lines.  And no matter the size, we would expect the master/main branch to enforce the
most important performance metrics for the whole team, ensuring no code ever flows in that damages a core scenario. 
-->
一个小型团队可以在整个团队中建立一套标准的基准，
而一个更大的团队可能需要沿着分支分割内环测试。
无论规模大小，我们都希望主干/主分支能够为整个团队强制执行最重要的性能指标，
确保没有任何损害核心场景的代码进入。

<!-- 
In some cases, we might leave running certain pre-commit tests to the developer's discretion.  (Note, this does not mean
running pre-commit tests altogether is optional -- only a particular set of them!)  This might be the case if, for
example, the test covered a lesser-used component and we know the nightly tests would catch any post-commit regression.
In general, when you have a strong performance culture, it's okay to trust judgement calls sometimes.  Trust but verify. 
-->
在某些情况下，我们可能会将某些提交前（pre-commit）测试留给开发者来自行决定
（注意，这并不意味着运行提交前测试完全是可选的，而只是有一组是特定于此的），
例如，如果测试覆盖了较少使用的组件并且我们知道每夜测试会捕获任何提交后回归。
一般来说，当你拥有强大的性能文化时，有时候可以信任这种判断。信任但要验证。

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
让我们来看几个具体的例子。
性能测试的范围通常从微观遍布宏观，因此从精确定位回归来源的角度看，他们通常也从更容易遍布到更难上 
（微测试只测量一种，所以波动往往更容易理解，而宏测试则需测量整个系统，通过波动找到目标往往要费点力气）。
网络服务器团队可能在最里面的预提交测试套件中，包括一系列的微测试和宏测试，比如说：
每个请求分配的字节数（微），请求响应时间（微）……
可能还有其他六个微到中间规模的基准测试……以及[TechEmpower](https://www.techempower.com/benchmarks/)（宏）。
感谢实验室的资源，测试并行以及强大的GitHub webhooks，使得这些测试都在15分钟内完成，
并能很好地集成到你的pull request和代码审查流程中，这一点是相当不错的。
但这显然不能完美的覆盖，也许每天晚上，TechEmpower需要运行4个小时，以便能在更长的时间内测量性能并识别泄漏，
开发者可能通过提交前测试，但却在这个更长的测试中失败。
因此，团队允许按需运行此测试，所以好的开发者可以避免丢脸。
但即使错误发生了，却再也不是一种抱怨或迫害的文化，是怎样那就是怎样。

<!-- 
This leads me to back to the zero tolerance rule. 
-->
这导致我回退到了零容忍规则上。

<!-- 
Barring exceptional circumstances, regressions should be backed out immediately.  In teams where I've seen this succeed,
there were no questions asked, and no IOUs.  As soon as you relax this stance, the culture begins to break down.  Layers
of regressions pile on top of one another and you end up ceding ground permanently, no matter the best intentions of the
team.  The commit should be undone, the developer should identify the root cause, remedy it, ideally write a new test if
appropriate, and then go through all the hoops again to submit the checkin, this time ensuring good performance. 
-->
除非出现特殊情况，否则出现回归则应立即撤销。
在我看到这个成功的团队中，这一点是毫无疑问的，也没有任何借口，
一旦你放松这种立场，文化就会开始崩溃。
无论团队的最佳意图是什么，如果性能回归都堆积在一起，那么你最终再也回不到最初的状态。
对于回归的提交应该撤消，开发者需确定根本原因，对其进行补救。
理想情况下，如果合适的话，编写一个新的测试，然后再次通过所有测试环以提交签入代码，并在这次确保良好的性能。

<!-- 
## Measurement, Metrics, and Statistics 
-->
## 测量，指标和统计

<!-- 
Decent engineers intuit.  Good engineers measure.  Great engineers do both. 
-->
体面的工程师靠直觉，优秀的工程师靠测量，伟大的工程师两种都做到。

<!-- 
Measure what, though? 
-->
那到底测量什么？

<!-- 
I put metrics into two distinct categories: 
-->
我将度量分为两个不同的类别：

<!-- 
* *Consumption metrics.*  These directly measure the resources consumed by running a test.
* *Observational metrics.*  These measure the outcome of running a test, *observationally*, using metrics "outside" of
  the system. 
-->
* *消耗指标*：它们直接测量运行测试所消耗的资源；
* *观察指标*：它们使用系统“外部”的度量标准来衡量运行测试的结果。

<!-- 
Examples of consumption metrics are hardware performance counters, such as instructions retired, data cache misses,
instruction cache misses, TLB misses, and/or context switches.  Software performance counters are also good candidates,
like number of I/Os, memory allocated (and collected), interrupts, and/or number of syscalls.  Examples of observational
metrics include elapsed time and cost of running the test as billed by your cloud provider.  Both are clearly important
for different reasons. 
-->
一个消耗指标的例子是硬件性能计数器，比如说完成指令（instruction retired），
未命中的数据缓存，未命中的指令缓存，未命中的TLB和/或上下文切换等。
软件性能计数器也是很好的例子，例如I/O数量，分配（和收集）的内存，中断和/或系统调用的数量等。
观察度量的示例包括运行时间和来自云提供商的运行测试成本开销。
处于多种原因，两者显然都很重要。

<!-- 
Seeing a team measure time and time alone literally brings me to tears.  It's a good measure of what an end-user will
see -- and therefore makes a good high-level test -- however it is seriously lacking in the insights it can give you.
And if there's no visibility into variance, it can be borderline useless. 
-->
逐字观察一整个团队的测量时间本身就非常耗费精力，
虽然它可以很好地衡量最终用户将会看到什么，并因此进行良好的高级测试， 
但它可以为你提供的洞察力严重缺乏，
如果没有对变化的可见性，那么可能是无用的。

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
对于试图理解为什么会发生变化的工程师而言，消耗指标显然更有帮助。
在我们上面的Web服务器团队的例子中，假设请求响应时间增加了30%，
所有测试报告告诉我们的都是时间。
开发者确实可以尝试在本地重现场景，并手动缩小原因范围，
但是由于实验室与本地硬件的差异，这样做可能会很繁琐并耗费时间，而且可能并不完美。
相反，如果是完成指令和分配内存的报告与时间回归报告一起呈现呢？
从它们那里，可以很容易地看出，每个请求突然分配了256KB的内存，这在之前是没有出现的。
结合考虑最近的提交，这可以使工程师在增加提交堆积在最顶层并进一步使问题难以理解之前，
很容易及时地快速查明并找到罪魁祸首。而这就好像printf调试所提供的一样。

<!-- 
Speaking of printf debugging, telemetry is essential for long-running tests.  Even low-tech approaches like printfing
the current set of metrics every so often (e.g., every 15 seconds), can help track down where something went into the
weeds simply by inspecting a database or logfile.  Imagine trying to figure out where the 4-hour web server test went
off the rails at around the 3 1/2 hour mark.  This can can be utterly maddening without continuous telemetry!  Of
course, it's also a good idea to go beyond this.  The product should have a built-in way to collect this telemtry out
in the wild, and correlate it back to key metrics.  [StatsD](https://github.com/etsy/statsd) is a fantastic option. 
-->
说到printf调试，遥测对于长时间运行的测试至关重要。
即使是低技术含量的方法，例如每隔一段时间（例如每15秒）打印一组当前的度量值，
也可以简单地通过检查数据库或日志文件来追踪到陷入困境的位置。
想象一下，试图找出4小时网络服务器测试在3个半小时左右的轨道上脱轨的地方，
如果没有连续的遥测，这可能真是令人头疼的一件事！
当然，在这上面走的更远也是一个好主意，产品应具有内置的方式在收集此遥测，并将其与关键指标相关联。 
这方面，[StatsD](https://github.com/etsy/statsd)是一个很棒的选择。

<!-- 
Finally, it's important to measure these metrics as scientifically as possible.  That includes tracking [standard
deviation](https://en.wikipedia.org/wiki/Standard_deviation), [coefficient of variation (CV)](
https://en.wikipedia.org/wiki/Coefficient_of_variation), and [geomean](https://en.wikipedia.org/wiki/Geometric_mean),
and using these to ensure tests don't vary wildly from one run to the next.  (Hint: commits that tank CV should be
blocked, just as those that tank the core metric itself.)  Having a statistics wonk on your team is also a good idea! 
-->
最后，尽可能科学地衡量这些指标非常重要。
这包括跟踪[标准差](https://en.wikipedia.org/wiki/Standard_deviation)，
[变异系数（CV）](https://en.wikipedia.org/wiki/Coefficient_of_variation)
和[几何平均值](https://en.wikipedia.org/wiki/Geometric_mean)，
并使用这些参数来确保两次测试结果不会相差太远。
（提示：大幅度改变CV的提交应该被阻止，就像那些大幅度改变核心指标本身一样）。
对你的团队进行统计数据研究也是一个不错主意！

<!-- 
## Goals and Baselines 
-->
## 目标和基线
<!-- 
Little of the above matters if you lack goals and baselines.  For each benchmark/metric pair, I recommend recognizing
four distinct concepts in your infrastructure and processes: 
-->
如果你缺乏目标和基线，那么上述的方法所产生的作用很小。
对于每个基准/指标对，我建议你在基础架构和流程中识别四个不同的概念：

<!-- 
* *Current*: the current performance (which can span multiple metrics).
* *Baseline*: the threshold the product must stay above/below, otherwise tests fail.
* *Sprint Goal*: where the team must get to before the current sprint ends.
* *Ship Goal*: where the team must get to in order to ship a competitive feature/scenario. 
-->
* *当前*：当前性能（可以跨越多个指标）；
* *基线*：产品必须保持高于/低于的阈值，否则测试失败；
* *迭代目标*：在当前迭代周期结束之前团队必须到达的位置；
* *交付目标*：团队必须到达此位置，才能发布有竞争力的功能/方案。

<!-- 
Assume a metric where higher is better (like throughput); then it's usually the case that Ship Goal &gt;= Sprint Goal
&gt;= Current &gt;= Baseline.  As wins and losses happen, continual adjustments should be made. -->
对于一个越高越好的指标（如吞吐量）来讲，那么通常情况下的情况是，交付目标 &gt;= 迭代目标 &gt;= 当前 &gt;= 基线。
随着输赢比较的不断发生，应该不断进行调整。

<!-- 
For example, a "baseline ratcheting" process is necessary to lock in improvements.  A reasonable approach is to ratchet
the baseline automatically to within some percentage of the current performance, ideally based on standard deviation
and/or CV.  Another approach is to require that developers do it manually, so that all ratcheting is intentional and
accounted for.  And interestingly, you may find it helpful to ratchet in the other direction too.  That is, block
commits that *improve* performance dramatically and yet do not ratchet the baseline.  This forces engineers to stop and
think about whether performance changes were intentional -- even the good ones!  A.k.a., "confirm your kill." 
-->
例如，“基线棘轮效应”的过程应该在改进的过程中被锁定。
一种合理的方法是将基线自动调整到当前性能的某个百分比内，理想情况是基于标准偏差和/或变异系数的。
另一种方法是要求开发者手动执行此操作，以便所有棘轮操作都是有意且深思熟虑后做出的决定。
有趣的是，你可能会发现在另一个方向上棘轮也是有帮助的；
也就是说，阻止那些显著*提高*性能却没有提高基线的提交。
这便迫使工程师停下来思考性能变化是否是有意的，即使是好的变化也应如此！这也被称为“确认杀戮”。

<!-- 
It's of course common that sprint goals remain stable from one sprint to the next.  All numbers can't be improving all
the time.  But this system also helps to ensure that the team doesn't backslide on prior achievements. 
-->
从一个迭代周期到下一个迭代周期，目标保持稳定当然是很常见的；
所有数字都无法永远处于增长的状态，但是这个系统也有助于确保团队不会退步到以前所取得的结果上。

<!-- 
I've found it useful to organize sprint goals behind themes.  Make this sprint about "server performance."  Or "shake
out excessive allocations."  Or something else that gives the team a sense of cohesion, shared purpose, and adds a
little fun into the mix.  As managers, we often forget how important fun is.  It turns out performance can be the
greatest fun of all; it's hugely measurable -- which engineers love -- and, speaking for myself, it's a hell of a time
to pull out the scythe and start whacking away!  It can even be a time to learn as a team, and to even try out some fun,
new algorithmic techniques, like [bloom filters](https://en.wikipedia.org/wiki/Bloom_filter). 
-->
我发现组织主题背后的迭代目标很有用。
制定名为“服务器性能”或者“摆脱过多的分配”的这个迭代周期，
能让团队有一种凝聚力和共同目标，并混合添加一点其他有趣的东西。
作为经理，我们经常忘记有趣的重要性。
事实证明，性能可以是所有人的最大乐趣，并且它也是非常容易测量，这一点上工程师都很喜欢。
而且，对我自己来说，这是一个可以暂时放下工作和休息的时刻！这甚至可以作为一整个团队学习的时间，
以尝试一些有趣的新算法技术，比如说[布隆过滤器](https://en.wikipedia.org/wiki/Bloom_filter)。

<!-- 
Not every performance test needs this level of rigor.  Any that are important enough to automatically run pre-commit
most certainly demand it.  And probably those that are run daily or monthly.  But managing all these goals and baselines
and whatnot can get really cumbersome when there are too many of them.  This is a real risk especially if you're
tracking multiple metrics for each of your benchmarks. 
-->
并非每项性能测试都需要这种严格程度，
但任何足够重要的测试在自动运行提交前肯定都需要它，
也许那些每天或每月运行的测试也会如此。
但管理所有这些目标和基线，以及当它们太多时，可能会变得非常麻烦。
这是会成为一个真正的风险，特别是如果你要跟踪每个基准测试的多个指标。

<!-- 
This is where the idea of ["key performance indicators" (KPIs)](https://en.wikipedia.org/wiki/Performance_indicator)
becomes very important.  These are the performance metrics important enough to track at a management level, to the whole
team how healthy the overall product is at any given time.  In my past team who built an operating system and its
components, this included things like process startup time, web server throughput, browser performance on standard
industry benchmarks, and number of frames dropped in our realtime audio/video client, including multiple metrics apiece
plus the abovementioned statistics metrics.  These were of course in the regularly running pre- and post-commit test
suites, but rolling them up in one place, and tracking against the goals, was a hugely focusing exercise. 
-->
这就是[“关键绩效指标”（KPI）](https://en.wikipedia.org/wiki/Performance_indicator)的理念变得非常重要的原因。
这些性能指标非常重要，足以使得管理层跟踪整个团队在任何给定时间内整体产品的健康度。
在我过去构建操作系统及其组件的团队中，这些指标包括进程启动时间，Web服务器吞吐量，
浏览器在标准业界基准测试上的性能以及实时音频/视频客户端中丢帧数等，这包括多个指标以及上述统计指标。
这些当然是包含在定期运行的提交前和提交后的测试套件中，
但将它们放在同一个地方并对目标进行跟踪，是一个非常聚焦的工作。

<!-- 
# In Summary 
-->
# 总结

<!-- 
This post just scratches the surface of how to do good performance engineering, but I hope you walk away with at least
one thing: doing performance well is all about having a great performance culture. 
-->
这篇文章只是简单介绍了如何进行良好的性能工程，
但我希望你能理解到一件事：在性能做到出色就需要一个良好的性能文化。

<!-- 
This culture needs to be infused throughout the entire organization, from management down to engineers, and everybody in
between.  It needs to be transparent, respectful, aggressive, data-driven, self-critical, and relentlessly ambitious.
Great infrastructure and supporting processes are a must, and management needs to appreciate and reward these, just as
they would feature work (and frequently even more).  Only then will the self-reinforcing flywheel get going. 
-->
这种文化需要贯穿整个组织，从管理层到工程师，以及介于两者之间的每个人。
它需要透明，尊重，积极，数据驱动，自我批评和勃勃雄心。
伟大的基础设施和支持流程也是必须的，管理层需要能对这些有所欣赏并进行奖励，
就像他们对待功能一样（并需要经常做的更好）。
只有这样，自增长的飞轮才会开始运转。

<!-- 
Setting goals, communicating regularly, and obsessively tracking goals and customer-facing metrics is paramount. 
-->
设定目标，定期沟通，痴迷地追踪目标以及面向客户的指标也是至关重要的。

<!-- It's not easy to do everything I've written in this article.  It's honestly very difficult to remember to slow down and
be disciplined in these areas, and it's easy to trick yourself into thinking running as fast as possible and worrying
about performance later is the right call.  Well, I'm sorry to tell you, sometimes it is.  You've got to use your
intuition and your gut, however, in my experience, we tend to undervalue performance considerably compared to features. 
-->
要达到我在本文中所写的所有内容并不容易。
老实说，记得在这些领域放慢速度并坚持这些原则是很难的一件事情，
而忽悠自己尽快完成功能并把关于性能的担心留到以后却很容易。
好吧，我很遗憾地告诉你，有时它就是这样的。
你必须依靠你的直觉和勇气，但是，根据我的经验，与功能相比，我们往往倾向于低估性能。

<!-- 
If you're a manager, your team will thank you for instilling a culture like this, and you'll be rewarded by shipping
better performing software on schedule.  If you're an engineer, I guarantee you'll spend far less time scrambling, more
time being proactive, and more time having fun, in a team obsessed over customer performance.  I'd love to hear what you
think and your own experiences establishing a performance culture. 
-->
如果你是一名经理，你的团队会感谢你灌输这样的文化，并且将以按计划交付性能更佳的软件作为回报。
如果你是一名工程师，并处于一个痴迷于客户性能的团队中，我保证你处于仓促之中的时间会更少，
而处于积极主动状态的时间更多，以及获得更多快乐的时间。
同时，我很想知道你的想法以及你自己建立性能文化的经历。
