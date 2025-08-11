---
layout: post
title:  "Unlocking Ractors: generic instance variables"
date:   2025-08-11 10:03:51 +0100
categories: ruby performance
---

In two previous posts, I explained that one of the big blockers for Ractors' viability is that while they're supposed
to run fully in parallel, in many cases, they'd perform worse than a single thread because there were numerous codepaths
in the Ruby virtual machine and runtime that were still protected by the global VM lock.

I also explained how I removed two of these contention points, [the `object_id` method](/ruby/performance/2025/04/26/unlocking-ractors-object-id.html),
and [class instance variables](/ruby/performance/2025/05/24/unlocking-ractors-class-variables.html).

Since then, the situation has improved quite drastically, as numerous other contentious points have been either eliminated or reduced by me and my former teammates.
I'm not going to make a post for each of them, as in most cases it boils down to the same [RCU technique](https://en.wikipedia.org/wiki/Read-copy-update)
I explained in the post about class instance variables.

But there's one such contention point I find interesting and that I'd like to write about: the generic instance variables table.

## How Instance Variables Work

As a Ruby user, you are likely familiar with the idea that everything is an object, and that is somewhat true, but that doesn't mean all objects are equal.
I already touched on that subject in some of my previous posts, so I'll do it quickly.

In the context of instance variables, in the Ruby VM you essentially have 3 or 4 types of objects, depending on how you count.

First, you have the "immediates", small integers (`1`), booleans (`true`, `false`), static symbols (`:foo`, but not dynamic symbols like ``"bar".to_sym``), etc.
These are called immediates because they don't actually exist in memory; they don't have an allocated object slot on the heap.  Their reference *is* their value.
In other words, they're just [tagged pointers](https://en.wikipedia.org/wiki/Tagged_pointer).

Hence, they can't have instance variables, and Ruby will treat them as if they were frozen to maintain the illusion of parity with other objects:

```ruby
>> 42.instance_variable_set(:@test, 1)
(irb):2:in 'Kernel#instance_variable_set': can't modify frozen Integer: 42 (FrozenError)
```

Then you have the more regular `T_OBJECT`, for your user-defined classes.
In the case of `T_OBJECT`, instance variables are stored inside the object's slot like an array.
Consider the following object with 3 instance variables:

```ruby
class Foo
  def initialize
    @a = 1
    @b = 2
    @c = 3
  end
end
```

It will fit in the base `40B` object slot.
`16B` is being used for the object's flags and a pointer to its class, and the remaining `24B` is used for the three instance variable references:

| flags    | klass   | @a | @b | @c |
|----------|---------|----|----|----|
| T_OBJECT | 0xffeff | 1  | 2  | 3  |

In some cases, if an instance variable is added later and the slot is full, the Ruby VM may have to allocate a separate memory
region and "spill" the instance variables there, but this is actually fairly rare. The VM keeps track of how many variables
the instances of each class have, so if Ruby ever has to spill, every future instance of that class will be allocated in a larger slot.

The third type of objects are `T_CLASS` and `T_MODULE`. Since that was the topic of my previous post, I'll be quick.
Class instance variables are laid out like for `T_OBJECT` except they're in a "companion" slot.

```ruby
class Foo
  @a = 1
  @b = 2
  @c = 3
end
```

The layout of the class itself stores a reference to that "companion" slot:

| flags   | klass   | obj_fields | ... | ... |
|---------|---------|------------|-----|-----|
| T_CLASS | 0xffeaa | 0xffdddd   |     |     |

And that other slot is laid out exactly like a `T_OBJECT`, except its type is `T_IMEMO` for "Internal Memory":

| flags          | klass   | @a | @b | @c |
|----------------|---------|----|----|----|
| T_IMEMO/fields | 0xffeaa | 1  | 2  | 3  |

That's a type of object that, as a Ruby user, you can't directly interact with, nor even get a reference to; they're basically invisible.
But they are used internally by the VM to store various data in memory managed by the GC instead of using manual memory management with `malloc` and `free`.

And then you have all the other objects. `Hash`, `Array`, `String`, etc.
For these, the space inside the object slot is already used.
For example, a `String` slot is used to store the string `length`, `capacity`, and if it's small enough, the bytes that compose the string itself, otherwise a pointer to a manually allocated buffer.

Yet, Ruby allows you to define any instance variables you want on a string:

```ruby
>> s = "test"
>> s.instance_variable_set(:@test, 1)
>> s.instance_variable_get(:@test)
=> 1
```

