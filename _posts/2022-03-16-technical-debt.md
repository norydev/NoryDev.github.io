---
layout: blogpost
title: Tell me you have technical debt without telling me you have technical debt
subhead: One must imagine sisyphus happy
excerpt: I always struggled to explain technical debt to non technical people. I tried to find examples but it always fell short in one way of another. Until today.
categories: code
date: 2022-03-16 10:00:00 +0300
imgclass: scarab
---

### Day 0

Alice is not her real name but she is a real – and very competent – software engineer. She works at ⭐MOVING-FAST⭐ startup and is tasked with implementing a pretty complex feature. The feature can be split into 2 sub-parts; 🪄PRETTY-MAGIC sub-feature which is simpler to implement and already bringing quite some value to the product, and 🔮VERY-MAGIC sub-feature which is much harder to implement but adding very distinctive value to the product.

Alice makes an estimate on how long it would take to implement the whole feature. However, she only gets about half that time allocated by her manager. So she intelligently decides to ship 🪄PRETTY-MAGIC sub-feature in a well tested and functioning manner first. And then with the rest of the time, gets pretty far on 🔮VERY-MAGIC: it works in the most optimistic flow where all data is valid and the user does not try anything funny. But it breaks in every other case, and this breakage renders part of the product unusable until some engineer manually fixes it for the offended user.

So that's not really production ready, is it? But the time allocated has ran out, so Alice ships this feature behind a feature flag: it's there in production, but nobody is using it. It still needs work, but at least the code is present for all to see and ready to be built upon. The current state of the feature is well documented. Alice now moves onto other projects.

### Day 120

Fast forward to 4 months later, a new software engineer, Bob (again, not his real name) is hired at ⭐MOVING-FAST⭐

### Day 180

Bob has been at the company for 2 months and is now tasked to take over 🔮VERY-MAGIC feature. His goal is to "finish what was left off by Alice on Day 0". Bob digs into the code and the documentation but can't quite make the feature work up to what is documented. Luckily for him, Alice still works at ⭐MOVING-FAST⭐ so Bob can reach out. Together they find out that some critical parts that used to work, aren't working anymore.

After digging into it, they realize that circa Day 110, a big part of 🔮VERY-MAGIC feature was completely removed from the codebase. What happened is, a totally different team, while porting their 🎈AWESOME feature to a new model, didn't port the 🔮VERY-MAGIC part of it. Why? "We were short on time and facing a strict deadline. Since this code was behind a feature flag and unused, we thought it's not a priority and we would do it later".

You guess what happened next, right? Later equals never.

So now Bob makes an estimation that it would take him about 2 days of work to make the port of the 🔮VERY-MAGIC part of 🎈AWESOME feature.

### Here is the bill

Let's sum up: that is 1 day of exploration + 2 days of work = 3 days of engineering time, just to get back to the state of the feature at Day 0. And that's considering all goes well and there isn't more surprizes along the way.

On Day 0, ⭐MOVING-FAST⭐ took a debt by not allocating Alice time to finish the feature completely. The interest of the debt 6 months later is at least 3 days of engineering time (a.k.a. hundreds of dollars). And the missed revenue by not having the feature live for 6 months (hard to estimate).

I don’t think I could make up a better example of what technical debt is.

(Plot twist: the Bob of this particular episode is me. Maybe next time I'll be Alice.)
