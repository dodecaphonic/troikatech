---
layout: post
title: "Maybe Haskell"
date: 2015-04-02 08:06
comments: true
categories: book fp haskell
---

The programming world is one of trends and fashions. One week you're on the top of the world for using that NoSQL database, and then you're very wrong the next; one day it's all about Rails, the next it's node.js, now it's Go. Using [Hacker News][hn] as a compass seemingly means discarding everything you're doing now to follow the next big thing.

Like fashion, though, sometimes one of those new things is actually well-rounded, makes a mark and becomes permanent. Also like fashion, the new thing might be an old thing that people are rediscovering or just now ready to adopt. Judging by what's on the specialized news, [Functional Programming][fp] is the old-new rage that's changing the world and is here to stay.

It's no wonder: more enlightened programmers and language designers have sprinkled some of the joys of that paradigm upon our OO tools, making us giggle with happiness when chaining `map`s and `inject`s and taking blocks to change a method's behavior, or using anonymous and higher-order functions in tired and uncool languages of yesteryear, feeling more productive all the way. It's so transformative to think in pipelines and in functional composition that we end up wanting to know how to learn more and feel _even better_. Functional Programming called, and you answered.

But lo!, what is a catamorphism? What the hell is a Category, and why does it need a theory? Why did someone put a Monad in my burrito? Is Functor just a funny word Erik Meijer says?

Let's face it: it can be daunting. None of the usual landmarks of what we call _programming_ are there to guide you through learning, and it's easy to feel inadequate and, dare I say it, intellectually inferior. Fear not: [Pat Brisbin][patbrisbin] knows this, and is here to help.

## The Book

["Maybe Haskell"][maybehaskell], written by the aforementioned mr. Brisbin, is a book of the short and sweet kind. It quickly acknowledges that it probably will not be the definitive guide on any of the subjects it talks about, and moves right on to the material.

From the gates, the author explains referential transparency and uses equational reasoning to show you how a name for an expression (let's say `add(x, y)`) can be replaced safely by the expression itself (`x + y`). This will become a tool later on to clarify that what seems so elaborate is actually pretty straightforward. It's very effective, because it unfolds everything that looks so terse and codified into its components, and those components into their components, working as both a calming device ("see how simple it is? It's just about context") and an illustration of the power of function composition.

It then gets to its main thread: what is the `Maybe` type and how is it built? What does it mean to adopt `Maybe` in a code base, and how do you deal with it without having every function in your system taking and returning other `Maybe`s? Even if you've heard of or applied `Maybe`, it might give you ideas and reveal unknown subtleties &mdash; especially if all you've learned about it has been self-directed.

From that on you'll hit three head-scratchers in sequence: Functors, Applicatives and Monads. It begins with showing you ways of not infecting your code with `Maybe` everywhere and ends with calling functions that take multiple `Maybe`s, dealing with the possibility of one or more of them not being there. The path is of full of little joys and insights to savor, and you'll get what a Monad _does_ by the end of it (even if the answer to what it _is_ goes through endofunctors and other details).

What I greatly enjoyed was the "Other Types" section. It's brief, but tackles how you can use the same building blocks to improve designs and make errors and side-effects more explicit. While I knew most of the benefits and the material, I thought about the complete novice and how that section could spark new ideas. I didn't "ooh" and "aah" because it was new to me: I did because it will hook a lot of casually interested people who perhaps got the book because it came from someone in the Ruby world and aren't very invested in the ideas of FP yet. It will definitely make _Ruby_, if nothing else, better.

In the end, even if "Maybe Haskell" just explains enough of what the language can do to support the examples, the quiet imponence and lack of pretense of the language become very evident. As the text progresses, you'll see the ivory tower where FP wizards live for what it really is: a building that begins on the same ground you and I step on, made out of very solid and simple materials. Luckily we have Pat gently guiding us to that conclusion.


[hn]: http://news.ycombinator.com
[patbrisbin]: https://twitter.com/patbrisbin
[maybehaskell]: http://maybe-haskell.com
[fp]: http://en.wikipedia.org/wiki/Functional_programming
