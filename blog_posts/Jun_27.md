---
title: Type holes in TypeScript
published: true
description:
tags: functional, typescript
---

Sandy Maguire recently wrote a nice article about type holes ([Implement With Types, Not Your Brain!](https://reasonablypolymorphic.com/blog/typeholes/index.html)), this blog post is a porting to TypeScript.

## What's a type hole?

> The idea is to implement the tiny part of a function that you know how to do, and then ask the compiler for help on the rest of it. It's an iterative process. It's a discussion with the compiler. Each step of the way, you get a little closer to the right answer, and after enough iterations your function has written itself — even if you're not entirely sure _how_.

TypeScript doesn't support type holes but they can be kind of simulated.

# A first example

Let's see the first example from the article

```ts
declare function jonk<A, B>(
  ab: (a: A) => B,
  ann: (an: (a: A) => number) => number
): (bn: (b: B) => number) => number
```

In order to simulate a type hole I'm going to use the following function declaration

```ts
declare function _<T>(): T
```

Let's put it into the `jonk`'s body

```ts
function jonk<A, B>(
  ab: (a: A) => B,
  ann: (an: (a: A) => number) => number
): (bn: (b: B) => number) => number {
  return _()
}
```

If you move the mouse over the "type hole" `_` you can see what TypeScript infers for its type parameter `T`

```
(bn: (b: B) => number) => number
```

So what is the type checker telling us? Two things

- The expression we want to replace `_()` with must have type `(bn: (b: B) => number) => number`.
- We have some local binds (`ab`, `ann`, `jonk`, and their types) that we can use to help with the implementation.

Since our hole has type `(bn: (b: B) => number) => number` we should bind the `bn` in a lambda (I'll just write the function body from now on)

```ts
return bn => _() // inferred type: number
```

What's the new inferred type? `number`. How can we produce a `number`? We can use `ab`, `ann` or `bn`. Since both `ann` and `bn` return a `number` let's choose `ann` as a guess

```ts
return bn => ann(_()) // inferred type: (a: A) => number
```

Our new hole has a function type, so let's introduce a lambda

```ts
return bn => ann(a => _()) // inferred type: number
```

We need to produce a `number` again, let's choose `bn` this time

```ts
return bn => ann(a => bn(_())) // inferred type: B
```

Now we need to produce a `B`. We have a function that can do that, `ab: (a: A) => B`

```ts
return bn => ann(a => bn(ab(_()))) // inferred type: A
```

Finally, we have a hole whose type is `A`. Since we have an `A` (the `a` parameter) let's just use that

```ts
function jonk<A, B>(
  ab: (a: A) => B,
  ann: (an: (a: A) => number) => number
): (bnn: (b: B) => number) => number {
  return bn => ann(a => bn(ab(a)))
}
```

Now we have a complete implementation, almost driven by the type checker.

# The second example

Let's tackle the second example of the blog post: `zoop`

```ts
declare function zoop<A, B>(
  abb: (a: A) => (b: B) => B,
  b: B,
  as: Array<A>
): B
```

We notice that `as` has type `Array<A>`, let's "pattern match" on that using `foldLeft`

```ts
import { foldLeft } from 'fp-ts/lib/Array'
import { pipe } from 'fp-ts/lib/pipeable'

function zoop<A, B>(
  abb: (a: A) => (b: B) => B,
  b: B,
  as: Array<A>
): B {
  return pipe(
    as,
    foldLeft(
      () => _(), // inferred type: B
      (head, tail) => _() // inferred type: B
    )
  )
}
```

We need to produce a `B` for the "nil" case (i.e. when the array is empty). Since we have a `B` let's just use that (I'll just write the function body from now on)

```ts
return pipe(
  as,
  foldLeft(
    () => b,
    (head, tail) => _() // inferred type: B
  )
)
```

Again we want to produce a `B` for the other case and we want to call `abb`. Since it takes two arguments, let's give it two holes

```ts
return pipe(
  as,
  foldLeft(
    () => b,
    (head, tail) =>
      abb(
        _() // inferred type: A
      )(
        _() // inferred type: B
      )
  )
)
```

`head` has type `A` so let's use it

```ts
return pipe(
  as,
  foldLeft(
    () => b,
    (head, tail) =>
      abb(head)(
        _() // inferred type: B
      )
  )
)
```

Now we must to produce a `B` and we want to use `tail` which has type `Array<A>`. Our only option is using `zoop` itself

```ts
function zoop<A, B>(
  abb: (a: A) => (b: B) => B,
  b: B,
  as: Array<A>
): B {
  return pipe(
    as,
    foldLeft(
      () => b,
      (head, tail) => abb(head)(zoop(abb, b, tail))
    )
  )
}

// p.s. `zoop` is `reduceRight`
```

> The reason why this works is known as theorems for free, which roughly states that we can infer lots of facts about a type signature (assuming it's correct.)
