---
title: "Let's Implement RxJS Observables - Part 1"
date: 2021-06-21T09:40:00+03:00
draft: false
description: Implementing a completely functional Observable from first principles. A step-by-step guide of the ideas inside.
---

# Let's Implement RxJS Observables - Part 1

## Why?

> "If you can't explain it simply, you don't understand it well enough"
_- Albert Einstein_

Or, in our case, if we can't implement it, we don't understand it well enough. So let's try to.

## What do we want?

We want to create an object like `Promise`, but which can produce more than one value. Promises are perfect for modeling *single-result* asynchronous operations. However, they cannot model *multiple-result* operations. _You can model a `setTimeout` with a promise, but you cannot model `setInterval` with a promise._

When would we want multiple-result operations? For example, with a promise, you can load data for a React component once. After that, you'd want to create a new promise (e.g. with a `useEffect` hook), to restart the operation. With an observable, you can construct one observable object and use that for the whole lifetime of a component.

## The `returnThreeValues` function

Let's say we want to create a function which returns 3 values with one second intervals between them:

```ts
const observable = returnThreeValues();
```

How would we wait for the values? Well, with a promise we could use `then` (`promise.then(singleValue => ...)`). Let's do something similar but call it `subscribe`:

```ts
const observable = returnThreeValues();

observable.subscribe(number => console.log(number));
```

Now we can try to implement `returnThreeValues`:

```ts
function returnThreeValues() {
  const subscribers = [];

  subscribers.forEach(next => next(1));

  setTimeout(() => {
    subscribers.forEach(next => next(2));
  }, 1000);

  setTimeout(() => {
    subscribers.forEach(next => next(3));
  }, 2000);

  return {
    subscribe: (returnNextValue) => {
      subscribers.push(returnNextValue);
    }
  };
}

const observable = returnThreeValues();

observable.subscribe(number => console.log(number));
// Produces:
// 2
// 3
```

The above code misses the first value because it comes before we are able to subscribe to the stream. We start the work as soon as we create the observable, before we have a chance to subscribe.

## It is lazy

How does a promise solve this problem? It does by buffering the value. Whenever a promise is fulfilled, the value is **stored** inside the promise. Can we do the same with our observable - keep *all the values* in case any late subscribers come? Sure, but that would potentially require a lot of memory. Imagine an observable that produces 1GB of data when created - it would have to buffer this forever.

Luckily, we can solve the problem another way. What if instead of buffering values, we just started the work when we subscribe? This way it would be impossible for the subscribe to miss anything. Let's just move the logic inside of the `subscribe`:

```ts
function returnThreeValues() {
  return {
    subscribe: (next) => {
      next(1);

      setTimeout(() => next(2), 1000);
      setTimeout(() => next(3), 2000);
    }
  };
}

const observable = returnThreeValues();

observable.subscribe(number => console.log(number));
// Produces:
// 1
// 2
// 3
```

The code also became much simpler as we don't have to maintain a list of subscribers anymore.

As a side-effect of the above, our observable became lazy - if nobody subscribes to it, then it never does any work (it never calls `setTimeout`). Another side-effect is that if we have multiple subscribers, they start separate instances of the same work:

```ts
function returnThreeValues() {
  return {
    subscribe: (next) => {
      console.log('started doing work');

      next(1);

      setTimeout(() => next(2), 1000);
      setTimeout(() => next(3), 2000);
    }
  };
}

const observable = returnThreeValues();

observable.subscribe(number => console.log('first: ', number));
// stared doing work
// first: 1

observable.subscribe(number => console.log('second: ', number));
// stared doing work
// second: 1

// ...1 second later...

// first: 2
// second: 2

// ...1 second later...

// first: 3
// second: 3
```

## It is a recipe for how to do the operation, it does not *do* the operation

The side-effect described above is actually useful. A function that returns an Observable is like an action factory - it prepares the operation but does not start it - this is done by the returned function. It is essentially the same as this:

```ts
function buildOperation(n) {
  return () => {
    for (let i = 0; i < n; i++) {
      console.log(i + 1);
    }
  };
}

const log3Times = buildOperation(3);

// No logs yet

log3Times();
// 1, 2, 3

log3Times();
// 1, 2, 3
```

A function producing an observable is the same, except the function to perform the operation is called `subscribe`:

```ts
function buildOperation(n) {
  return {
    subscribe: () => {
      for (let i = 0; i < n; i++) {
        console.log(i + 1);
      }
    }
  };
}

const log3Times = buildOperation(3);

// No logs yet

log3Times.subscribe();
// 1, 2, 3

log3Times.subscribe();
// 1, 2, 3
```

