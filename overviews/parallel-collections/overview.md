---
layout: overview-large
title: Overview

disqus: true

partof: parallel-collections
num: 1
---

**Aleksandar Prokopec, Heather Miller**

## Motivation

Amidst the shift in recent years by processor manufacturers from single to
multi-core architectures, academia and industry alike have conceeded that
_Popular Parallel Programming_ remains a formidable challenge.

Parallel collections were included in the Scala standard library in an effort
to facilitate parallel programming by sparing users from low-level
parallelization details, meanwhile providing them with a familiar and simple
high-level abstraction. The hope was, and still is, that implicit parallelism
behind a collections abstraction will bring reliable parallel execution one
step closer to the workflow of mainstream developers.

The idea is simple-- collections are a well-understood and frequently-used
programming abstraction. And given their regularity, they're able to be
efficiently parallelized, transparently. By allowing a user to "swap out"
sequential collections for ones that are operated on in parallel, Scala's
parallel collections take a large step forward in enabling parallelism to be
easily brought into more code.

Take the following, sequential example, where we perform a monadic operation
on some large collection:

    val list = (1 to 10000).toList
    list.map(_ + 42)
    
To perform the same operation in parallel, one must simply invoke the `par`
method on the sequential collection, `list`. After that, one can use a
parallel collection in the same way one would normally use a sequential
collection. The above example can be parallelized by simply doing the
following:

    list.par.map(_ + 42)

The design of Scala's parallel collections library is inspired by and deeply
integrated with Scala's (sequential) collections library (introduced in 2.8).
It provides a parallel counterpart to a number of important data structures
from Scala's (sequential) collection library, including:

* `ParArray`
* `ParVector`
* `mutable.ParHashMap`
* `mutable.ParHashSet`
* `immutable.ParHashMap`
* `immutable.ParHashSet`
* `ParRange`
* `ParCtrie` (`Ctrie`s are new in 2.10)

In addition to a common architecture, Scala's parallel collections library
additionally shares _extensibility_ with the sequential collections library.
That is, like normal sequential collections, users can integrate their own
collection types and automatically inherit all of the predefined (parallel)
operations available on the other parallel collections in the standard
library.

## Some Examples

To attempt to illustrate the generality and utility of parallel collections,
we provide a handful of simple example usages, all of which are transparently
executed in parallel.

_Note:_ Some of the following examples operate on small collections, which
isn't recommended. They're provided as examples for illustrative purposes
only. As a general heuristic, speed-ups tend to be noticeable when the size of
the collection is large, typically several thousand elements.

#### map

Using a parallel `map` to transform a collection of `String` to all-uppercase:

    scala> val lastNames = List("Smith","Jones","Frankenstein","Bach","Jackson","Rodin").par
    lastNames: scala.collection.parallel.immutable.ParSeq[String] = ParVector(Smith, Jones, Frankenstein, Bach, Jackson, Rodin)
    
    scala> lastNames.map(_.toUpperCase)
    res0: scala.collection.parallel.immutable.ParSeq[String] = ParVector(SMITH, JONES, FRANKENSTEIN, BACH, JACKSON, RODIN)

#### fold

