---
title: Refactor Any Curried Function with Only Two Basic Functions in Typescript (or Any Language)
date: '2023-02-26'
tags: ['typescript', 'functional programming', 'haskell', 'refactor', 'semantic-editor-combinators']
draft: false
summary: How to refactor any curried function by combining two basic functions, `compose` and `flip`, in a generic, mechanical and purely functional way. The arguments of any curried function can be swapped and/or altered at any (nested) level.
images: []
layout: PostSimple
canonicalUrl: refactor-any-curried-function
---

Suppose you have a curried function that takes four arguments, for example:

```ts
const getPaymentAmount =
  (price: number) =>
  (quantity: number) =>
  (discount: number) =>
  (taxRate: number): number =>
    (price * quantity - discount) * (1 + taxRate)

assert.deepStrictEqual(getPaymentAmount(10)(2)(5)(0.2), 18)
```

Often, a programmer would want to extend this `getPaymentAmount` function _without modifying_ it. For example, one may want to produce a new function:

- **with a fixed discount**: by moving the third argument `discount` to the top level, so one can partially apply it away; or
- **that accepts an argument of another type**: by changing the type of the fourth argument `taxRate` from `number` to `string`.

A naive way would be to wrap around the function with a new function like so:

```ts
const getPaymentAmountWrapped =
  (/* moved to top level */ discount: number) =>
  (price: number) =>
  (quantity: number) =>
  (taxRate: /* changed from number to */ string) =>
    getPaymentAmount(price)(quantity)(discount)(parseFloat(taxRate))

const getPaymentAmountNoDiscount = getPaymentAmountWrapped(0)

assert.deepStrictEqual(getPaymentAmountNoDiscount(10)(2)('0.2'), 24)
```

Though it works, the code is very redundant. The programmer has to repeat the same boilerplate wrapping everytime in similar situations.

Is there a way to refactor and/or extend _any_ curried function in a generic, concise, and mechanical way?

The answer is yes.

