---
layout: post
title: "You broke the build!"
date: 2013-10-12 21:24:00
comments: true
categories: blog
tags: [process, tdd, coworkers]
---
A coworker and I share rank as managers of a small team of Rails developers, an oddity created by a brief stint I had working for another company and then coming back. I used to be his manager, but when I returned my role was constrained to the technical side of things, while he took over caring for the team and client relations. It is an arrangement that suits me better, to be frank, as it lets me focus on what I do best.

Both of us still work on code. Our approaches differ both because of our relative gaps in experience (I've worked on more projects, and with Ruby specifically for a lot longer) and our personalities. The last point does not generate contention per se, but leads to clearly different stances when it comes to quality in our work: my ultimate goal is that everyone folds their branches back to <code>master</code> with tests and properly factored code; he knows tests are a force for good, but often lets them slide for different reasons.

Recently we've discussed this and the effects it has on the rest of the team. One of the junior developers has become completely test-averse, carelessly breaking parts of the suite and shrugging it off. While I have called attention to many instances of the thing, the separation of responsibilities means it's up to him to make changes to the process and accountability, which he vowed to do.

That is, until he merged a branch he worked on for almost two months, clearly never running the tests, and broke the build.

### Why am I so angry?

My frustration skyrocketed when I got the notification from the CI server (and many others from [Code Climate][cc] about the decline in quality, but that's a subject for another post). It took me a few minutes to regain my composure, get the code, confirm **NO TESTS** were running, and write to him that the build was badly broken. He retorted with an excuse that surely made a lot of sense to him ("I needed to share some code with the junior dev"), but only got me more pissed. It was 10pm on a Friday night, and I decided to let it go.

The whole thing left a bad taste in my mouth, though. The experience I had at the other company was terrifying precisely because a mission-critical Rails app went up without a single test, resulting in very long working hours and awful bugs in production &mdash; all in a culture that eschewed personal responsibility and poo-pooed the thought of having a _process_. Maybe I've been burnt a lot more than the young guns working with me now, but I just know it will bite us down the road, and that it will result in dissatisfied customers, long nights and me not even having the energy to say "I told you so". For now he can say our code isn't in production. But it isn't YET.

That, in the end, is what makes me angry. That otherwise intelligent people can't see that they're gonna pay tenfold for the cost they are avoiding now by not learning and following best practices, and that testing, while certainly not flawless, helps you sleep peacefully knowing that the code you changed didn't break anything in the suite and the build.

### How can I make it better?

I'm not yet sure. I'm often afraid that it hinges on a personal commitment to learning and improving your craft. Other times, though, I wonder if it isn't just an absence of mentors who are patient and didactic enough to sit beside someone that isn't convinced and work him or her through it, and of company cultures that accept that the time it takes to make things better is not money wasted.

It doesn't help that a lot of people still don't know the first thing about things you and I take for granted. Most of them think _"Refactoring"_ is a menu on their IDEs, or empty jargon to use when code needs some TLC. Forgive them, [father Fowler][refactoring], for they do not know what they are doing. Basic concepts may have to be more evenly spread before we can talk about building the next step.

One thing I can personally improve is on communicating clearly to my teammates and people close to me is that I care about these practices, making it clear that my interlocutor is not expected to know everything about it (I certainly don't). I also have to understand it may be an unwelcome message at times, and that I should just take the time it needs.

### Final thoughts

It's hard to not cry foul when people don't do what they say they're going to. The situation that triggered this logorrhea, now that it's through, must be assessed not as some act of malice, but simply as the disconnect between discourse and what is perceived to be a pressing matter. And to be fair, this particular instance had a happy ending: after my frustrated emails, the dude went back and made things right with the code and the build is bright green. Hopefully it will be ok to talk about it openly as both the wrong and the right thing to do, and prove compelling enough for the junior devs and other coworkers to feel engaged.

[cc]: https://codeclimate.com/
[refactoring]: http://www.refactoring.com/
