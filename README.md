# Proposal: Back out .return() from iterators and generators

ES2015 adds generators and the iteration protocol. One unusual feature in
ES2015 is the `.return()` method on the iteration protocol, which is supported
by generators. In iteration, `.return()` should be called before exiting
iteration early. On generators, `.return()` makes the next `yield` expression
act like an early function return.

I'd argue that both of these features are unnecessary.

## Why generators don't need `.return()`

### Generator return does not provide a basis for promise cancellation

### Generator `.return()` does not add more power to the language

### Generator `.return()` causes additional overhead to all generator usage

## Why iterators don't need `.return()`

### Iterator `.return()` is largely motivated by generator `.return()`

### Iterator `.return()` as a resource management mechanism would be better served by `try`/`finally` or a new mechanism

### Iterator `.return()` causes additional overhead for all iterator usage
