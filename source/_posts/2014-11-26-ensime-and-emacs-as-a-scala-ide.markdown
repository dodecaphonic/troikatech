---
layout: post
title: "ENSIME and Emacs as a Scala IDE"
date: 2014-11-26 10:38
comments: true
categories: [scala, emacs, ide, ensime, emacsrocks, clubedosposts]
---

_"Maybe Emacs is not enough."_

That popped up in my mind, and it scared me. I knew Scala was a different beast; I knew there was probably a lot I was missing out on by using my tried-and-true workflows; I knew that IntelliJ was supposed to be amazing. Still, thinking Emacs-the-almighty was not enough frightened me.

When I started on this slow path towards learning FP, I had been using dynamic languages almost exclusively for almost 14 years, with a short stop in C++-land for a couple of them. I was used to a world of mostly ok tools centered on a REPL, and it was fine &mdash; programming is more thinking than typing and clicking, that whole _spiel_. But I had never really done anything in a good type system, and, frankly, it was time I knew how the rest of the world leveraged their tools in order to work more comfortably and effectively.

With that in mind, I evaluated the Typesafe IDE and IntelliJ IDEA 12 and 13, finding a lot of good in both tools (and a few problems, some discussed in my post about the [Reactive Programming course][reactive]). Still, after a few good days with each option, I was tempted to just go back to Emacs and rely on my memory (and [Dash][dash]) for the API signatures, do all my code refactorings by hand and use the `sbt console` for quick explorations.

Then I found out I could have the cake and eat it too.

## Enter ENSIME

ENSIME (__ENhanced Scala Interaction Mode for Emacs__) is a project that gives Emacs IDE-like capabilities. It performs type and error-checking as you write code, provides symbol inspection, facilities for browsing your codebase and performing automated refactorings. It accomplishes all that using the [Scala Presentation Compiler][scalaprescomp], a lightweight version of the infrastructure that goes only as far as needed to resolve types, find errors and do semantic highlighting.

Setting it up is super simple. Using MELPA, install the `ensime` package. Then add the following to your Emacs config:

```
(require 'ensime)
(add-hook 'scala-mode-hook 'ensime-scala-mode-hook)
```

Then add the plugin to your global `sbt` config (e.g. `~/.sbt/0.13/plugins/plugins.sbt`):

``` scala
resolvers += Resolver.sonatypeRepo("snapshots")

addSbtPlugin("org.ensime" % "ensime-sbt" % "0.1.5-SNAPSHOT")
```

And then, in your project directory, run `sbt gen-ensime` (requires sbt >= 0.13.5). It will resolve the dependencies, install the ENSIME plugin and leave everything ready to go.

Now, when you open a buffer, you're gonna see the following in your mode line:

{% img https://dl.dropboxusercontent.com/s/zss0kz5lr8hvhmr/2014-11-27%20at%2010.22.png ENSIME Disconnected %}

Use `M-x ensime` to start a connection. It might take a few seconds for it to do what it must to analyze your project, but you'll eventually see the mode line change to show it's ready to work.

## Code completion

One of the cool things ENSIME provides is real code completion, based on the type you're dealing with. Instead of the usual `M-/` cycling, you can explore an API by looking at the method signatures and documentation. Here's the thing in action:

{% img https://s3.amazonaws.com/troikatech/ensime_as_ide/completion.gif Completing %}

## Type inspection

Sometimes Scala's type inference engine gets confused, giving you something too broad or too narrow for your needs; other times, you just want to know the type of a `val`. Worry not: ENSIME can tell you what has been inferenced just by putting the cursor over the token you want and pressing `C-c C-v i` (works a bit like `:t` in `ghci`).

{% img https://s3.amazonaws.com/troikatech/ensime_as_ide/type_at_point.gif Inspecting types %}

You can also show uses of a symbol by pressing `C-c C-v r`.

## Automated Refactorings

ENSIME offers six simple, but extremely useful automated refactorings:

- Inline Local
- Extract Local
- Extract Method
- Rename
- Organize Imports
- Import Type at Point

{% img https://s3.amazonaws.com/troikatech/ensime_as_ide/refactoring.gif Refactoring %}

Of all of these, _Import Type at Point_ is the only one I'd consider flaky. It resolves the type perfectly, but inserts the `import` statement inline. I don't know if that's configurable. Otherwise, it works as many other automated tools: finds each change, shows you the substitution, asks you to ok it.

## Navigation

You can use `M-.` and `M-*`, normally associated with finding tags, to move inside your project.

{% img https://s3.amazonaws.com/troikatech/ensime_as_ide/navigation.gif Navigation %}

You can also jump from implementation to test, and vice versa.

## `scala` and `sbt` integration

If you press `C-c C-v s`, an sbt console will be launched. A lot of my usual Ruby workflow of running specs from keybinds and jumping quickly to the REPL can be reproduced with this feature.

For instance, when you want to run all tests, you press `C-c C-b T`. When you wish only to invoke `testQuick`, you use `C-c C-b t`. 

There's keybinds for changing a region or a buffer, too &mdash; useful both for playing with code and exercising your Emacs gymnastics.

## Finally

ENSIME has been fun to work with. It allows me to focus on code and work comfortably with my (admittedly small) projects. It's a great showcase of Emacs capabilities, and has led a couple of hardcore vim-using friends to show admiration.

If you're doing Scala and don't want to commit to an IDE, but wish to have more of a modern environment, please try ENSIME. I even hear there's vim and jEdit clients.

[reactive]: /blog/2014/01/12/reactive
[dash]: http://kapeli.com/dash
[scalaprescomp]: http://scala-ide.org/docs/dev/architecture/presentation-compiler.html
