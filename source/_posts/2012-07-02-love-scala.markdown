---
layout: post
title: "Things I Love About Scala"
tags:
  - Concurrency
  - Algorithms  
  - Scala
  - Java  
---

Scala makes development in a static language productive and
concurrency easy, while allowing you to tap into the power of the JVM,
giving you access to the same concurrency primitives available for
Java, should you need to. I'm now a fan.

<!-- more -->

## Syntactic Sugar

Say that we've got a web service that calls out to other web services,
or does something else expensive (like calculating the digits of PI
:)) and you want to cache the returned results in a map of some sort.

Our client-side usage of this class will be like:

{% highlight scala %}
val cache = new Cache

val value = cache("some-key-here") {
  // do something expensive ...
  Thread.sleep(100)
  
  // returns result
  "Hello world!"
}
{% endhighlight %}

First, let's implement a naive version:

{% highlight scala %}
class NaiveCache {
  private[this] var map = Map.empty[String, Any]

  def apply(key: String)(process: => Any): Any = 
    map.get(key) match {
      case Some(value) => value
      case None => {
        val value = process
        map = map + (fullKey -> value)
        value
      }
    }
}
{% endhighlight %}

This looks pretty simple. We've got a Map that stores the results and
an `apply` method that takes as input a Key and a function `process()`
that returns the value of some computation.

But it's not OK that this function returns Any reference, because we
are in a static language and I hate casting, which in this case is
totally unnecessary. The right return type of `apply()` could be
inferred from its parameters. So we'll fix that by adding a generic
variable:

{% highlight scala %}
class NaiveCache {
  private[this] var map = Map.empty[String, Any]

  def apply[T: Manifest](key: String)(process: => T): T = {
    val fullKey = key + "::" + manifest[T].toString

    map.get(fullKey) match {
      case Some(value) => value.asInstanceOf[T]
      case None => {
        val value = process
        map = map + (fullKey -> value)
        value
      }
    }
  }
}
{% endhighlight %}

We've added the generic type T to our `apply()` which is inferred at
the call-site from `process()`. We still want our map to hold
anything, however in order to do that:

1. when we fetch the value from our map, we need to cast it to T in
   order to prevent compiler errors
2. if we do that, but then we call our `apply()` method with the same
   key but with a different `process()`, then we can end up
   with `ClassCastExceptions`, so to prevent it we need to make the
   return type of `process()` part of our key

