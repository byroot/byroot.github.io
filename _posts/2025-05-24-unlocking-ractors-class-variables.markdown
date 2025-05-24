---
layout: post
title:  "Unlocking Ractors: class instance variables"
date:   2025-05-24 10:03:51 +0100
categories: ruby performance
---

In [a previous post about ractors](/ruby/performance/2025/02/27/whats-the-deal-with-ractors.html), I explained why
I think it's really unlikely you'd ever be able to run an entire application inside a ractor, but that they could
still be situationally very useful to move CPU-bound work out of the main thread, and to unlock some parallel algorithm.

But as I mentioned, this is unfortunately not yet viable because there are many known implementation bugs that can lead
to interpreter crashes, and that while they are supposed to execute in parallel, the Ruby VM still has one true global
lock that Ractors need to acquire to perform certain operations, making them often perform worse than the equivalent
single-threaded code.

One of these remaining contention points is class instance variables and class variables, and given it's quite frequent
for code to check a class or module instance variable as some sort of configuration, this contention point can have a very
sizeable impact on Ractor performance, let me show you with a simple benchmark:

```ruby
module Mod
  @a = @b = @c = 1

  def self.compute(count)
    count.times do
      @a + @b + @c
    end
  end
end

ITERATIONS = 1_000_000
PARALLELISM = 8

if ARGV.first == "ractor"
  ractors = PARALLELISM.times.map do
    Ractor.new do
      Mod.compute(ITERATIONS)
    end
  end
  ractors.each(&:take)
else
  Mod.compute(ITERATIONS * PARALLELISM)
end
```

This simplistic micro-benchmark just add three module instance variables together repeatedly.
In one mode it does it serialy in the main thread, and if the `ractor` argument is passed, it does as many loop, but with 8
parallel ractors.
Hence in a perfect world, using the Ractors branch should be close to 8 times faster.

However, if you run this benchmark on Ruby's master branch, this isn't the result you'll get:

```bash
$ hyperfine -w 1 './miniruby --yjit ../test.rb' './miniruby --yjit ../test.rb ractor'
Benchmark 1: ./miniruby --yjit --disable-all ../test.rb
  Time (mean ± σ):     252.4 ms ±   1.2 ms    [User: 250.2 ms, System: 1.6 ms]
  Range (min … max):   249.9 ms … 253.8 ms    11 runs

Benchmark 2: ./miniruby --yjit --disable-all ../test.rb ractor
  Time (mean ± σ):      2.005 s ±  0.013 s    [User: 2.098 s, System: 6.963 s]
  Range (min … max):    1.992 s …  2.027 s    10 runs

Summary
  ./miniruby --yjit ../test.rb ran
    7.94 ± 0.06 times faster than ./miniruby --yjit ../test.rb ractor
```

That's right, instead of being 8 times faster, the branch that uses Ractors ended up being 8 times slower.
This is because to read a module or class instance variables, secondary ractors have to acquire the VM lock,
which is a costly operation in itself, and worse, they end up waiting a lot to obtain the lock.

So what can we do about it?

## Language Semantic

Before we delves into how this lock could be removed or reduced, let's review how class instance variables behave with ractors.

Given that classes are global, their instance variables are too, hence they are essentially global.
Because of this, Ractors can't let you do everything with them, otherwise, it would be a way to work around Ractors isolation.

The first rule is that only the main Ractor is allowed to set class instance variables:

```ruby
class Test
  class << self
    attr_accessor :var
  end
end

Test.var = 1 # works

Ractor.new do
  # works
  p Test.var

  # raises Ractor::IsolationError: can not set instance variables
  # of classes/modules by non-main Ractors
  Test.var = 2
end.take
```

So secondary ractors can read instance variables on classes and modules, but can't write them.

The second rule is that they can only read instance variables on classes if the object stored in that variable is shareable:

```ruby
class Test
  class << self
    attr_accessor :var1, :var2
  end

  @var1 = {}.freeze
  @var2 = {}
end

Ractor.new do
  # works:
  p Test.var1

  # raises Ractor::IsolationError: can not get unshareable values from
  # instance variables of classes/modules from non-main Ractors
  p Test.var2
end.take
```

## Reducing Contention

