---
layout: post
title: "Property-based testing in Ruby"
date: 2014-04-02 10:32
comments: true
categories: blog
tags: [ruby, fp, quickcheck, testing, haskell, scala]
---
For the past year or so I have slowly been dipping my feet into the vast functional programming seas. From taking the awesome [Coursera][coursera] [offerings][fpcoursera] [from Typesafe][reactivecoursera] to slowly working through Rúnar Bjarnason's and Paul Chiusano's _[Functional Programming in Scala][fpinscala]_, my mind has been expanding proportionally to the time I dedicate to learning its ways. It has been incredibly rewarding and humbling.

One such reward has been coming into direct touch with property-based testing. This technique, first developed by [QuickCheck][quickcheck] in Haskell-land, spins automated testing on its head: instead of codifying what is proper behavior by asserting that the outputs for given inputs match what is expected, the tester establishes logical properties about what should happen and lets the tool generate loads of inputs to check if they hold. If something goes wrong, the tool will then try to find the smallest test input that breaks the property (_falsifies_ it), a process called _shrinking_; if it can't find anything, you can sigh with relief and think about what to scrutinize next.

Having a QuickCheck-like tool at your disposal can be incredibly powerful. The more complex the software or the algorithm, the greater the likelihood of your carefully curated unit and integration tests having blind spots. [Basho][basho], for instance, [have written about the stark realization that their worker pool library was full of subtle bugs by using QuickCheck for Erlang][bashoqc], and you can find [other][telecomqc] [instances][volvoqc] of how the technique helped make better software.

I don't know about you, but when I come in contact with stuff like that I immediately think of how improved parts of my day job would be if I just could apply it. Considering that my daily duties are conducted in Ruby, I felt it was time I explored the subject in that realm.

## A contrived setup that hopefully shows how it can work out

Let's say we've decided to implement our own linked list class in Ruby. We would probably start our implementation with something like this:

``` ruby "List" https://gist.github.com/dodecaphonic/9934064#file-list-rb
require "singleton"

class Nil
  include Singleton
  
  def empty?; true; end
  
  def to_s
    "Nil"
  end
end

class Cons
  def initialize(head, tail = Nil.instance)
    @head = head
    @tail = tail
  end

  attr_reader :head, :tail

  def empty?; false; end

  def to_s
    "(#{head} . #{tail.to_s})"
  end
end
```

Using that _very_ convenient API, we can build lists:

``` ruby
>> l = Cons.new(1, Cons.new(2, Cons.new(3, Nil.instance)))
>> l.to_s # => "(1 . (2 . (3 . Nil)))"
```

We know that, in a linked list, adding to the head is O(1), while appending to the end is O(n). So we build algorithms that respect its efficiency guarantees. However, when we, say, map this list into another list, it results in the following situation:

``` ruby "List" https://gist.github.com/dodecaphonic/9934064#file-list-rb
def do_something_amazing(list, acc = Nil.instance)
  super_value = list.head * 1337
  if list.tail.empty?
    acc
  else
    do_something_amazing(list.tail, List.new(super_value, acc))
  end
end

>> do_something(l).to_s # => "(4011 . (2674 . (1337 . Nil)))"
```

Processing things from head to tail means the list ends up reversed. It's common, then, to reverse it back when we're done processing, to preserve the order an external user would expect. Let's add a <code>reverse</code> method to a <code>List</code> helper module:

``` ruby "List" https://gist.github.com/dodecaphonic/9934064#file-list-rb
module List
  def self.reverse(list, acc = Nil.instance)
    if list.empty?
      acc
    else
      reverse(list.tail, Cons.new(list.head, acc))
    end
  end
end
```

So when we try to reverse what was created in <code>do_something_amazing</code>, we get what we need:

``` ruby
List.reverse(do_something_amazing(l)).to_s # => "(1337 . (2674 . (4011 . Nil)))"
```

Awesome. I think this is enough for us to start exploring properties. If you're getting bored, take a sip of coffee and come back when you're ready. There's a few cool tricks below the fold.

## Testing the old way

Being the good developers we are, we are covering that code with tests:

``` ruby "List Tests" https://gist.github.com/dodecaphonic/9934064#file-list_test-rb
class ListTest < MiniTest::Test
  def test_reversing_lists
    assert_equal "(3 . (2 . (1 . Nil)))",
      List.reverse(Cons.new(1, Cons.new(2, Cons.new(3)))).to_s
    assert_equal "(9 . (400 . (321 . (1 . (10 . Nil)))))",
      List.reverse(Cons.new(10, Cons.new(1, Cons.new(321, Cons.new(400, Cons.new(9)))))).to_s
    assert_equal "Nil", List.reverse(Nil.instance).to_s
    assert_equal "(1 . Nil)", List.reverse(Cons.new(1)).to_s
  end
end
```