In Java this is hard to do, because in Java the generic types get
erased at compile-time and so aren't available at runtime. Scala has
the same behavior, however in Scala we can specify that we want the
[Manifest](http://www.scala-lang.org/api/current/index.html#scala.reflect.Manifest)
for our type T, which will give us access to the erasure of T at
runtime. Thus we can use the name of type T in composing our key. 

So these will now work with no cast exceptions, because the 2 calls to
cache will work on 2 different keys:

{% highlight scala %}
// key will be ... some-string::String
val value1: String = cache("some-string") { "Hello world!" }

// key will be ... some-string::Int
val value2: Int = cache("some-string") { 92 }
{% endhighlight %}

## Concurrency

I also mentioned concurrency. Can instances of this class be used in a
multi-threaded context safely?

Yes definitely, with maybe a small adjustment to our map
declaration. It's better if we marked our map as being `@volatile`,
because when writing a value to a volatile variable, the write creates
a memory fence that ensures the value will get published to other
threads immediately (instead of being cached in a processor registry
or something):

{% highlight scala %}
class NaiveCache {
  @volatile
  private[this] var map = Map.empty[String, Any]
  
  //...
{% endhighlight %}

Something is a little off. Between fetching the key from the map and
updating the map with a new value, we can have a race problem, so in
case of multiple threads pounding on this cache you can end up with
waisted resources from threads doing duplicate work, although this may
be acceptable for a cache.

The advantages of this implementation are:

- simple to understand
- no locking anywhere, for anything; the logic is completely non-blocking
- our cache never gets into an inconsistent state

Try doing that with a `java.util.HashMap`. The difference here is that
this Map is completely immutable. Its internal state can never be
corrupted by multiple threads reading and writing to it because on
every write a new Map is created and the reference to the old one gets
replaced. Replacing that reference is also an atomic operation.

So this means:

- we've got no lock overhead (only a volatile variable that's much
  cheaper than a synchronization block)
- we've got no lock contention to speak of 
- we've got no deadlocks possible

But say you want to fix the race condition that results in duplicate
effort. Many developers would just use the Java Monitor Pattern and
`synchronize` the whole `apply()` method. But this means that multiple
threads won't be able to read from this cache in parallel, which in
the case of a cache I don't think it's an acceptable trade-off.

In Java you can solve this by using a
[ReentrantReadWriteLock](http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/locks/ReentrantReadWriteLock.html).
This is a pair of 2 locks that you can acquire, one for reads and one
for writes. So by using it you can ensure that you can have multiple
threads reading from your datastructure, but when you want to make
writes then you acquire the `writeLock()`, which blocks every other
threads that synchronize on the same lock from both reading or
writing, until the writeLock gets released. And this is perfectly
acceptable for a cache.

However when using an immutable Map, you don't need a
`ReentrantReadWriteLock`. Our original code can just use a simple
mutex only on writes:

{% highlight scala %}
class SafeCache {
  @volatile
  private[this] var map = Map.empty[String, Any]
  private[this] val lock = new AnyRef

  def apply[T: Manifest](key: String)(process: => T): T = {
    val fullKey = key + "::" + manifest[T].toString

    map.get(fullKey) match {
      case Some(value) => 
	    value.asInstanceOf[T]
		
      case None => lock.synchronized {	  
	    // we can have a race condition here, so if the key is already
	    // present when the lock is acquired, then do nothing else
		
	    if (map.contains(fullKey))
          return apply(key)(process)

        val value = process
        map += (fullKey -> value)
        value
      }
    }
  }
}
{% endhighlight %}

Interesting to note that reading is still completely
non-blocking. This would be in contrast with using a
`ReentrantReadWriteLock`, which would block all reads from happening
when a thread acquires the `writeLock()`.

Aren't immutable data-structures great? And you can do the same thing
in Java. Checkout the
[immutable collections](https://code.google.com/p/guava-libraries/wiki/ImmutableCollectionsExplained)
from [Google's Guava](https://code.google.com/p/guava-libraries/).

## Non-blocking Results : Futures

When people talk about parallelism and concurrency on top of Scala,
they talk about Actors and [Akka](http://akka.io/). Akka is great for
actors-based concurrency, but that's not what I want to talk about. 

In our case I want to make the call to `apply()` non-blocking, after
all we might deal with potentially expensive computations here.

Akka provides
[Futures and Promises](http://doc.akka.io/docs/akka/2.0.1/scala/futures.html)
which is a very light and very composable way of specifying concurrent
operations. These are soon to be integrated within the Scala standard
library (in version 2.10).

First, we'll use these imports from Akka:

{% highlight scala %}
import akka.dispatch.{ExecutionContext, Await, Future}
import akka.util.duration._
{% endhighlight %}

Then our class now becomes:

{% highlight scala %}
class CachedFuture(implicit val ec: ExecutionContext) {
  @volatile
  private[this] var map = Map.empty[String, Future[Any]]
  private[this] val lock = new AnyRef

  def apply[T: Manifest](key: String)(process: => T): Future[T] = {
    val fullKey = key + "::" + manifest[T].toString

    map.get(fullKey) match {
      case Some(future) => 
        future.asInstanceOf[Future[T]]

      case None => lock.synchronized {
        if (map.contains(fullKey))
          return apply(key)(process)

        val future = Future(process)
        map += (fullKey -> future)
        future
      }
    }
  }
}
{% endhighlight %}

The differences are:

- instead of processing the value and storing the result, we are
  creating and storing a Future reference  
- the constructor of our class now takes an implicit parameter that
  references an `ExecutionContext`, under which these Futures will get
  executed (think of Futures as Thread instances, with the
  ExecutionContext being responsible for starting those threads)
  
Here's a main method for testing:

{% highlight scala %}
object Main extends App {
  implicit val ec = ExecutionContext.fromExecutorService(
    Executors.newCachedThreadPool())

  val cachedFuture = new CachedFuture

  // this is now non-blocking
  val future = cachedFuture("some-key") {
     Thread.sleep(100)
	 "Hello world!"
  }
  
  // we now block for a result
  val greeting: String = Await.result(future, 3.seconds)

  println(greeting)

  // the threads created by the execution context above are foreground
  // threads, so they'll block the main thread from exiting (you can fix this,
  // but I chose not to for simplicity)
  
  ec.shutdown()
}
{% endhighlight %}

This example isn't much, however the greatest thing about Futures is
that these objects behave like collections, responding to filter, map
and flatMap. So these objects are composable.

Here's something you can do:

{% highlight scala %}
  val responses = List.fill(10000) {
    cachedFuture(random.nextInt(1000).toString) {
      Thread.sleep(100)
      random.nextInt(1000)
    }
  }

  val futureSum = Future.sequence(responses).map(_.sum)
  
  println(Await.result(futureSum, 10 seconds))
{% endhighlight %}

We are creating 10,000 (cached) futures, that return random integers
from 0 to 1000.

We then create another future that's the combination of those 10,000
futures we've created, with its result being a List of Integers. Well,
after we apply map on it, the result will be the sum of those 10,000
integers.

And then we block until the result is available.

Did I mention that futures are collections that respond to `filter`,
`map` and `flatMap`? This means you can also do something like this:

{% highlight scala %}
  val word1 = cachedFuture("word-1") {
    Thread.sleep(1000)
    "Hello"
  }
  val word2 = cachedFuture("word-2") {
    Thread.sleep(1000)
    "World!"
  }

  val concatenate = for {
    w1 <- word1
    w2 <- word2
  } yield w1 + " " + w2

  println(Await.result(concatenate, 2.seconds))
{% endhighlight %}

This is just a small and dumb example, but the possibilities for
composing concurrent tasks are awesome.  And this API is also
available for Java:
[Futures (Java)](http://doc.akka.io/docs/akka/2.0.2/java/futures.html).

Everything I described is possible within Java and Java 8 should make
things more interesting. But I love how easy and intuitive Scala makes
this.
