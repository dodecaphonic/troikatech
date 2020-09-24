---
layout: post
title: "fp-ts overview: Error handling, the functional way (part 1)"
date: 2020-09-24 10:15
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
  - either
---

In most codebases, error-handling means _exceptions_. A team will carefully considering potential problems, then create sets of exceptions by extending some error class that will be to signify something in the application domain went wrong.

The fact exceptions can come from deep within the call stack and bubble up very far from their point of origin often make debugging them hard. They also affect code reuse, as every path that includes exception-throwing code also becomes exceptional — a fact that is usually hidden and forces (careful and caring) developers to perpetually keep documentation up-to-date at every level of the affected call stack to avoid surprising behavior.

Thinking about the matter might lead us to two conclusions:

- Not all exceptions are alike ("Out of memory" is not the same as a domain-specific error that can be tracked and handled);
- We would benefit from treating errors less like landmines and more like integral parts of our code.

These are not original thoughts. Programmers have tried finding ways to make error handling a top concern for decades, and a few different designs are present in mainstream(-ish) languages. Java has [checked exceptions][checkedexceptions]; Go has the infamous `if err != nil` pattern; Erlang APIs often return an `err` that you check before advancing.

These are often inconvenient enough that they lead either to sloppiness (errors being ignored or supressed under the assumption that they're not important or severe enough) or confusion (the happy path becoming muddled by error checking code).

Modern statically-typed functional programming helps us with that by giving us structures and tools that act as a living document of what might wrong, let us defer handling errors, and make that happy path easy to write (and, more importantly, clear to read). One of these tools is `Either`.

# Pushing errors to the forefront

The key is making errors _values_ like any other. Unlike relying on implicit behavior, we return an `Either<E, A>`, which gives us a _left_ branch (the `E` type, normally used to denote failures), and a _right_ branch (the `A` type, for our desired outcomes).

That doesn't imply we have to write tentative code, verifying at every step whether we have a failure or a success. Because `Either` favors its right branch, we are able to build data processing pipelines that operate safely, only reaching for the result at the latest possible moment. In practice, this means _we can't forget to deal with failures_. 

Here's an example:

```typescript
import { pipe } from "fp-ts/lib/pipeable";
import * as E from "fp-ts/lib/Either";

import Either = E.Either;

type Transaction = unknown;
type Balance = unknown;
type StatementError =
  | "invalid bank account"
  | "missing account number"
  | "malformed header"
  | "transaction can't be zeroes";

interface Statement {
  readonly transactions: Transaction[];
}

declare function parseBankStatement(rawStatement: string): Either<StatementError, Statement>;
declare function validateTransactions(transactions: Transaction[]): Either<StatementError, Transaction[]>;
declare function buildBalance(transactions: Transaction[]): Balance;

const balanceFromRawStatement = (
  rawStatement: string
): Either<StatementError, Balance> =>
  pipe(
    parseBankStatement(rawStatement),
    E.map((s) => s.transactions),
    E.chain(validateTransactions),
    E.map(buildBalance)
  );
```

We move data from step to step as if nothing had gone wrong. If `parseBankStatements` returns a `Left`, everything else is a no-op; if `validateTransactions` returns a `Left`, the last `E.map` will be skipped. Whatever happens, we don't have to mix error-handling with the main logic.

# A note about Either and _errors_

The `E` in `Either<E, A>` can be whatever we want. It doesn't even have to be an "error", per se, though the default short-circuiting semantics associated with the left branch remain whatever type ends up being used.

# Employing Either

As with many things in fp-ts, we will use `pipe` often. This is mostly for type inferencing reasons, though it also helps us in keeping code declarative by not introducing bindings to hold intermediary steps.

## Putting things in Eithers

We use `right` to create a value in the right branch, `left` in the left branch:

```typescript
import { pipe } from "fp-ts/lib/pipeable";
import * as E from "fp-ts/lib/Either";

import Either = E.Either;

const goodValue: Either<Error, string> = E.right("Good");
const badValue: Either<Error, string> = E.left(new Error("Bad"));
```

## Working with Eithers when we have Rights

We can use `map` (from Functor) or `chain` (from Monad). The main practical difference is that the former allows us to transform the value while keeping it in the right branch, and the latter confers us the power to decide whether to keep on the right or move to the left (i.e. we can decide a computation should be treated as an error from then on).

```
const betterValue = pipe(
  goodValue,
  E.map(value => `${value} is now 'better'`)
); // this is a changed 'right'

const worseValue = pipe(
  goodValue,
  E.chain(value => E.left(new Error(`Nothing can be ${value} in 2020`)))
); // this is now a 'left'
```

## Working with Eithers when we have Lefts

We can modify the error using `mapLeft`:

```typescript
const crypticError: Either<number, string> = pipe(
  worseValue,
  E.mapLeft((err) => err.message.length)
);
```

Provide alternative values:

```typescript
const improvedValue = pipe(
  worseValue,
  E.alt(() => E.right("Back to 2015"))
);
```

Provide alternative values while peeking at the error:

```typescript
const optimisticValue = pipe(
  worseValue,
  E.orElse((err) => E.right(`${err.message}. But there's always 2021.`))
);
```

Note: we don't necessarily have to provide a `right`. We can map an error to another error, for instance.

## Working with existing code that might throw exceptions

TypeScript operates under the rules of JavaScript, and our application will inevitably integrate with third-party code that creates exceptions. We could use `try...catch` and do our own wrapping, but `Either.tryCatch` already takes care of that pattern for us:

```typescript
const safeParseJson = (str: string): Either<Error, unknown> =>
  E.tryCatch<Error, unknown>(
    () => JSON.parse(str),
    (err) => (err instanceof Error ? err : Error("unexpected error when parsing json"))
  );