_You can find all the code in this post and play around with them at [Typescript Playground](https://www.typescriptlang.org/play?jsx=0&ts=4.9.5#code/PTAEAkEMGMGsE9QHdKIC4HtkYE60jhgK4B2AJqAAYkZkCmAXJAM7N05qWgC2tRANnVA00oSADdIAS36QARoNBSSoACrwADnWbQcUjaIAKs+AHNCpMgCgQagBbFTd4RlFocROv0QAienQ1QZncpaFE6AEciSH4fUGg7OjhlUwAaJVEkGX5sPFAAM1xQYhwgqW4NRToAD0gKwWZQOS8MJCtoDBJgsVZ2UQBeUABvK1Ax0H8NAGUQsIBRKJiGUAAKSGXSWBokEnS5DZIt1pIASlB+gD5h0fHbqXzVyFAAQn7BuTOR2+-xtDtCJDCOiAuY4Qg4FaUAAkQ0gAF8Xm9QDC5HDKCcbj84Zjvh0uhhBAA6fgYUyQmHw85IlFojG3bHY9qdbqmOhoQyobh0EhoACCvFIAxxKw0emgjGERG4zRwZ0uONuK0WPKkaHgyxIUplcouCp+qzIUh0xB5Gq17B1ev14xWaEg1QASpA0BLNdKLWb3aV5dbfTbRaEhAAqUDKtCqxAAWgmRo6grOIZWAEZQABqUB2x3OugnADcVisLDYHEJkxmYrQC2i-BWONZ7M53L5Ap5yYADCcVgAmTsAVk7bcJPdSOKTAA4rBimV1RB0Khg2OccQAeABC6QAwhcVvllit9qBVzrQBvLdbl7zt6Y9+tQLzj0fzrq-WM1st78sN0+rb78isyZAJxTni3QPIMKzVJ62rLMEegkKYT6gNUhKYOWKQrBiIGiAh4EHnIGAEnQkCnFB7CIXIoAAPygCmyxttO3R2HueEEYIxHHrBKTnPEGDzmwO6dqYU5Fn0pZ0AEaHzIsNY4nYtoeDmI63D4SY+JOBZYQU-D6EutwXuk64ntuu6PO+x77ssj6XCeZ76hZh7mbe96IX+gH7sBzKzp00DOtxEEwSE8HmeqQSBUJiHVGmoDwAWIklmWsyVtJta3Pk2kaCseI+WgnY+II+RoD4uV6E4hUnEp4w+CVdhoPlhXqQxogAPpJl2ADMAAsuk2i1AVweFPp2U1XZ9Sktl+isTVtaNQXfi+3yTR1M0Dc+836i1UXDZtbWbR1GmeaAw1Jp13Fpfok2tZ1wm9PF4nTIlVYxCl4xHZ1Kw+F2RXvapuVtV9PgdUVFVjCp7WAw1mk4NoAgDDxfF0PtM5afoXZTcsABi6WoztgxQ8wMM7ulHlIy1bVdl1gxnRo2MXWD13FmgYkSQ9yU4qT5Pff9f25Z9uWA+VOKg51anE6B6VtU1S2gJj+gSxToB4wTiv8GghP6EB+aaS15M48jGhy7TV2xTdjMJRWj0ybc2sdW1nM8-9-PvdzwOgEL4Oi6IVMAGxNQA7Bj6U+773HK6rocrOHkfQyrasaEB8f5o1h31hy8Bcjy-ImrDVOxzTKeNhnLY5ZhB1NfnadNpngoAHIYAAIrGWfcWXbKp+nzZZysHaFibTP3ebrNW+X7dVzytcN8agrtp2PZd0OAu3OTEMHaKSS8RoC5CJT6WZevm9nCwoCGFDc4b2wiPdAQphSjyjSDKvp+bxfohXzfaDMCNd44Nf3C3123GRWsuHV+v934QSAknEBt8pa8m-m-Zg8tAFXCjvjGOwC4GgOYOA+Oz8ghSgpsKW8bptRzTsgeYhFpSETWgKRWUVC1orDILQ8aa1bhPHTBRdM0AorWFYT8JO+NuAdQAOqqjsGheC3EoHvw6iKAgbAACSPJOyCI6vTUSZtQhJWrM9MYqjRF-AkWSJMM9OxtT5kDUc9EPbJ1bgXDugoDHiLClIjB0C5E4DYJjDAzpBJ2IroXLO6jbrMwHjotmw9K5FycUY6e3Y+y5UHLzF244Gq2FUAATUMHMKYBY1RaGlljKa3F9KHk3OkOu24cQmTfHecyB4rJXEyp+Y8ddJyIVqc5ayzSbIdIaa0vJmghAy31pLEpvIDLlNAHXdIcwqmpRvGZPpllzI0N6d0ph0zjxzHad0py9SVkdM2XXVZLTEI7KsPk4Zgc-bjMmSeCpsz0jo3SAAcXSOAeZ4wan7OWQ5DpazTxHOWCcjpEo5jmRMujcy15QCvOPOAXZTTfndP6QCs5GyQXmXBTC5Y8KOlQoRYMgpx817w26mMUpq4ryLLqYhRpP5lxbh3MxQ51kgWDQmiiq4G5iVCFgT-P+FLQBriagZGlqw5BNUsk1B8LC9ITKMro38tKulNLReyn8jklmoulYeWViFeWXKGV-QVMjhXLjrmK6ZErGF6qta0+V4wqVTLmcq60PydXqrZU0wF5ljnbK1R07lkqfWrD9Uc+1BrrJzCAA)._

## `compose` and `flip`

Let's quickly go over two basic and commonly used functions in functional programming languages such as Haskell.

### `compose`

`compose` combines two functions and outputs a new one.

If you have a function `f :: b -> c` and a function `g :: a -> b`, then you can get a new function `h :: a -> c` by composing `f` and `g`.

In Haskell, it is written tersely as:

```haskell
compose :: (b -> c) -> (a -> b) -> a -> c
compose f g = \x -> f (g x)
```

In Typescript, it looks a bit more verbose:

```ts
const compose =
  <B, C>(f: (b: B) => C) =>
  <A>(g: (a: A) => B) =>
  (a: A): C =>
    f(g(a))

const f = (x: number): string => x.toString()
const g = (b: boolean): number => (b ? 1 : 0)
const h: (b: boolean) => string = compose(f)(g)

assert.deepStrictEqual(h(true), '1')
```

### `flip`

`flip` swaps the first argument and the second argument of a function:

```haskell
flip :: (a -> b -> c) -> b -> a -> c
flip f a b =  f b a
```

In Typescript :-

```ts
const flip =
  <A, B, C>(f: (a: A) => (b: B) => C) =>
  (b: B) =>
  (a: A) =>
    f(a)(b)

const concat = (x: string) => (y: string) => x + y

assert.deepStrictEqual(flip(concat)('left')('right'), 'rightleft')
```

## Swap Any Arguments at Any Level

Here comes the fun part.

If we want to swap the first and the second arguments of any function, we can apply `flip` :-

```ts
const _1234 = (_1: string) => (_2: string) => (_3: string) => (_4: string) => _1 + _2 + _3 + _4

const _2134 = flip(_1234)

assert.deepStrictEqual(_2134('2')('1')('3')('4'), '1234')
```

But what if we want to swap the second `_2` and the third `_3` arguments? We will need to apply `flip` to the _result_ of the outermost function.

### Semantic Editor Combinators

[Semantic editor combinators](http://conal.net/blog/posts/semantic-editor-combinators) ("**SEC**") is a term coined by Conal Elliot. It describes an alternative way to interpret, _inter alia_, `compose` and `flip`.

The `compose` function, under the lens of **SEC**, has the effect of getting the _result_ out of a function, when viewing `compose` as _unary_ instead of _binary_.

The best way to gain some intuitions is to see its effects.

First, let's rename `compose` with a more intuitive label:

```ts
const result = compose
```

Then, to swap the second `_2` and the third `_3` arguments, we can compose `flip` with `result` :-

```ts
const flip2_3: Flip2_3 = result(flip)
const _1324 = flip2_3(_1234)

assert.deepStrictEqual(_1324('1')('3')('2')('4'), '1234')
```

_Note: For readability, I omitted some typings from the code snippets. You can find them in the Typescript Playground link above._

Note that we need to spell out the type `Flip2_3` for Typescript. This is because, unfortunately, Typescript's type system could not infer the type for us (vs Haskell, which does).

One naturally wonders if the third argument `_3` and the fourth argument `_4` could similarly be swapped. Indeed we can, we just need to apply `result` one more time!

```ts
const flip3_4: Flip3_4 = result(result(flip))
const _1243 = flip3_4(_1234)

assert.deepStrictEqual(_1243('1')('2')('4')('3'), '1234')
```

In fact, you can go crazy and create functions that swap arguments at arbitrarily deep nested level:

```ts
const flip6_7: Flip6_7 = result(result(result(result(result(flip)))))
```

One can easily build an arsenal of these functions and export them as a library.

### Application

Back to our original example: to move the argument `discount` to the top, we can do so mechanically:

```ts
const _getPaymentAmount = flip(flip2_3(getPaymentAmount))
const _getPaymentAmountNoDiscount = _getPaymentAmount(0)

assert.deepStrictEqual(_getPaymentAmountNoDiscount(10)(2)(0.2), 24)
```

## Alter any arguments at any level

Let's take a step further.

Instead of swapping arguments, what if we want to _alter_ the arguments? In our example above, we want to convert `taxRate` from `string` to `number` before feeding it to the `getPaymentAmount` function.

First, note that if we apply `flip` to `compose`, the result is `precompose` :-

```haskell
precompose :: (a -> b) -> (b -> c) -> a -> c
precompose = flip compose
```

In Typescript :-

```ts
const precompose = flip(compose) as Precompose
```

As a dual with `result` a.k.a `compose`, we can think of `precompose` as returning the`arguments`

```ts
const argumnts = precompose
```

Again we can create arbitrarily deep level as we want:

```ts
const argumnts2: Argumnts2 = (x) => result(argumnts(x))
const argumnts4: Argumnts4 = (x) => result(result(result(argumnts(x))))

const sum4 = (a: number) => (b: number) => (c: number) => (d: number) => a + b + c + d
const sum4WithString = argumnts4(parseInt)(sum4)

assert.deepStrictEqual(sum4WithString(1)(2)(3)('4'), 10)
```

Note that we changed the fourth argument of the `sum4` function from `number` to `string` by just precomposing with `parseInt`!

## Application

Ok, back to our initial example. The application is mechanical :-

```ts
const _getPaymentAmountWithString = argumnts4(parseFloat)(getPaymentAmount)

assert.deepStrictEqual(_getPaymentAmountWithString(10)(2)(5)('0.2'), 18)
```

The fourth argument `taxRate` now takes a `string` instead of a `number`, as promised.

## Conclusion

We covered a lot of grounds.

With two basic functions `compose` and `flip`, we can generate a bunch of functions that swap and/or alter arguments at any level, for any curried function. In addition, this approach comes with the following benefits:

- **Purely functional**: this means that it comes with all the benefits of FP: referential transparency, testability, conciseness, etc.
- **Generic & mechanical**: as mentioned above, one could write a bunch of these functions and export them as a library. Then, it can be applied anywhere.
- **Universal**: finally, since the underlying mechanics is "just maths", it is applicable to any programming languages, not just Typescript.