To allow this, the VM has an internal hash table, which used to be called the `genivar_tbl`, for Generic Instance Variables Hash-Table, and that I renamed into `generic_fields_tbl_` as part of my work on `object_id`.

I previously explained how this works in [my post about the `object_id`](/ruby/performance/2025/04/26/unlocking-ractors-object-id.html#generic-instance-variables)
method, but I'll reexplain here with a bit more detail, as it's really the core topic.

Once again, I'll use Ruby pseudo-code to make it easier:

```ruby
module GenericIvarObject
  GENERIC_FIELDS_TBL = Hash.new.compare_by_identity

  def instance_variable_get(ivar_name)
    if ivar_shape = self.shape.find(ivar_name)
      RubyVM.synchronize do
        if buffer = GENERIC_FIELDS_TBL[self]
          buffer[ivar_shape.index]
        end
      end
    end
  end
end
```

In that global hash, the keys are the reference to the objects, and the values are pointers to manually allocated buffers.
Inside the buffer, there is an array of references just like in a `T_OBJECT` or a `T_IMEMO/fields`.

This isn't ideal for multiple reasons.

First, having to do a hash-lookup is way more expensive than reading at an offset like we do for `T_OBJECT`, or even chasing a reference
like we do for `T_CLASS` and `T_MODULE`.

But worse, if we're in a multi-ractor scenario, we have to acquire the VM lock for the whole operation.
First, because that hash-table is global and not thread-safe, then because we must ensure that another Ractor can't free that manually allocated buffer while we're reading it[^1].

So now, you probably understand the problem.
Any code that reads or writes an instance variable in an object that isn't a direct descendant of `Object` (actually `BasicObject`) nor `Module` is a contention point for Ractors.

## Surely That Isn't Common?

Before I dig into what can be changed, you may wonder if it even matters.

And it's a very fair question. Developer time isn't unlimited, hence the question of whether it is worth removing a contention
points boil down to how hot a code path it is, and how hard it is to fix it.

When I started looking at this, it was from the angle of `T_STRUCT`.
I wanted the instance variable of `Struct` and `Data` objects
not to be contention points, e.g., it's not that rare to see `Struct` being used as some sort of code generator:

```ruby
Address = Struct.new(:street, :city) do
  def something_else
    @something_else ||= compute_something
  end
end
```

Because `Struct.new` and `Data.define` don't create `T_OBJECT` but `T_STRUCT` objects.
In these, the space inside the slot is used for the declared fields, not for the ivars.

Another pattern I expected was C extensions. When a Ruby C extension needs to expose an API, it uses the `TypedData` API, which allows to create `T_DATA` objects.
But it's not rare for extensions to do as little as possible in C, and to extend that C class with some Ruby.

An example of that is the `trilogy` gem, which [defines a bunch of C methods](https://github.com/trilogy-libraries/trilogy/blob/16667c95e8c2716a16e69e8325d6b0cb615591e2/contrib/ruby/ext/trilogy-ruby/cext.c#L1141-L1153)

```c
RUBY_FUNC_EXPORTED void Init_cext(void)
{
    VALUE Trilogy = rb_const_get(rb_cObject, rb_intern("Trilogy"));
    rb_define_alloc_func(Trilogy, allocate_trilogy);

    rb_define_private_method(Trilogy, "_connect", rb_trilogy_connect, 3);
    rb_define_method(Trilogy, "change_db", rb_trilogy_change_db, 1);
    rb_define_alias(Trilogy, "select_db", "change_db");
    rb_define_method(Trilogy, "query", rb_trilogy_query, 1);
    //...
}
```

But [then augment that C class with Ruby code](https://github.com/trilogy-libraries/trilogy/blob/16667c95e8c2716a16e69e8325d6b0cb615591e2/contrib/ruby/lib/trilogy.rb):

```ruby
class Trilogy
  def initialize(options = {})
    options[:port] = options[:port].to_i if options[:port]
    mysql_encoding = options[:encoding] || "utf8mb4"
    encoding = Trilogy::Encoding.find(mysql_encoding)
    charset = Trilogy::Encoding.charset(mysql_encoding)
    @connection_options = options
    @connected_host = nil

    _connect(encoding, charset, options)
  end
end
```

That's a pattern I really like, as it allows to write less C and more Ruby, so I would have hated having to complexify some C extensions so that they'd perform better under ractors.

Then you have a few classics, [like `ActiveSupport::SafeBuffer`](https://github.com/rails/rails/blob/3235827585d87661942c91bc81f64f56d710f0b2/activesupport/lib/active_support/core_ext/string/output_safety.rb#L19-L73),
which is a subclass of `String` with a `@html_safe` instance variable:

```ruby
module ActiveSupport
  class SafeBuffer < String
    def initialize(str = "")
      @html_safe = true
      super
    end

    # ...snip
  end
end
```

So it's not that rare for code to inherit from core types, and it can end up in hot spots.
Even though I would recommend avoiding it as much as possible, for reasons other than performance, sometimes it's the pragmatic thing to do, so users do it.

## Some Data Points

Regardless, I was quite convinced that improving this code path would be useful and started working on it.
But later on, I was asked to provide some data, so while I'm breaking the chronology here, let me share it with you.

I started by doing my favorite hack in the VM, a good old print gated by an environment variable:

```c
  if (getenv("DEB")) {
      fprintf(stderr, "%s\n", rb_obj_info(obj));
  }
```

Then I modified the [`yjit-bench` suite](https://github.com/Shopify/yjit-bench/) to set `ENV["DEB"] = "1"` at the start
of the benchmarks loops, as I'm more interested in runtime codepaths than in boottime ones.

I then ran the [`shipit`](https://github.com/Shopify/shipit-engine) benchmark while redirecting STDERR to a file:

```bash
$ bundle exec ruby benchmark.rb 2> /tmp/ivar-stats.txt
```

And did some quick number crunching with `irb`:

```ruby
File.readlines("/tmp/ivar-stats.txt", chomp: true).tally.sort_by(&:last).reverse
```

Here are some results. It's a very vanilla Rails 8 application, nothing fancy:

```ruby
[
 ["VM/thread", 4886969],
 ["T_HASH", 229501],
 ["SQLite3::Backup", 122531],
 ["T_STRING", 70597],
 ["xmlDoc", 23625],
 ["T_ARRAY", 9039],
 ["OpenSSL/Cipher", 2800],
 ["xmlNode", 2025],
 ["encoding", 358],
 ["time", 199],
 ["proc", 68],
 ["T_STRUCT", 38],
 ["OpenSSL/X509/STORE", 3],
 ["Psych/parser", 2],
 ["set", 1],
]
```

`T_STRUCT` was there as I expected, but entirely dwarfed by other types.
For the ones that aren't obvious:

- `"VM/Thread"` is literally `Thread` instances.
- `xmlNode` and `xmlDoc` are `nokogiri` objects.
- Anything that doesn't start with `T_`, is a `T_DATA`.

The `T_HASH` I definitely didn't expect, and it wasn't clear where it was coming from. So I did another hack:

```c
if (getenv("DEB") && TYPE_P(obj, T_HASH) && (rand() % 1000) == 0) {
    rb_bug("here");
}
```

The `rb_bug` function causes the RubyVM to abort and print its crash report, which does contain the Ruby level-backtrace.
With that, I figured these were [`Rack::Utils::HeaderHash`](https://github.com/rack/rack/blob/9163ac3f5fac795179f9935e2ba6533a0ca1cf82/lib/rack/utils.rb#L436-L449) instances.

As for the `T_ARRAY`, it seems like it was mostly from [`ActiveSupport::Inflector::Inflections::Uncountables`](https://github.com/rails/rails/blob/bb3ddbf032c3a24c2c94f911c8c5ca9f6939c6d9/activesupport/lib/active_support/inflector/inflections.rb#L33-L37)

And for `"VM/Thread"` it comes from [`ActiveSupport::IsolatedExecutionState`](https://github.com/rails/rails/blob/bb3ddbf032c3a24c2c94f911c8c5ca9f6939c6d9/activesupport/lib/active_support/isolated_execution_state.rb#L7-L8).

All the rest was various `T_DATA` defined by C extensions, like the `trilogy` example I shared.

I ran a few other benchmarks from the `yjit-bench` repo, and often found similar generic instance variable usages.

So to answer the question, while it's not that big of a hotspot, I believe it's used enough to be worth optimizing, especially for `T_DATA`,
and not just because of Ractors.

## Shaped Structs

But as I said, before I got all that data, my sight was set on `T_STRUCT`.
Struct objects are laid out very similarly to `T_OBJECT` except that the space is used for "members" instead of instance variables.

For instance, the following struct:

```ruby
struct = Struct.new(:field_1, :field_2).new(1, 2)
```

Would be laid out as is:

| flags    | klass   | field_1 | field_2 | - |
|----------|---------|---------|---------|---|
| T_STRUCT | 0xbbeaa | 1       | 2       |   |

Hence, my initial idea was that if we were to encode the struct's layout using shapes like we do for instance variables, we'd
be able to collocate members and variables together so that:

```ruby
MyStruct = Struct.new(:field_1, :field_2) do
  def initialize(...)
    super
    @c = 1
end
```

Could be laid out as:

| flags    | klass   | field_1 | field_2 | @c |
|----------|---------|---------|---------|----|
| T_STRUCT | 0xffeaa | 1       | 2       | 3  |

Which would be perfect. Everything would be embedded in the object slot, so we'd have minimal memory usage and access times.

Unfortunately, after putting some more thought into it, I realized that was a major problem with it: complex shapes.
I [previously wrote at length on what complex shapes are](https://railsatscale.com/2023-10-24-memoization-pattern-and-object-shapes/#shape_too_complex), so very quickly,
in the Ruby VM, shapes aren't garbage collected, so if some code generates a lot of different shapes, Ruby will deoptimize the object and use a hash table to store
its instance variables. It also does the same if the program uses all the possible shape slots.

So if `Struct` members were encoded with shapes, we'd need to have many fallback code paths to handle complex structs,
and for some of the struct APIs, that is straight out impossible, because Struct objects can be treated like arrays:

```ruby
>> Struct.new(:a, :b).new(1, 2)[1]
=> 2
```

In such a case, all we have is the member offset, so if the struct was deoptimized into a hash, we wouldn't be able to look up members by index anymore, short of keeping a reverse index, but that's really a lot of extra complexity.
So I abandoned this idea.

## Shape Offset

A few days later, I was brainstorming with Étienne Barrié, and we thought of a simpler solution.
Instead of encoding struct members in shapes, we could introduce a new type of shape to encode at which offset the instance variables start.

As often mentioned, shapes are a tree, so an object with variables `@a -> @b -> @c -> @d`, the shape tree would look like:

```ruby
ROOT_SHAPE
  \- Ivar(name: :@a, index: 0, capacity: 3)
    \- Ivar(name: :@b, index: 1, capacity: 3)
      \- Ivar(name: :@c, index: 2, capacity: 3)
        \- Ivar(name: :@d, index: 3, capacity: 8)
```

With offset shapes, the same instance variable list, but for a struct with two members, would look like:

```ruby
ROOT_SHAPE
  \- Offset(index: index: 1, capacity: 3)
    \- Ivar(name: :@a, index: 2, capacity: 3)
      \- Ivar(name: :@b, index: 3, capacity: 8)
        \- Ivar(name: :@c, index: 4, capacity: 8)
          \- Ivar(name: :@d, index: 5, capacity: 8)
```

Here again, we'd need to handle the case where the Ruby VM ran out of shapes, but at least only the instance variables
would be deoptimized into a hash table, the struct members would still be laid out like an array, saving a ton of complexity.

That being said, while I still think this is a good idea, it's a fairly big project with some uncertainties.
So when I evoked this solution with Peter Zhu, he suggested something much simpler.

## Direct References

The annoying thing with generic instance variables isn't so much that they aren't embedded inside the object's slot, but that to find the companion slot, you need to go through that global hash table.

Of course, if they were embedded, it would mean better data locality, which is good for performance, but that really isn't much compared to the hash-lookup, so a single pointer chase would already be a major win.

Hence, Peter's suggestion was to just use empty space in struct slots to keep a direct reference to the buffer that holds
the instance variables, and since structs are basically fixed-size arrays, we can store that reference right after the
last struct member.

In pseudo-code, it would be more or less:

```ruby
class Struct
  def instance_variable_get(ivar)
    if __slot_capacity__ > size
      self[size].instance_variable_get(ivar)
    else
      # use the generic instance variables table
    end
  end
end
```

That's essentially the same strategy as with classes and modules.

At least on paper, that was quite easy because [a few weeks prior, I had refactored the generic instance variables to use the same underlying managed object as classes](https://github.com/ruby/ruby/pull/13626): `T_IMEMO/fields`.

Once again, [I paired with Étienne Barrié to implement that idea](https://github.com/ruby/ruby/pull/14095), but the resulting PR was way larger and more complex than I had hoped for, because of a lack of encapsulation.

In many places across the VM, when dealing with instance variables, you have a similar big `switch/case` statement with
a branch for each of the 3 or 4 possible types of object layouts.
So making `T_STRUCT` different would mean adding one more code path in all these places, which would leave me with a bad taste in my mouth.

That's why I backtracked a bit and decided to start by [refactoring the generic instance variables table, so that all accesses go through a very small number of functions](https://github.com/ruby/ruby/pull/14107).
After that, all reads and writes to the table went through mostly just two functions, making it the perfect place to specialize the behavior for struct objects.

As a bit of a sidenote, the more I work on the Ruby VM, the more I realize the challenging part isn't to come up with a brilliant idea,
or a clever algorithm, but the sheer effort required to refactor code without breaking everything.
The C language doesn't have a lot of features for abstractions and encapsulation, so coupling is absolutely everywhere.

Anyways, with that refactoring done, I was able to re-implement [the same pull request we did with Étienne, but half the size](https://github.com/ruby/ruby/pull/14129), most of it being just tests, documentation, and benchmarking code.

Now the generic instance variable lookup function looks like this:

```c
VALUE
rb_obj_fields(VALUE obj, ID field_name)
{
    RUBY_ASSERT(!RB_TYPE_P(obj, T_IMEMO));
    ivar_ractor_check(obj, field_name);

    VALUE fields_obj = 0;
    if (rb_shape_obj_has_fields(obj)) {
        switch (BUILTIN_TYPE(obj)) {
          case T_STRUCT:
            if (LIKELY(!FL_TEST_RAW(obj, RSTRUCT_GEN_FIELDS))) {
                fields_obj = RSTRUCT_FIELDS_OBJ(obj);
                break;
            }
            // fall through
          default:
            RB_VM_LOCKING() {
                if (!st_lookup(generic_fields_tbl_, (st_data_t)obj, (st_data_t *)&fields_obj)) {
                    rb_bug("Object is missing entry in generic_fields_tbl");
                }
            }
        }
    }
    return fields_obj;
}
```

When dealing with a `T_STRUCT` and if there's some unused space in the slot, we entirely bypass the `generic_fields_tbl` and `RB_VM_LOCKING`.

And to ensure we don't fall in the fallback path too much, we modified the Struct allocator to allocate a large enough slots for structs
that have instance variables:

```c
static VALUE
struct_alloc(VALUE klass)
{
    long n = num_members(klass);
    size_t embedded_size = offsetof(struct RStruct, as.ary) + (sizeof(VALUE) * n);
    if (RCLASS_MAX_IV_COUNT(klass) > 0) {
        embedded_size += sizeof(VALUE);
    }

    // snip...
}
```

As a result, instance variable accesses in structs are now noticeably faster, even when no ractor is involved:

```
compare-ruby: ruby 3.5.0dev (2025-08-06T12:50:36Z struct-ivar-fields-2 9a30d141a1) +PRISM [arm64-darwin24]
built-ruby: ruby 3.5.0dev (2025-08-06T12:57:59Z struct-ivar-fields-2 2ff3ec237f) +PRISM [arm64-darwin24]
warming up.....

|                      |compare-ruby|built-ruby|
|:---------------------|-----------:|---------:|
|member_reader         |    590.317k|  579.246k|
|                      |       1.02x|         -|
|member_writer         |    543.963k|  527.104k|
|                      |       1.03x|         -|
|member_reader_method  |    213.540k|  213.004k|
|                      |       1.00x|         -|
|member_writer_method  |    192.657k|  191.491k|
|                      |       1.01x|         -|
|ivar_reader           |    403.993k|  569.915k|
|                      |           -|     1.41x|
```

That was a satisfying change.

## Generalizing to Other Types

Now that we had a working pattern, the question was where else could we apply it.

I definitely knew instance variables on `T_STRING` are rather common, given I'm very familiar with `ActiveSupport::SafeBuffer`, so I thought about pulling a similar trick for them.

Unfortunately, what made this possible with `T_STRUCT` is that they are essentially fixed-size arrays.
Which means we know that whatever free space is left in the slot won't ever be needed in the future.

Whereas other types like `T_STRING` and `T_ARRAY` are variable size.
If you start storing a reference in free space at the end of the slot, you then need to be very careful that if the user appends
to the string or array, it won't overwrite that reference. That's much harder to do and probably not worth the extra complexity.

But one of my favorite things with Ruby and Rails is to be able to optimize from both ends.
If some pattern Rails uses isn't very performant, I can try to optimize Ruby, but I can also just change what Rails does.

In the case of `ActiveSupport::SafeBuffer`, all we're storing is just a boolean: `@html_safe = true`, and eventually, if something is appended to the buffer, the flag will be flipped.
But appends into safe buffers are very rare.

Most of the time, `String#html_safe` is only used as a way to tag the string, to indicate that it doesn't need to be escaped when it's later appended into another buffer. In other words, the overwhelming majority of instances never flip that flag.

Based on that knowledge, [I changed that variable to be a negative](https://github.com/rails/rails/pull/55352).
Instead of starting with `@html_safe = true`, we can start with `@html_unsafe = false`, and since referencing an instance
variable that doesn't exist evaluates to `nil`, which is also falsy, we can simply not set the variable at all.

The result made `String#html_safe` twice as fast, even when no Ractor is started:

```
ruby 3.5.0dev (2025-07-17T14:01:57Z master a46309d19a) +YJIT +PRISM [arm64-darwin24]
Calculating -------------------------------------
    String#html_safe (old)     6.421M (± 1.6%) i/s  (155.75 ns/i) -     32.241M in   5.022802s
    String#html_safe          12.470M (± 0.8%) i/s   (80.19 ns/i) -     63.140M in   5.063698s
```

I guess this is a good example of [mechanical sympathy](https://en.wiktionary.org/wiki/mechanical_sympathy)[^2], the more you know about how the tools you are using work, the more effectively you can use them.

And now that I learned about `ActiveSupport::Inflector::Inflections::Uncountables`, I should probably change it in a similar way.

But the one other type that I thought was worth attention to was `T_DATA`.

## TypedData

Until just a few months ago, `T_DATA` slots were fully used; here's the `RTypedData` C struct in Ruby 3.4,
I added some annotations with the size of each field:

```c
struct RTypedData {
    /** The part that all ruby objects have in common. */
    struct RBasic basic; // 16B

    /**
     * This field  stores various  information about how  Ruby should  handle a
     * data.   This roughly  resembles a  Ruby level  class (apart  from method
     * definition etc.)
     */
    const rb_data_type_t *const type; // 8B

    /**
     * This has to be always 1.
     *
     * @internal
     */
    const VALUE typed_flag; // 8B

    /** Pointer to the actual C level struct that you want to wrap. */
    void *data; // 8B
};
```

Just quickly, the first `16B` was used for the common header all Ruby objects share, `8B` was used to store a pointer
to another struct that gives information to Ruby on what to do with this object, for instance, how to garbage collect it.

And then two other `8B` values, one pointing to arbitrary memory a C extension might have allocated, and then `typed_flag`.
If you read the comment associated with `typed_flag`, you may wonder what purpose it can possibly serve.

It's there because `RTypedData` is the newer API for C extensions that was introduced in 2009 by Koichi Sasada.
Historically, when you needed to wrap a piece of native memory in a Ruby object, you'd use the `RData` API, and you had to
supply:

  - A pointer to the memory region.
  - A marking function for the GC.
  - A free function for the GC.

That older, deprecated API is still there today, and you can see the struct that backs it up:

```c
/**
 * @deprecated
 *
 * Old  "untyped"  user  data.   It  has  roughly  the  same  usage  as  struct
 * ::RTypedData, but lacked several features such as support for compaction GC.
 * Use of this struct is not recommended  any longer.  If it is dead necessary,
 * please inform the core devs about your usage.
 *
 * @internal
 *
 * @shyouhei tried to add RBIMPL_ATTR_DEPRECATED for this type but that yielded
 * too many warnings  in the core.  Maybe  we want to retry  later...  Just add
 * deprecated document for now.
 */
struct RData {

    /** Basic part, including flags and class. */
    struct RBasic basic;

    /**
     * This function is called when the object is experiencing GC marks.  If it
     * contains references to  other Ruby objects, you need to  mark them also.
     * Otherwise GC will smash your data.
     *
     * @see      rb_gc_mark()
     * @warning  This  is  called  during  GC  runs.   Object  allocations  are
     *           impossible at that moment (that is why GC runs).
     */
    RUBY_DATA_FUNC dmark;

    /**
     * This function is called when the object  is no longer used.  You need to
     * do whatever necessary to avoid memory leaks.
     *
     * @warning  This  is  called  during  GC  runs.   Object  allocations  are
     *           impossible at that moment (that is why GC runs).
     */
    RUBY_DATA_FUNC dfree;

    /** Pointer to the actual C level struct that you want to wrap. */
    void *data;
};
```

So in various places in the Ruby VM, when you interact with a `T_DATA` object, you need to know if it's a `RTypedData` or a `RData`
before you can do much of anything with it.

That's where `typed_flag` comes in. It's at the same offset in the `RTypedData`struct as the `dfree` pointer in the `RData` struct, and for various reasons, it's impossible for a legitimate C function pointer to be strictly equal to `1`.

That's why `typed_flag` is always `1`, it allows us to check if a `T_DATA` is typed by checking `rdata->dfree == 1`.

Now you might wonder why I'm telling you all of this.
Well, it's because that `typed_flag` field is using `8B` of space to store exactly `1bit` of information, and that has buggered me for several years.

Even though truth be told, the comment is outdated, and the field can also sometimes be `3` as [we piggy-backed on it with Peter Zhu last year to implement embedded TypedData objects](https://railsatscale.com/2025-06-03-implementing-embedded-typeddata-objects/).
But that's still 32 times more than needed, so if someone could think of a better place to store these two bits, that would free and entire `8B` to store a direct reference to the `T_IMEMO/fields`.

## Enter Set Man

Well, it turns out that someone did earlier this year.

Just before the RubyKaigi developer meeting, Jeremy Evans [proposed to turn `Set` into a core class, and to reimplement it in C](https://bugs.ruby-lang.org/issues/21216), and that was accepted.
Later during the conference, he asked me to [review his usage of the RTypedData API](https://github.com/ruby/ruby/pull/13074), and I suggested a bunch of improvements to make `Set`
objects smaller and reduce pointer chasing by leveraging embedded RTypedData objects.

But turns out that there was a bit of an annoying tradeoff here. The `RTypedData` struct is `40B` large, but when used embedded, we recycle the `data` pointer, so it's only `32B` large,
and the `set_table` struct Jememy needed to store is `56B`, for a total of `88B`, which is a particularly annoying number.

Not because of the meaning some distasteful people attribute to it, but because it is just `8B` too large to fit in a standard `80B` GC slot, hence if we marked it as embeded, the footprint would grow from `40 + 56 = 96B` to `160B` with lots of wasted space.

In all honesty, it wasn't a massive problem unless your application is using a massive amount of sets, but it seems that it really bothered Jeremy.

What he came up with a couple of weeks later was that [he moved these two bits of memory into the low bits of `RTypedData.type` and `RData.dmark`](https://github.com/ruby/ruby/pull/13190),
freeing `8B` per embedded TypedData object and allowing `Set` objects to fit in 80B.

Here again, the assumption was that because of alignment rules, the three lower bits of pointers can't ever be set, so we can store our own information in there.

But now, I think [this space could be put to better use to store a reference to a companion `T_IMEMO/fields`](https://github.com/ruby/ruby/pull/14134), so we could skip the global instance variables table.
The problem is that here again it's a matter of tradeoff. We can waste some memory to save some CPU cycles, which is better is really just a judgment call.

Just like this issue bothered Jeremy a few months back, it now bothered me, and I went searching for a way to save another `8B` in `Set` objects.

## Shrinking Set

Hence, I started to stare at the `struct set_table` while frowning my eyebrows in the hope of spotting some redundant or superfluous member I could eliminate:

```c
struct set_table {
    /* Cached features of the table -- see st.c for more details.  */
    unsigned char entry_power, bin_power, size_ind;
    /* How many times the table was rebuilt.  */
    unsigned int rebuilds_num;
    const struct st_hash_type *type;
    /* Number of entries currently in the table.  */
    st_index_t num_entries;
    /* Array of bins used for access by keys.  */
    st_index_t *bins;
    /* Start and bound index of entries in array entries.
       entries_starts and entries_bound are in interval
       [0,allocated_entries].  */
    st_index_t entries_start, entries_bound;
    /* Array of size 2^entry_power.  */
    set_table_entry *entries;
};
```

I was first attracted to the trio of `num_entries`, `entries_start`, and `entries_bound`. All of these are `8B` integers, so if I could eliminate just one of them, I'd be set[^3].

Without being really intimate with the set implementation, I guessed that surely, if you know how many entries you have, you don't need both the offset of the start and end of the entries list.
So in theory, I could just replace every reference to `entries_bound` by `entries_start + num_entries`.

What I do when I experiment with code I'm not fully familiar with, is that I try to prove my assumptions.
Here I wrote a small helper function:

```c
static inline st_index_t
set_entries_bound(const struct set_table *set)
{
    RUBY_ASSERT(set->entries_start + set->num_entries == set->entries_bound);
    return set->entries_bound;
}
```

And then went over the code to replace all the direct accesses to `set->entries_bound` by my helper, and tried to run the test suite to see if that `RUBY_ASSERT` would trip or not.

Well, turns out it wasn't that simple... After seeing the test suite light up like a Christmas tree, I dug into the code
helped by the backtraces in the crash reports, and realized the `entries_bound` doesn't always match the entries' size,
There is even a comment about it in the code:

```c
    /* Do not update entries_bound here.  Otherwise, we can fill all
       bins by deleted entry value before rebuilding the table.  */
```

So that was a bust, and I went back to the drawing board.

After some more staring and eyebrow frowning, I got another idea.

Ruby's hash-tables (Ruby sets are hash-sets) are ordered.
Hence, you can see them as the combination of a regular unordered hash table and a classic array. The hash-table values are just offset into that array.

Here, the hash-table part is the `st_index_t *bins`, and the array part is `set_table_entry *entries`.

Both of these are memory regions allocated with `malloc`, and they are grown and shrunk at the same time when you add or remove elements from the set.

Hence, if we can know how large one of them is, we could allocate both with a single `malloc`, and then access the other by simply skipping over the first one.

In this case, the size of `set_table.bins` is indicated by `set_table.bin_power`:

```c
/* Return size of the allocated bins of table TAB.  */
static inline st_index_t
set_bins_size(const set_table *tab)
{
    return features[tab->entry_power].bins_words * sizeof (st_index_t);
}
```

That's how [with a relatively small patch, I was able to save 8B from `struct set_table`](https://github.com/ruby/ruby/commit/9250ece276bae357a6ac42cb832c67bbfab0eb01),
which could allow us to keep `Set` objects in `80B` slots even if we make embedded `RTypedData` `32B` again.

However, I still need to run some benchmarks to make sure this patch wouldn't degrade set performance significantly.

## Lookup Cache

For some remaining types like `T_STRING`, `T_ARRAY`, or `T_HASH`, it's unlikely we'll ever find spaces in their slots for an extra reference.
So I had another idea to speed up accesses and reduce contention.

The core of the assumption is that whenever we look up the instance variables of an object, there is a high chance that the next lookup will be for the same object.

So what if we kept a cache of the last object we looked up, and its associated `T_IMEMO/fields`?

In pseudo-ruby:

```ruby
module GenericIvarObject
  GENERIC_FIELDS_TBL = Hash.new.compare_by_identity

  def instance_variable_get(ivar_name)
    if ivar_shape = self.shape.find(ivar_name)
      fields_obj = if Fiber[:__last_obj__] == self
        Fiber[:__last_fields__]
      else
        Fiber[:__last_obj__] = self
        Fiber[:__last_obj__] = RubyVM.synchronize do
          GENERIC_FIELDS_TBL[self]
        end
      end

      fields_obj.instance_variable_get(ivar_name)
    end
  end
end
```

Given that the cache is in fiber local storage, we don't need to protect it with a lock.

I have [a draft patch for that idea](https://github.com/ruby/ruby/pull/14132) that I need to polish and benchmark, but I like that it's quite simple.

## Future Work

Ultimately, for the remaining cases, it would be good if the Ruby VM had a proper concurrent-map implementation to allow lock-free lookups into the generic instance variables table.
However, concurrent maps are *hard*, so it might not happen any time soon.

In the meantime, for the more important types like `T_STRUCT` and `T_DATA`, we now have solutions, either already merged or potentially soon to be, and for others, we have a way to reduce how often we look up the table.
And all that improves performance for both single-threaded and multi-ractor applications, so it's a win-win.

My biggest concern with Ractors is that at some point we'd significantly impact single-threaded performance for the benefit of Ractors, so when we find optimizations that improve both use-cases, I'm particularly happy.

[^1]: You might think that since an object can't be visible by more than one ractor unless it is frozen, then this isn't a concern. But actually, since `object_id` is now essentially a memoized instance variable, it can happen.
[^2]: There was [a pretty good talk on that subject](https://www.youtube.com/watch?v=wCOuJB6MEQo) at Euruko 2024.
[^3]: Pun intended.
