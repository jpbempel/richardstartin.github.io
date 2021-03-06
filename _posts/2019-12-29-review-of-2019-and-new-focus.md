---
title: "Review of 2019 and New Focus"
layout: post
date: 2019-12-29
---

This is my first and probably last meta post on this blog; the end of the year is as good a time as any to write such a thing.
I have written very few posts this year, and this post starts with a quick recap of what I did write, in case you missed it.

1. TOC 
{:toc}

#### [Does Inlined Mean Streamlined Part 1: Escape Analysis](/posts/does-inlined-mean-streamlined-part-1-escape-analysis)

This was supposed to be the first post in a series where I muse about the almost memetic obsession with inlining amongst performance focused Java users.
Inlining _is_ the daddy of all optimisations, but in some cases, it's really not as important as you might think.
Inlining allows optimising analysis to consider larger chunks of code, widening its scope.
If methods which allocate can be inlined, escape analysis can be performed in a wider context, and may well lead to the elimination of  otherwise escaping  objects.
If inlining is prevented, this can't happen, and your code will slow down ever so slightly (though how much depends).
In this post I argue that this benefit might not often be enough to bother designing for, and the cost of missing out can be contrived to be similar to the differential cost in write barrier cost incurred in changing the garbage collector.
Calling a post "part 1" dooms "part 2" to failure, which would have argued that there's often no point in worrying about inlining reduction operations (such as hash codes) given the paucity of common sub-expression elimination.

#### [Classifying Documents](/posts/classifying-documents)

I have always been interested in eliminating the part of my job which entails writing and reading brain dead application logic, and always prefer to impose constraints on a solution to writing it out explicitly.
Even so, I am yet to find a rules engine that I don't hate using or find I can't use because it's too slow.
This post is about a simple rule engine I wrote, which is fast, but I do hate using.
I later found out that the approach I used (_linear bitvector scan_) was [proposed for policy based packet forwarding in the 90s](https://www.cse.iitb.ac.in/~krithi/courses/631/lakshmanan.pdf), and [similar ideas have been experimented with for an eBPF replacement for iptables](https://sebymiano.github.io/documents/19-eBPF-Iptables-Demo.pdf), so the data structure is in good company.

#### [Vectorised Byte Operations](/posts/vectorised-byte-operations)

Whilst Graal is the cool new thing in JIT compilers, there are developers plugging new and powerful optimisations into C2.

<blockquote class="twitter-tweet"><p lang="nl" dir="ltr">Graal vs C2 <a href="https://t.co/iM5MWvEZIB">pic.twitter.com/iM5MWvEZIB</a></p>&mdash; Vsevolod (@qwwdfsad) <a href="https://twitter.com/qwwdfsad/status/829419306320556033?ref_src=twsrc%5Etfw">February 8, 2017</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

#### [XXHash](/posts/xxhash)

XXHash and XXHash64 are relatively old hash functions, but they work really well, and are based on the principle of replacing total order with partial order when you have a large chunk of data to process.
I have a go at vectorising the main loops in XXHash using the Project Panama Vector API, but it doesn't work well, likely because the algorithm isn't designed with this in mind.
XXHash64 and XXH3 have wider internal states, and probably benefit more.

#### [Multidimensional Switches](/posts/multidimensional-switches)

The linear bitvector scan thing works with 32 bit or 64 bit words too, and I reckon it would be perfect for branch-free pattern matching in the new switch expressions Java is getting.
The state of the art in this space is decision trees, and there is a lot of [theory](http://moscova.inria.fr/~maranget/papers/ml05e-maranget.pdf) about how to reduce the size of tree to keep code size small.
I would like to think that my idea smokes decision trees at this scale, and that the impact on code size of a few integers is negligible, but it's an outlandish claim, and doing a decent comparison would take more time than I have.

#### [Dependency Management as Revenue Capture](/posts/dependency-management-as-revenue-capture)

Lots of companies with lots of money use open source libraries, but the authors don't get paid to maintain the libraries.
Every now and then you see people lumped with some major bedrock library trying to get donations so they can carry on doing what they do.
Dependency management solutions could be put in place to meter usage of libraries, create products out of bundles of dependencies which are paid for on a subscription basis, and cut royalties to authors from the revenue stream.
I realised this was a terrible idea when I told a friend who is a corporate lawyer, and his eyes lit up.

#### [Finding Bytes](/posts/finding-bytes)

It's easy to demonstrate the impact of SIMD on numerics, but having worked in finance for a long time, I can say with great certainty that quant libraries will always be written in C++, often with Python wrappers.
I expect there will be very little traditional demand for fast numerics on the JVM.
I'm convinced that the difference SIMD will make to Java is in all the boring stuff: things like base 64 encoding, compression, encryption, and parsing.
This post looks at finding the next byte in some input (i.e. `String.indexOf`), and I find this operation can go so much faster using explicit vectorisation using the Vector API.

### New Focus

I'm really quite bored of writing about the things I write about, but find I learn a lot just writing this blog.
I often discover what I don't know just by trying to write down what I think I do know.
Working as a software developer for several years, I have gained many practical skills, such as testing and profiling systems that I wish were more interesting, but have lost other knowledge through lack of use.

In software development we often talk in terms of dynamical systems - _service rate_, _backpressure_, _warmup_ - but lack the tools to reason about system dynamics quantitatively.
I have always been interested in system dynamics and spent a lot of time at university studying dynamical systems and stochastic processes, applying these ideas to biological and information systems,  but have lost this knowledge and the thought processes it makes possible.
I am revisiting these topics as if I had been a software developer when I first encountered them, and will start to write about modeling information systems; the motivation to write being to find gaps in understanding so I can fix (or accept) them.
If you were reading these posts for the micro-optimisations - seriously or in jest - you will find the new posts a bit different, but you have been warned!
