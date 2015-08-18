---
layout: post
title: "Decoding larger JSON objects in Elm 0.15"
date: 2015-08-17 22:31
comments: true
categories: elm fp applicative elm0.15 thinking applicative thinking-functionally
---

[Elm][elm] is pretty cool. It's a functional programming language with a [focus on usability][mainstream], strongly-typed but unceremonious, with nice type inferencing, good documentation and great stewardship from its creator, [Evan Czaplicki][evan].

It's so cool I've given an excited talk about it at work after only a couple of weeks of fiddling with it. And whenever I speak about tech, I try to add a demo or two to tie things together and make points clearer. That led to [elm-forecast][elm-forecast], a tiny app showing how to call APIs, decode JSON and display things on the screen.

![What elm-forecast looks like][superui]

## The problem

[Dark Sky's JSON API][darksky] offers detailed weather information for most of the world. It has up-to-the minute data for some locations, and powers a lot of nice weather apps, like [forecast.io][forecastio] and [Weather Timeline][wtime]. My app was also going to be nice, so I picked it as my data source.

I started by wrapping the current forecast definition as a [record][record]:

```haskell
type alias Forecast = { time : Int
                      , summary : String
                      , icon : String
                      , precipIntensity : Float
                      , precipProbability : Float
                      , temperature : Float
                      , windSpeed : Float
                      , windBearing : Float
                      , humidity : Float
                      , visibility : Float
                      , cloudCover : Float
                      , pressure : Float
                      , ozone : Float
                      }
```

Record types marry a lot of the feel of JavaScript objects with static types (the things after the colons).

If you're familiar with dynamic languages, the next step will seem alien: instead of just calling something like `JSON.parse(obj)` and referencing its fields, we have to tell Elm how to make a typed `Forecast` out of the serialized data.

Let's see what it looks like with a smaller object:

```haskell
$ elm repl
Elm REPL 0.4.2 (Elm Platform 0.15.1)
  See usage examples at <https://github.com/elm-lang/elm-repl>
  Type :help for help, :exit to exit
> import Json.Decode as Json exposing ((:=))
> type alias Point = { x: Float, y: Float }
> serialized = "{\"x\": -43.123, \"y\": -22.321}"
"{\"x\": -43.123, \"y\": -22.321}" : String
> pointDecoder = Json.object2 \
|   Point \
|   ("x" := Json.float) \
|   ("y" := Json.float)
<function> : Json.Decode.Decoder Repl.Point
> Json.decodeString pointDecoder serialized
Ok { x = -43.123, y = -22.321 } : Result.Result String Repl.Point
```

The code above defines a type `Point` and a `Json.Decode.Decoder` `pointDecoder`, which takes care of deserializing an object with two fields (`object2`) and returning a `Point`. As you can see, no types have been declared, yet Elm has inferred every single one of them.

`Json.Decode` has functions from `object1` to `object8`, capable of building objects with one up to eight fields. What to do with `Forecast`, that has **13**? _"Throw away five things, it's just an example app you're building to learn Elm"_, thought the lazy author. Luckily, thirst for knowledge (and a little guilt) averted that course, and I relied on what little function programming I know to _almost_ get there using [`Json.Decoder.andThen`][andthen]. Since _almost_ is actually _not quite_, to Google I went. [A thread with a recipe from mr. Czaplicki himself][elmdiscuss] offered the following solution:

```haskell
import Json.Decode as Json

apply : Json.Decoder (a -> b) -> Json.Decoder a -> Json.Decoder b
apply func value =
    Json.object2 (<|) func value
```

Let's see it in action:

```haskell
> newPointDecoder = Json.map Point ("x" := Json.float) `apply` ("y" := Json.float)
<function> : Json.Decode.Decoder Repl.Point
> Json.decodeString newPointDecoder serialized
Ok { x = -43.123, y = -22.321 } : Result.Result String Repl.Point
```

With `apply`, you can chain as many decoders as you like and build `objectN`. But how does it work?

## A detour to the world of Partial Application

In Elm, like in Haskell, every function is _curried_. What it means, in practice, is that every function takes a single argument and returns a value, which can in turn be another function, taking a single argument, returning a value, and so forth. I'll define a function `add` that (oh, how impressive) adds two numbers:

```haskell
> add x y = x + y
<function> : number -> number -> number
> add 2 3
5 : number
```

It looks like a function call with two arguments, like you see in most other languages. But look at the type signature the compiler inferred: `add : number -> number -> number`. What do the arrows represent? Well, they tell you exactly that the paragraph above tries to explain. Let's see:

```haskell
> add2 = add 2
<function> : number -> number
> add2 3
5 : number
```

When defining `add2`, I've _partially_ applied `add` with `2`, getting another function (now from `number -> number`). Calling that function will then result in a final number, the `5` literal that amazes people all over the world. This very characteristic helps you build `apply`.

In the example a few paragraphs above, `Point` is a function with the signature `Float -> Float -> Point`. That means that if I try to use it with a single decoder, it will move closer to getting an actual `Point`, but not get there yet:

```haskell
> Json.map Point ("x" := Json.float)
<function> : Json.Decode.Decoder (Float -> Repl.Point)
```

Looking at the type signature, it's structure that decodes a `Float` and returns another structure that can decode a function `Float -> Point`. If I tried to do the same thing with a type constructor that took more arguments, say `Float -> String -> Bool -> String -> Value`, the first step would yield a Decoder with type `(String -> Bool -> String -> Value)` &mdash; solved for the first parameter, still waiting for a resolution for the next three.

What `apply` does then is leverage the fact that you can progressively get to your final value by applying function by function, taking care of spitting out every every step as a `Json.Decoder`. There's a name for this pattern of having a function in a box and applying it to values in other boxes: it's an [Applicative functor][applicative]). If you've read a bit about the language, you know that Elm shies away from the burden of a Math-y, Haskell-y lexicon. The great thing is that by hiding the words but showing things in practice, you end up building an intuition for how the concepts can be *useful*.