Usually when dealing with lock contention issues, the first solution is to turn one big lock into multiple finer-grained locks.
In our simplistic benchmark, all ractors are accessing variables on the same module, so that wouldn't help, but we could
assume that in more realistic scenarios, they'd access the variables of many different modules and, hence wouldn't fight as much
for the same one.

But the way I envision Ractors being used in real-world cases, at least initially, is for running small pieces of
code in parallel, with an API approaching futures:

```ruby
futures = []
futures << Ractor.new { fetch_and_compute_prices }
futures << Ractor.new { fetch_and_compute_order_history }
...
futures.map(&:take)
```

As such I actually expect Ractors to commonly access the same module or class variables over and over, so introducing more finely grained locks isn't very enticing.

Another possibility would be to use a [read-write lock](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock),
given only the main ractor can "write" variables, all secondary ractors could acquire the read lock concurrently.
But from previous experience, while read-write locks do allow concurrent read threads not to stall, they're still quite
costly when contented because all threads have to atomically increment and decrement the same value and that isn't good
for the CPU cache.
It's a fine solution when the operation you are protecting is a relatively slow one, but in our case, reading an instance
variable is extremely cheap, so any kind of lock, even an uncontended one, will be disproportionally costly and ruin performance.

That's why the only reasonable solution is to find a way to not use a lock at all.

## How do Instance Variables Work

To understand how we could make instance variables lock-free, we must first understand how they work.
As is now tradition, I'll try to explain it using Ruby pseudo code, starting with instance variable reads:

```ruby
class Module
  def instance_variable_get(variable_name)
    if RubyVM.main_ractor?
      # The main ractor is the only one allowed to write instance variables
      # hence it doesn't need to lock because we know no one else could be
      # concurrently modifying `@shape` or `@fields`
      if field_index = @shape.field_index_for(variable_name)
        @fields[field_index]
      end
    else
      # Secondary ractors must lock the VM even for reads because the main Ractor
      # could be modifying `@shape` or `@fields` concurrently.
      RubyVM.synchronize do
        if field_index = @shape.field_index_for(variable_name)
          value = @fields[field_index]
          raise Ractor::IsolationError unless Ractor.shareable?(value)
          value
        end
      end
    end
  end
end
```

I'm not going to explain how shapes work here, as I already explained it in multiple previous posts.
The only thing you really need to know is that instance variables are stored in a continuous array, and shapes
keep track of the offset at which each variable is stored. They also are immutable, so you can query them concurrently.

As a result, reading an instance variable only amount of querying the shape tree to figure out if that particular variable exists,
and if it does, what its index is. After that, we read the variable at the specified offset in the `@fields` array of the
object.

However, on secondary Ractors, we additionally need to lock the VM to ensure the shape and the fields are consistent,
but that will be clearer once I explain how writing instance variables works.

```ruby
class Module
  def instance_variable_set(variable_name, value)
    raise FrozenError if frozen?
    # The main ractor is the only one allowed to write instance variables
    raise Ractor::IsolationError unless RubyVM.main_ractor?

    RubyVM.synchronize do
      if field_index = @shape.field_index_for(variable_name)
        # The variable already exists, we replace its value
        @fields[field_index] = value
      else
        # The variable doesn't exist, we have to make a shape transition
        next_shape = @shape.add_instance_variable(variable_name)

        if next_shape.capacity > @shape.capacity
          # @fields is full, we need to allocate a larger one
          new_fields = Memory.allocate(size: next_shape.capacity)
          new_fields.replace(@fields) # copy content
          @fields, old_fields = new_fields, @fields

          # The fields array is manually managed memory, so it needs to be freed explicitly
          Memory.free(old_fields)
        end

        @fields[next_shape.field_index] = value
        @shape = next_shape
      end
    end
  end
end
```

As you can see, the fields array has a given size, if we're adding a new instance variable, we may need to allocate
a larger one and swap the two, as well as change the object's shape.

That is why we need to lock the VM, we can't let another ractor read an instance variable while we're doing this because
it would run into all sorts of race conditions:

  - It could be reading inside `old_fields` while we're freeing it, causing a use-after-free bug.
  - It could be reading inside `old_fields` using the new shape, causing an out-of-bounds read.
  - It could be reading inside `new_fields` using the new shape, but before we've written the new value, causing an uninitialized memory read.

