---
layout: post
title:  "A Monad in Practicality: Controlling Time"
snip:   How Monads help you compose asynchronous operations.
---

Use numbered headers: True

<!-- * * * -->

Concurrency is quite a big deal, specially these days where everything must be
“web scale.” There are several models of concurrency, each with their own pros
and cons, in this article I present a composable alternative to the widespread
non-blocking model of concurrency, which generally uses
[Continuation-Passing style][cps], where each computation accepts a
“continuation” or “callback,” instead of returning the result of the
computation to the caller.

However, in a language where some computations return a value to the caller,
and some take a continuation as an argument, we can't really apply our
well-established and general compositional operators, such as
[function composition][], because the rules they rely on have been broken. This
is why, in platforms like Node.js, code quickly descends into callback-hell and
spaghetti of callsite-specific functionality — we lose all of the
organisational patterns we are used to.

None the less, it is possible for a non-blocking function to return the result
to its caller **even thought it still doesn't know what the result is**! This
really old concept (which appeared around 1976) is sometimes called *promise*,
*future*, *delay*, or *eventual*, depending on your library and programming
language.

In this blog post I'll describe what a *future* value is, how it forms a monad,
what are the benefits of a *future* forming a monad, and how they may be used
for abstracting over the concurrency problem in a composable way.

> **Note**: Previous knowledge of monads or category theory are not necessary
> to read this blog post, even though this article builds upon my
> [previous article on monads in JavaScript][failures]. Some knowledge of
> JavaScript is necessary, however.

[cps]: http://matt.might.net/articles/by-example-continuation-passing-style/
[function composition]: http://en.wikipedia.org/wiki/Function_composition
[failures]: http://robotlolita.github.io/2013/12/08/a-monad-in-practicality-first-class-failiures.html



# Table of Contents
 *  TOC
{:toc}

## Introduction

Non-blocking concurrency is all the hype these days, specially with platforms
such as Node.js, which are focused on I/O bound applications, a perfect fit for
this form of concurrency. Unfortunately, most of these platforms don't provide
good primitives for working with this form of concurrency, as is the case with
Node, where people must rely on naïve Continuation-Passing style to schedule
some actions to occur after other actions.

As previously discussed, this naïve use of *callbacks* leads to operations that
can not be composed with the primitives we already use for composing
synchronous computations, such as function composition. More so, in languages
like JavaScript, these asynchronous operations can't be integrated with the
usual syntactic control-flow structures, such as conditional branching, loop
constructions, or exception handling.