const yayJson = safeParseJson("}");
```

## Taking things out of Eithers

It's not possible to reach into an Either and take the value or the error out. We have to help the compiler understand what is actually possible based on the runtime value we have at hand.

One way to do it is using the type guards defined in fp-ts's Either:

```typescript
if (E.isLeft(worseValue)) {
  // worseValue.left will be available
} else {
  // worseValue.right will be available
}

if (E.isRight(worseValue)) {
  // worseValue.right will be available
} else {
  // worseValue.left will be available
}
```

This may be useful in some situations, but it won't fit with our pipelines as well as the alternatives. Instead, we should use the helper functions defined in Either

### getOrElse

`getOrElse` requires us to define a way to build an `A` from an `E`:

```typescript
const mehValue = pipe(
  worseValue,
  E.getOrElse((err) => `I used to be ${err}. Now I'm free`)
);
```

### fold

`fold` requires us to provide mappings from `E` and `A` to a common type `B`:

```typescript
const answer = pipe(
  improvedValue,
  E.fold(
    () => 42,
    (value) => value.length
  )
);
```

`fold` is more powerful than `getOrElse`, because you can perform transformations (i.e. `getOrElse` requires you to provide a value of the same type `A` as in the `Either`, while `fold` allows you to return a `B`).

# Variations [defined in the library][fptsmodules]:

Either is useful in other contexts — when performing IO, when performing async computations, when performing computations within a context/environment, when performing async computations within a context/environment, et cetera —, and fp-ts ships with a few different monad stacks that include it:

- `IOEither<E, A>`
- `TaskEither<E, A>`
- `ReaderEither<E, A>`
- `ReaderTaskEither<R, E, A>`
- `StateReaderTaskEither<S, R, E, A>`

# Coming soon (in blogs, code for "this is the last post for a while")

In part 2, we'll talk about calling functions when you have more than one `Either`, accumulating errors (instead of short-circuiting), pulling things inside-out, and writing confident and expressive code using `Either`. Stay tuned.

[fptsmodules]: https://gcanti.github.io/fp-ts/modules/
[checkedexceptions]: https://en.wikibooks.org/wiki/Java_Programming/Checked_Exceptions
