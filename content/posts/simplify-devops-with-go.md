+++
date = '2026-07-10T12:56:53+02:00'
title = 'Simplify DevOps With Go'
weight = 1
+++

Look, it's simple. One of the main jobs of a DevOps[^1] engineer is to keep reducing the unnecessary complexity and bringing clarity. And this is not simple.

> Any intelligent fool can make things bigger and more complex...  

-- E.F. Schumacher

To be able to simplify one needs to become very experienced, very senior. Here's a pattern I've noticed that makes it difficult. I often take on[^2] a task, that's at least partly new to me. There's a new tool, framework, platform or programming language involved. And there's usually a deadline[^3]. So this approach seems reasonable:

1. Learn the new stuff ~~well~~.
2. Build a prototype.
3. ~~Build the real solution~~.
4. ~~Refactor it~~.
5. ~~Refactor it again~~.
6. Deploy.

Now, the problem is that the first step looses the "well" part and steps 3, 4, and 5 are often skipped. Sometimes because there's no time. Sometimes because I'm lazy[^4]. One can mitigate this by asking for more time and by fighting the laziness[^5]. But I should like to focus on a third option here: spending less time learning.

The best way to spend less time learning something is, of course, to already know (most of) it. And that's why I prefer to invest deeply in concepts and in one good programming language and reuse[^6] them wherever possible.

For me, that language is Go.

And this site is about using Go to simplify DevOps work: writing small tools, automating repetitive tasks, building infrastructure, and replacing unnecessary complexity with simple, maintainable software.

[^1]: Here, "DevOps" refers to the infrastructure and automation role. Depending on the company, it may also be called Platform Engineer, Cloud Engineer, SRE, or Sysadmin.
[^2]: Or I'm given one.
[^3]: Often self-imposed.
[^4]: Yes, LLMs tend to ecourage this.
[^5]: Although laziness can also be a virtue: https://bcantrill.dtrace.org/2026/04/12/the-peril-of-laziness-lost/
[^6]: And learn more or relearn them.