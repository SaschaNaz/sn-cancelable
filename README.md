# cancelable [![Travis-CI build result](https://travis-ci.org/SaschaNaz/sn-cancelable.svg?branch=master)](https://travis-ci.org/SaschaNaz/sn-cancelable)
This proposal tries replacing cancellation token from [cancelable-promise](https://github.com/domenic/cancelable-promise) with cancellation chain object, to achieve more automatic cancellation propagation.

###### TODO
- `chain` keyword may be able to be confused with normal promise `then` chain.

## Differences from [cancelable-promise](https://github.com/domenic/cancelable-promise)

- `Promise.cancelable(async (chain) => { /* ... */ });`
- `promise[@@cancel]`

New `CancelableChain` object is passed to `Promise.cancelable` input callback. This object can store other promises and cancel them when its underlying promise gets canceled. Its constructor is exposed to make a standalone chain instead of Promise dependant one.

```ts
interface CancelableChain {
  constructor(): CancelableChain;

  /*
   * `chain()` stores objects that supports `@@cancel` and call it
   * if cancellation is requested.
   */
  <T>(cancelable: CancelablePromise<T>): Promise<T>;
  (cancelable: Cancelable): Promise<void>;

  cancel(): void; // same as current `cancel` parameter to shorten the parameter list
  
  canceled: boolean; // true when underlying promise is canceled
  tillCanceled: Promise<void>; // resolves when underlying promise gets canceled
  /*
   * throws CancelError when underlying promise gets canceled, otherwise returns nothing
   */
  throwIfCanceled: void;
  
  [[chainedList]]: Cancelable[]; // stored cancelables.
}

// A cancelable is an object that supports Symbol.cancel.
interface Cancelable {
  [@@cancel](): void;
}
interface Promise<T> extends Cancelable {}
```

## Use

```js
function inner() {
  return Promise.cancelable(async (chain) => {
    await a();
    chain.throwIfCanceled();
    await b();
  });
}

function outer() {
  return Promise.cancelable(async (chain) => {
    await chain(inner()); // cancels inner() when parent promise gets canceled
  });
}
```

```js
function inner() {
  return Promise.cancelable(async (chain) => {
    await a();
    if (!chain.canceled) {
      await b();
    }
  });
}
```

```js
function inner() {
  return Promise.cancelable(async (chain) => {
    const state = { foo: false }
    chain.tillCanceled.then(() => state.foo = true);
    await c(state);
  });
}
```

## Syntax sugar

A `cancelable function` has a new `chain` keyword in its function scope.

```js
cancelable function inner() {
  await a();
  chain.throwIfCanceled(); // chain as keyword, a form like `new.target`
  await b();
}

cancelable function outer() {
  chain inner(); // store inner() promise to cancellation chain
}
```

```js
cancelable function inner() {
  await a();
  if (!chain.canceled) { // chain as keyword
    await b();
  }
}
```

## Token style to chain style

`chain(promise)` returns input promise so that promise.then can happen after chaining.

```js
let chain = new CancelableChain();
chain(fetch())
  .then(() => chain(process()));

chain.cancel();
```

This example with `.then()` works basically same as the following example.

```js
let chain = new CancelableChain();
chain(fetchAndProcess());

chain.cancel();

cancelable function fetchAndProcess() {
  chain fetch();
  chain process();
}
```
