---
title: Interoperability with non functional code using fp-ts
published: true
description:
tags: #functional, #typescript
---

Sometimes you are forced to interoperate with code not written in a functional style, let's see how to deal with it.

## Sentinels

Use case: an API that may fail and returns a special value of the codomain.

Example: `Array.prototype.findIndex`

Solution: `Option`

```ts
import { Option, none, some } from 'fp-ts/lib/Option'

function findIndex<A>(as: Array<A>, predicate: (a: A) => boolean): Option<number> {
  const index = as.findIndex(predicate)
  return index === -1 ? none : some(index)
}
```

## `undefined` and `null`

Use case: an API that may fail and returns `undefined` (or `null`).

Example: `Array.prototype.find`

Solution: `Option`, `fromNullable`

```ts
import { Option, fromNullable } from 'fp-ts/lib/Option'

function find<A>(as: Array<A>, predicate: (a: A) => boolean): Option<A> {
  return fromNullable(as.find(predicate))
}
```

## Exceptions

Use case: an API that may throw.

Example: `JSON.parse`

Solution: `Either`, `tryCatch2v`

```ts
import { Either, tryCatch2v } from 'fp-ts/lib/Either'

function parse(s: string): Either<Error, unknown> {
  return tryCatch2v(() => JSON.parse(s), reason => new Error(String(reason)))
}
```

## Random values

Use case: an API that returns a non deterministic value.

Example: `Math.random`

Solution: `IO`

```ts
import { IO } from 'fp-ts/lib/IO'

const random: IO<number> = new IO(() => Math.random())
```

## Synchronous side effects

Use case: an API that reads and/or writes to a global state.

Example: `localStorage.getItem`

Solution: `IO`

```ts
import { Option, fromNullable } from 'fp-ts/lib/Option'
import { IO } from 'fp-ts/lib/IO'

function getItem(key: string): IO<Option<string>> {
  return new IO(() => fromNullable(localStorage.getItem(key)))
}
```

Use case: an API that reads and/or writes to a global state and may throw.

Example: `readFileSync`

Solution: `IOEither`, `tryCatch2v`

```ts
import * as fs from 'fs'
import { IOEither, tryCatch2v } from 'fp-ts/lib/IOEither'

function readFileSync(path: string): IOEither<Error, string> {
  return tryCatch2v(() => fs.readFileSync(path, 'utf8'), reason => new Error(String(reason)))
}
```

## Asynchronous side effects

Use case: an API that performs an asynchronous computation.

Example: reading from standard input

Solution: `Task`

```ts
import { createInterface } from 'readline'
import { Task } from 'fp-ts/lib/Task'

const read: Task<string> = new Task(
  () =>
    new Promise<string>(resolve => {
      const rl = createInterface({
        input: process.stdin,
        output: process.stdout
      })
      rl.question('', answer => {
        rl.close()
        resolve(answer)
      })
    })
)
```

Use case: an API that performs an asynchronous computation and may reject.

Example: `fetch`

Solution: `TaskEither`, `tryCatch`

```ts
import { TaskEither, tryCatch } from 'fp-ts/lib/TaskEither'

function get(url: string): TaskEither<Error, string> {
  return tryCatch(() => fetch(url).then(res => res.text()), reason => new Error(String(reason)))
}
```

## Playground

Check out [this repo](https://github.com/tychota/fp-ts-playground) by [Tycho Tatitscheff](https://twitter.com/TychoTa) containing the source code and tests.