Although there are different alternatives for this problem, some of which
includes [delimited continuations][] and
[communicating sequential processes][], *futures* are by far the cheapest
implementation in most languages, since they don't require additional language
support besides threads **or** higher-order functions. A naïve implementation
of futures allows asynchronous computations to be composed just as synchronous
computations can be. However, co-monadic futures are a necessary extension to
integrate these computations with the language's syntactic structure — such a
concept is the basis for [C#'s async/await semantics][async-await].

This article is intended as a general introduction to the concept of Futures,
and how they change the possibilities of composing asynchronous computations
(IOW, how to think with futures), as well as a description of its practical
usage in JavaScript using [my implementation of monadic futures][data.future],
therefore most of the article assumes familiarity with the JavaScript language,
although you should be able to follow with an understanding of higher-order
programming just fine.

[delimited continuations]: http://axisofeval.blogspot.com.br/2011/07/some-nice-paperz.html
[communicating sequential processes]: http://www.usingcsp.com/
[async-await]: http://research.microsoft.com/en-us/um/people/gmb/papers/cs5full.pdf
[data.future]: https://github.com/folktale/data.future


## Non-blocking computations and CPS

In Node.js, non-blocking computations are expressed in terms of
Continuation-Passing style, which is the simplest solution for a language
supporting higher-order functions. You can easily invent this model of
computation in these languages yourself if you live by the following maxim:

> **No function should ever return to its caller.**

Now, if a function can't return a value, what are they even good for? Or
rather, how do we use a function that can't return a value? Well, instead of
letting the language semantics define where the control goes after the function
has the value, we explicitly pass the next computation (or *current
continuation*, or *callback*) to the function we're calling. So, this:

{% highlight js %}
function add(a, b) {
  return a + b
}
{% endhighlight %}

Becomes this:

{% highlight js %}
function add(a, b, continuation) {
  continuation(a + b)
}
{% endhighlight %}

Or, to make the duality even more obvious, if you pretend that `return` isn't a
keyword:

{% highlight js %}
function add(a, b, return) {
  return(a + b)
}
{% endhighlight %}

If you consider `return` to be an internal function that knows where to move
the flow of execution in your program, then a *continuation* just makes that
function explicit. But just by doing so it gives us so much more power, because
now we're not bound to the control flow semantics of our language anymore,
which allows a function taking a continuation to be either synchronous or
asynchronous, needing no changes in the user of that code. On the other hand,
we lose the guarantee that in a sequence of actions, each one will always
execute before the previous one, or that calls to such computations will be
thread-safe. In a language with continuations, these concerns are shifted from
the compiler or interpreter, to each computation.

{% highlight js %}
// Since these are functions in CPS style, there's no guarantees about
// which of these operations will be executed first.
add(1, 2, print)
add(3, 4, print)
add(5, 6, print)

// Thus, to sequence actions, you must provide the action to run in
// sequence as a continuation:
add(1, 2, function(result) {
  add(result, 3, print)  // this will always execute after 1 + 2
})
{% endhighlight %}

With computations following the Continuation-Passing style, it's pretty
straight-forward to write a program that uses non-blocking
computations. However, given that JavaScript's semantics enforce synchronous
execution, CPS does not magically transform a blocking computation into a
non-blocking one. Instead, the computation needs to explicitly execute the
blocking work on a separate thread:

{% highlight js %}
function read(pathname, continuation) {
  // Here we use a primitive that's not present in most JavaScript
  // implementations. It creates a new thread, and executes some
  // code inside of that thread. This allows the main thread to
  // continue doing its business as usual
  Thread.spawn(function() {
    // This is a blocking call, but it occurs in a separate thread
    // so it doesn't block the main thread.
    var contents = io.read(pathname)

    // Once we're done, the continuation is scheduled to be called
    // with the contents of the file in the main thread.
    // Note that we can't just execute it right away in the main
    // thread because it might be busy.
    Thread.main.schedule(function(){ continuation(contents) })
  })
}
{% endhighlight %}

Lastly, since Continuation-Passing style forgoes the language's own
control-flow semantics to provide the basis for an expressive,
bring-your-own-control-flow framework, we can't combine the usual syntactical
constructs for control-flow, such as if/else or try/catch, with our CPS
computations. New control-flow structures need to be built up on top of this
new framework.

In Node, instead of building a new exception mechanism, they use a much simpler
model, somewhat similar to Shell's error handling, where all continuations take
not just the result of the computation, but also any error that might have
occurred. In this model, the previous computation would be written as:

{% highlight js %}
function read(pathname, continuation) {
  Thread.spawn(function() {
    try {
      var contents = io.read(pathname)
      Thread.main.schedule(function(){ continuation(null, contents) })
    catch (error) {
      Thread.main.schedule(function(){ continuation(error, null) })
    }
  })
}

// And you would use it as:

read('/foo/bar', function(error, contents) {
  if (error)  handleError(error)
  else        handleContents(contents)
})
{% endhighlight %}


## Futures as placeholders for eventual values

While Continuation-Passing style is definitely an interesting start for
building your own control-flow structures, it's too low level for us to work
with directly in JavaScript, whose primitives are not written in terms of
continuations. As such, we would prefer a more abstract way of dealing with
non-blocking computations, such that we're able to compose them with our
regular synchronous computations using the primitives we've already got!

For this to work, our asynchronous functions need to be able to return a value,
but we don't want them to block until they've figured out what the value should
be, and they can't return the value before they've figured out what it should
be! ...or can't they? Well, enter the concept of *futures*. A Future represents
the eventual result of an asynchronous computation, such that a it may return
a useful value to the caller and fill in the details later.

In other words, instead of writing this:

{% highlight js %}
function read(pathname, continuation) {
  ...
}

read('/foo/bar', function(error, contents) { ... })
{% endhighlight %}

We can write this:

{% highlight js %}
function read(pathname) {
  ...
}

var contentsF = read('/foo/bar')
{% endhighlight %}

So far, just by using futures we've managed to exchange the explicit passing of
continuations our code is already looking like it used to. Functions return
values. We can assign those values to variables. BAM. Everything is right, uh?
But what happens when we try to use those values? Well, it depends on how you
implement them. In multiple-threaded environments, reading from a future will
usually block the current thread until some other thread fills in the
value.

While composable, a naïve implementation of blocking futures poses the same
problems as regular blocking computations. Furthermore, this doesn't work in a
single-threaded environment. If we combine our concrete representation of
future values with the Continuation-Passing style, we get part of the
compositionality benefits, without having to deal with the blocking
problem. Plus, it works great in a single-threaded environment.

One of the simplest implementations of future would be to just return a
function whose only parameter is a continuation:

{% highlight js %}
function read(pathname) {
  return function(continuation) {
    ...
  }
}

var contentsF = read('/foo/bar')
contentsF(function(error, contents) {
  if (error)  handleError(error)
  else        handleContents(contents)
})
{% endhighlight %}


## A monadic formulation of futures

Albeit the previous example of implementation was simple and usable, in
JavaScript we can benefit much more from a slightly different implementation:
one where our future forms a [monad][]. To this end, we'll need to create an
object that fulfils the interface described in the [fantasy land][]
specification. And why would we do that? Well, by doing this simple thing we
get a lot of abstractions for free! (and we get nice properties for reasoning
about our programs too).

> If you're not familiar with the concepts of Category Theory, or have never
> heard the term *monad* before, don't worry, **monads are just monoids in a
> category of endofunctors,** or rather, **a monad is a functor together with
> two natural transformations**. You could even say that **a FuzzyWuzzy is
> just a Vogon in the centre of Megabrantis Cluster!**
>
> In other words, it doesn't matter to you, the programmer, what a *monad*
> is. In fact, for the rest of this article you can replace any occurrence of
> *monad* by whichever word you like the most. If you're interested in knowing
> all about monads, go read
> [Phil Wadler's "Monads For Functional Programming"][monad] paper.

Moving on, to make our future a monad, we need to implement two methods:

 -  `of(a) → Future(a)`: creates a future holding anything (including other
    monads).
    
 -  `chain(a → Future(b)) → Future(b)`: transforms the value inside a monad
    into another monad of the same type.

Furthermore, to ensure that we can compose cleanly all of the different things
that implement this interface, there are some rules that everyone needs to play
by. With this we can compose all the different types of monads in complex ways,
and be sure that we can understand and have guarantees about what will
happen. The rules are (where `m` is an instance of the future):

 -  Associativity: `m.chain(f).chain(g)` should be equivalent to `m.chain(x => f(x).chain(g))`;
 -  Left identity: `m.of(a).chain(f)` should be equivalent to `f(a)`;
 -  Right identity: `m.chain(m.of)` should be equivalent to `m`.

And that's all it takes for something to be fulfil the monad interface. The
only difference to an interface in a language like Java is that besides the
type constraints we also have these *laws* that dictate how this object needs
to behave in certain circumstances. I'll leave the implementation of such
object as an exercise to the reader, and use my [data.future][] implementation
for the rest of this article.



[monad]: http://homepages.inf.ed.ac.uk/wadler/papers/marktoberdorf/baastad.pdf
[fantasy land]: https://github.com/fantasyland/fantasy-land

## The composition of futures

With monadic futures, our now widely-used example becomes:

{% highlight js %}
function read(pathname) {
  return new Future(function(reject, resolve) {
    ...
  })
}

var contentsF = read('/foo/bar')
contentsF.chain(handleContents).orElse(handleError)
{% endhighlight %}

> **Note**: the referred future implementation is pure, as such `chain` does
> not run the computation, but returns a description of the computation that
> should be performed. To run the computation you must invoke the
> `fork(errorHandler, successHandler)` method. Check out
> [Runar's talk on Purely Functional I/O][pure-io] to understand why this
> matters.

Again, like Continuation-Passing style computations, the value of `contentsF`
is not dependent on the order in which our source code is arranged. This
disentangles computations and time, which are so usually coupled in an
imperative language. Therefore, it doesn't matter which order you arrange your
source code, the results must be the same:

{% highlight js %}
var a = read('/foo')
var b = read('/bar')

// is the same as
var b = read('/bar')
var a = read('/foo')
{% endhighlight %}

Of course, the example above would also yield the same value in a pure
imperative program. But this formulation of futures force you not to rely on
the implicit ordering of your source code, because there's no way to know when
each value will be available. So, the following program is automatically
excluded, and with it a class of complexities:

{% highlight js %}
var a = read('/foo')
var b = a + read('/bar')  // can't access `a` yet.
{% endhighlight %}

If one needs to access the value of a future, they need to do so explicitly by
invoking the `chain` method and providing a computation to run once the value
is made available. In essence, futures force you to give up on the implicit
dependency on time (and source order), and buy into explicit dependency on
actual values.

Furthermore, the existence of different methods for sequencing successful and
erroneous values, together with the monad semantics, allow us to short-circuit
computations that we can't perform, and automatically propagate the errors back
up to the caller, similar to what a `try/catch` construct would do for
synchronous code. In fact, one of the advantages of monads over plain
functions/continuations is the ability to ignore some of the painful
explicitness of functional programming sometimes.


{% highlight js %}
function divide(a, b) {
  return new Future(function(reject, resolve) {
                      if (b === 0)  reject(new Error('Division by 0'))
                      else          resolve(a / b) })
}

function reject(a) {
  return new Future(function(reject, resolve) {
                      reject(a) })
}

function resolve(a) {
  return new Future(function(reject, resolve) {
                      resolve(a) })
}

var aF = divide(9, 3)
// => Future(_, 3)
var bF = divide(2, 1)
// => Future(_, 2)
var cF = aF.chain(function(a) {
                    return bF.chain(function(b) {
                                      return resolve(aF + bF) })})
// => Future(_, 5)
var dF = aF.chain(function(a) {
                    return divide(a, 0).chain(function(b) {
                                                return resolve(a + b) })})
// => Future(Error('Division by 0'), _)
var eF = dF.orElse(function(error) {
                     return reject(new Error('boo')) })
// => Future(Error('boo'), _)
{% endhighlight %}

> **Note**: both `chain` and `orElse` are used to sequence asynchronous
> computations, as such they are not the best fit for running synchronous
> computations in the value the future holds. Using `map` and `rejectedMap`,
> you could make these cases much simpler. E.g.:
>
>     dF.rejectedMap(function(e){ return new Error('boo') })

[pure-io]: http://www.infoq.com/presentations/io-functional-side-effects

## Common abstractions

Good, we can compose computations, but working with anything has become
incredibly bothersome. Some of this is due to JavaScript's verbosity in
function declarations, some is due to limitations in abstraction power. Either
way, we can lessen the burden by taking advantage of the monadic abstraction
and functional combinators.

For example, suppose that we need to concatenate two files we've read from the
disk, but the contents of those files are inside a future. The usual way to go
about it would be to nest the chain calls in different closures:

{% highlight js %}
var aF = read('/foo')
var bF = read('/bar')
var cF = aF.chain(function(a) {
                    return bF.chain(function(b) {
                                      return resolve(a + b) })})
{% endhighlight %}

But since this is a common pattern, we could easily write a combinator that
abstracts over this:

{% highlight js %}
function chain2(aM, bM, f) {
  return aM.chain(function(a) {
                    return bM.chain(function(b) {
                                      return new Future(function(_, resolve) {
                                                          resolve(f(a, b)) })})})
}

var aF = read('/foo')
var bF = read('/bar')
chain2(aF, bF, function(a, b){ return a + b })
{% endhighlight %}

In fact, this pattern is **so** common, that people have already written this
combinator for us, and by having our future form a monad we can use it for
free. The combinator is called `liftM`, and as you can guess, it takes some
monads and a regular function, and makes this function work on the values of
those monads.

By currying this `liftM` combinator, and using the `flip` combinator, we can
easily use function composition for working with monads:

{% highlight js %}
// Promotes an unary function to a function on monads
var liftM = curry(function(aM, f) {
                    return aM.chain(function(a) {
                                      return aM.of(f(a)) })})

// Promotes a binary function to a function on monads
var liftM2 = curry(function(aM, bM, f) {
                     return aM.chain(function(a) {
                                       return bM.chain(function(b) {
                                                         return bM.of(f(a, b)) })})})

// Inverts the argument order for a binary function
var flip = curry(function(f, a, b) {
                   return f(b, a) })

// Now we can use `liftM`, partial application and the `flip` combinator to
// easily promote any regular function into functions on monads:
function toUpper(a) {
  return a.toUpperCase()
}
var toUpperM = flip(liftM)(toUpper)
toUpperM('foo') // => 'FOO'

var readAsUpperM = compose(toUpperM, read)
readAsUpper('/foo/bar')
{% endhighlight %}




## Ad-hoc parallelism and non-determinism
## Syntactical composition with co-monads
## Conclusion
## Libraries
## References and additional reading