Let's go back to `Json.object2`. It expects `(a -> b -> c) -> Decoder a -> Decoder b -> Decoder c` &mdash; a function from type `a` to `b` to `c` and `Decoder`s for `a` and `b`, yielding a `Decoder c`. In our definition `pointDecoder` in the beginning of this post, we matched that to a tee, as `Point` can be seen as a function taking two `Floats` (`a` and `b`) and returning a `Point` record (`c`). But `a`, `b` or `c` can also be a function! In fact, that's exactly what we've seen above with `Json.Decode.Decoder (Float -> Repl.Point)`. Thus, when we say:

```haskell
> Json.object2 (<|) func value
```

and replace `func` with `Json.Decoder.Decode (Float -> Point)` and `value` with `("y" := Json.float)`, we'll end up with a `Decoder` built of applying what's coming out of `value` to `Float -> Point`, arriving at `Decoder Point`. If we manually try to build the same chain, it looks like this:

```haskell
> import Json.Decode as Json exposing ((:=), andThen)
> type alias Point = { x: Float, y: Float }
> partialDecoder = Json.succeed(Point) `andThen` \
    (\f -> ("x" := Json.float) `andThen` \
    (\x -> Json.succeed <| f x))
<function> : Json.Decode.Decoder (Float -> Repl.Point)
> decoderPoint = partialDecoder `andThen` \
    (\f -> ("y" := Json.float) `andThen` \
    (\y -> Json.succeed <| f y))
<function> : Json.Decode.Decoder Repl.Point
```

Cool, right? Now that you and I understand the technique, we can go back to the gif above and marvel at how poor my CSS skills are.

## No magic

What I find the most refreshing as I dive into functional programming is that there's (usually) no magic. If you start peeling the layers, there's just functions brought together to perform amazing things. `apply` here is exactly that: the power of a few functions allowing you to convert arbitrarily large structures into a nice type Elm can understand. In a world of "factory this" "IoC container that", you can't help but smile. And it REALLY REALLY REALLY improves your programming everywhere: I'm a fan of saying my Ruby is much better and more maintainable after I decided to learn the functional ways because it's true. Hopefully you can find the same joy.


[mainstream]: https://www.youtube.com/watch?v=oYk8CKH7OhE
[elm]: http://elm-lang.org/
[pragcourse]: https://pragmaticstudio.com/
[purescript]: http://www.purescript.org/
[evan]: http://evan.czaplicki.us/
[elm-forecast]: https://github.com/dodecaphonic/elm-forecast
[darksky]: https://developer.forecast.io/
[forecastio]: http://forecast.io
[wtime]: https://play.google.com/store/apps/details?id=com.samruston.weather&hl=en
[superui]: https://s3.amazonaws.com/troikatech/elm_json/elm-forecast.gif
[andthen]: http://package.elm-lang.org/packages/elm-lang/core/1.0.0/Json-Decode#andThen
[elmdiscuss]: https://groups.google.com/forum/m/#!topic/elm-discuss/2LxEUVe0UBo
[applicative]: https://wiki.haskell.org/Typeclassopedia#Applicative
[record]: http://elm-lang.org/docs/records
