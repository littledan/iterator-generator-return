# Proposal: Back out .return() from iterators and generators

ES2015 adds generators and the iteration protocol. One unusual feature in
ES2015 is the `.return()` method on the iteration protocol, which is supported
by generators. In iteration, `.return()` should be called before exiting
iteration early. On generators, `.return()` makes the next `yield` expression
act like an early function return.

I'd argue that both of these features are unnecessary.

## Why generators don't need `.return()`

### Generator `.return()` does not provide a basis for promise cancellation

Promise cancellation is more like an exception. It cancellation propagate outward from a cancelled promise, not stop at the function boundary. There's no meaningful 'return value' from a cancelled promise. Cancellation is not something that makes sense to catch and halt and prevent the cancellation from going through. The cancellation needs to go all the way up.

It's not clear whether cancellation will be modelled by a specially tagged exception, a new exception-like abrupt completion, or a .NET-style token that is passed around, but what's clear is that it won't be analogous to generator `.return()`. Everyone I've spoken with who is working on cancellable promises seems to be in agreement about this (but I'm happy to amend this document with more views!).

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
x.next(1);
x.next(2);
let r = x.return(100);
```

So there's no fundamental real power added to the language (the way
that, say, adding generators in the first place gives). The only
change is in terms of what we encourage in our sugary constructs.

If you have an arbitrary generator lying around, not through this construct, and you want to get it to exit early and free all the resources that `.return()` would free, then `.throw()` will *often*, but not always work for edge cases. One thing that would work is creating a protocol like above. See the 'resource management' section below for more thoughts on mechanisms for this.

### Generator `.return()` causes additional overhead to all generator usage

With generator `.return()`, all yield statements might return from the generator, depending on how they are used. This basically means an implementation that looks like the above emulation--after every yield, check to see if some sort of `return` flag is set somewhere, and return if so. While it's theoretically possible, it'll be extremely complicated for implementations to properly deopt on this when appropriate in order to eliminate the conditionals.

## Why iterators don't need `.return()`

### Iterator `.return()` is largely motivated by generator `.return()`

If we did decide to get rid of generator `.return()`, we wouldn't have any iterators in the standard library which implemented `.return()`, and it would be a feature just for user-level libraries. And there aren't any user-level libraries banging at our door requesting this feature.

### Iterator `.return()` as a resource management mechanism would be better served by `try`/`finally` or a new mechanism

Say an iterator acquires some resources, and ...

### Iterator `.return()` causes additional overhead for all iterator usage

The iteration protocol shows up in multiple places in the ES2015 spec, not just in `for`/`of` loops, but also in destructuring bind. These callsites all have to be aware of how to call `.return()`, observably checking for the method and invoking it if it's there. For `for`/`of` loops, ...

## What should be done?

### Work out cancellation

### Revert `.return()` from the ES2016 spec

### Go back and add things later if needed

Yes!