We're pretty confident that's enough, even if it was kind of boring to do manually. That amount of testing would let us go home and sleep soundly.

## Testing the QuickCheck way

First, we'll need something like QuickCheck in Ruby. The best, most idiomatic, most maintained, least-Monad-y thing I have found is [Rantly][rantly]. It has both primitive value generation built-in and property testing with shrinking. We'll skip over the basic API and go straight to defining a property to check if my algorithm is really bullet-proof. To aid in the creation of lists from Arrays, we'll add a new helper:

``` ruby "List" https://gist.github.com/dodecaphonic/9934064#file-list-rb
module List
  # ...
  def self.from_values(*values)
    values.reverse.inject(Nil.instance) { |ls, v| Cons.new(v, ls) }
  end
end
```

To check that it works, let's change the existing tests and see if they still pass:

``` ruby "List Tests" https://gist.github.com/dodecaphonic/9934064#file-list_test-rb
class ListTest < MiniTest::Test
  def test_reversing_lists
    assert_equal "(3 . (2 . (1 . Nil)))",
      List.reverse(List.from_values(1, 2, 3)).to_s
    assert_equal "(9 . (400 . (321 . (1 . (10 . Nil)))))",
      List.reverse(List.from_values(10, 1, 321, 400, 9)).to_s
    assert_equal "Nil", List.reverse(Nil.instance).to_s
    assert_equal "(1 . Nil)", List.reverse(List.from_values(1)).to_s
  end
end
```

``` 
Run options: --seed 48889

# Running:

.

Finished in 0.001256s, 796.1783 runs/s, 3184.7134 assertions/s.

1 runs, 4 assertions, 0 failures, 0 errors, 0 skips
```

Great. Now to the newfangled thing. As I mentioned before, writing a property to check requires us to think differently than we would with regular unit tests. Your formulation should state something logical, something that does not rely on specific inputs. Following that guideline, we can reason about reversing lists in the following manner:

``` ruby "List Tests" https://gist.github.com/dodecaphonic/9934064#file-list2_test-rb
  # ...
  def test_reversing_by_property
    property {
      length = range(0, 1_000_000)
      List.from_values(array(length) { integer })
    }.check { |list|
      assert_equal list.to_s, List.reverse(List.reverse(list)).to_s
    }
  end
  # ...
```

The meat is in the <code>check</code> block. Determining that a list has been reversed correctly requires us to check if reversing it again gets us back to the original list. To seed our check, we build a <code>property</code> block that creats an array with a random length between 0 and 1_000_000, itself filled with random integers. Let's run the tests again:

```
$ bundle exec ruby list.rb
Run options: --seed 17130

# Running:

.
..........
success: 100 tests
.

Finished in 121.969127s, 0.0164 runs/s, 0.8527 assertions/s.

2 runs, 104 assertions, 0 failures, 0 errors, 0 skips
```

It took a while (we wanted to be thorough, with those million-item arrays), but we're pretty sure it works. I'm a believer and I'm stoked; when I look at you, however, I see a face that says "look, it's cool and all, but isn't it _kind of worthless_? The tests we had were telling us the same thing, and we only needed the power of our minds to generate the correct inputs. Why go through so much trouble?"

Well, what about those times when ours minds fail us?

## Catching a bug with Rantly

Let's say you're excited about building your own data structures and want to wrap that linked list inside a very inefficient Set. You mutter to yourself that you should make sure items are not inserted twice, which for now seems to be the main difference between Sets and Lists as storage containers.

You build a little more structure into what you already have, adding a <code>prepend</code> method and inlining <code>reverse</code> into a <code>List</code> base class:

```ruby "List 2" https://gist.github.com/dodecaphonic/9934064#file-list2-rb

class List
  def to_s
    raise "Don't use this directly, fool"
  end

  def empty?; true; end

  def prepend(v)
    Cons.new(v, self)
  end

  def reverse(acc = Nil.instance)
    if empty?
      acc
    else
      tail.reverse(Cons.new(head, acc))
    end
  end

  def self.from_values(*values)
    values.reverse.inject(Nil.instance) { |ls, v| Cons.new(v, ls) }
  end  
end

class Nil < List
  include Singleton

  def to_s
    "Nil"
  end
end

class Cons < List
  def initialize(head, tail = Nil.instance)
    @head = head
    @tail = tail
  end

  attr_reader :head, :tail

  def empty?; false; end

  def to_s
    "(#{head} . #{tail.to_s})"
  end
end
```

To check if an item exists, you add a <code>contains?</code> method:

