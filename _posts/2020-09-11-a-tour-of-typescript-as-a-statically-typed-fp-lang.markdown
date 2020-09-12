---
layout: post
title: "A Tour of TypeScript as a Typed Functional Programming Language"
date: 2020-09-11 16:25
comments: true
categories: blog
tags:
  - haskell
  - fp
  - typescript
  - fp-ts
  - io-ts
  - functional
  - functionalprogramming
---

_(edited for clarity in 2020-09-11T00:28:15Z)_

If you've been curious about (or studying) statically typed functional programming for a while, I bet you've asked yourself these questions:

- How much do I have to learn to apply it successfully?
- Will I ever stop being surprised by how much I don't know?
- Can I get paid to work in that style?

Digging deeper and becoming more knowledgeable is a fun effort by itself. The third question is more delicate, nuanced, and probably anxiety-inducing than the first two: Haskell (or Haskell-adjacent) jobs are scarce, and your best bet is trying to land a Scala job in an FP-friendly organization.

But there's a dearth of Scala jobs, as well, and most of them are in the land of data science. Instead of growing despondent, here's my proposition: the easiest path to marrying statically-typed functional programming and your source of income is to go extremely mainstream and use TypeScript.

## Does TypeScript pass muster?

Yes! You don't even have to squint that hard.

TypeScript gives you enough tools to make illegal states unrepresentable at compile time, and its type system, by virtue of having to model JavaScript with its every wart, is very interesting and powerful. Its features include:

- First-class functions;
- [Parametric polymorphism][tsgenerics], with rich constructs such as conditional types;
- Algebraic Data Types.

Expressing these concepts might not be as succint as in Haskell, PureScript or Elm, but it's possible with tolerable noise, resulting in the same compounding benefits you would find in friendlier environs.

What follows is an overview of the basics of using TypeScript successfully as a Haskell impersonator. Some of the examples will rely on the [fp-ts][fpts] ecosystem.

## Thinking in transformations

One major way FP-style has infected the mainstream is in computations that are built as chainable expressions. In JavaScript, for instance, it would be quite natural to write something like the code below:

```javascript
const animals = [
  { name: "giraffe", type: "mammal" },
  { name: "elephant", type: "mammal" },
  { name: "crocodile", type: "reptile" },
  { name: "emu", type: "bird" },
  { name: "gecko", type: "reptile" }
];

const reptilesWithG = animals
  .filter(a => a.class === "reptile")
  .map(a => a.name)
  .filter(name => name.startsWith("g"));
```

It has a lot of what's good about functional programming. Data pipelines, higher-order functions, composability. But why solving problems using that approach interesting?

One way to view it is that it focuses on the _what_ instead of on the _how_. It doesn't introduce bindings at the same level as the code you're interested in — you don't care about the intermediate steps between `animals` and `reptilesWithG`. Existing higher-order functions (`map` and `filter`) take you to your destination.

That immediately provides a big benefit: a single expression can be refactored easily. It would be straightforward to fuse the two filters. Extracting a function taking `animals` and calling it in place could be done in a pinch.

Data transformation pipelines rely on _composability_ — combining functions that do part of a job in order to build richer behavior. This in turn requires each part to be referentially transparent. Types only enhance these characteristics, as each function will be mechanically verified by the compiler, and pipelines will be correctly connected when the compiler is happy.

There are different functional programming libraries for TypeScript, and each implements its pipelining functions. If you use fp-ts, `pipe` (and `flow`) will be your bread and butter:

```typescript

import { pipe } from "fp-ts/lib/pipeable";
import * as A from "fp-ts/lib/Array";

interface Animal {
  name: string;
  type: string;
}

const animals: Animal[] = [
  { name: "giraffe", type: "mammal" },
  { name: "elephant", type: "mammal" },
  { name: "crocodile", type: "reptile" },
  { name: "emu", type: "bird" },
  { name: "gecko", type: "reptile" }
];

const reptilesWithG = pipe(
  animals,
  A.filter(a => a.class === "reptile" && a.name.startWith("g")),
  A.map(a => a.name)
);
```

