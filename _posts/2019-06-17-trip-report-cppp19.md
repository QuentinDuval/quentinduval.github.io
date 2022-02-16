---
layout: post
title: "CPPP19 Trip report - The first edition of the first french C++ conference"
description: ""
excerpt: "Last Saturday marked the first occurrence of the CPPP, the first conference dedicated to C++ on the French soil, organized by Frédéric Tingaud and Joel Falcou."
categories: [functional-programming]
tags: [Functional-Programming, C++]
---

Last Saturday marked the first occurrence of the CPPP, the first ever conference dedicated to C++ on the French soil, organized by [Frédéric Tingaud](https://twitter.com/FredTingaudDev) and [Joel Falcou](https://twitter.com/joel_f).

It was one of the lucky to be there for this very first edition, and it was quite an special moment to experience the birth of what promises to be one of the great C++ conference of tomorrow.

In this post, I would like to share with you my trip report at CPPP 2019, talks I felt were interesting (actually, all of them were interesting, so that’s easy) and what I enjoyed from the spirit of the conference.

## The talks and the speakers - the heart of the conference

The conference was short (it lasted one day) but full of interesting talks and interesting speakers. Here is a short feedback on the ones I was lucky to attend, hoping that it will encourage you to see more of it when the talks will be online (and maybe come to Paris to attend next year).

### Emotional Code

[Kate Gregory](https://twitter.com/gregcons) started this first edition with a keynote on Emotional Code. I believe this is the same talk she gave at the [ACCU this year](https://www.youtube.com/watch?v=uloVXmSHiSo).

This talk showed some great examples of the impact of the emotional state of developers on the code they produce. It showed how we should avoid judging others solely based on the code they wrote, for we rarely know the social context in which things happened, and this social context matters a lot.

Fear, working environments with lack of consideration, short-gun deadlines, or misguided metrics, can explain a great deal why people never dare refactoring code, wrote a piece of code without a second look, refuse to give feedback, or avoid participating in some team or company activities.

On the contrary, inspiring figures showing great confidence in the ability to break down and accomplish a daunting task, emphatic or supporting colleagues and manager, and great team dynamic will make most of the difference between a successful project and a failed one.

I only regret that the end of the talk seemed to focus mostly on improving ourselves as individual. I would have loved to hear more about how a group can learn to complete and accept each other’s weaknesses, or influence as a group the culture in which they are in, or just when to quit and find a better working environment. Maybe in a future talk, in which case I would be quite interested.

### The state of compile time regular expressions

[Hana Dusíková](https://twitter.com/hankadusikova) gave one of the most technically impressive talk of the conference itself.

From LL(1) parsers to non deterministic finite automatons, from constexpr functions to template meta-programming, this talk was at the same time very interesting and very rewarding. It was also very intellectually demanding: it goes fast and I wish I had a way to pause during the talk, but the good news is that we can do this on Youtube after the talks are online.

Toward the end of the talk, the performance metrics about the many REGEX engines available as library, as well as the performance measures on constexpr function were quite interesting as well. I did not know that std::regex was so inappropriate for performance intensive tasks.

### Modules: what you should know

Gabriel Dos Reis talks bout the genesis of the Modules proposal, explains to us the state of the proposal, gives us some good practices on how to use Modules in the coming C++20, as well as some perspectives on what this proposal will change in the C++ world.

Should we use “import” or “include” for header files in our modules? What remains to be done for build systems to cope with modules? What are the unique opportunities that modules offers for tooling (*) or improving compilation times? Why are modules one of the most important change since the apparition of classes in C++?

This talk will give you a broad overview of these numerous topics, as well as some funny anecdotes on why and how modules originated at Microsoft.

_(*) Gabriel Dos Reis proposes a standardized format for the result of compilation of modules, which would allow tools to exploit it, and also to build other languages on top of it (much like .class in Java allowed to build languages like Kotlin, Clojure or Scala on top of the JVM). This was quite interesting._

### The anatomy of an exploit

[Patricia Aas](https://twitter.com/pati_gallardo) shows us some tricks that have been known to be used to exploit a program, by putting it in an unexpected state and trying to hack it from there.

If you are new to secure coding practices, this talk will show you why you need to pay attention to this, and how easy it becomes to exploit a program if you did not care enough.

Through a bunch of example and demos, it demonstrate the basics of an attack by buffer overflow, shell code, tries to give a picture of the mindset of an attacker, and explain why fuzzing is so interesting as attacker or as developer to defend against such attacks.

And as a bonus, the slides are beautiful and the presentation both fun and joyful. It never hurts.

### Identifying Monoids: Exploiting compositional structure in code

The last talk I attended was from [Ben Deane](https://twitter.com/ben_deane) and concerned one of my favorite topic, functional programming and more specifically in this talk, Monoids and Foldables.

In this talk, he shows plenty of examples of Monoids, from the basic ones (integer under addition/multiplication, or strings under concatenation) to more advanced ones:

Program options or command line arguments (maps in general)
Statistics: top N, min, max, mean, histograms and more
Or more advanced data structure like HyperLogLog or Parsers
The talk also shows some example of Foldables (data structures you can summarize to a single value – or alternatively, data structures on which you can implement an equivalent of `std::accumulate`) and how it combines well with Monoids, giving us the ability to fold a collection of monoidal values to a single value.

The associative property of a Monoids allows us to reorder instructions (select a parenthesisation strategy) when folding them, and therefore take advantage of parallized or optimization tricks we could not afford otherwise.

### Bonus: Adding a new clang tidy check - Live Demo

[Jeremy Demeule](https://twitter.com/jeremydemeule) presented how to implement a new check and fix-it to Clang-Tidy, with a live demo, on a non-trivial example, from the beginning to the end.

If you are curious on how you can automate changes on a large codebase (for instance to refactor some part of your code to use the newest C++ features) with a tooling much more powerful that simple regex, you might be interested in watching this workshop.

Jeremy shows how to use clang-query to discover the clang AST interactively and implement your matcher, demonstrates how to generate a fix-it, and gives you a basic workflow you can follow to successfully test and deploy your checks on production.

In summary, if you are new to clang-tidy and are interested in getting started quickly and on the right foot, this is a really good talk to start from.

## Final words on the spirit of the conference

A conference is about talks, but not only. It is also about the people there, the venue, the food, and the spirit, and everything that makes people enjoy their stay and enjoy their conversations.

And I must say that everything on that regards was quite impressive.

The venue was very nice and very well located. The food was quite good (thanks a lot for the vegetarian risotto). And above all, the spirit was really nice.

I really loved the fact that the organizers had taken into consideration the attendees. A simple color coding on the badges allowed to identify those who were okay to appear on photos from those who did not, as well as allowing to identify the french speaker from the English speaker, helping everyone to chat.

This was a nice idea that I wish could be ported to all other conferences.

And I wish a good long life to the CPPP.
