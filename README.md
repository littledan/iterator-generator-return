# Proposal: Back out `.return()` from iterators and generators

by Daniel Ehrenberg (@littledan)

[Summarizing slide deck](https://docs.google.com/presentation/d/13KkqTTz9s2ZZWIF57PWsoQELiYa3Zf150cC9VhqAW60/edit)

ES2015 adds generators and the iteration protocol. One unusual feature in
ES2015 is the `.return()` method on the iteration protocol, which is supported
by generators. In the iteration protocol, `.return()` should be called before exiting
iteration early. On generators, `.return()` makes the next `yield` expression
act like an early function return.

I'd argue that both of these features are unnecessary.

## Why generators don't need `.return()`

### Generator `.return()` does not provide a basis for promise cancellation

Promise cancellation is more like an exception. It cancellation propagate outward from a cancelled promise, not stop at the function boundary.

It's not clear whether cancellation will be modelled by a specially tagged exception, a new exception-like abrupt completion, or a .NET-style token that is passed around, but what's clear is that it won't be analogous to generator `.return()`. Everyone I've spoken with who is working on cancellable promises seems to be in agreement about this.

### Generator `.return()` does not add more power to the language

All usage of `.return()` from a generator could be emulated by a sufficiently motivated generator and user according to a particular protocol. For an example of how this emulation would work, see this generator which is not in terms of `.return()`:

```js
function* foo(accumulator) {
  while (true) {
    let {return: r, payload: p} = yield;
    if (r) return r;
    accumulator.push(p);
  }
}

let accumulator = []
let x = foo(accumulator);
x.next();
x.next({payload:1});
x.next({payload:2});
let {value: r} = x.next({return: 100});
```

Then accumulator will be `[1,2]`, and `r` will be `100`. This is just like
if we wrote with the current ES2015 language:

```js
function* foo(accumulator) {
  while (true) {
    accumulator.push(yield);
  }
}

let accumulator = []
let x = foo(accumulator);
x.next();
x.next(1);
x.next(2);
let r = x.return(100);
```

So there's no fundamental real power added to the language (the way
that, say, adding generators in the first place gives). The only
change is in terms of what we encourage in our sugary constructs.

### Generator `.return()` causes additional overhead to all generator usage

With generator `.return()`, all yield statements might return from the generator, depending on how they are used. This basically means an implementation that looks like the above emulation--after every yield, check to see if some sort of `return` flag is set somewhere, and return if so. Even if it's theoretically possible to optimize out, it'll be pretty complicated to provide a separate IP for 'return' and make it free. The likely outcome for a while is slower generators in many browsers.

## Why iterators don't need `.return()`

### Iterator `.return()` is largely motivated by generator `.return()`

If we did decide to get rid of generator `.return()`, we wouldn't have any iterators in the standard library which implemented `.return()`, and it would be a feature just for user-level libraries. And I don't know of any user-level libraries banging at our door requesting this feature.

### Iterator `.return()` as a resource management mechanism would be better served by `try`/`finally` or a new mechanism

Most iterators are themselves iterable. That is, you can get the iterator out of the iterable, store it in a local variable, and then do a `for`/`of` loop over it. This means that, if an iterator absolutely must be the thing that allocates resources, it could be stored in a local variable, and then freed from the `finally` branch of a `try`/`finally`. Sure, the semantics of `.return()` are such that it only gets called when needed (not in a normal exit), but the only implementor of `.return()` in the ES2015 spec (generators) is resilient against being called when generator has already completed. It doesn't seem like such a big burden on users to do something similar for their own resource management arrangement around `try`/`finally`.

It's true that resource management through `for`/`of` loops is more terse than using `try`/`finally`. But I'd argue that most resources will not be represented by iterators, users are using `try`/`finally` successfully today for similar cases, and if we wanted to make a nicer-looking mechanism, we should do so in a way which is not restricted to iterators. For example, maybe a `[Symbol.close]()` method could be called at the end of a scope for specially declared variables, with special syntax for marking the iterator of a `for`/`of` loop that way.

### Iterator `.return()` resource management seems to be mostly for blocking I/O, but we are moving towards async I/O

Iterator `.return()` is built for a scenario like this: Say an iterator synchronously acquires some resources in the `.[Symbol.iterator]()` method, and these are typically freed when the iterator runs through. However, when being disposed of early, the explicit `.return()` method provides another entrypoint for freeing the resource. Some example use cases include a database cursor, where the foreign resources need to be freed after iterating through the database, and an iterator over file lines, where the file should be closed when the iterator is closed.

As far as I can tell, these resource management cases would tend to involve blocking. While it could sound nice to handle this case of resource management well, it seems a bit out of step with the direction that JavaScript is going in, with Promises/async/await for most of these kinds of operations. Even if this is useful in the shell script case, I'd argue that we as a committee should be trying to drive the language towards a point where those async mechanisms work well in a shell script, rather than building features for a synchronous, blocking world.

There are a lot of advantages to
async/await, not least of which is that it frees us from having to
think about shared state concurrency within the same realm. I am
afraid that if we encourage much uses of blocking, we may eventually
feel forced to provide more mechanisms for dealing with it, like
threads (or leave users of those systems with something that feels
very incomplete).

### Iterator `.return()` causes additional overhead for all iterator usage

The iteration protocol shows up in multiple places in the ES2015 spec, not just in `for`/`of` loops, but also in destructuring bind. These callsites all have to be aware of how to call `.return()`, observably getting the method property and invoking it if it's there. For `for`/`of` loops, a sort of `try`/`catch` statement is required around all loops. This adds overhead because, in several JS implementations, `try`/`catch` is not supported in certain compiler tiers, and even if it is supported in a particular tier, it might not be cost-free in that tier. Further, the semantics of early return in `for`/`of` loops cannot be desugared to any other existing exception construct, so it can't be implemented via a simple desugaring and requires something more complex.

## What should be done?

### Revert `.return()` from the ES2016 spec

It's not too late to remove `.return()`. No browser has shipped this feature in the ES2015 form yet, so the removal should be web-compatible. Given the lack of motivation and implementation burden that `.return()` has, let's remove it from ES2016.

### Work out cancellation

Promise cancellation is an extremely important feature which informs the design of `async`/`await` and, to the extent that generators continue to parallel async functions, impacts generators. Once we know how cancellation works, it'd help us figure out what we need to add to the language or leave in the language, and we won't need to hold onto guesswork about what might or might not be useful.

### Go back and add things later if needed

I think it'd be web-compatible to add some forms of resource management on top later. While anyone can, in theory, detect anything with Proxies, TC39's goal isn't absolute backwards-compatibility, but rather web-compatibility. Symbols give us a web-compatible extension point where more methods can be put on existing objects. Another possibility to consider is marking `for`/`of` loops specially if some cleanup action needs to be taken on early exit--ES2015 adds several contextual keywords, and it should be possible to add more if needed.