`pipe` passes the results of one step as an argument to the next step. `animals` will be the input for `A.filter`, and the result of said filtering will be the input for `A.map`.

This way of writing the code detaches the Array transformation functions from functions in the `animals` object. This is good: not only it makes steps recombinable, it also helps cement the idea that sequences of transformations are not restricted to things in collections. fp-ts makes it easy to do the same thing with asynchronous computations, for instance:

```typescript
import * as T from "fp-ts/lib/Task";

const addPopulation = (animal: Animal): Task<AnimalWithPopulation> => pipe(
  fetchPopulation(animal.name),
  T.map(population => { ...animal, population })
);
```

It also will aid in building your intuitions for the similarities between different structures and computational contexts. `A.map` and `T.map` are possible because `Array` and `Task` are both `Functors`, for instance, but you don't even need to know that for them to be useful. Understanding that changing data in `Task` requires `map` and changing data in `Array` also requires `map` will possibly help in in intuiting that `Either` might also require `map`.

## Immutability

JavaScript is not immutable by default. A few tools exist (`Object.freeze`, non-writable properties) to avoid mutation at runtime, but they usually restrict shallow mutations only, and require discipline when dealing with data (e.g. you can't forget a call to `freeze`).

TypeScript gives you a few tools to avoid the temptation of changing data in place. You can, for instance, declare interface fields as `readonly`:

```typescript
interface Animal {
  readonly name: string;
}

const animal = { name: "Tiger" };

animal.name = "Liger"; // error TS2540: Cannot assign to 'name' because it is a read-only property.
```

As well as declaring Arrays as `readonly` (or `ReadonlyArray<T>`):

```typescript
const answers: readonly string[] = ["foo", "bar", "baz"];
answers[42] = "what is the question?"; // error TS2542: Index signature in type 'readonly string[]' only permits reading.
```

And function parameters as `Readonly`:

```typescript
interface Room {
  name: string;      // not readonly
  occupancy: number; // not readonly
}

function changeOccupancy(room: Readonly<Room>): Room { 
  room.occupancy = 110; // error TS2540: Cannot assign to 'occupancy' because it is a read-only property.
  
  return room;
}
```

You can even [make a type recursively immutable][recursiveim]. 

## Type system features

TypeScript has a structural type system, instead of a nominal one. If you have an `interface Friend { name: string; age: 42; }` and a `function isAdult(subject: { age: number }): boolean`, passing a `Friend` to `isAdult` will work, because it requires `age: number` and `Friend` has an `age: number`.

This begets interesting modelling tools and capabilities. Remember `addPopulation`? It took a type called `AnimalWithPopulation`. Here's one way you could define it in TypeScript:

```typescript
interface AnimalWithPopulation extends Animal {
  population: number;
}
```

And here's another:

```typescript
type AnimalWithPopulation = Animal & {
  population: number;
};
```

It might look like a case of "six of one, half a dozen of the other", but isn't. What the second form gives you is a way to [augment types][tsadvanced] (TypeScript calls them _intersection types_) without creating class hierarchies, wrappers or decorators.

Say you have an e-commerce site that has some functionality only available for registered customers. A registered customer must have an email and an address, and the presence of an email and the presence of an address are concepts already defined in other places. You could define it as so:

```typescript
type Customer = { lastViewedPage: string };
type HasEmail = { email: string };
type HasAddress = { address: Address };
type HasName = { firstName: string, lastName: string };
type RegisteredCustomer = Customer & HasName & HasEmail & HasAddress;
```

Admittedly this is a contrived example, but I hope it illustrates an interesting possibility: you can build your domain model in such a way that the code acts only on the pieces it cares about:

```typescript
function generateLabel(recipient: HasName & HasAddress): PackageLabel {
  // ...
}

function prepareShipment(recipient: HasName & HasEmail & HasAddress): Shipment {
  const label = generateLabel(recipient);
    
  // ...
}
```

Another interesting aspect of its type system are literal types:

```typescript
type John = "John";
type Role = "admin" | "guest" | "regular" | "manager";
type Floor = 1 | 2 | 3 | 4 | 5 | 6;
```

Code specifying `Role` or `Floor` will only take those literals. You can't pass any `number` as a Floor, or any `string` as a Role (though there's a way to [narrow those types to the subset specified in the union][tsguards]). `John` can only be "John".

