---
title: Getting started with fp-ts: Either vs Validation
published: true
description:
tags: functional, typescript
---

# The problem

Say you must implement a web form to signup for an account. The form contains two field: `username` and `password` and the following validation rules must hold:

- `username` must not be empty
- `username` can't have dashes in it
- `password` needs to have at least 6 characters
- `password` needs to have at least one capital letter
- `password` needs to have at least one number

# Either

The `Either<L, A>` type represents a computation that might fail with an error of type `L` or succeed with a value of type `A`, so is a good candidate for implementing our validation rules.

For example let's encode each `password` rule

```ts
import { Either, left, right } from 'fp-ts/lib/Either'

const minLength = (s: string): Either<string, string> =>
  s.length >= 6 ? right(s) : left('at least 6 characters')

const oneCapital = (s: string): Either<string, string> =>
  /[A-Z]/g.test(s)
    ? right(s)
    : left('at least one capital letter')

const oneNumber = (s: string): Either<string, string> =>
  /[0-9]/g.test(s) ? right(s) : left('at least one number')
```

We can chain all the rules using... `chain`

```ts
const validatePassword = (
  s: string
): Either<string, string> =>
  minLength(s)
    .chain(oneCapital)
    .chain(oneNumber)
```

Because we are using `Either` the checks are **fail-fast**. That is, any failed check shortcircuits subsequent checks so we will only ever get one error.

```ts
console.log(validatePassword('ab'))
// => left("at least 6 characters")

console.log(validatePassword('abcdef'))
// => left("at least one capital letter")

console.log(validatePassword('Abcdef'))
// => left("at least one number")
```

However this could lead to a bad UX, it would be nice to have all of these errors be reported simultaneously.

The `Validation` type may help here.

# Validation

The `Validation<L, A>` type is much like `Either<L, A>`, it represents a computation that might fail with an error of type `L` or succeed with a value of type `A`, but as opposed to `Either` is able to **collect multiple failures**.

In order to do that we must tell `Validation` how to **combine two values** of type `L`.

That's what [Semigroup](https://dev.to/gcanti/getting-started-with-fp-ts-semigroup-2mf7) is all about: combining two values of the same type.

For example we can pack the errors into a list.

The `'fp-ts/lib/Validation'` module provides a `getApplicative` function that, given a semigroup, returns an [Applicative](https://dev.to/gcanti/getting-started-with-fp-ts-applicative-1kb3) instance for `Validation`

```ts
import { getArraySemigroup } from 'fp-ts/lib/Semigroup'
import { getApplicative } from 'fp-ts/lib/Validation'

const applicativeValidation = getApplicative(
  getArraySemigroup<string>()
)
```

However in order to use `applicativeValidation` we must first redefine all the rules so that they return a value of type `Validation<Array<string>, string>`.

Instead of rewriting all the previous functions, which is cumbersome, let's define a [combinator](https://dev.to/gcanti/functional-design-combinators-14pn) that converts a check outputting an `Either<L, A>` into a check outputting a `Validation<Array<L>, A>`

```ts
import {
  Validation,
  fromEither
} from 'fp-ts/lib/Validation'

function toValidation<L, A>(
  check: (a: A) => Either<L, A>
): (a: A) => Validation<Array<L>, A> {
  return a => fromEither(check(a).mapLeft(l => [l]))
}

const minLengthV = toValidation(minLength)
const oneCapitalV = toValidation(oneCapital)
const oneNumberV = toValidation(oneNumber)
```

Let's put all together, I'm going to use the `sequenceT` helper which takes `n` actions and does them from left-to-right, returning the resulting tuple

```ts
import { sequenceT } from 'fp-ts/lib/Apply'

function validatePasswordV(
  s: string
): Validation<Array<string>, string> {
  return sequenceT(applicativeValidation)(
    minLengthV(s),
    oneCapitalV(s),
    oneNumberV(s)
  ).map(() => s) // we can discard the resulting tuple and just return the input here
}

console.log(validatePasswordV('ab'))
// => failure(["at least 6 characters", "at least one capital letter", "at least one number"])
```

# Appendix

Note that the `sequenceT` helper is able to handle actions with different types:

```ts
import {
  Validation,
  success,
  failure
} from 'fp-ts/lib/Validation'

interface Person {
  name: string
  age: number
}

// Person constructor
const toPerson = ([name, age]: [
  string,
  number
]): Person => ({
  name,
  age
})

const validateName = (
  s: string
): Validation<Array<string>, string> =>
  s.length === 0 ? failure(['Invalid name']) : success(s)

const validateAge = (
  s: string
): Validation<Array<string>, number> =>
  isNaN(+s) ? failure(['Invalid age']) : success(+s)

function validatePerson(
  name: string,
  age: string
): Validation<Array<string>, Person> {
  return sequenceT(applicativeValidation)(
    validateName(name),
    validateAge(age)
  ).map(toPerson)
}
```