Now, if you are not familiar with C, or another low-level programming language, you might be thinking that I'm exaggerating.
After all, updating the shape is the last operation, so surely cases 2 and 3 aren't possible.

Well, I got some bad news...

## Memory Model

Multithreaded programming is tricky, but even more so when allowing multiple threads to read and write the same memory,
because processors have all sorts of caches, hence a variable doesn't only reside in one place in your RAM.

It can also be copied in the CPU L1/L2/etc caches, or even in the CPU registers.
When one thread writes into a variable, it's not immediately visible to all other threads, the write will take a while
to propagate back to the RAM.
Worse, if you write into multiple variables in a specific order, it's not even guaranteed other threads will witness these changes
in the same order.

Let's consider a simple multi-threaded program:

```ruby
Point = Struct.new(:x, :y)

treasure = nil

thread = Thread.new do
  while true
    if treasure
      puts "Treasure is at #{treasure.x.inspect} / #{treasure.y.inspect}"
      break
    end
  end
end

point = Point.new
point.x = 12
point.y = 24
treasure = point

thread.join
```

As a Ruby programmer, you likely expect this program to print `Treasure is at 12 / 24`, and you'd be correct.
After all, we fully initialize the `Point` instance before updating the `treasure` global variable to point to it.

But if we were to write a similar program in C, the output could be any of:

  - `Treasure is at 12 / 24`
  - `Treasure is at nil / 24`
  - `Treasure is at 12 / nil`
  - `Treasure is at nil / nil`

