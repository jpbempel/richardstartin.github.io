---
ID: 10865
title: Data Driven Logic
author: Richard Startin
post_excerpt: ""
layout: post
redirect_from:
  - /data-driven-logic/
published: true
date: 2018-04-29 18:02:33
tags: java pattern-matching
---
I really don't like reading or writing blocks of `if-else` statements. They make my eyes glaze over. Rumour has it that processors don't like executing them either, though that's less true now than it once was. There are two problems with these blocks of statements, and neither one of them is performance:

<ol>
	<li>They are hard to read and tend to have subtle dependencies on line order.</li>
	<li>They can't be treated as data, and can't be executed remotely unless you do something weird like serialise <em>code</em>.</li>
</ol>

Since I started programming in Java, I have been aware of the existence of rule engines, but I have never heard of a single case of "soft coding" working out. In my own experience, every time I have been involved in the implementation of a system with a DSL to empower business analysts to control the business logic, there has been low stakeholder participation during the design of the DSL and developers have ended up writing the business logic anyway. The most excruciating aspect of this is that it dilutes accountability for testing by blurring the boundaries between the application and user input. Perhaps your experience differs. However, rule engines can eradicate cyclomatic complexity in application code, and systems consisting of straight line code (with high test coverage) tend to do what they are supposed to. Soft coding isn't the appeal of rule engines, getting rid of the `if-else` blocks is. If you squint at rule engines the right way, they look <em>data driven</em> and they start to get exciting. I can't see value in anything more complicated than a decision table.

You can represent a block of `if-else` statements as a decision table by considering every possible branch as a line in the table. Your decision table doesn't need to be exhaustive: there can be cases where you fall through and throw an exception or choose a default. This can be quite hard to write in imperative code, and you may need to throw the same exception in multiple places, set flags, or otherwise rely on line order to achieve this. Decision tables have a really nice property: if you want to start treating certain cases as exceptional, you just delete the line from the table.

Decision tables are very similar in character to case classes in Scala, or to the weaker `when` expressions present in Kotlin, but decision tables can be allowed to grow much larger. I wouldn't allow a match expression with 50,000 cases through a code review even if someone had the energy to write one deviously enough to come in under the maximum byte code method size.

I looked at several implementations of decision tables on GitHub and saw a lot of clean code, but not a lot of textbook computer science. Every single implementation iterated through a list of rules checking the rule against the input data. I have implemented a password strength checker like this in the past (I know! I probably shouldn't have done this myself!) which is fine because a strong password checker might have at most a dozen rules. What if you work in adtech and want to classify the people you track (how do you sleep at night?) as members of one or many of 50,000 clusters which can be described in terms of regions of, say, a 50 dimensional feature space? Imagine your task is to guess which cluster your quarry belongs to in a few microseconds at most. You won't get far if you iterate through thousands of rules. 

I implemented a small library in the evenings over the last couple of weeks called <a href="https://github.com/richardstartin/bitrules" rel="noopener" target="_blank">bitrules</a>. This was based on some <a href="https://richardstartin.github.io/posts/fast-and-simple-rules-engine-with-roaringbitmap/" rel="noopener" target="_blank">ideas</a> I had about using RoaringBitmap for decision tables last year. The idea is very simple: think of a list of rules with constrained attributes as a matrix, and transpose that matrix and loop through the <em>attributes</em> during classification. This is a similar approach to that taken in <a href="https://richardstartin.github.io/posts/blocked-signatures/" rel="noopener" target="_blank">blocked signatures</a>, a search technique used in BitFunnel, which translates an expensive signature scan to a random access. In the case of bitrules, for each constraint on each attribute, bits are removed from a mask of potentially matching rules. These bitsets are intersected sequentially, resulting in a bitset rapidly decreasing in cardinality. Because I used RoaringBitmap, rapid reduction in cardinality means a rapid reduction in size, which means cache friendliness. There are a few tricks in the code like using range encoding for range attributes, so that range queries can be evaluated with a single bitset intersection. I plan to implement a hopefully efficient serialisation format so the table can be sent to another server and used for classification remotely.

I don't actually know how fast this code is: performance is context sensitive, and I shy away from making "performance measurements". It's best suited to cases where there are a large number of rules (thousands) and I bet it's really fast when there are ~50,000 segments in a ~50 dimensional space. I don't even have a use case right now for bitrules: it was just fun writing the code. I have started releasing it to <a href="https://search.maven.org/#artifactdetails%7Cuk.co.openkappa%7Cbitrules%7C0.1.3%7Cjar" rel="noopener" target="_blank">maven central</a>, while I can't guarantee its fitness for purpose, perhaps it may be of some use to someone else.
