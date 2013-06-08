---
layout: post
title: "Currency Arbitrage in Python"
description: ""
category: finance
tags: [python, finance, theory]
---
{% include JB/setup %}

Yesterday, there was a [post on Hacker News](https://news.ycombinator.com/item?id=5842349) about solving a currency arbitrage problem in Prolog. The problem was originally [posted](http://priceonomics.com/jobs/puzzle/) by the folks over at Priceonomics. Spoiler alert - I solve their puzzle in this post.

I've actually solved this puzzle before, on a final in my undergraduate algorithms class. I remember being proud of myself for coming up with the solution, and for remembering a trick from high school precalculus that allowed me to get there. (Side note: I now see this trick used basically all the time.) Of course, this was for an algorithms class, so I never actually implemented it.

The solution is this: structure the currency network as a directed graph, with exchange rates on the edges. Then, find a cycle where multiplying the weights together gives you a number > 1. Two things to note - first, this is like solving a longest path problem. Second, it's not quite that because longest paths are about *additive* weight and we're looking for *multiplicative* weight.

The trick I mentioned above involves taking the `log` of the exchange rates and finding an increasing cycle. We have to remember that the sum of `log`s of some terms is the same as the `log` of the product of those terms. To suit the algorithm we use, taking the negative of the `log` of the weights and finding a negative weight cycle. It turns out that the [Bellman-Ford algorithm](http://en.wikipedia.org/wiki/Bellman%E2%80%93Ford_algorithm) will do just this.

Originally, I had plans to redo the author's Prolog solution with the Bellman-Ford algorithm. While I'm a big fan of [declarative programming and the first order logic](http://boom.cs.berkeley.edu/), I chickened out and decided to redo things in good old python.

The goal here was to write in concise, idiomatic python with enough structure to be readable, and use common python libraries as much as possible. So, I use `urllib2` and `json` to grab and parse data from Priceonomics, and `NetworkX` to structure the data as a graph and run the Bellman-Ford algorithm. 

Of course, it turns out that by default `NetworkX` raises an exception when a negative cycle is detected, so I had to [modify it](https://github.com/networkx/networkx/pull/886) to return the results of the algorithm when a cycle is detected.

Once I did that, though - I wound up with 50 lines of pretty simple python. Most of the works is actually done in interpreting the results of the Bellman-Ford algorithm.

<script src="https://gist.github.com/etrain/5736212.js">
</script>