Summing via `fold` on a `ParArray`:

    scala> val parArray = (1 to 1000000).toArray.par    
    parArray: scala.collection.parallel.mutable.ParArray[Int] = ParArray(1, 2, 3, ...
    
    scala> parArray.fold(0)(_ + _)
    res0: Int = 1784293664

#### filter

Using a parallel `filter` to select the last names that come alphabetically
after the letter "K".

    scala> val lastNames = List("Smith","Jones","Frankenstein","Bach","Jackson","Rodin").par
    lastNames: scala.collection.parallel.immutable.ParSeq[String] = ParVector(Smith, Jones, Frankenstein, Bach, Jackson, Rodin)
    
    scala> lastNames.filter(_.head >= 'J')
    res0: scala.collection.parallel.immutable.ParSeq[String] = ParVector(Smith, Jones, Jackson, Rodin)


## Semantics

While the parallel collections abstraction feels very much the same as normal
sequential collections, it's important to note that its semantics differs,
especially with regards to side-effects and non-associative operations.

In order to see how this is the case, first, we visualize _how_ operations are
performed in parallel. Conceptually, Scala's parallel collections framework
parallelizes an operation on a parallel collection by recursively "splitting"
a given collection, applying an operation on each partition of the collection
in parallel, and re-"combining" all of the results that were completed in
parallel. In general, there is no guarantee on which order results are re-
combined-- they're combined in the order that they're completed.

These concurrent, and "out-of-order" semantics of parallel collections lead to
the following two implications:

1. **Side-effecting operations can lead to non-determinism**
2. **Non-associative operations lead to non-determinism**

### Side-Effecting Operations

Given the _concurrent_ execution semantics of the parallel collections
framework, operations performed on a collection which cause side-effects
should generally be avoided, in order to maintain determinism. A simple
example is by using a transformer method, like `foreach` to increment a var
declared outside of the closure which is passed to `foreach`.

    scala> var sum = 0
    sum: Int = 0

    scala> val list = (1 to 1000).toList.par
    list: scala.collection.parallel.immutable.ParSeq[Int] = ParVector(1, 2, 3,…
    
    scala> list.foreach(sum += _); sum
    res01: Int = 467766
    
    scala> var sum = 0
    sum: Int = 0
    
    scala> list.foreach(sum += _); sum
    res02: Int = 457073    
    
    scala> var sum = 0
    sum: Int = 0
    
    scala> list.foreach(sum += _); sum
    res03: Int = 468520    
    
Here, we can see that each time `sum` is reinitialized to 0, and `foreach` is
called again on `list`, `sum` holds a different value. The source of this non-
determinism is a _data race_-- concurrent reads/writes to the same mutable
variable.

In the above example, it's possible for two threads to read the _same_ value
in `sum`, to spend some time doing some operation on that value of `sum`, and
then to attempt to write a new value to `sum`, potentially resulting in an
overwrite (and thus, loss) of a valuable result, as illustrated below:

    ThreadA: read value in sum, sum = 0                value in sum: 0
    ThreadB: read value in sum, sum = 0                value in sum: 0
    ThreadA: increment sum by 760, write sum = 760     value in sum: 760
    ThreadB: increment sum by 12, write sum = 12       value in sum: 12


### Non-Associative Operations

Given this _"out-of-order"_ semantics, one must be careful to perform only
associative operations in order to avoid non-determinism. That is, given a
parallel collection, `pcoll`, one should be sure that when invoking a higher-
order function on `pcoll`, such as `pcoll.reduce(func)`, the order in which
`func` is applied to the elements of `pcoll` can be arbitrary. A simple, but
obvious example is a non-associative operation such as subtraction:

    scala> val list = (1 to 1000).toList.par
    list: scala.collection.parallel.immutable.ParSeq[Int] = ParVector(1, 2, 3,…
    
    scala> list.reduce(_-_)
    res01: Int = -228888
    
    scala> list.reduce(_-_)
    res02: Int = -61000
    
    scala> list.reduce(_-_)
    res03: Int = -331818
    
In the above example, we take a `ParVector[Int]`, invoke `reduce`, and pass to
it `_-_`, which simply takes two unnamed elements, and subtracts the second
from the first. Due to the fact that the parallel collections framework spawns
threads which, in effect, independently perform `reduce(_-_)` on different
sections of the collection, the result of two runs of `reduce(_-_)` on the
same collection will not be the same.

_Note:_ Often, it is thought that, like non-associative operations, non-
commutative operations passed to a higher-order function on a parallel
collection likewise result in non-deterministic behavior. This is not the
case. The _"out of order"_ semantics of parallel collections only means that
the operation will be executed out of order (that is, non-sequentially, in
time), it does not mean that the result will be re-"_combined_" out of order--
on the contrary, results will generally always be reassembled _in order_. A
simple example is string concatenation-- an associative, but non-commutative
operation:

    scala> val strings = List("abc","def","ghi","jk","lmnop","qrs","tuv","wx","yz").par
    strings: scala.collection.parallel.immutable.ParSeq[java.lang.String] = ParVector(abc, def, ghi, jk, lmnop, qrs, tuv, wx, yz) 
    
    scala> val alphabet = strings.reduce(_++_)
    alphabet: java.lang.String = abcdefghijklmnopqrstuvwxyz

For more on how parallel collections split and combine operations on different
parallel collection types, see the [Architecture]({{ site.baseurl }}/overviews
/parallel-collections/architecture.html) section of this guide. 

## Mutable & Immutable Parallel Collections
difference between mutable and immutable parallel collections

## Helpful Hints
* Lists, and avoiding copying. 