``` ruby "List with contains" https://gist.github.com/dodecaphonic/9934064#file-list2-rb
class List
  # ..
  def contains?(v); false; end
  # ..
end

class Cons < List
  # ..
  def contains?(v)
    head == v || tail.contains?(v)
  end
  # ..
end
```

Then you write your immutable Set and matching tests:

``` ruby "A dumb set implementation" https://gist.github.com/dodecaphonic/9934064#file-set-rb
class DumbSet
  def initialize(storage = Nil.instance)
    @storage = storage
  end

  attr_reader :storage
  private     :storage

  def push(v)
    if !storage.contains?(v)
      DumbSet.new(storage.prepend(v))
    else
      self
    end
  end
  alias_method :<<, :push

  def contains?(v)
    storage.contains?(v)
  end

  def to_a
    values = []
    list   = storage
    until list.empty?
      values << list.head
      list = list.tail
    end
    values
  end
end

class DumbSetTest < Minitest::Test
  def setup
    @s = (((DumbSet.new << 1) << 2) << 3)
  end

  attr_reader :s

  def test_contains
    assert s.contains?(3)
    assert s.contains?(2)
    assert s.contains?(1)
    assert !s.contains?(4)
  end

  def test_uniqueness
    assert_equal [-32, 1, 2, 3], (s << -32 << -32 << -32).to_a.sort
  end
end
```

And because I spotted you writing new code and yelled "HEY USE RANTLY IT'S SO COOL YIPEE", you add some property tests:

``` ruby 
class DumbSetTest < Minitest::Test
  # ...
def test_contains_property
    property {
      array(range(0, 100)) { integer }
    }.check { |vs|
      s = vs.inject(DumbSet.new) { |ds, v| ds << v }
      assert vs.all? { |v| s.contains?(v) }
    }
  end

  def test_uniqueness_property
    property {
      array(range(0, 100)) { integer }
    }.check { |vs|
      ns = vs.inject(DumbSet.new) { |ds, v| ds << v }
      rs = vs.inject(ns) { |ds, v| ds << v }
      assert_equal vs.sort, ns.to_a.sort
    }
  end
  # ...
end  
```

It looks good:

```
$ bundle exec ruby set_test.rb
Run options: --seed 15625

# Running:


..........
success: 100 tests
..
..........
success: 100 tests
..

Finished in 0.119717s, 33.4121 runs/s, 1720.7247 assertions/s.

4 runs, 206 assertions, 0 failures, 0 errors, 0 skips
```

You then implement the removal of items:

``` ruby "Set with delete" https://gist.github.com/dodecaphonic/9934064#file-set-rb
class DumbSet
  # ...
  def delete(v)
    ls = storage
    ns = DumbSet.new

    while !ls.empty?
      if ls.head != v
        ns = ns << v
      end

      ls = ls.tail
    end

    ns
  end
  # ...
end

class DumbSetTest < Minitest::TestCase
  # ...
  def test_delete
    os = (((DumbSet.new << 1) << 2) << 3)
    ns = os.delete(1337)
    assert_equal [1, 2, 3], ns.to_a.sort
    ns = os.delete(3)
    assert_equal [1, 2], ns.to_a.sort
    ns = ns.delete(2)
    assert_equal [1], ns.to_a.sort
    ns = (ns << 432).delete(1)
    assert_equal [432], ns.to_a.sort
    ns = ns.delete(432)
    assert_equal [], ns.to_a.sort
  end
  # ...
end
```

Your tests pass, but this time you don't listen to me about adding another property. You're just not that convinced they're worth their salt, and it looks good enough to ship with all the tests you've added. The Pokémon Collecting app you work on can benefit from it right now, instead of 20 minutes from now. To production it goes.

Time goes by, and you've forgotten about me and our little adventure. Your system is humming along and moved on to maintenance mode. Days have been kind of slow, so you decide to add an optimization you've read about in Hacker News, detailing how a node.js program got a 10x speedup. You modify your delete method accordingly:

``` ruby "Set Tests" https://gist.github.com/dodecaphonic/9934064#file-set_test-rb
  # ...
  def delete(v)
    ls  = storage
    tmp = DumbSet.new

    while !ls.empty?
      if (ls.head != v) && (ls.head < 1500) # secret performance trick
        tmp = tmp << ls.head
      end

      ls = ls.tail
    end

    tmp
  end
  # ...
```

CI still reports all green.

A few days later, you receive a report from a User telling she deleted their Pokémon with power level 3, but her Pokémons with levels 4013, 1551 and 20000 disappeared. Your first line of defense &mdash; your tests &mdash; have not caught any issues. Sweating bullets and drowning in emails from stakeholders and other Pokémon fiends, you're about to collapse.

And then you remember: what about trying to express a property to see if it holds?

