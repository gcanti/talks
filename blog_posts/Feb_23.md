---
title: Functional design: how to make the `time` combinator more general
published: true
description:
tags: functional, typescript
series: Functional design
---

In the [last article](https://dev.to/gcanti/functional-design-combinators-14pn) I wrote a `time` combinator which mimics the analogous Unix command: given an action `IO<A>`, we can derive an action `IO<A>` that prints to the console the elapsed time

```ts
import { IO } from 'fp-ts/lib/IO'
import { now } from 'fp-ts/lib/Date'
import { log } from 'fp-ts/lib/Console'

export function time<A>(ma: IO<A>): IO<A> {
  return now.chain(start =>
    ma.chain(a =>
      now.chain(end =>
        log(`Elapsed: ${end - start}`).map(() => a)
      )
    )
  )
}
```

There are two problems with this combinator though:

- is not flexible, i.e. consumers can't choose what to do with the elapsed time
- works with `IO` only

In this article we'll tackle the first problem.

## Adding flexibility by returning the elapsed time

Instead of always logging, we can return the elapsed time along with the computed value

```ts
export function time<A>(ma: IO<A>): IO<[A, number]> {
  return now.chain(start =>
    ma.chain(a =>
      now.map<[A, number]>(end => [a, end - start])
    )
  )
}
```

Now a user can choose what to do with the elapsed time by defining its own combinators.

We could still log to the console...

```ts
export function withLogging<A>(ma: IO<A>): IO<A> {
  return time(ma).chain(([a, millis]) =>
    log(`Result: ${a}, Elapsed: ${millis}`).map(() => a)
  )
}
```

Usage

```ts
import { randomInt } from 'fp-ts/lib/Random'

function fib(n: number): number {
  return n <= 1 ? 1 : fib(n - 1) + fib(n - 2)
}

const program = withLogging(randomInt(30, 35).map(fib))

program.run()
/*
Result: 14930352, Elapsed: 127
*/
```

...or just ignore the elapsed time...

```ts
export function ignoreSnd<A>(ma: IO<[A, unknown]>): IO<A> {
  return ma.map(([a]) => a)
}
```

...or, for example, only keep the fastest of a non empty list of actions

```ts
import {
  Semigroup,
  fold,
  getMeetSemigroup
} from 'fp-ts/lib/Semigroup'
import { contramap, ordNumber } from 'fp-ts/lib/Ord'
import { getSemigroup } from 'fp-ts/lib/IO'

export function fastest<A>(
  head: IO<A>,
  tail: Array<IO<A>>
): IO<A> {
  const ordTuple: Ord<[A, number]> = contramap(
    ([_, elapsed]) => elapsed,
    ordNumber
  )
  const semigroupTuple = getMeetSemigroup(ordTuple)
  const semigroupIO = getSemigroup(semigroupTuple)
  const fastest = fold(semigroupIO)(time(head))(
    tail.map(time)
  )
  return ignoreSnd(fastest)
}
```

Usage

```ts
fastest(program, [program, program])
  .chain(a => log(`Fastest result is: ${a}`))
  .run()
/*
Result: 5702887, Elapsed: 49
Result: 2178309, Elapsed: 20
Result: 5702887, Elapsed: 57
Fastest result is: 2178309
*/
```

In the [next article](https://dev.to/gcanti/functional-design-tagless-final-332k) we'll tackle the second problem by introducing a powerful style of programming: tagless final.

## Appendix

The implementation of `fastest` is quite dense, let's see the relevant bits:

1. its signature ensures that we provide a non empty list of actions

```ts
//  at least one action --v            v--- possibly other actions
function fastest<A>(head: IO<A>, tail: Array<IO<A>>): IO<A>
```

2. `contramap` is an `Ord` combinator: given an instance of `Ord` for `T` and a function from `U` to `T`, we can derive an instance of `Ord` for `U`

Here `T = number` and `U = [A, number]`

```ts
// from `Ord<number>` to `Ord<[A, number]>`
const ordTuple = contramap(
  ([_, elapsed]) => elapsed,
  ordNumber
)
```

3. `getMeetSemigroup` transforms an instance of `Ord<T>` into an instance of `Semigroup<T>` which, when combining two values, returns the smaller.

```ts
// from `Ord<[A, number]>` to `Semigroup<[A, number]>`
const semigroupTuple = getMeetSemigroup(ordTuple)
```

4. `getSemigroup` is a `Semigroup` combinator: given an instance of `Semigroup` for `T`, we can derive an instance of `Semigroup` for `IO<T>`

```ts
// from `Semigroup<[A, number]>` to `Semigroup<IO<[A, number]>>`
const semigroupIO = getSemigroup(semigroupTuple)
```

5. `fold` reduces a non empty list of actions using the provided `Semigroup`

```ts
// from a non empty list of `IO<[A, number]>` to `IO<[A, number]>`
const fastest: IO<[A, number]> = fold(semigroupIO)(
  time(head)
)(tail.map(time))
```

6. Finally we ignore the elapsed time and return just the value

```ts
// from `IO<[A, number]>` to `IO<A>`
return ignoreSnd(fastest)
```