When combining literal types with objects and unions, we get Algebraic Data Types.

## Making impossible states unrepresentable: modelling with Algebraic Data Types (ADTs)

Take the common necessity of fetching data asynchronously. A desirable UX can be described in four scenarios:

- While fetching the data, you would like to display a loading indicator;
- When you have data, you would like to render it;
- When fetching fails for some reason, you would want to display the error in the UI;
- When refreshing data, you would like to display the loading indicator over the existing data;

You might, then, create this model:

```typescript
interface PageData<T> {
  isLoading: boolean;
  data?: T;
  error?: Error;
}
```

with these conditionals:

- if `data` is present, render it;
- if `isLoading` is true, render the loading indicator (if `data` is also present, render it over the rendered data);
- if `error` is present, render the error.

It looks fine at a first glance, but it's not bulletprof. One day, `data` and `error` could both be present by mistake, which could then result in the wrong UI because of error-prone conditionals like `if (!isLoading && !error && data) { ... }`.

By thinking things through, it becomes clear that some of the data is only relevant in some of the states of the fetching process. `error` is only relevant when there's an `Error`, and `data` is only relevant when actual `data` exists or a refresh is happening. If neither is present, the initial loading is happening.

You _could_ write tests to keep these in check, but [tests can't prove the absence of bugs][c2absence]. If a mistake not covered by them got introduced, it would only be found during runtime (possibly in production). You should, instead, leverage the compiler to ensure you get what you need at the right time, and _only_ what you need:

```typescript
interface FetchingData {
    _tag: "FETCHING_DATA";
};

interface FetchedData<T> {
    _tag: "FETCHED_DATA";
    data: T;
}

interface FetchingFailed {
    _tag: "FETCHING_FAILED";
    error: Error;
}

interface RefreshingData<T> {
    _tag: "REFRESHING_DATA";
    data: T;
}

type FetchData<T> =
  | FetchingData
  | FetchedData<T>
  | RefreshingData<T>
  | FetchingFailed;
  
const fetchingData: FetchData<never> = 
  { _tag: "FETCHING_DATA "};
  
const fetchedData = <T>(data: T): FetchData<T> => ({
  _tag: "FETCHED_DATA", 
  data 
});

const refreshingData = <T>(data: T): FetchData<T> => ({ 
  _tag: "REFRESHING_DATA", 
  data 
});

const fetchingFailed = (error: Error): FetchData<never> => ({ 
  _tag: "FETCHING_FAILED", 
  error 
});
```

`FetchData<T>` is a generic type that is either `FetchingData`, `FetchedData<T>`, `RefreshingData<T>` or `FetchingFailed`. It cannot be more than one of them at the same time. Furthermore, you cannot make assumptions about the data: the only thing the alternatives have in common is the `_tag` field, which should be used to [discriminate between the alternatives][discriminatedunion]. You have to check that field's value in order to be allowed to reach into the objects.

That means you can't have `error` and `data` at the same time. Nor can you have `error` and `isLoading`. There's no way to have an invalid state:

```typescript
const renderData: IO<void> = (state: FetchedData<string>) => {
  switch (state._tag) {
    case "FETCHING_DATA":
      return renderIsLoading();
    case "FETCHED_DATA":
      return renderData(state.data);
    case "FETCHING_FAILED":
      return renderError(state.error);
    case "REFRESHING_DATA":
      return renderRefreshing(state.data);
  }
}

const state = fetchedData("oh data my data");

renderData(state);
console.log(state.data);
  // Property 'data' does not exist on type 'FetchData<string>'.
  // Property 'data' does not exist on type 'FetchingData'.

```

It's not uncommon to hear something to the tune of "that's a lot of code". I usually point out that many tests will not have to be written, and that it's a lot clearer than nested if statements or large conditionals.

## fp-ts

[fp-ts][fpts] gives you many of the tools you get in Haskell, PureScript or Scala, and a lot of convenience functions to lift your regular TypeScript to a transformation-friendly style. It encodes higher-kinded types using [Lightweight higher-kinded polymorphism][lighterweight], since TypeScript doesn't have that feature. You get used to specifying HKT instances by hand pretty quickly.

Due to TypeScript's type inferencing limitations, a lot of your code will use `pipe` (it's not unlike using `|>` in Elm). This might make it confusing at times, and you'll eventually develop strategies and sensibilities regarding when to extract helpers and break large computations.