``` ruby
  # We'll add at most 10 unique items and then delete the first
  # 2. If there's anything wrong, this will blow up.
  def test_delete_property
    property {
      array(10) { range(0, 5000) }.uniq
    }.check { |values|
      os = values.inject(DumbSet.new) { |s, v| s << v }
      ds = values[0..1].inject(os) { |s, v| s.delete(v) }
      assert_equal (values.size - 2), ds.to_a.size
    }
  end
```

You run it and it explodes:

```
$ bundle exec ruby set_test.rb
Run options: --seed 46455

# Running:


..........
success: 100 tests
..
failure: 0 tests, on:
[384, 437, 120, 718, 1850, 4579, 3178, 4191, 533, 2669]
F
..........
success: 100 tests
...

Finished in 0.093858s, 63.9264 runs/s, 2248.0769 assertions/s.

  1) Failure:
DumbSetTest#test_delete_property [set_test.rb:69]:
Expected: 8
  Actual: 2

6 runs, 211 assertions, 1 failures, 0 errors, 0 skips
```

What? How come you've only got 2 when you expected 8? Well, there must be something wrong with delete, after all. Let's take that array and try it on an _pry_ session to see what happens:

```
[1] pry(main)> values = [384, 437, 120, 718, 1850, 4579, 3178, 4191, 533, 2669]
=> [384, 437, 120, 718, 1850, 4579, 3178, 4191, 533, 2669]
[2] pry(main)> os = values.inject(DumbSet.new) { |s, v| s << v }
=> #<DumbSet...>
[3] pry(main)> values[0..1].inject(os) { |s, v| s.delete(v) }.to_a
=> [718, 533]
```

Wait a minute! Should delete also remove everything that's over 1000-ish? Is there anything in the code that stipulates such a thing? Maybe that node.js optimization was not so great after all. Let's remove it and run the tests:

```
$ bundle exec ruby set_test.rb
Run options: --seed 2727

# Running:

.
..........
success: 100 tests
..
..........
success: 100 tests
.
..........
success: 100 tests
..

Finished in 0.099329s, 60.4053 runs/s, 3120.9415 assertions/s.

6 runs, 310 assertions, 0 failures, 0 errors, 0 skips
```

Voilà: properties have saved the day, and you've learned not to trust Hacker News bravado ever again.

## Is using Rantly the same as using QuickCheck or ScalaCheck?

Sort of. For one, you have to write your own generators every time you want something other than basic types, while both QuickCheck and ScalaCheck can figure out a lot by themselves. This can make expressing what you mean a lot easier, and you don't spend time debugging your <code>property</code> blocks in search of mistakes. That said, writing a generator for your own types requires only that you instantiate them in the <code>property</code> blocks with other auto-generated values.

Shrinking is not as good in Rantly. It works ok a lot of the time, but it could be improved. On the surface, from skimming the algorithms used in ScalaCheck and Rantly, it doesn't _seem_ that different, but over that side of the line the patterns in minimization seem easier to spot.

There's also no mechanism to test stateful code. ScalaCheck has [Commands][sccommands] to help in modeling state changes, and I'm aware [PropEr][proper] and [QuickCheck for Erlang][qcerlang] also provide something in that direction.

One minor thing is that integration with RSpec and MiniTest could be improved. Its output pollutes the test run, and on large suites it becomes hard to know the relationship between a failed property and a failed test. It should be easy to fix for anyone motivated. On that note, there's no ready-made extension for MiniTest (although adding one is trivial enough that I'm sending a PR to fix it).

## Final considerations

I hope I have proven, even if with a craptastic example, that property-testing can aid you in writing better Ruby code. Our community is great when it comes to using tests both as a design and as a verification tool, and QuickCheck (via Rantly) is a new way of thinking about them. You should keep your TDD/BDD to carve out your objects and responsibilities, but also add property checks where suitable to strengthen your confidence in the system. 


[fpinscala]: http://www.manning.com/bjarnason/
[fpcoursera]: https://www.coursera.org/course/progfun
[reactivecoursera]: https://www.coursera.org/course/reactive
[quickcheck]: http://en.wikipedia.org/wiki/QuickCheck
[basho]: http://basho.com
[bashoqc]: http://basho.com/quickchecking-poolboy-for-fun-and-profit/
[telecomqc]: http://www.quviq.com/documents/erlang001-arts.pdf
[volvoqc]: http://www.autosar.org/download/conferencedocs11/12_AUTOSAR_ModelBased_Quviq.pdf
[emailrx]: http://www.regular-expressions.info/email.html
[rantly]: https://github.com/hayeah/rantly
[coursera]: http://coursera.org
[sccommands]: https://github.com/rickynils/scalacheck/wiki/User-Guide#stateful-testing
[proper]: https://github.com/manopapad/proper
[qcerlang]: http://www.quviq.com/