That's why an observable is often described to be like a function - because it is a function. When you `subscribe`, you *call* the function that somebody has prepared. This function then does work and produces values.

## Where does it end?

Good question. Let's get back to a previous example:

```ts
function returnThreeValues() {
  return {
    subscribe: (next) => {
      next(1);

      setTimeout(() => next(2), 1000);
      setTimeout(() => next(3), 2000);
    }
  };
}

const observable = returnThreeValues();

observable.subscribe(number => console.log(number));
// 1
// 2
// 3
```

How do we know when the observable stops producing values? With the promise it was simple - we get one value, then it is done. If we don't know the number of values to wait for, we need some way for the operation (the observable) to tell us.

An obvious choice is to just provide another callback:

```ts
function returnThreeValues() {
  return {
    subscribe: (next, complete) => {
      next(1);

      setTimeout(() => next(2), 1000);
      setTimeout(() => {
        next(3);
        complete();
      }, 2000);
    }
  };
}

const observable = returnThreeValues();

observable.subscribe(
  number => console.log(number),
  () => console.log('Done. No more values!')
);
// 1
// 2
// 3
// Done. No more values!
```

We just added another callback and made sure to call it when we decide we won't produce any more values.

## What if something breaks?

With promises, we have a way to signal that the async operation failed. We can do the same here with another callback:

```ts
function returnThreeValues() {
  return {
    subscribe: (next, error, complete) => {
      next(1);

      setTimeout(() => next(2), 1000);
      setTimeout(() => {
        try {
          doWorkThatCanFail();

          next(3);
          complete();
        } catch (e) {
          error(e);
        }
      }, 2000);
    }
  };
}

const observable = returnThreeValues();

observable.subscribe(
  number => console.log(number),
  error => console.log('Work failed. No more values!')
  () => console.log('Done. No more values!')
);
// 1
// 2
// Work failed. No more values!
```

What happens after an error, though? Can we continue emitting values? Do we have to "complete"? How do we know we've recovered from the error?

To make everything simpler, let's just say that after an error there can be no more values. An since there can be no more values, we don't need to call the `complete` callback. This is what rxjs does as well.

## Extracting `Observable` from `returnThreeValues`

We've created a way for the function `returnThreeValues` to return more than one value asynchronously. Can we separate this mechanism and make it more generic? Remember how promises are created manually?

```ts
const thisIsAPromise = new Promise((resolve, reject) => {
  // code that calls either resolve or reject
});
```

We can do something very similar for our observable:

```ts
const thisIsAnObservable = new Observable(({ next, error, complete }) => {
  // code that calls next, then error or complete
});
```

The only difference is the number of callbacks we pass to the constructor function. With this, we can write the `returnThreeValues` as follows:

```ts
function returnThreeValues() {
  return new Observable(({ next, error, complete }) => {
    next(1);

    setTimeout(() => next(2), 1000);
    setTimeout(() => {
      try {
        doWorkThatCanFail();

        next(3);
        complete();
      } catch (e) {
        error(e);
      }
    }, 2000);
  });
}
```

How would we implement the `Observable` constructor then? We just write this:

```ts
function Observable(producer) {
  return {
    subscribe: (next, error, complete) => {
      producer({ next, error, complete });
    }
  };
}
```

Compare this to a very minimal implementation of a promise:

```ts
function Promise(producer) {
  let state = 'waiting';
  let value = undefined;

  const subscribers = [];

  producer(resolveValue => {
    state = 'resolved';
    value = resolveValue;

    subscribers.forEach(sub => sub.complete(value));
  }, rejectValue => {
    state = 'rejected';
    value = rejectValue;

    subscribers.forEach(sub => sub.error(value));
  });

  return {
    then: (complete, error) => {
      if (state === 'resolved') {
        complete(value);
      } else if (state === 'rejected') {
        error(value);
      } else {
        subscribers.push({ complete, error });
      }
    }
  }
}
```

The promise's implementation is much more complex. It has to manage state. An observable does not store state - it just has to pass an observer (consumer, subscriber) to a producer. Pure and functional.

You will sometimes see the following definition for an observable: "An observable is a function tying a producer to an observer." This is what it means.

## What if it doesn't end? How do we stop listening?

Remember that we allow our producer to produce any number of values. We can even create an observable that never stops producing values:

