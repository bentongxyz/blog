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
type Flip2_3 = <A, B, C, D>(f: (a: A) => (b: B) => (c: C) => D) => (a: A) => (c: C) => (b: B) => D

const flip2_3: Flip2_3 = result(flip)

const _1324 = flip2_3(_1234)
assert.deepStrictEqual(_1324('1')('3')('2')('4'), '1234')
```

Note that we need to spell out the type `Flip2_3` for Typescript. This is because, unfortunately, Typescript's type system could not infer the type for us (vs Haskell, which does).

One naturally wonders if the third argument `_3` and the fourth argument `_4` could similarly be swapped. Indeed we can, we just need to apply `result` one more time!

```ts
type Flip3_4 = <A, B, C, D, E>(
  f: (a: A) => (b: B) => (c: C) => (d: D) => E
) => (a: A) => (b: B) => (d: D) => (c: C) => E

const flip3_4: Flip3_4 = result(result(flip))
const _1243 = flip3_4(_1234)
assert.deepStrictEqual(_1243('1')('2')('4')('3'), '1234')
```

In fact, you can go crazy and create functions that swap arguments at arbitrarily deepl nested level:

```ts
type Flip6_7 = <A, B, C, D, E, F, G, H>(
  f: (a: A) => (b: B) => (c: C) => (d: D) => (e: E) => (f: F) => (g: G) => H
) => (a: A) => (b: B) => (c: C) => (d: D) => (e: E) => (g: G) => (f: F) => H
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
type Precompose = <A, B>(g: (a: A) => B) => <C>(f: (b: B) => C) => (a: A) => C
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

Ok, back to our example. The application is mechanical :-

```ts
const _getPaymentAmountWithString = argumnts4(parseFloat)(getPaymentAmount)
assert.deepStrictEqual(_getPaymentAmountWithString(10)(2)(5)('0.2'), 18)
```

THe fourth argument `taxRate` now takes a `string` instead of a `number`, as promised.

## Conclusion

We covered a lot of grounds.

With two basic functions `compose` and `flip`, we can generate a bunch of functions that swap and/or alter arguments at any level, for any curried function. In addition, this approach comes with the following benefits:

- **Purely functional**: this means that it comes with all the benefits of FP: referential transparency, testability, conciseness etc.
- **Generic & mechanical**: as mentioned above, one could write a bunch of these functions and export them as a library. Then, it can be applied anywhere.
- **Universal**: finally, since the underlying mechanics is "just maths", it is applicable to any programming languages, not just Typescript.
