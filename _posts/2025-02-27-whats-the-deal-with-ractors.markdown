---
layout: post
title:  "What's The Deal With Ractors?"
date:   2025-02-27 09:03:51 +0100
categories: ruby performance
---

I want to write a post about [Pitchfork](https://rubygems.org/gems/pitchfork), explaining where it comes from, why it
is like it is, and how I see its future.
But before I can get to that, I think I need to share my mental model on a few things, in this case, Ractors.

When Ractors were announced 4 or 5 years ago, many people expected we'd quickly see a Ractor-based web server,
some sort of Puma but with Ractors instead of threads.
Yet this still hasn't happened, except for a few toy projects and experiments.

Since this post series is about giving context to Ruby HTTP servers design constraints, I think it makes sense to share
my view on Ractors viability.

## What Are They Supposed to Be?

The core idea of Ractors is relatively simple, the goal is to provide a primitive that allows true in-process parallelism,
while still not fully [removing the GVL](/ruby/performance/2025/01/29/so-you-want-to-remove-the-gvl.html).

As I mentioned in depth in a previous post, operating without a GVL would require synchronization (mutexes) on every
mutable object that is shared between threads.
Ractors' solution to that problem is not to allow sharing of mutable objects between Ractors.
Instead, they can send each other copies of objects, or in some cases "move" an object to another Ractor, which means they
can no longer access it themselves.

This isn't unique to Ruby, it's largely inspired by the [Actor model](https://en.wikipedia.org/wiki/Actor_model), like
the Ractor name suggests, and many languages in the same category as Ruby have a similar construct or are working on one.
For instance, JavaScript has [Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API),
and Python has been working on [subinterpreters](https://peps.python.org/pep-0734/) for a while.

And it's no surprise because it makes total sense from a language evolution perspective.
If you have a language that has prevented in-process parallelism for a long time, a Ractor-like API allows you to introduce (constrained) parallelism in a way that isn't going to break existing code, without having to add mutexes everywhere.

But even in languages that have free threading, shared mutable state parallelism is seen as a major foot gun by many,
and message-passing parallelism is often deemed safer, for instance, channels in Go, etc. 

Applied to Ruby, this means that instead of having a single Global VM Lock that synchronizes all threads,
you'd instead have many Ractor Locks, that each synchronize all threads that belong to a given Ractor.
So in a way, since the Ruby 3.0 release that introduced Ractors, on paper the GVL is somewhat already gone,
even though as we'll see later, it's more subtle than that.

And this can easily be confirmed experimentally with a simple test script:

```ruby
require "benchmark"
Warning[:experimental] = false

def fibonacci(n)
  if n == 0 || n == 1
    n
  else
    fibonacci(n - 1) + fibonacci(n - 2)
  end
end

def synchronous_fib(concurrency, n)
  concurrency.times.map do
    fibonacci(n)
  end
end

def threaded_fib(concurrency, n)
  concurrency.times.map do
    Thread.new { fibonacci(n) }
  end.map(&:value)
end

def ractor_fib(concurrency, n)
  concurrency.times.map do
    Ractor.new(n) { |num| fibonacci(num) }
  end.map(&:take)
end

p [:sync, Benchmark.realtime { synchronous_fib(5, 38) }.round(2)]
p [:thread, Benchmark.realtime { threaded_fib(5, 38) }.round(2)]
p [:ractor, Benchmark.realtime { ractor_fib(5, 38) }.round(2)]
```

Here we use the Fibonacci function as a classic CPU-bound workload and benchmark it in 3 different ways.
First without any concurrency, just serially, then concurrently using 5 threads, and finally concurrently using 5 Ractors.

If I run this script on my machine, I get these results:

```ruby
[:sync, 2.26]
[:thread, 2.29]
[:ractor, 0.68]
```

As we already knew, using threads for CPU-bound workloads doesn't make anything faster because of the GVL, however using Ractors we can benefit from some parallelism.
So this script proves that, at least to some extent, the Ruby VM can execute code in parallel, hence the GVL is not so
global anymore.

But as always, the devil is in the details.
Running a pure function like `fibonacci`, that only deals with immutable integers, in parallel is one thing, running
a full-on web application, with hundreds of gems and a lot of global states, in parallel is another.

## Shareable Objects

Where Ruby ractors are significantly different from most similar features in other languages, is that Ractors share the global
namespace with other Ractors.

To create a `WebWorker` in JavaScript, you have to provide an entry script:

```javascript
myWorker = new Worker("worker.js")
```

WebWorkers are created from a blank slate and have their own namespace, they don't automatically inherit all the constants
defined by the caller.

Similarly, Python's sub-interpreters as defined in PEP 734, start with a clean slate.

So both JavaScript's WebWorker and Python's sub-interpreters have very limited sharing capabilities and are more akin to light subprocesses, but with an API that allows passing each other's objects without needing to serialize them.

Ruby's Ractors are more ambitious than that.
From a secondary Ractor, you have visibility on all the constants and methods defined by the main Ractor:

```ruby
INT = 1

Ractor.new do
  p INT # prints 1
end.take
```

But since Ruby cannot allow concurrent access to mutable objects, it has to limit this in some way:

```ruby
HASH = {}

Ractor.new do
  p HASH # Ractor::IsolationError
  # can not access non-shareable objects in constant Object::HASH by non-main Ractor.
end.take
```

So all objects are divided into shareable and unshareable objects, and only shareable ones can be accessed by secondary ractors.
In general, objects that are frozen, or inherently immutable are shareable as long as they don't reference a non-shareable object.

In addition, some other operations, such as assigning class instance variables aren't allowed from any ractor other than
the main one:

```ruby
Ractor.new do 
  class Foo
    class << self
      attr_accessor :bar
    end
  end
  Foo.bar = 1 # Ractor::IsolationError
  # can not set instance variables of classes/modules by non-main Ractors
end.take
```

So Ractors' design is a bit of a double-edged sword.
On one hand, by having access to all the loaded constants and methods, you don't have to load the same code multiple
times, and it's easier to pass complex objects from one ractor to the other, but it also means that not all code may be
able to run from a secondary ractor.
Actually, a lot, if not most, existing Ruby code can't run from a secondary Ractor.
Something as mundane as accessing a constant that is technically mutable, like a String or Hash, will raise an `IsolationError`,
even if you never attempted to mutate it.

Something as mundane and idiomatic as having a constant with some defaults is enough to make your code not Ractor compatible,
e.g.:

```ruby
class Something
  DEFAULTS = { config: 1 } # You'd need to explictly freeze that Hash.

  def initialize(options = {})
    @options = DEFAULTS.merge(options) # => Ractor::IsolationError
  end
end
```

That's one of the main reasons why a Ractor-based web server isn't really practical for anything more than a trivial application.

If you take Rails as an example, there is quite a lot of legitimate global states, such as the routes, the database schema
cache, or the logger. Some of it could probably be frozen to be accessible by secondary ractors, but for things
like the logger, the Active Record connection pool, and various caches, it's tricky.

To be honest, I'm not even sure how you could implement a Ractor safe connection pool with the current API, but I may
be missing something. Actually, that's probably a good illustration of the problem, let's try to implement a Ractor-compatible connection pool.

## A Ractor Aware Connection Pool

The first challenge is that you'd need to be able to move connections from one ractor to another, something like:

```ruby
require "trilogy"

db_client = Trilogy.new
ractor = Ractor.new { receive.query("SELECT 1") }
ractor.send(db_client, move: true)
p ractor.take
```

If you try that you'll get a `can not move Trilogy object. (Ractor::Error)`.
This is because as far as I'm aware, there is no way for classes implemented in C to define that they can be moved to
another ractor. Even the ones defined in Ruby's core, like `Time` can't:

```ruby
Ractor.new{}.send(Time.now, move: true) # can not move Time object. (Ractor::Error)
```

The only thing C extensions can do is define that a type can be shared between Ractors once it is frozen, using the
`RUBY_TYPED_FROZEN_SHAREABLE` flag, but that wouldn't make sense for a database connection.

A way around this is to encapsulate that object inside its own Ractor:

```ruby
require "trilogy"

class RactorConnection
  def initialize
    @ractor = Ractor.new do
      client = Trilogy.new
      while args = Ractor.receive
        ractor, method, *args = args
        ractor.send client.public_send(method, *args)
      end
    end
  end

  def query(sql)
    @ractor.send([Ractor.current, :query, sql], move: true)
    Ractor.receive
  end
end
```

When we need to perform an operation on the object, we send a message telling it what to do,
and give it our own ractor so it can send the result back.

It really is a huge hack, and perhaps there is a proper way to do this, but I don't know of any.

Now that we have a "way" to pass database connections across ractors, we need to implement a pool.
Here again, it is tricky, because by definition a pool is a mutable data structure, hence it can't
be referenced by multiple ractors.

So we somewhat need to use the same hack again:

```ruby
class RactorConnectionPool
  def initialize
    @ractor = Ractor.new do
      pool = []
      while args = Ractor.receive
        ractor, method, *args = args
        case method
        when :checkout
          ractor.send(pool.pop || RactorConnection.new)
        when :checkin
          pool << args.first
        end
      end
    end
    freeze # so we're shareable
  end

  def checkout
    @ractor.send([Ractor.current, :checkout], move: true)
    Ractor.receive
  end

  def checkin(connection)
    @ractor.send([Ractor.current, :checkin, connection], move: true)
  end
end

CONNECTION_POOL = RactorConnectionPool.new

ractor = Ractor.new do
  db_client = CONNECTION_POOL.checkout
  result = db_client.query("SELECT 1")
  CONNECTION_POOL.checkin(db_client)
  result
end
p ractor.take.to_a # => [[1]]
```

I'm not going to go further, as this implementation is quite ridiculous, I think this is enough to make my point.

For Ractors to be viable to run a full-on application in, Ruby would need to provide at least a few basic data structures
that would be shareable across ractors, so that we can implement useful constructs like connection pools.

Perhaps some `Ractor::Queue`, maybe even some `Ractor::ConcurrentMap`, and more importantly, C extensions
would need to be able to make their types movable.

## What Ractors Could Be Useful For

So while I don't believe it makes sense to try to run a full application inside Ractors, I still think Ractors could be
very useful even with their current limitations.

For instance, in my previous post about the GVL, I mentioned how some gems do have background threads, one example being
[`statsd-instrument`](https://github.com/Shopify/statsd-instrument/blob/6fd8c49d50803bbccfcc11b195f9e334a6e835e9/lib/statsd/instrument/batched_sink.rb#L163),
but there are others like open telemetry and such.

These gems all have a similar pattern, they collect information in memory, and periodically serialize and send it down
the wire. Currently, this is done using a thread, which is sometimes problematic because the serialization part holds
the GVL, hence can slow down the threads that are responding to incoming traffic.

This would be an excellent pattern for Ractors, as they'd be able to do the same thing without holding the main Ractor's
GVL and it's mostly fire and forget.

I only mean this as an example I know well, I'm sure there's more.
The key point is that while Ractors in their current form can hardly be used as the main execution primitive, they can certainly be used for parallelizing lower-level functions inside libraries.

But unfortunately, in practice, it's not really a good idea to do that today.

## Also There Are Many Implementation Issues

If you attempt to use Ractors, Ruby will display a warning:

```
warning: Ractor is experimental, and the behavior may change in future versions of Ruby!
Also there are many implementation issues.
```

And that's not an overstatement.
As I'm writing this article, there are 74 open issues about Ractors.
A handful are feature requests or minor things, but a significant part are really critical bugs such as
segmentation faults, or deadlocks.
As such, one cannot reasonably use Ractors for anything more than small experiments.

Another major reason not to use them even in these cases that are perfect for them, is that quite often, they're not
really running in parallel as they're supposed to.

## One More Global Lock

As mentioned previously, on paper, the true Global VM Lock is supposedly gone since the introduction of Ractors in Ruby 3.0
and instead, each ractor has its own "GVL". But this isn't actually true.

There are still a significant number of routines in the Ruby virtual machine that do lock all Ractors.
Let me show you an example.

Imagine you have 5 millions small JSON documents to parse:

```ruby
# frozen_string_literal: true
require 'json'

document = <<~JSON
  {"a": 1, "b": 2, "c": 3, "d": 4}
JSON

5_000_000.times do
  JSON.parse(document)
end
```

Doing so serially takes about 1.3 seconds on my machine:

```bash
$ time ruby --yjit /tmp/j.rb

real	0m1.292s
user	0m1.251s
sys	0m0.018s
```

As unrealistic as this script may look, it should be a perfect use case for Ractor. In theory, we could spawn
5 Ractors, have each of them parse 1 million documents, and be done in 1/5th of the time:

```ruby
# frozen_string_literal: true
require 'json'

DOCUMENT = <<~JSON
  {"a": 1, "b": 2, "c": 3, "d": 4}
JSON

ractors = 5.times.map do
  Ractor.new do
    1_000_000.times do
      JSON.parse(DOCUMENT)
    end
  end
end
ractors.each(&:take)
```

But somehow, it's over twice as slow as doing it serially:

```bash
/tmp/jr.rb:9: warning: Ractor is experimental, and the behavior may change in future versions of Ruby! Also there are many implementation issues.

real	0m3.191s
user	0m3.055s
sys	0m6.755s
```

What's happening is that in this particular example, JSON has to acquire the true remaining VM lock for each key in
the JSON document.
With 4 keys, a million times, it means each Ractor has to acquire and release a lock 4 million times.
It's almost surprising it only takes 3 seconds to do so.

For the keys, it needs to acquire the GVL because it inserts string keys into a Hash, and as I explained in
[Optimizing Ruby's JSON, Part 6](/ruby/json/2025/01/12/optimizing-ruby-json-part-6.html), when you do that Ruby will
look inside the interned string table to search for an equivalent string that is already interned.

I used the following Ruby pseudo-code to explain how it works:

```ruby
class Hash
  def []=(key, value)
    if entry = find_entry(key)
      entry.value = value
    else
      if key.is_a?(String) && !key.interned?
        if interned_str = ::RubyVM::INTERNED_STRING_TABLE[key]
          key = interned_str
        elsif !key.frozen?
          key = key.dup.freeze
        end
      end

      self << Entry.new(key, value)
    end
  end
end
```

In the above example `::RubyVM::INTERNED_STRING_TABLE` is a regular hash that could cause a crash if it was accessed
concurrently, so Ruby still acquires the GVL to look it up.

If you look at [`register_fstring` in `string.c`](https://github.com/ruby/ruby/blob/d4b8da66ca9533782d2fed9762783c3e560f2998/string.c#L538-L570)
(`fstring` is the internal name for interned strings), you can see the very obvious `RB_VM_LOCK_ENTER()` and
`RB_VM_LOCK_LEAVE()` calls.

As I'm writing this, there are 42 remaining calls to `RB_VM_LOCK_ENTER()` in the Ruby VM, many are very rarely hit and not
much of a problem, but this one demonstrates how even when you have what is a perfect use case for Ractors, besides their constraints,
it may still not be advantageous to use them yet.

## Conclusion

In his RubyKaigi 2023 talk about the state of Ractors, Koichi Sasada who's the main driving force behind them, mentioned that
Ractors suffered from [some sort of a chicken and egg problem](https://youtu.be/Id706gYi3wk?si=DaECpXT2lEMO7kiA&t=878).
By his own admission, Ractors suffer from many bugs, and often don't actually deliver the performance they're supposed to,
hence very few people use them enough to be able to provide feedback on the API, and I'm afraid that almost two years later,
my assessment is the same on bugs and performance.

If Ractors bugs and performance problems were fixed, it's likely that some of the provided feedback would lead to some of their
restrictions to be lifted over time.
I personally don't think they'll ever have little enough restrictions for it to be practical to run a full application inside a Ractor, hence that a Ractor-based web server would make sense, but who knows, I'd be happy to be proven wrong.

Ultimately, even if you are among the people who believe that Ruby should just try to remove its GVL for real rather
than to spend resources on Ractors, let me say that a large part of the work needed to make Ractors perform well,
like a concurrent hash map for interned strings, is work that would be needed to enable free threading anyway, so it's not
wasted.