```ts
function interval(timeBetweenValues) {
  return new Observable(subscriber => {
    let counter = 1;

    setInterval(() => {
      subscriber.next(counter++);
    }, timeBetweenValues);
  });
}
```

We don't ever call `complete` or `error` because we don't ever stop emitting values. How would we use this?

```ts
const observable = interval(1000);

observable.subscribe(count => console.log(count));

// 1
// ...1 second passes...
// 2
// ...1 second passes...
// 3
// ...1 second passes...
// 4
// ...
// ...
// ...
// 1000
// ...
// 999999999999
// ...
```

This code will never complete. Once we've subscribed, we need to wait for all the values, even if that means waiting forever.

Can we stop listening somehow? We need a way to "unsubscribe". We need to be able to do this:

```ts
const observable = interval(1000);
const { unsubscribe } = observable.subscribe(count => console.log(count));

setTimeout(() => {
  console.log('done listening for values');
  unsubscribe();
}, 5000);

// 1
// 2
// 3
// 4
// 5
// done listening for values
```

Let's implement it then. The subscribe function needs to return an object with an `unsubscribe` function:

```ts
function Observable(producer) {
  return {
    subscribe: (next, error, complete) => {
      let subscriber = { next, error, complete };

      producer({
        next: value => subscriber && subscriber.next(value),
        error: error => subscriber && subscriber.error(error),
        complete: () => subscriber && subscriber.complete(),
      });

      return {
        unsubscribe: () => {
          subscriber = undefined;
        }
      };
    }
  };
}
```

This way when we call `unsubscribe` we are no longer notified of any new values that are produced. We just stop listening. However, the work continues to be performed, even if nobody is listening.

We need to stop the `setInterval` we started. We need a way to "tell" the producer that we don't need any more values, so that it can call `clearInterval`.

What if the producer tells the observable how its operation can be cleaned up? It can do that by returning a cleanup function:

```ts
function interval(timeBetweenValues) {
  return new Observable(subscriber => {
    let counter = 1;

    const interval = setInterval(() => {
      subscriber.next(counter++);
    }, timeBetweenValues);

    // This will be called when values are no longer needed (on unsubscribe)
    return () => {
      clearInterval(interval);
    };
  });
}
```

How would we need to change the `Observable` constructor for this to work? We just call the function:

```ts
function Observable(producer) {
  return {
    subscribe: (next, error, complete) => {
      let subscriber = { next, error, complete };

      // We save the cleanup function...
      const cleanup = producer({
        next: value => subscriber && subscriber.next(value),
        error: error => subscriber && subscriber.error(error),
        complete: () => {
          subscriber && subscriber.complete();

          // ...which we call on complete...
          cleanup();
        },
      });

      return {
        unsubscribe: () => {
          subscriber = undefined;

          // ...and on unsubscribe.
          cleanup();
        }
      };
    }
  };
}
```

Here is the complete usage example, with logging for clarity:

```ts
function interval(timeBetweenValues) {
  return new Observable(subscriber => {
    let counter = 1;

    const interval = setInterval(() => {
      console.log('producing value', counter);
      subscriber.next(counter++);
    }, timeBetweenValues);
    console.log('interval started');

    return () => {
      console.log('interval stopped');
      clearInterval(interval);
    };
  });
}

const observable = interval(1000);
const subscription = observable.subscribe(count => console.log(count));

setTimeout(() => {
  console.log('done listening for values');
  subscription.unsubscribe();
}, 5500);

// interval started
// producing value 1
// ...1 second passes...
// producing value 2
// ...1 second passes...
// producing value 3
// ...1 second passes...
// producing value 4
// ...1 second passes...
// producing value 5
// done listening for values
// interval stopped
```

Success! We now have a mechanism to stop listening to any stream at any time. If we stop listening, the stream has a way to interrupt itself.

This can be used with finite streams as well - we just need the correct cleanup function.

## Are we done?

**The code above is a fully-functional implementation for RxJS-like `Observable`.**
_Yes, RxJS itself includes a lot more code to allow for more flexibility and edge-cases. However, the `Observable` above is also fully functional and works much in the same way._

RxJS also has additional functionality like subjects and operators, some of which we'll implement in Part 2. There, we'll make this work:

```ts
const observable = interval(1000).pipe(
  map(x => x * 10),
  take(3)
);

observable.subscribe(
  number => console.log(number),
  () => {},
  () => console.log('complete');
);

// 10
// ...1 second passes...
// 20
// ...1 second passes...
// 30
// complete
```
