---
ID: 10806
title: Parallel Bitmap Aggregation
author: Richard Startin
post_excerpt: ""
layout: post
redirect_from:
  - /parallel-bitmap-aggregation/
published: true
date: 2018-04-01 20:39:52
tags: roaring
---
A bitmap index represents predicates over records as sets consisting of the integer identities of each record satisfying each predicate. This representation is actually a few decades out of date, and systems like <a href="https://github.com/pilosa" rel="noopener" target="_blank">Pilosa</a> use much more sophisticated data structures, and Sybase had even more on offer back in the 90s. But the chances are, if you've rolled your own bitmap index, you've used equality encoding and have a bitmap per indexed predicate. <a href="https://github.com/RoaringBitmap/RoaringBitmap" rel="noopener" target="_blank">RoaringBitmap</a> is a great choice for the bitmaps used in this kind of data structure, because it offers a good tradeoff between bitmap compression and performance. It's also <em>succinct</em>, that is, you don't need to decompress the structure in order to operate on it. With the naive index structure described, it's likely that you have many bitmaps to aggregate (union, intersection, difference, and so on) when you want to query your index. 

RoaringBitmap provides a class `FastAggregation` for aggregations, and the method `FastAggregation.and` is incredibly fast, particularly given its apparent simplicity. This reflects a nice property of set intersection, that the size of the intersection cannot increase and tends to get smaller as the number of sets increases. Unions and differences are different: the problem size tends to increase in magnitude as the number of contributing sets increases. While `FastAggregation.or` and `FastAggregation.xor` are highly optimised, not a lot can be done about the fact each additional set makes the problem bigger. So it may be worth throwing some threads at the problem, and this gets more attractive as you add more dimensions to your index. You can, of course, completely bypass this need by reading some database research and sharing bitmaps between overlapping predicates.

I <a href="https://github.com/RoaringBitmap/RoaringBitmap/pull/211" rel="noopener" target="_blank">implemented</a> the class `ParallelAggregation` in RoaringBitmap, but I'm not convinced the technique used performs as well as it could do. RoaringBitmap stores the 16 bit prefix of each integer in a sorted array, with the rest of each integer in that 16 bit range stored in a container at the same index in another array. This makes the structure very easy to split. The implementation I worked on seeks to exploit this by grouping all the containers by common key as a `SortedMap<Short, List<Container>>` before executing each reduction (i.e. `Function<List<Container>, Container>`) in parallel in a `ForkJoinPool`. This results in a reasonable speedup of 2x-6.5x compared to `FastAggregation` on an 8 core machine, but it uses quite a lot of temporary memory just to set the problem up. I don't think it should be possible to beat this approach without grouping the containers by key somehow, but I suspect there are lighter weight approaches which use less memory and give better throughput. Perhaps this would be an interesting problem to work on?