Why? Well, this has to do with [memory models](https://en.wikipedia.org/wiki/Memory_model_(programming)).
In order to optimize your code, compilers sometimes may have to change the order of memory reads and writes.
So for programmers to be able to write correct programs, they need to know what the compiler can and cannot do, and that's
what a language memory model defines. In the case of C, the memory model is very lax, and compilers are allowed to reorder
reads and writes very extensively.

And it's not only about the compilers. CPUs too can reorder read and write operations.
The `x86` (AKA Intel) memory model is quite strict, so it doesn't reorder much, but the `arm64` memory model is much more lax,
so even if your compiler generated the native code in the same order, your CPU could execute them out of order,
giving you unpredictable results.

To work around this problem, C compilers and CPUs provide ["barriers"](https://en.wikipedia.org/wiki/Barrier_(computer_science)).
You can insert them in your code to enforce that reads and write can't be reordered across such barriers, allowing
you to ensure that all threads will observe memory in a consistent way.

## Atomic Write

From a programmer's perspective, it's generally exposed as "atomic" read and write operations, and it's understood by the
compiler and CPU that memory operations cannot be reordered across atomic operations.

So going back to our `instance_variable_set` implementation, we can fix two of the three race conditions by using an atomic
write:

```ruby
class Module
  def instance_variable_set(variable_name, value)
    raise FrozenError if frozen?
    # The main ractor is the only one allowed to write instance variables
    raise Ractor::IsolationError unless RubyVM.main_ractor?

    if field_index = @shape.field_index_for(variable_name)
      # The variable already exists, we replace its value
      @fields[field_index] = value
    else
      # The variable doesn't exist, we have to make a shape transition
      next_shape = @shape.add_instance_variable(variable_name)

      if next_shape.capacity > @shape.capacity
        # @fields is full, we need to allocate a larger one
        new_fields = Memory.allocate(size: next_shape.capacity)
        new_fields.replace(@fields) # copy content
        old_fields = @fields
        # Ensure `@fields` isn't updated before its content has been filled
        Atomic.write { @fields = new_fields }

        # The fields array is manually managed memory, so it needs to be freed explicitly
        Memory.free(old_fields)
      end

      @fields[next_shape.field_index] = value
      @shape = next_shape
    end
  end
end
```

With this simple change, we now guarantee that the new `@fields` will be visible to other threads before the new `@shape` is.

They may still see the old `@shape` with the new `@fields`, but that's acceptable because all the offsets `@shape` may point to
contain the same values. Pretty neat. Now we only need to find a solution for the use-after-free problem.

## Our Friend The Garbage Collector

So our problem is that after we swap the old `@fields` array for the new one, we must free the old array to not leak memory.
But if there is no synchronization, we can't guarantee that another thread doesn't have a reference to the old array in its
registers or caches, so it may try to read from it after it was freed, and that might lead to a segmentation fault.

Hence, we must wait until there's no longer any reference to the old array before freeing it, and if you think about it
that's exactly what a garbage collector does, and lucky for us, Ruby already has one.

So the solution to avoid use-after-free is to use an actual Ruby `Array` instead of manually allocated memory,
this way we no longer have to free it explicitly, the garbage collected will take care of it later:

```ruby
class Module
  def instance_variable_set(variable_name, value)
    raise FrozenError if frozen?
    # The main ractor is the only one allowed to write instance variables
    raise Ractor::IsolationError unless RubyVM.main_ractor?

    if field_index = @shape.field_index_for(variable_name)
      # The variable already exists, we replace its value
      @fields[field_index] = value
    else
      # The variable doesn't exist, we have to make a shape transition
      next_shape = @shape.add_instance_variable(variable_name)

      if next_shape.capacity > @shape.capacity
        # @fields is full, we need to allocate a larger one
        new_fields = Array.new(next_shape.capacity)
        new_fields.replace(@fields) # copy content
        old_fields = @fields
        # Ensure `@fields` isn't updated before its content has been filled
        Atomic.write { @fields = new_fields }
      end

      @fields[next_shape.field_index] = value
      @shape = next_shape
    end
  end
end
```

Now, if another thread is currently reading inside the old `@fields`, it doesn't matter because it will remain valid
memory until the garbage collector notices it's no longer referenced by anyone.

And just like that, we now have fully lock-free class instance variable reads and writes!

Well... no. Because we overlooked two complications.

## Removing Instance Variables

Perhaps you don't know about it, because it's quite a rare thing to do, but in Ruby, you can remove an object's instance variables:

```ruby
class Test
  p instance_variable_defined?(:@foo) # => false
  @foo = 1
  p instance_variable_defined?(:@foo) # => true

  remove_instance_variable(:@foo)
  p instance_variable_defined?(:@foo) # => false
end
```

And while this is an extremely rare operation, it can happen, hence we must handle it in a thread safe way.

Let's look at its pseudo-implementation:

```ruby
class Module
  def remove_instance_variable(variable_name)
    removed_index = @shape.field_index_for(variable_name)

    # The variable didn't exist in the first place
    return unless removed_index

    next_shape = @shape.remove_instance_variable(variable_name)

    # Shift fields left
    removed_index.upto(next_shape.fields_count) do |index|
      @fields[index] = @fields[index + 1]
    end

    @shape = next_shape
  end
end
```

So when removing an instance variable, we get a new shape that is shorter than the previous one, which means that
all the variables indexed after the one we removed are now lower, so we need to shift all the fields.

To better illustrate, consider the following code:

```ruby
@a = 1
@b = 2
@c = 3
remove_instance_variable(:@b)
```

In the snippet above, `@fields` will change from `[1, 2, 3]` to `[1, 3]`, and that's not really possible to do this in a thread-safe way.

We could, of course, do this shifting in a copy of `@fields`, and then swap `@fields` atomically, but one major problem would remain: the old shape and the new
shape are fundamentally incompatible.

If you are accessing `@c` using the old fields with the new shape, you will get `2` which is incorrect.

If you are accessing `@c` using new fields with the old shape, you will get whatever is outside the array, or perhaps a segmentation fault.

So in this case, we can't rely on clever ordering of writes to keep a consistent view of the instance variables for all ractors.

For the anecdote, this isn't how the initial implementation of object shapes in Ruby worked.

Early in Ruby 3.2 development, `#remove_instance_variable` wouldn't produce a shorter shape, but instead
a child shape of type `UNDEF` that would record that the variable at offset `1` needs to be considered not defined.

However it was found that this could cause an infinite amount of shapes to be created by misbehaving code:

```ruby
obj = Object.new
loop do
  obj.instance_variable_set(:@foo)
  obj.remove_instance_variable(:@foo)
end
```

So instead [the implementation was changed to rebuild the shape tree](https://github.com/ruby/ruby/pull/6866).

That previous implementation would have been useful in this case, as it would have prevented this race condition.
But ultimately it doesn't matter, because there is another complication I didn't mention.

## Complex Shape

The other major complication I deliberately overlooked in my explanation thus far, is the existence of complex shapes.

Since shapes are append-only, Ruby code that defines instance variables in random order or often removes instance variables
can potentially generate an infinite combination of shapes, and each shape uses some amount of memory.

That's why Ruby keeps track of how many shape variations a given class causes, and after a specific threshold (currently 8),
Ruby gives up and marks the class as "too complex".

If you run this script on a recent Ruby, you will see a performance warning:

```ruby
Warning[:performance] = true

class TooComplex
  def initialize
    10.times do |i|
      instance_variable_set("@iv_#{i}", i)
      remove_instance_variable("@iv_#{i}")
    end
  end
end

TooComplex.new
```

```
/tmp/complex.rb:6: warning: The class TooComplex reached 8 shape variations,
instance variables accesses will be slower and memory usage increased.
It is recommended to define instance variables in a consistent order,
for instance by eagerly defining them all in the #initialize method.
```


When this happens, any operation on an instance of that class that would result in a new shape being created instead results
in some sort of "singleton" shape, known as the complex shape, and in that case instance variables are stored in a Hash
instead of being stored in an array. It's slower and uses more memory, but limits the creation of new shapes.

So the real `#instance_variable_get` and `#instance_variable_set` implementations are more complicated than what I described at the start of the post.
In reality, they look more like this:

```ruby
class Module
  def instance_variable_get(variable_name)
    if @shape.too_complex?
      @fields[variable_name] # @fields is is Hash
    elsif field_index = @shape.field_index_for(variable_name)
      @fields[field_index] # @fields is an Array
    end
  end

  def instance_variable_set(variable_name, value)
    raise FrozenError if frozen?
    # The main ractor is the only one allowed to write instance variables
    raise Ractor::IsolationError unless RubyVM.main_ractor?

    if shape.too_complex?
      return @field_index[variable_name] = value
    end

    if field_index = @shape.field_index_for(variable_name)
      # The variable already exists, we replace its value
      @fields[field_index] = value
    else
      # The variable doesn't exist, we have to make a shape transition
      next_shape = @shape.add_instance_variable(variable_name)

      if next_shape.too_complex?
        new_fields = {}
        @shape.each_ancestor do |shape|
          new_fields[shape.variable_name] = @fields[shape.field_index]
        end

        @fields = new_fields
        @shape = next_shape

        return @fields[variable_name] = value
      end

      if next_shape.capacity > @shape.capacity
        # @fields is full, we need to allocate a larger one
        new_fields = Array.new(next_shape.capacity)
        new_fields.replace(@fields) # copy content
        old_fields = @fields
        # Ensure `@fields` isn't updated before its content has been filled
        Atomic.write { @fields = new_fields }
      end

      @fields[next_shape.field_index] = value
      @shape = next_shape
    end
  end
end
```

And this code is now riddled with race conditions because regular and complex shapes are radically different,
even in the happy path case where we're adding a new instance variable, we might turn `@fields` from an array into
a `Hash`.
So if `@shape` and `@fields` aren't perfectly synchronized together, we might end up trying to access a Hash
like an Array, and vice-versa, which will likely end up in a VM crash.

## 128bit Atomics

One solution could have been to ensure `@shape` and `@fields` are written atomically together, but unfortunately in this case
it isn't really possible.

First, because it would require to write two pointer-sized (64bit) values in a single atomic operation, which is possible
on some modern CPUs using SIMD instruction, but Ruby supports many different platforms, and there is no way all of them
would have support for it.

And second, because the constraint with this is that both fields need to be contiguous.
You can't atomically write two pointer-sized values that are distant from each other.
Semantically you are treating two contiguous 64bit values are a single 128bit one, and for reasons I won't get into here,
`@shape` and `@fields` can't be made contiguous.

## Delegation

That's where it came to me that we could instead bundle the `@shape` and `@fields` in their own GC-managed object,
so that when we have to update both atomically, we can work on a copy and then swap the pointer:

```ruby
class Module
  def instance_variable_get(variable_name)
    @fields_object&.instance_variable_get(variable_name)
  end

  def instance_variable_set(variable_name, value)
    raise FrozenError if frozen?
    # The main ractor is the only one allowed to write instance variables
    raise Ractor::IsolationError unless RubyVM.main_ractor?

    new_fields_object = @fields_object ? @fields_object.dup : Object.new
    new_fields_object.instance_variable_set(variable_name, value)
    Atomic.write { @fields_object = new_fields_object }
  end

  def remove_instance_variable(variable_name)
    raise FrozenError if frozen?
    # The main ractor is the only one allowed to write instance variables
    raise Ractor::IsolationError unless RubyVM.main_ractor?

    new_fields_object = @fields_object ? @fields_object.dup : Object.new
    new_fields_object.remove_instance_variable(variable_name)
    Atomic.write { @fields_object = new_fields_object }
  end
end
```

It really is that trivial. Instead of storing instance variables in the class or module, we store them in a regular `Object`,
and on mutation, we first clone the current state, do our unsafe mutation, and finally atomically swap the `@fields_object` reference.

Of course, doing it exactly like this would cause a huge increase in object allocation, so in the actual code I added lots
of special cases to directly mutate the existing object rather than to copy it when it is safe to do so, but conceptually
this is [exactly what my current patch is doing](https://github.com/byroot/ruby/commit/989bce8eef24c6dc6aeb7495d7c57c4324016e72).

That patch is mostly a proof of concept, in the end, I don't think we should use an actual `T_OBJECT` for various reasons,
but I already have a follow-up patch that replaces it with a `T_IMEMO`, which is an internal type invisible to Ruby users.

With this solution I was able to remove the locks around class instance variables, and now the ractor version
of the micro-benchmark runs almost 3 times faster than the single-threaded version:

```
$ hyperfine -w 1 './miniruby --yjit ../test.rb' './miniruby --yjit ../test.rb ractor'
Benchmark 1: ./miniruby --yjit ../test.rb
  Time (mean ± σ):     166.3 ms ±   1.1 ms    [User: 164.4 ms, System: 1.5 ms]
  Range (min … max):   164.0 ms … 168.5 ms    18 runs

Benchmark 2: ./miniruby --yjit ../test.rb ractor
  Time (mean ± σ):      59.3 ms ±   2.6 ms    [User: 211.4 ms, System: 1.5 ms]
  Range (min … max):    57.9 ms …  67.7 ms    48 runs

Summary
  ./miniruby --yjit ../test.rb ractor ran
    2.80 ± 0.12 times faster than ./miniruby --yjit ../test.rb
```

That's still far from the 8 times faster you might expect, but profiling indicates that it's now a scheduling problem,
which we'll eventually fix too, and it's still over 13 times faster than on Ruby 3.4:

```
$ hyperfine -w 1 'ruby --disable-all --yjit ../test.rb ractor' './ruby --disable-all --yjit ../test.rb ractor'
Benchmark 1: ruby --disable-all --yjit ../test.rb ractor
  Time (mean ± σ):     772.3 ms ±   9.0 ms    [User: 1023.8 ms, System: 1325.6 ms]
  Range (min … max):   759.3 ms … 790.5 ms    10 runs

Benchmark 2: ./ruby --disable-all --yjit ../test.rb ractor
  Time (mean ± σ):      56.8 ms ±   1.4 ms    [User: 205.7 ms, System: 1.6 ms]
  Range (min … max):    55.8 ms …  65.6 ms    50 runs

Summary
  ./ruby --disable-all --yjit ../test.rb ractor ran
   13.59 ± 0.36 times faster than ruby --disable-all --yjit ../test.rb ractor
```

Hopefully, I'll get this merged in the next couple of weeks.

## Won't This Increase Memory Usage?

You may be thinking that this is all well and good, but that using another object to store classes and modules instance
variables in another object will increase Ruby's memory usage.

Well, probably not. Previously the `@fields` memory was managed by `malloc`, and while it depends on which implementation
of `malloc` you are using, most of them will have an overhead of `16B` per allocated pointer, which is exactly the overhead
of a Ruby object.

So overall it shouldn't cause memory usage to increase.

## Cherry On Top

This solution has another incidental benefit, which is that it fixes both a bug and a performance regression recently introduced
when [the new Namespace feature was merged](https://bugs.ruby-lang.org/issues/21311).

Under namespaces, core classes are supposed to have a different set of instance variables, and frozen status, in each namespace,
but this doesn't work well at all with shapes because right now the shape is stored in the object header, hence all objects
including classes and modules, only have a single shape.

By delegating instance variable management to another object, classes can now have one `@fields_object` per namespace,
encompassing both the shape and the fields, hence properly namespace class instance variables.

It wasn't at all a motivation for this change, but it's a nice side effect.
