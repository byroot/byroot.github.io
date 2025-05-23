---
layout: post
title:  "Unlocking Ractors: object_id"
date:   2025-04-26 11:03:51 +0100
categories: ruby performance
---

In [a previous post about ractors](/ruby/performance/2025/02/27/whats-the-deal-with-ractors.html), I explained why I think it's really unlikely you'd ever be able to run an entire application inside a ractor, but that they could
still be situationally very useful to move CPU-bound work out of the main thread, and to unlock some parallel algorithm.

But as I mentioned, this is unfortunately not yet viable because there are many known implementation bugs that can lead
to interpreter crashes, and that while they are supposed to execute in parallel, the Ruby VM still has one true global
lock that Ractors need to acquire to perform certain operations, making them often perform worse than the equivalent
single-threaded code.

But things are evolving rapidly.
Since then, there is now a team of people working on fixing exactly that: tackling known bugs and eliminating or reducing the remaining contention points.

The one example I gave to illustrate this remaining contention, was the `fstring_table`, which in short is a big internal
hash table used to deduplicate strings, which Ruby does whenever you use a String as a key in a Hash.
Because looking into that table while another Ractor is inserting a new entry would result in a crash (or worse),
until last week Ruby had to acquire the remaining VM lock whenever it touched that table.

But [John Hawthorn recently replaced it with a lock-free Hash-Set](https://bugs.ruby-lang.org/issues/21268), and now this
contention point is gone. If you re-run the JSON benchmarks from the previous post using the latest Ruby master,
the Ractor version is now twice as fast as the single-threaded version, instead of being 3 times slower.

This still isn't perfect though, as the benchmark uses 5 ractors, hence in an ideal world should be almost 5 times faster
then the single-threaded example, so we still have a lot of work to do to eliminate or reduce the remaining contention
points.

One of such remaining contention points, that you likely didn't suspect would be one, is
[the `#object_id` method](https://docs.ruby-lang.org/en/3.4/Object.html#method-i-object_id).
And on my way back from RubyKaigi, I started working on tackling it.

But before we delve into what I plan to do about it, let's talk about how this method came to be a contention point.

## A Little Bit Of History

Up until Ruby 2.6, the `#object_id` implementation used to be quite trivial:

```c
VALUE
rb_obj_id(VALUE obj)
{
    if (STATIC_SYM_P(obj)) {
        return (SYM2ID(obj) * sizeof(RVALUE) + (4 << 2)) | FIXNUM_FLAG;
    }
    else if (FLONUM_P(obj)) {
      return LL2NUM((SIGNED_VALUE)obj);
    }
    else if (SPECIAL_CONST_P(obj)) {
      return LONG2NUM((SIGNED_VALUE)obj);
    }
    return LL2NUM((SIGNED_VALUE)(obj) / 2);
}
```

Of course, it's C so it might be a bit cryptic to the uninitiated, but in short, for the common case of a heap allocated
object, its `object_id` would be the address where the object is stored, divided by two.
So in a way, `#object_id` used to return you an actual pointer to the object.

This made implementing the lesser-known counterpart of `#object_id`, [`ObjectSpace._id2ref`](https://docs.ruby-lang.org/en/2.5.0/ObjectSpace.html#method-c-_id2ref),
just as trivial, multiply the `object_id` by two, and here you go, you now have a pointer to the corresponding object.

```ruby
s = "I am a string"
ObjectSpace._id2ref(s.object_id).equal?(s) # => true
```

But there was actually a major problem with that implementation, which is that the Ruby heap is composed of standard-size slots.
When an object is no longer referenced, the GC reclaims the object slot and will most likely re-use it for a future object.

Hence if you were to hold onto an `object_id`, and use `ObjectSpace._id2ref`, it's not actually certain the object you get
back is the one you got the `object_id` from, it might be a totally different object.

It also meant that if you are holding onto an `object_id` as a way to know if you've already seen a given object,
you may run into some false positives.

That's why [in 2018 there was already a feature request to deprecate both `#object_id` and `_id2ref`](https://bugs.ruby-lang.org/issues/15408).
Back then Matz agreed to deprecated `_id2ref` for Ruby 2.7, but pointed out that removing `#object_id` would be too much of a breaking change,
and that it is a useful API.
However, this somehow fell through the cracks, and `_id2ref` was never formally deprecated, which is [something I'd like to
do for Ruby 3.5](https://github.com/ruby/ruby/pull/13157).

I'm not certain why `_id2ref` was added initially, given that `git blame` points to [a commit from 1999 that was generated by cvs2svn](https://github.com/ruby/ruby/commit/210367ec889).
But if I had to guess, I'd say it was added for `drb` which today remains the only significant user of that API in the stdlib, but [even that is about to change](https://github.com/ruby/drb/pull/35).

## GC Compaction

Regardless of why `_id2ref` was added, that major flaw in its design became a blocker for Aaron Patterson when [he implemented
GC compaction in Ruby 2.7](https://bugs.ruby-lang.org/issues/15626).
Since GC compaction implies that objects can be moved from one slot to another, `#object_id` could no longer be derived from
the object address, otherwise, it wouldn't remain stable.

What Aaron did is conceptually simple:

```ruby
module Kernel
  def object_id
    unless id = ObjectSpace::OBJ_TO_ID_TABLE[self]
      id = ObjectSpace.next_obj_id
      ObjectSpace.next_obj_id += 8
      ObjectSpace::OBJ_TO_ID_TABLE[self] = id
      ObjectSpace::ID_TO_OBJ_TABLE[id] = self
    end
    id
  end
end

module ObjectSpace
  def self._id2ref(id)
    ObjectSpace::ID_TO_OBJ_TABLE[id]
  end
end
```

In short, Ruby added two internal Hash tables. One of them with objects as keys and IDs as values, and the inverse for the other.
Whenever you access an object's ID for the first time, a unique ID is created by incrementing an internal counter,
and the relation between the object and its ID is stored in the two hash tables.

As a Ruby user, you can observe this change easily by printing some `object_id`:

```ruby
p Object.new.object_id
p Object.new.object_id
```

Up to Ruby 2.6, the above code will print some large and seemingly random integers such as `50666405449360`, whereas on
Ruby 2.7 onwards, it will print small integers, likely `8` and `16`.

This change both solved the historical issue with `_id2ref` and allowed the GC to keep stable IDs when moving objects from one
address to the other, but made `object_id` way more costly than it used to be.

Ruby's hash-table implementation stores 3 pointer-sized numbers per entry.
One for the key, one for the value, and one for the hashcode:

```c
struct st_table_entry {
    st_hash_t hash;
    st_data_t key;
    st_data_t record;
};
```

And given every `object_id` is stored in two hash-tables, that makes for a total of `48B` (plus some change) per `object_id`.
That's quite a lot of memory for just a small number.

In addition, accessing the `object_id` now requires doing a hash lookup, when before it was a simple division, and whenever
the GC frees or moves an object that has an ID, it needs to update these two hash-tables.

To be clear, I don't have any evidence that these two tables cause significant memory or CPU overhead in real-world Ruby applications.
I'm just saying that `#object_id` is way more expensive than one might expect.

## Entering Ractors

Then later on, when Koichi Sasada implemented Ractors since now multiple ractors could attempt to access these two hash-tables
concurrently, [he had to add a lock around them in `#object_id`](https://github.com/ruby/ruby/commit/da3438a5045), turning
`#object_id` in a contention point:

```ruby
module Kernel
  def object_id
    RubyVM.synchronize do
      unless id = ObjectSpace::OBJ_TO_ID_TABLE[self]
        id = ObjectSpace.next_obj_id
        ObjectSpace.next_obj_id += 8
        ObjectSpace::OBJ_TO_ID_TABLE[self] = id
        ObjectSpace::ID_TO_OBJ_TABLE[id] = self
      end
      id
    end
  end
end

module ObjectSpace
  def self._id2ref(id)
    RubyVM.synchronize do
      ObjectSpace::ID_TO_OBJ_TABLE[id]
    end
  end
end
```

At this point, you may wonder if it's really a big deal.
After all, `#object_id` is used a bit for debugging, but not so much in actual production code.
And this is mostly true, but it does come up in real-world code, e.g. [in the `mail` gem](https://github.com/mikel/mail/blob/d1d65b370b109b98e673a934e8b70a0c1f58cc59/lib/mail/message.rb#L1698),
[in `rubocop`](https://github.com/rubocop/rubocop/blob/4a611564c4e1d8ec12a8e45e96490465e5141605/lib/rubocop/cop/variable_force/branch.rb#L129-L131),
and of course [quite a bit in Rails](https://github.com/rails/rails/blob/99e27fa586af7db2b5334124a62eb3a464cdffd8/activesupport/lib/active_support/cache/strategy/local_cache.rb#L213-L215).

But calling `Kernel#object_id` isn't the only way you might rely on an object ID.

The [`Object#hash`](https://docs.ruby-lang.org/en/3.4/Object.html#method-i-hash) method for example rely on it:

```c
static st_index_t
objid_hash(VALUE obj)
{
    VALUE object_id = rb_obj_id(obj);
    if (!FIXNUM_P(object_id))
        object_id = rb_big_hash(object_id);

    return (st_index_t)st_index_hash((st_index_t)NUM2LL(object_id));
}

VALUE
rb_obj_hash(VALUE obj)
{
    long hnum = any_hash(obj, objid_hash);
    return ST2FIX(hnum);
}
```

Common value classes such as `String`, `Array` etc, do define their own `#hash` method that doesn't rely on the object ID,
but all other objects that are compared by identity by default will end up using `Object#hash`, hence accessing the `object_id`.

For instance here's a quite class `#hash` implementation from one of Rails classes:

```ruby
#  activerecord/lib/arel/nodes/delete_statement.rb
  def hash
    [self.class, @relation, @wheres, @orders, @limit, @offset, @key].hash
  end
```

It absolutely isn't obvious, but here we're hashing a `Class` object, and classes are indexed by identity like a default object:

```ruby
>> Class.new.method(:hash).owner
=> Kernel
>> Object.new.method(:hash).owner
=> Kernel
```

Hence the above code currently requires to lock the entire virtual machine, just to produce a hashcode.

## Deoptimization

So what could we do to remove or reduce the need to synchronize the entire virtual machine when accessing object IDs?

Well first, given that `ObjectSpace._id2ref` is very rarely used, and will likely be marked as deprecated soon,
we can start by optimistically not creating nor updating the `id -> object` table until someone needs it, which hopefully
won't be the case in the vast majority of programs:

```ruby
module Kernel
  def object_id
    RubyVM.synchronize do
      unless id = ObjectSpace::OBJ_TO_ID_TABLE[self]
        id = ObjectSpace.next_obj_id
        ObjectSpace.next_obj_id += 8
        ObjectSpace::OBJ_TO_ID_TABLE[self] = id
        if defined?(ObjectSpace::ID_TO_OBJ_TABLE)
          ObjectSpace::ID_TO_OBJ_TABLE[id] = self
        end
      end
      id
    end
  end
end

module ObjectSpace
  def self._id2ref(id)
    RubyVM.synchronize do
      unless defined?(ObjectSpace::ID_TO_OBJ_TABLE)
        ObjectSpace::ID_TO_OBJ_TABLE = ObjectSpace::OBJ_TO_ID_TABLE.invert
      end
      ObjectSpace::ID_TO_OBJ_TABLE[id]
    end
  end
end
```

This doesn't remove the lock yet, but assuming your program never calls `ObjectSpace._id2ref` it removes some work
from inside the lock, hence it shouldn't be held as long.
And even if you don't use Ractors, it should slightly reduce memory usage as well as remove work for the GC,
as demonstrated by a micro-benchmark:

```
benchmark:
  baseline: "Object.new"
  object_id: "Object.new.object_id"
```

```
compare-ruby: ruby 3.5.0dev (2025-04-10T09:44:40Z master 684cfa42d7) +YJIT +PRISM [arm64-darwin24]
built-ruby: ruby 3.5.0dev (2025-04-10T10:13:43Z lazy-id-to-obj d3aa9626cc) +YJIT +PRISM [arm64-darwin24]
warming up..

|           |compare-ruby|built-ruby|
|:----------|-----------:|---------:|
|baseline   |     26.364M|   25.974M|
|           |       1.01x|         -|
|object_id  |     10.293M|   14.202M|
|           |           -|     1.38x|
````

As always, when possible, the most efficient way to speed up some code is to not call it if you can avoid it.

If you're curious to see the actual implementation, [you can have a look at the pull request](https://github.com/ruby/ruby/pull/13115).

## Inline Storage

But while saving a bit of memory and CPU is nice, we're still not significantly reducing contention, so what else could we do?

The crux of the issue here is that the `object_id` is stored in a centralized hash table, and as long as it will be the case,
synchronization will be required, short of implementing a lock-free hash table, but this is quite tricky to do.
Much trickier than a hash-set John used for the `fstring_table`.

But more importantly, a centralized data structure to store all the IDs of all objects isn't great for locality anyway.
More so, needing to do a hash lookup to access an object's property is quite costly, when conceptually it should be stored directly
inside the object.

If you think about it, `object_id` isn't very different from an instance variable:

```ruby
module Kernel
  def object_id
    @__object_id ||= ObjectSpace.generate_next_obj_id
  end
end
```

You'd need the id generation to be thread-safe, which is easily done using an atomic increment operation, but other than that,
assuming the object isn't one of the special objects that is accessible from multiple ractors, you can mutate it to store the
`object_id` without having to lock the entire VM.

However, as is tradition, nothing is ever that simple.

## Final Shapes

Since Ruby 3.2, objects use shapes to define how their instance variables are stored.

Here again, let's use some pseudo-Ruby code to illustrate the basics of how they work.

To start, shapes are a tree-like structure. Every shape has a parent (except the root one)
and 0-N children:

```ruby
class Shape
  def initialize(parent, type, edge_name, next_ivar_index)
    @parent = parent
    @type = type
    @edge_name = edge_name
    @next_ivar_index = next_ivar_index
    @edges = {}
  end

  def add_ivar(ivar_name)
    @edges[ivar_name] ||= Shape.new(self, :ivar, ivar_name, next_ivar_index + 1)
  end
end
```

With this, when the Ruby VM has to execute code such as:

```ruby
class User
  def initialize(name, role)
    @name = name
    @role = role
  end
end
```

It can compute the object shape on the fly such as:

```ruby
# Allocate the object
object = new_object
object.shape = ROOT_SHAPE

# add @name
next_shape = object.add_ivar(:@name)
object.shape = next_shape
object.ivars[next_shape.next_ivar_index - 1] = name

# add @role
next_shape = object.add_ivar(:@role)
object.shape = next_shape
object.ivars[next_shape.next_ivar_index - 1] = role
```

This method may seem surprising, but it's actually very efficient for various reasons I won't get into here,
because I wrote [another post about it a bit over a year ago](https://railsatscale.com/2023-10-24-memoization-pattern-and-object-shapes/),
go read it if you are curious to know more.

But how instance variables are laid out isn't the only thing that shapes record. They also keep track of how large an object
is, hence how many instance variables it can store, as well as whether it has been frozen.

Still in pseudo-Ruby code, it looks like this:

```ruby
class Shape
  def add_ivar(ivar_name)
    if @type == :frozen
      raise "Can't modify frozen object"
    end
    @edges[ivar_name] ||= Shape.new(self, :ivar, ivar_name, next_ivar_index + 1)
  end

  def freeze
    @edges[:__frozen] ||= Shape.new(self, :frozen, nil, next_ivar_index)
  end
end
```

So `frozen` shapes are final. It is expected that a shape of type `frozen` won't ever have any children.

But in the case of `object_id`, we want to be able to store the id on any object, regardless of whether they are frozen
or not. So the first step is to modify shapes to allow that, [which I did in a relatively simple commit](https://github.com/Shopify/ruby/commit/ca92bbe4f646658f9a420e61089cf5d6e27a5a71).

But here too there was a bit of a complication. In a few cases, for instance when calling `Object#dup`, Ruby needs to find
the unfrozen version of a shape. Previously, since frozen shapes couldn't possibly have children, it was quite simple:

```ruby
class Object
  def dup
    new_object = self.class.allocate
    if self.shape.type == :frozen
      new_object.shape = self.shape.parent
    else
      new_object.shape = self.shape
    end
    # ...
  end
end
```

Once you allow frozen shapes to have children, this operation becomes more involved, as you now need to go up the tree
to find the last non-frozen shape, then reapply all the child shapes you wish to carry over.

After this small refactoring was done, I could introduce a new type of shape: `SHAPE_OBJ_ID`, which behaves very similarly
to instance variable shapes:

```ruby
class Shape
  def object_id
    # First check if there is an OBJ_ID shape in ancestors
    shape = self
    while shape.parent
      return shape if shape.type == :obj_id
      shape = shape.parent
    end

    # Otherwise create one.
    @edges[:__object_id] ||= Shape.new(self, :obj_id, nil, next_ivar_index + 1)
  end
end
```

And just like this, we're now able to reserve some inline space inside any object to store the `object_id`,
and in *some cases* we're able to access an object's ID fully lock-free.

## Lock Free Shapes

Why I'm saying *in some cases* is because there are still a number of limitations.

First, since shapes are mostly immutable, we can access an object's shape, and all its ancestors without taking a lock.
However, finding or creating a shape's child currently still requires synchronizing the VM.
So even if my patch was to be applied, Ruby would still lock when accessing an object's ID for the very first time,
it would only be lock-free on subsequent accesses.

Being able to find or create child shapes in a lock-free way would be useful way beyond the `object_id` use case, so
hopefully we'll get to it in the future, I haven't yet dedicated much thought to it, but I'm hopeful we can find
a solution. But even if we can't do it lock-free, I think we could at least use a dedicated lock for it, so we wouldn't
contend with all the other code paths that synchronize the entire VM, only paths that do the same operation.

Then, if the object is potentially shared between ractors, we also still need to acquire the lock before storing the ID,
as otherwise, concurrent writes may cause a race condition. Given we need to both update the object's shape and write
the `object_id` inside the object, we can't do it all in an atomic manner.

Finally, not all objects store their instance variables in the same way.

## Generic Instance Variables

As a Rubyist, you likely know that in Ruby everything is an object, but that doesn't mean all objects are equal.

In the context of instance variables, there are essentially three types of objects: `T_OBJECT`, `T_CLASS/T_MODULE` and
then all the rest.

`T_OBJECT` are your classic objects that inherit from the `BasicObject` class. Their instance variables are stored
inline directly inside the object slot, as long as it's large enough. If it ends up overflowing, then a separated memory
location is allocated, and instance variables are moved there, the object slot then only contains a pointer to that auxiliary memory.

`T_CLASS` and `T_MODULE` as their name suggests are all instances of the `Class` and `Module` classes. These are much
larger than regular objects, as they need to keep track of a lot of things, such as their method table, a pointer to the
parent class, etc:

```ruby
>> ObjectSpace.memsize_of(Object.new)
=> 40
>> ObjectSpace.memsize_of(Class.new)
=> 192
```

As such, they never store their instance variables inline, they always store them in auxiliary memory, and they have
dedicated space in their object slot to store the auxiliary memory pointer:

```c
# internal/class.h
struct rb_classext_struct {
    VALUE *iv_ptr; // iv = instance variable
    // ...
}
```

And finally, there are all the other objects, such as `T_STRING`, `T_ARRAY`, `T_HASH`, `T_REGEXP`, etc.
None of these have free space in their slot to store inline variables, and not even space to store the auxiliary memory
pointer.

So what does Ruby do when you do add an instance variable to such objects? Well, it stores it in a Hash-table of course!

In pseudo-Ruby, it would look like this:

```ruby
module GenericIvarObject
  class GenericStorage
    attr_accessor :shape
    attr_reader :ivars

    def initialize
      @ivars = []
    end
  end

  def instance_variable_get(ivar_name)
    store = RubyVM.synchronize do
      GENERIC_STORAGE[self] ||= GenericStorage.new
    end

    if ivar_shape = store.shape.find(ivar_name)
      store.ivars[ivar_shape.next_ivar_index - 1]
    end
  end
end
```

As you probably have noticed or even guessed, since this is yet another global hash table, any access needs to be synchronized,
which means that for objects other than `T_OBJECT`, `T_CLASS` and `T_MODULE`,
my patch replaces one global synchronized hash with another...

So perhaps for these, keeping the original `object -> id` table would be preferable, that's something I still need to figure out.

### Conclusion

My patch isn't finished. I still have to figure out how to best deal with "generic" objects, and probably refine the
implementation some more, and perhaps it won't even be merged at all in the end.

But I wanted to share it because explaining something helps me think about the problem,
and also because while I don't think `object_id` is currently the biggest Ractor bottleneck,
it's a good showcase of the type of work that needs to be done to make Ractors more parallel.

If you are curious about the patch, here's [what it currently looks like as of this writing](https://github.com/ruby/ruby/compare/master...byroot:ruby:object_id-in-shape-snapshot).

Similar work will have to be done for other internal tables, such as the symbol table and the various method tables.

