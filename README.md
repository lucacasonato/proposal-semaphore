## Semaphores for JavaScript

Stage: 0

Champion: Luca Casonato (@lucacasonato)

Author: Luca Casonato (@lucacasonato)

This is a proposal for adding semaphores to JavaScript.

### Motivation

Semaphores are a synchronization primitive that can be used to control access to
a shared resource. In many languages, these are used to coordinate access to
shared resources across multiple threads. In JavaScript, this is less common -
but you often want to coordinate access to a shared resource across multiple
asynchronous operations on the same thread.

For example, you might want to limit the number of concurrent HTTP requests
you're making to a server, or the number of concurrent disk writes you're
performing. You might want to ensure that only one operation is reading from a
file at a time, or that only one operation is writing to a file at a time.

### Prior Art

Most languages have a semaphore construct.

- In C#, the
  [`Semaphore` class](https://learn.microsoft.com/en-us/dotnet/api/system.threading.semaphore)
  is used to control access to a shared resource.
- In Python, the
  [`threading.Semaphore`](https://docs.python.org/3/library/threading.html#semaphore-objects)
  class is used to control access to a shared resource.
- In Java, the
  [`Semaphore` class](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Semaphore.html)
  is used to control access to a shared resource.
- In Rust, the
  [`tokio::sync::Semaphore`](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html)
  struct is used to control access to a shared resource.

There is a good mix here between synchronous and asynchronous APIs - Java,
Python, and C# all provide synchronous APIs that block the current thread until
a token is available, while Rust provides an asynchronous API that returns a
future that resolves when a token is available.

There are multiple NPM packages that provide a semaphore implementation for
JavaScript.

- The [`semaphore`](https://www.npmjs.com/package/semaphore) package provides a
  callback based semaphore API.
- The [`@shopify/semaphore`](https://www.npmjs.com/package/@shopify/semaphore)
  package provides a promise based semaphore API that looks very similar to the
  one proposed here. The main difference is that `release()` returns a promise
  that resolves when the token is re-acquired - I have not seen this pattern in
  other semaphore implementations, so it is not included in this proposal.

### Proposal

This proposal adds a `Semaphore` class to JavaScript. A `Semaphore` has a
`limit` property, which is the maximum number of "tokens" that can be acquired
from the semaphore at once.

A `Semaphore` has an `acquire()` method. This method returns a promise that
resolves when a token is available from the semaphore. If the semaphore is at
its limit, the promise will not resolve until a token is available. The
`acquire()` method returns a `Sempahore.Permit` object.

The `Sempahore.Permit` object has a synchronous `release()` method, which
releases the token back to the semaphore. Additionally, the `Sempahore.Permit`
object also implements the disposable protocol by having a `[Symbol.dispose]`
method (which is an alias for `release()`).

```js
const semaphore = new Semaphore(5);

async function doWork() {
  const guard = await semaphore.acquire();
  // Do some work
  guard.release();
}
```

The `Semaphore` class also has a `with()` method, which is a convenience method
for acquiring a token, doing some work, and then releasing the token.

```js
const semaphore = new Semaphore(5);

async function doWork() {
  await semaphore.with(async () => {
    // Do some work
  });
}
```

```ts
interface Semaphore {
  new (limit: number): Semaphore;

  limit: number;
  acquire(): Promise<Semaphore.Permit>;
  with<T>(fn: () => Promise<T>): Promise<T>;
}

namespace Semaphore {
  interface Permit {
    release(): void;
    [Symbol.dispose](): void;
  }
}
```

### Open Questions

#### Should an explicit `acquire()` and `release()` mechanism be provided?

An explicit `acquire()` and `release()` mechanism is more flexible, but it has
the footgun of a user forgetting to release the token. This can lead to
deadlocks if the `release()` method is not called.

This also poses the question of whether if `Semaphore.Permit` is GC'd, should
the token be released back to the semaphore? This would prevent deadlocks, but
it would also make it harder to reason about the code - and it would expose GC
in a very prominent way.

#### Should the `Semaphore` have a way to query the number of tokens available?

Many languages provide a way to query the number of tokens available in a
semaphore. This can be useful for debugging purposes, but it can also be useful
for making decisions based on the number of tokens available. For example, if
you have a best effort tracing system, you might want to only trace a request if
there are tokens available in the tracing semaphore.

#### Should the `Semaphore` have a way to "try acquire" a token, returning `null` if the semaphore is at its limit?

Motivation for this is similar to the previous question - it can be useful for
making decisions based on the number of tokens available.

#### Should the `Semaphore` be sharable across agents in an agent cluster to allow for coordination across multiple agents (threads)?

This would enable simple cross agent coordination. On the web platform, this
means coordination between web workers.

If so, this would only be allowed on the web platform when shared memory is
available.

#### Should there be a way to cancel an acquisition or `with()` block?

This would allow for a way to cancel an acquisition if the reason for
acquisition is no longer valid. For example, a timeout could be implemented by
cancelling the acquisition if the timeout is reached.

This is not strictly necessary, as one can just immediately dispose the permit
object when `acquire()` resolves and the task is no longer needed, or by
immediately disposing the permit object in the `with()` block.

#### Should there be a synchronous `acquireSync()` method?

This would allow for synchronous code to acquire a token from the semaphore.
This may be useful for Wasm code that is compiled from native code that uses
semaphores.

On the web platform, like `Atomics.wait` this would not be allowed on the main
thread.

#### Should there be a way to acquire multiple tokens at once?

Many languages provide a way to acquire multiple tokens at once. This can be
useful when not all operations have the same cost. For example, you might want
to acquire 1 tokens for a read operation and 4 tokens for a write operation.

#### Name bikeshedding

- `Semaphore` vs `Atomics.Semaphore`
- `Semaphore.Permit` vs `Semaphore.Token` vs `Semaphore.Guard`