Your solutions can be very expressive, and the types will tell you a lot. In the following example, we'll take that list of animals from earlier, filter by their class and names, then fetch their Wikipedia pages concurrently and build a list of the results, failing the entire computation if an error occurs:

```typescript
import { flow } from "fp-ts/lib/function";
import { pipeable } from "fp-ts/lib/pipeable";
import * as A from "fp-ts/lib/Array";
import * as O from "fp-ts/lib/Option";
import * as TE from "fp-ts/lib/TaskEither";

import TaskEither = TE.TaskEither;

const reptilesWithGWikipediaArticles: TaskEither<WikipediaArticle[]> = pipe(
  animals,
  A.filterMap(
    flow(
      option.some, 
      option.filter(a => a.class === "reptile" && a.name.startsWith("g")),
      option.map(fetchWikipediaArticle)
    )
  ),
  A.sequence(TE.taskEither)
);
```

`TaskEither` defines an asynchronous computation that can fail. `filterMap` allows you to filter an array and change entries at the same time. `sequence` allows you to turn a list of asynchronous computations into an asynchronous computation with a list.

fp-ts forms a rich ecosystem of libraries that give you lenses, runtime encoding/decoding of types, parser combinators and other niceties you've probably heard about or used.

## Putting it all together: Functional TypeScript in the workplace

We've been successfully using all of the above (and very little more) at my day job for 10 months, building a robust HTTP API that's serving millions of requests per day, integrating with dozens of services. It goes weeks between serving 500s, and using proper types to model computations that can fail forces the team to handle errors and introduce fallback strategies.

I consider it a great technical success. But this is only part of the story: getting there required my team to find their footing and get comfortable.

Since I was the one tasked with getting the service off the ground, I built a proof-of-concept and presented it to them. _When I had the buy-in_, I immediately became responsible for training and helping everyone in getting along with the codebase. If you find yourself in the same situation, be prepared to pair program, change pedagogy, and be patient. Remember it's a very different way to do things. You should be able to show different solutions in different styles, as well as refrain from reviewing negatively code that could be more functional.

Would I be more satisfied working in Haskell? Perhaps. The fact is I wouldn't be doing this style of programming professionally, in a conservative technical setting, were it not for node.js being acceptable and TypeScript being adopted by other squads. This pairing makes for a good Trojan horse, and it's ergonomic enough that the experience ends up being quite pleasant. A lot more people end up being exposed to solving problems differently, to boot. At the end of the day, that's the biggest win.

[fpslack]: https://fpchat-invite.herokuapp.com/
[discriminatedunion]: https://www.typescriptlang.org/docs/handbook/advanced-types.html#discriminated-unions
[adts]: https://en.wikipedia.org/wiki/Algebraic_data_type
[tsgenerics]: https://www.typescriptlang.org/docs/handbook/generics.html
[tsguards]: https://www.typescriptlang.org/docs/handbook/advanced-types.html#type-guards-and-differentiating-types
[fpts]: https://gcanti.github.io/fp-ts/
[c2absence]: https://wiki.c2.com/?TestsCantProveTheAbsenceOfBugs
[tsadvanced]: https://www.typescriptlang.org/docs/handbook/advanced-types.html#intersection-types
[recursiveim]: https://www.sitepoint.com/compile-time-immutability-in-typescript/
[lighterweight]: https://www.cl.cam.ac.uk/~jdy22/papers/lightweight-higher-kinded-polymorphism.pdf
