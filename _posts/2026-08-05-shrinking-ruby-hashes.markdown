---
layout: post
title:  "Shrinking Ruby Hashes"
date:   2026-08-05 08:28:51 +0200
categories: ruby performance
---

As you may know, one area of Ruby performance optimization that particularly interests me is memory usage.
Given that most Ruby deployments rely on `fork`, improving Copy-on-Write performance is generally where you get the
biggest bang for your buck, but that only helps with the somewhat static part of an application heap.

A significant contributor to memory usage is also the transient memory that is allocated during a request or job
cycle and released soon after.
As such, it's also interesting to keep an eye out for opportunities to make various Ruby objects smaller.

And the Ruby object type that's probably the biggest contributor to memory usage is likely `Hash`.
`Hash` instances are extremely common in Ruby applications and libraries, from option hashes and keyword arguments
to logging and API responses.

They're so convenient that they're perhaps overused sometimes, especially since they're really not very memory-efficient.

So let's dig into how much memory they use, a bit of history of how we got there, and what we could do about it.

### Measuring Memory Usage

If you've read some of my previous posts, you are probably already familiar with the Ruby APIs that allow digging into memory usage.

The simplest one is `ObjectSpace.memsize_of(obj)`:

```ruby
>> Ruby::DESCRIPTION
=> "ruby 4.0.6 (2026-07-14 revision 03b6d3f889) +PRISM [arm64-darwin25]"
>> require 'objspace'
>> ObjectSpace.memsize_of({})
=> 160
```

So this tells us that an empty hash uses `160` bytes.
But without a comparison point, that doesn't mean much, so let's compare them to say, `Struct`:

```ruby
require 'objspace'
puts "Ruby: #{RUBY_VERSION}"

11.times do |size|
  struct_class = size.zero? ? Object : Struct.new(*size.times.map { |i| :"m_#{i}" })
  struct = ObjectSpace.memsize_of(struct_class.new)
  hash = ObjectSpace.memsize_of(Hash[size.times.map { |i| [i, i] }])
  diff = (hash.to_f / struct).round(1)
  puts "size: #{size} \tstruct: #{struct} \thash: #{hash} \tdiff: #{diff}x"
end
```

Which gives us:

```
Ruby 4.0.6
size:  0 	struct:  40 	hash: 160 	diff: 4.0x
size:  1 	struct:  40 	hash: 160 	diff: 4.0x
size:  2 	struct:  40 	hash: 160 	diff: 4.0x
size:  3 	struct:  40 	hash: 160 	diff: 4.0x
size:  4 	struct:  80 	hash: 160 	diff: 2.0x
size:  5 	struct:  80 	hash: 160 	diff: 2.0x
size:  6 	struct:  80 	hash: 160 	diff: 2.0x
size:  7 	struct:  80 	hash: 160 	diff: 2.0x
size:  8 	struct:  80 	hash: 160 	diff: 2.0x
size:  9 	struct: 160 	hash: 544 	diff: 3.4x
size: 10 	struct: 160 	hash: 544 	diff: 3.4x
```

As you can see, `Hash` uses 2 to 4 times as much memory as `Struct`[^1] or a Plain Old Ruby Object (PORO).

I could almost say to stop using `Hash` when a PORO could do and leave it at that (and that would be good advice),
but the goal is to dig into why `Hash` uses so much memory and what we can do about it.

If you studied hash tables, that may seem like I'm stating the obvious here.
Of course, hash tables need more memory!

What may be less obvious is that in the above example, up until size 8, Ruby's `Hash` instances aren't exactly hash tables.

So let's actually look at the implementation and its history to understand how we got here.

### Open Addressing

The `Hash` implementation changed a lot over Ruby's lifetime.

The first major change I remember was when the Hash-table implementation [was entirely rewritten by Vladimir Makarov for Ruby 2.4](https://bugs.ruby-lang.org/issues/12142).
It then changed from a more traditional design to [open-addressing](https://en.wikipedia.org/wiki/Open_addressing).
I'm not going to dig into the differences much, there are much better sources than me on that, but the thing I'll point out, though,
is that while that change made hashes noticeably faster, it also significantly increased the "header" size.

By header, I mean the C struct that keeps track of the entries and bins.
Prior to the change, the `st_table` struct was `48B`:

```c
#include <stdio.h>
#include <limits.h>

struct st_hash_type;
struct packed_entry;
struct st_table_entry;

typedef unsigned long long st_index_t;
#define ST_INDEX_BITS (sizeof(st_index_t) * CHAR_BIT)

struct st_table {
    const struct st_hash_type *type;
    st_index_t num_bins;
    unsigned int entries_packed : 1;
    st_index_t num_entries : ST_INDEX_BITS - 1;
    union {
      struct {
          struct st_table_entry **bins;
          void *private_list_head[2];
      } big;
      struct {
          struct st_packed_entry *entries;
          st_index_t real_entries;
      } packed;
    } as;
};

int main(int argc, char **argv)
{
    fprintf(stderr, "sizeof(struct st_table) = %ld\n", sizeof(struct st_table));
    return 0;
}
```

```
sizeof(struct st_table) = 48
```

In Vladimir's initial patch, that struct grew to `88B`, but after some improvements, when the patch actually
landed, it only grew to `64B`, and then [in a follow-up commit](https://github.com/ruby/ruby/commit/5714a26b90b40846733fb2a5764d4c61285f5ea1),
Nobu shrunk it down further to `56B`.

```c
#include <stdio.h>

struct st_hash_type;
struct st_table_entry;

typedef unsigned long long st_index_t;

struct st_table {
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
    struct st_table_entry *entries;
};

int main(int argc, char **argv)
{
    fprintf(stderr, "sizeof(struct st_table) = %ld\n", sizeof(struct st_table));
    return 0;
}
```

```
sizeof(struct st_table) = 56
```

Still an extra `16B` of baseline usage, yet, since the new implementation used some smart, dynamically sized integer offsets.
So, overall, small Hash memory usage was mostly decreased.

Back in Ruby 2.3, it was:

```
Ruby 2.3.8
size: 0 	struct:  40 	hash:  40 	diff: 1.0x
size: 1 	struct:  40 	hash: 232 	diff: 5.8x
size: 2 	struct:  40 	hash: 232 	diff: 5.8x
size: 3 	struct:  40 	hash: 232 	diff: 5.8x
size: 4 	struct:  72 	hash: 232 	diff: 3.2x
size: 5 	struct:  80 	hash: 232 	diff: 2.9x
size: 6 	struct:  88 	hash: 232 	diff: 2.6x
size: 7 	struct:  96 	hash: 552 	diff: 5.8x
size: 8 	struct: 104 	hash: 600 	diff: 5.8x
size: 9 	struct: 112 	hash: 648 	diff: 5.8x
size: 10 	struct: 120 	hash: 696 	diff: 5.8x
```

And in 2.4:

```
Ruby: 2.4.10
size: 0 	struct:  40 	hash:  40 	diff: 1.0x
size: 1 	struct:  40 	hash: 192 	diff: 4.8x
size: 2 	struct:  40 	hash: 192 	diff: 4.8x
size: 3 	struct:  40 	hash: 192 	diff: 4.8x
size: 4 	struct:  72 	hash: 192 	diff: 2.7x
size: 5 	struct:  80 	hash: 288 	diff: 3.6x
size: 6 	struct:  88 	hash: 288 	diff: 3.3x
size: 7 	struct:  96 	hash: 288 	diff: 3.0x
size: 8 	struct: 104 	hash: 288 	diff: 2.8x
size: 9 	struct: 112 	hash: 480 	diff: 4.3x
size: 10 	struct: 120 	hash: 480 	diff: 4.0x
```

A very positive change, but that's still, a lot of memory for a few keys.

### Array Tables

The next major change to hashes happened in Ruby 2.6.
Yimin Zhao, a Google Summer of Code student mentored by Koichi Sasada worked on adapting hashes to work with [the "transient heap"](https://bugs.ruby-lang.org/issues/14858).
The ticket description explains the concept pretty well, but in short, the idea is that since "most objects die young",
and that at the time, most Ruby objects would allocate extra memory with `malloc`. As such, allowing young objects to allocate memory
from a simple bump pointer allocator would speed things up.

Then at the end of GC marking, any surviving object pointing at memory allocated in the transient heap would reallocate it using the real `malloc` and copy the bytes over.
After that, the entire transient heap could be reset and re-used for free.
This is very much inspired by copying garbage collectors, but adapted for the Ruby VM of the time.

It's in that context that Yimin Zhao started working on [integrating the transient heap with Ruby's Hash](https://bugs.ruby-lang.org/issues/14989),
the problem, though, was that `st_table` isn't the backing structure for the `Hash` class. It is also used as a generic hash table
across the virtual machine, and is even exposed in Ruby's C API, so some extensions make use of it.

As such, refactoring it to use the transient heap would have been very tricky.
Instead, Yimin and Koichi took another approach.
They instrumented various benchmarks and [saw that 80% of the hashes had 8 entries or less](https://docs.google.com/spreadsheets/d/1xAjO_qb5K49aLnvk8SypGwO5Avtbm2X12cYb1d-n6Xs/edit?gid=0#gid=0).
For small tables like this, you don't necessarily need a real hash table, even a linear search of `O(n)` complexity
can beat a `O(1)` hash-table lookup when `N` is small enough.

Based on that idea, they refactored Ruby's `Hash` class to essentially be an Array up to 8 entries.
Now instead of always being backed by an `st_table`, [hashes up to size 8 were now backed by an `li_table`](https://github.com/ruby/ruby/commit/8f675cdd00e2c5b5a0f143f5e508dbbafdb20ccd#diff-d5867d8e382e49f5cdef27a4d24c1a4588954f96e00925092a586659bf1b1ba4R743-R744),
for "linear table", [later renamed `ar_table`](https://github.com/ruby/ruby/commit/e4c79d0d10429ac7d48641a66091f0292d807a9d) for "array table".

At that point, each entry would be composed of a key, a value, and a recorded hashcode, for a total of `24B` per entry, so `192B` plus
the base `40B` overhead for all objects, so `232B`:

```
Ruby: 2.6.10
size: 0 	struct: 40 	hash: 232 	diff: 5.8x
size: 1 	struct: 40 	hash: 232 	diff: 5.8x
size: 2 	struct: 40 	hash: 232 	diff: 5.8x
size: 3 	struct: 40 	hash: 232 	diff: 5.8x
size: 4 	struct: 72 	hash: 232 	diff: 3.2x
size: 5 	struct: 80 	hash: 232 	diff: 2.9x
size: 6 	struct: 88 	hash: 232 	diff: 2.6x
size: 7 	struct: 96 	hash: 232 	diff: 2.4x
size: 8 	struct: 104 	hash: 232 	diff: 2.2x
size: 9 	struct: 112 	hash: 928 	diff: 8.3x
size: 10 	struct: 120 	hash: 928 	diff: 7.7x
```

So small hashes became bigger again.

### Lower Byte Hashcode

But thankfully, in Ruby 2.7, [Koichi managed to shrink them again](https://bugs.ruby-lang.org/issues/15602) using a neat trick.
Instead of storing one `8B` hash code per entry, he changed it to only store the lower byte of the hash code.
The tradeoff is that this increases the collision probability to `0.39%` (`1/256`), but for such small tables,
that is likely already good enough.
The added upside is that it makes scanning the table much faster, and that it saves `7B` per entry, so `56B` per table,
quite a sizeable saving:

```
Ruby: 2.7.8
size: 0 	struct: 40 	hash: 40 	diff: 1.0x
size: 1 	struct: 40 	hash: 168 	diff: 4.2x
size: 2 	struct: 40 	hash: 168 	diff: 4.2x
size: 3 	struct: 40 	hash: 168 	diff: 4.2x
size: 4 	struct: 72 	hash: 168 	diff: 2.3x
size: 5 	struct: 80 	hash: 168 	diff: 2.1x
size: 6 	struct: 88 	hash: 168 	diff: 1.9x
size: 7 	struct: 96 	hash: 168 	diff: 1.8x
size: 8 	struct: 104 	hash: 168 	diff: 1.6x
size: 9 	struct: 112 	hash: 928 	diff: 8.3x
size: 10 	struct: 120 	hash: 928 	diff: 7.7x
```

### Variable Width Allocation

The next notable change after was the generalization of Variable Width Allocation (VWA) in Ruby 3.3[^2].
Prior to it, all Ruby objects were allocated as a fixed-size `40B` slot, and if they needed to store
anything more, each object would have to allocate more using the system `malloc`.

VWA changed that to allow multiple different sizes, all multiples of `40` following `40 * 2 ^ n`, so `40`, `80`, `160`, `320` and `640`,
which is perfect for small hashes backed by `ar_table`.

Instead of the `ar_table` being stored in a buffer allocated with `malloc`, it could now be stored inline in the slot,
saving the pointer that used to point at the `malloc` buffer, making it exactly `160B`:

```
Ruby: 3.3.7
size: 0 	struct: 40 	hash: 160 	diff: 4.0x
size: 1 	struct: 40 	hash: 160 	diff: 4.0x
size: 2 	struct: 40 	hash: 160 	diff: 4.0x
size: 3 	struct: 40 	hash: 160 	diff: 4.0x
size: 4 	struct: 80 	hash: 160 	diff: 2.0x
size: 5 	struct: 80 	hash: 160 	diff: 2.0x
size: 6 	struct: 80 	hash: 160 	diff: 2.0x
size: 7 	struct: 80 	hash: 160 	diff: 2.0x
size: 8 	struct: 80 	hash: 160 	diff: 2.0x
size: 9 	struct: 160 	hash: 544 	diff: 3.4x
size: 10 	struct: 160 	hash: 544 	diff: 3.4x
```

In reality, this saved a bit more than `8B` per hash, because `ObjectSpace.memsize_of` only reports the amount of memory that was requested by Ruby to `malloc`.
Various allocators have different strategies, but in general, there is some memory lost because of padding, and also some extra metadata the allocator has to keep.
So it's safe to assume the saving was in reality at least `24B` or even `32B` per hash.

The additional benefit is that previously, when the GC would find an unused Hash, it would need to call `free()` to release the allocated memory.
With VWA, since there's no external buffer, the GC can just mark that slot as free and move on, which is dramatically faster.

With `Hash` and many other types being refactored to use Variable Width Allocation, the transient heap became mostly useless
and was [entirely removed by Peter Zhu in Ruby 3.3](https://bugs.ruby-lang.org/issues/19730).

One notable downside, though, is that empty hashes are now 4 times bigger than they used to be, as they always reserve
a slot big enough to fit an `ar_table`. Gain some, lose some...

### Going Smaller

Recently, I started thinking whether we could make these small hashes even more compact.
In theory, `Hash` instances have a base footprint of `24B`: `16B` for the generic Ruby object header,
and`8B` for the default value (called `ifnone`).

Then hashes backed by an `ar_table` need an extra fixed `8B` to store the hash code.

From there, each entry in the hash needs `16B`: `8B` for the key reference, and `8B` for the value reference.
So the theoretical memory footprint of small hashes should be `32 + size * 2`:

```
Ruby: imaginary version
size: 0 	struct: 40 	hash:  32
size: 1 	struct: 40 	hash:  48
size: 2 	struct: 40 	hash:  64
size: 3 	struct: 40 	hash:  80
size: 4 	struct: 80 	hash:  96
size: 5 	struct: 80 	hash: 112
size: 6 	struct: 80 	hash: 128
size: 7 	struct: 80 	hash: 144
size: 8 	struct: 80 	hash: 160
size: 9 	struct: 160 	hash: 544
size: 10 	struct: 160 	hash: 544
```

But of course it's not that easy because of two limitations.

First, we have to round up to a size offered by the GC (`40`, `80`, `160`, ...).

Second, since you can append keys to a hash, it needs to be able to transition into an `st_table` if it runs out of space,
and as mentioned previously, a `st_table` is `56B`, so `24 + 56 => 80`, a Hash instance can't possibly be smaller than 80B.

But is that really the case?

### Knock-On Effects

Optimizations are fundamentally an iterative process.
An optimization that is impossible or too complex one day can become possible or even easy as the surrounding environment changes, and recently, quite a few things have changed in Ruby.

First, Ruby `4.1.0dev` now has more fine-grained slot sizes.
In addition to the existing power of `40` sizes, [thanks to Matthew Valentine-House](https://github.com/ruby/ruby/pull/16282),
it now offers powers of `32` all the way up to `1024`, allowing us to be much closer to the theoretical ideal I listed above.

The second recent change, I already mentioned in passing on this blog when I talked about a generic instance variable,
and how I shrunk the `set_table` struct by `8B`, well `set_table` is essentially a copy-paste of `st_table`, so I applied
the same optimization to it so they'd stay in sync, meaning `st_table` since Ruby 4.0 is actually `48B`.
Hence, if we could shave off another `8B`, `st_table` backed hashes could use a `64B` slot, which in turn would allow to
optimistically allocate `ar_table` backed hashes with only a capacity of 2, and they'd still be able to transition to an `st_table`
if needed.

And a third recent change in `4.1.0dev` is that [Peter Zhu made a significant change to object shapes](https://github.com/ruby/ruby/pull/17572),
so that now they always include the size of the slot, directly in the object itself.
It's a change I wanted to make for a long time, because while it has always been possible to query an object's slot size,
up until that refactoring, it was quite costly, hence it was better avoided in hotspots.
That being said, strings would frequently go through that codepath, but thanks to Peter's change,
[I was able to get rid of that and get a nice 8% speedup on micro-benchmarks](https://github.com/ruby/ruby/pull/17730).

### Data Locality

The reason querying an object's slot size was slow was mostly due to data locality.
The slot size was stored in the page metadata, and to get there, you first needed to find its address at the beginning
of the page.

```c
# gc/default.c

struct heap_page_header {
    struct heap_page *page;
};

struct heap_page {
    // snip...
    uint64_t slot_size_reciprocal;
};
```

The problem is that processors have become incredibly fast at crunching data, but memory latency hasn't progressed
as rapidly.
So when the processor needs to read data that isn't already in its cache, it has to stall for what is comparatively a very long time.

Wheras now that the information is inside the object header, it is almost guaranteed to already be in the processor cache,
given CPU cache lines are generally 64B (`x86_64`) or `128B` (Apple Silicon).
Making it practically free to access because we most likely already read some other information that is stored right next to it, like the object's type.

But that's enough of a digression.

### Immutability

The first idea I had to start tackling this problem with baby steps was that, given that the main blocker for allocating
hashes in smaller GC slots was the risk that they may need to grow, why not start with frozen hashes?

Since they're immutable, there are no concerns whatsoever that they may run out of space.
The downside of course is that they're not quite as frequent, but they're probably still more frequent than you think.

For instance:

```ruby
some_method({ a: 1, b: 2 })
```

In that example, the hash passed as an argument is mutable, so we can't optimize it.
However, there is a second, "hidden" hash in that snippet of code that you're probably not seeing, and it is frozen.

Let's disassemble this code:

```ruby
puts RubyVM::InstructionSequence.compile('some_method({ a: 1, b: 2 })').disasm
== disasm: #<ISeq:<compiled>@<compiled>:1 (1,0)-(1,27)>
0000 putself                                                          (   1)[Li]
0001 duphash                                {a: 1, b: 2}
0003 opt_send_without_block                 <calldata!mid:some_method, argc:1, FCALL|ARGS_SIMPLE>
0005 leave
```

As you can see, Ruby generated bytecode with the `duphash` instruction.
As its name suggests, it makes a copy (`dup`) of an already existing hash, and this hash is hidden from user space.
You can never get a reference to it, even with `ObjectSpace.each_object`.
The best you can do is somehow find it in the `ObjectSpace.dump_all` output.

Another case where such optimization could be performed would be literal hashes that are explicitly frozen, such as:

```ruby
DEFAULT_OPTIONS = {
  host: "localhost",
  port: 3456,
}.freeze
```

If we disassemble that expression:

```ruby
>> puts RubyVM::InstructionSequence.compile('{ a: 1, b: 2 }.freeze').disasm
== disasm: #<ISeq:<compiled>@<compiled>:1 (1,0)-(1,21)>
0000 opt_hash_freeze                        {a: 1, b: 2}, <calldata!mid:freeze, argc:0, ARGS_SIMPLE>(   1)[Li]
0003 leave
```

We can see the `opt_hash_freeze` instruction being used.
That's [an optimization I added two years ago with Étienne Barrié](https://bugs.ruby-lang.org/issues/20684), similar
in concept to frozen string literals.
When the compiler sees that you are immediately freezing an entirely literal hash, instead of making a copy of the hidden
hash and then freezing it, it "reveals" the hidden hash, and pushes it on the stack directly with no zero allocations nor copies.

Of course, I wouldn't expect even these two cases to amount to a massive amount of memory, but in my view, even a small gain is welcome,
granted it is easy to implement.
But more importantly, starting with frozen hashes would be a good way to validate the idea and start refactoring the code
to no longer assume all `ar_table` have a fixed size, before moving to harder changes.

[The patch itself was relatively simple](https://github.com/ruby/ruby/pull/16653), but I merged it before the new `32` based
pool sizes were introduced.
Running the associated benchmark on the current Ruby `4.1.0dev` now shows that frozen hashes
can be as small as `64B` or even `32B` for empty hashes:

```ruby
require "objspace"

p ObjectSpace.memsize_of({}.freeze) # => 32
p ObjectSpace.memsize_of({a: 1}.freeze) # => 64
p ObjectSpace.memsize_of({a: 1, b: 2}.freeze) # => 64
p ObjectSpace.memsize_of({a: 1, b: 2, c: 3}.freeze) # => 80
p ObjectSpace.memsize_of({a: 1, b: 2, c: 3, d: 4}.freeze) # => 96
p ObjectSpace.memsize_of({a: 1, b: 2, c: 3, d: 4, e: 5, }.freeze) # => 128
p ObjectSpace.memsize_of({a: 1, b: 2, c: 3, d: 4, e: 5, f: 6}.freeze) # => 128
p ObjectSpace.memsize_of({a: 1, b: 2, c: 3, d: 4, e: 5, f: 6, g: 7}.freeze) # => 160
p ObjectSpace.memsize_of({a: 1, b: 2, c: 3, d: 4, e: 5, f: 6, g: 7, h: 8}.freeze) # => 160
```

### Struct Packing

Now, if I wanted to allow the same optimization for mutable hashes, I needed to shrink `st_table` further so
that `st_table` backed hashes would also fit in `64B` slots, otherwise we could never cross the `80B` barrier,
making the optimization much less interesting.

As mentioned before, they were already theoretically `72B`, so all I needed to find was `8B`, and I had a few different ideas.
Let's look at the structs again:

```c
struct RHash {
    struct RBasic basic;    // 16B
    const VALUE ifnone;     // 8B
};

struct st_table {
    unsigned char entry_power, bin_power, size_ind; // 1B each, so 3B
    unsigned int rebuilds_num;                      // 4B
    const struct st_hash_type *type;                // 8B
    st_index_t num_entries;                         // 8B
    st_index_t entries_start, entries_bound;        // 8B each, so 16B
    struct st_table_entry *entries;                 // 8B
};
```

My first idea was to move the hash default value (`ifnone`) elsewhere.
Very few hashes have a default value, so I kind of find this wasteful to have this `8B` overhead in all hashes.

One possibility would have been to make it an instance variable of the hash, but as I explained in
[my previous post about generic instance variables](/ruby/performance/2025/08/11/unlocking-ractors-generic-variables.html),
the instance variables of most types, like `Hash`, are stored in a global table that requires synchronization between
ractors, so turning `ifnone` into an instance variable would have meant that default hash values would become a contention point for Ractors.
That felt too much like running some of my previous hard work, so I shelved the idea.
Perhaps one day the generic instance variables table will be contention-free, and then this idea will be more palatable.

My second idea was to get rid of `st_table.type`.
This member is a pointer to two functions, one for the hash function to use, the other for the comparison function to use.
In the case of `st_table` backed hashes, that pointer is always one of two values.
99% of the time, it's the default Ruby object hashing function, and in rare cases, if for "identity hashes" created
with `Hash#compare_by_identity`, it's a different pointer.

So, in theory, it would make a whole lot of sense not to store that in the `st_stable` and instead just pass it as an argument
whenever it is needed.

But here again it isn't that simple, because `st_table` is a public C API, hence it is used by a bunch of native gems,
and changing the API would break all of them.
The only possibility would be to essentially fork `st.h` to have a private internal version used solely by Ruby itself.
But even then, it could cause issues because the Ruby C API exposes the `RHASH_TBL` macro that by contract returns an
`st_table` pointer.
Even currently, when this macro is called on an `ar_table` backed hash, we have to convert it into an `st_table`.
So here again it would be very hard to pull off.

`RHASH_TBL` should really be deprecated and removed, but that would take a long time.

Yet another idea, this time suggested by John Hawthorn, was to use 32-bit integers for `num_entries`, `entries_start` and `entries_bound`.
This would have shrunk the struct by `12B` (more likely `8B` because of alignment rules), but would also have limited
hashes to ~4 billion entries.

To be fair, such a big hash would require `96GiB` of memory just for the entries list,
I can hardly imagine anyone processing such an amount of data using Ruby hashes, but still, having such an arbitrary limitation felt wrong to me.
And amusingly, [that question was debated 10 years ago already, when Vladimir Makarov submitted his patch](https://bugs.ruby-lang.org/issues/12142#note-17).

That's when, after chatting a bit more with John, we figured we could shrink `entries_start` without restricting the maximum size
of the tables.
But to explain why, I need to explain its purpose.

As mentioned previously, even in `st_table`, entries are stored linearly in the `entries` array:

```c
table->entries = {
  {.hash = 0x12, .key = 0x34, .value = 0x56},
  {.hash = 0x78, .key = 0x90, .value = 0x12},
  // snip...
}
```

This is the reason why Ruby hashes preserve their insertion order, and iterating a Hash with `Hash#each` is (almost)
as simple as iterating over that contiguous array.

Except that you can not only append to hashes, you can also delete from them, and when you do, you create holes
in that contiguous array.

So naively, whenever you would delete from a hash, you'd need to shift all the following entries to cover the gap,
and then update all the offsets.
That would be pretty costly, but more importantly, wouldn't have `O(1)` complexity.

So instead, to know which entries are present and which are deleted, the hash code of the entry is set to a special value (`0`):

```c
/* The reserved hash value and its substitution.  */
#define RESERVED_HASH_VAL (~(st_hash_t) 0)
#define RESERVED_HASH_SUBSTITUTION_VAL ((st_hash_t) 0)

static inline st_hash_t
normalize_hash_value(st_hash_t hash)
{
    /* RESERVED_HASH_VAL is used for a deleted entry.  Map it into
       another value.  Such mapping should be extremely rare.  */
    return hash == RESERVED_HASH_VAL ? RESERVED_HASH_SUBSTITUTION_VAL : hash;
}

/* Macros for marking and checking deleted entries given by their
   pointer E_PTR.  */
#define MARK_ENTRY_DELETED(e_ptr) ((e_ptr)->hash = RESERVED_HASH_VAL)
#define DELETED_ENTRY_P(e_ptr) ((e_ptr)->hash == RESERVED_HASH_VAL)
```

The consequence is that the iteration routine needs to skip over `RESERVED_HASH_VAL`.
In itself, that's not a huge deal, if you somehow delete a lot of elements from the start of the Hash,
which can happen if, for some reason, you are relying on the `Hash#shift` method,
that can cause a lot of extra work for the iteration routine.

It's to speed that case up that `entries_start` was added.
When you delete the first entry, `st` does recompute `entries_start` to save work later:

```c
/* Update the entries start of table TAB after removing an entry
   with index N in the array entries.  */
static inline void
update_range_for_deleted(st_table *tab, st_index_t n)
{
    /* Do not update entries_bound here.  Otherwise, we can fill all
       bins by deleted entry value before rebuilding the table.  */
    if (tab->entries_start == n) {
        st_index_t start = n + 1;
        st_index_t bound = tab->entries_bound;
        st_table_entry *entries = tab->entries;
        while (start < bound && DELETED_ENTRY_P(&entries[start])) start++;
        tab->entries_start = start;
    }
}
```

So this part of the struct isn't strictly required, and if present, it doesn't even need to be 100% accurate as long as it's never higher than it should be.
In other words, it doesn't need to be able to address the entirety of the `entries` array, and we can reasonably assume that it almost never really goes very high.

As such, it would be fine to make it smaller.
Worst-case scenario, for the odd hash with lots of deleted entries, it would be a little bit slower until the hash is rebuilt the next time another entry is inserted into it.

However, you might think that shrinking it isn't enough.
After all, I needed to reclaim a full `8B`, so making it smaller wouldn't work.

Well, if you haven't noticed, when I showed the `st_table` struct above, there was some free unused space in it:

```c
struct st_table {
    unsigned char entry_power, bin_power, size_ind; // 1B each, so 3B
    unsigned int rebuilds_num;                      // 4B
```

Here, the `3` `unsigned char` are followed by an `unsigned int`.
CPUs have all sorts of restrictions on which addresses they can read, called [alignment rules](https://en.wikipedia.org/wiki/Data_structure_alignment).
They vary a bit from one architecture to another, but the core of the idea is that you can't just read a word of a given size at any address.
The address must match the size of the memory being read.
For instance, `8B` values (like pointers) must be aligned on `8B`, meaning their address must be divisible by `8`.
Otherwise, depending on the CPU, it will either not work at all or be slower than it could be.

That's why compilers will sometimes "pad" structs, as in insert implicit holes between the different members.

In the case above, `rebuilds_num` is `4B` large and on most archs needs to be aligned to `4`.
But before it, we have `3` `1B` large members, so the compiler inserts a `1B` padding of essentially wasted space there.

So by making `entries_start` `1B` large, and moving it alongside `size_ind`, we actually save a full `8B`:

```c
struct st_table {
    unsigned char entry_power, bin_power, size_ind, entries_start; // 1B each, so 4B
    unsigned int rebuilds_num;                                     // 4B
    const struct st_hash_type *type;                               // 8B
    st_index_t num_entries;                                        // 8B
    st_index_t entries_bound;                                      // 8B
    struct st_table_entry *entries;                                // 8B
};
```

That's almost the whole patch.
The only extra complication was to add a condition to ensure `entries_start` will cap
at `255` rather than to overflow.
You can see [the full patch](https://github.com/ruby/ruby/pull/18149).

With that fairly small change, `st_table` backed hashes now fit in `64B` slot, making it more attractive to
make it possible to allocate `ar_table` hashes in smaller slots.

Could we go further and fit in a `40B` slot though?
As mentioned above, getting rid of `ifnone` and `type` could be doable in the long term if instance variables improve
and some C APIs are deprecated.
That would only leave 8 more bytes to find, which I'm sure is doable.

But for now, I think it's too soon.
We'll have to wait for some other parts of the codebase to evolve first.

### Dynamic AR Table Sizes

But either way, fitting `st_table` inside a `40B` slot would only allow for shrinking empty hashes.
Other than that, `64B` is enough to be as close as possible to the optimal size.

The problem now is that ever since `ar_table` was introduced, all the code has been written around it with that fixed size in mind.
So now there's a bit of a refactoring to do, to make it dynamic and update some assumptions.

I have [an unfinished patch](https://github.com/byroot/ruby/commit/e351efe54f5660f38192cb04f5f33fc19d25aa08) that still
has a bunch of bugs that I need to iron out, but it does work as expected on the happy path:

```
Ruby: ruby 4.1.0dev (2026-08-05T19:51:05Z hash-dynamic-ar-bo.. 095039c7b1) +PRISM [arm64-darwin25]
size: 0 	struct:  32 	hash:  64 	diff: 2.0x
size: 1 	struct:  32 	hash:  64 	diff: 2.0x
size: 2 	struct:  40 	hash:  64 	diff: 1.6x
size: 3 	struct:  64 	hash:  80 	diff: 1.3x
size: 4 	struct:  64 	hash:  96 	diff: 1.5x
size: 5 	struct:  64 	hash: 128 	diff: 2.0x
size: 6 	struct:  80 	hash: 128 	diff: 1.6x
size: 7 	struct:  80 	hash: 160 	diff: 2.0x
size: 8 	struct:  96 	hash: 160 	diff: 1.7x
size: 9 	struct:  96 	hash: 448 	diff: 4.7x
size: 10 	struct: 128 	hash: 448 	diff: 3.5x
```

The flip side of the coin, however, is that when starting from a smaller slot, hashes need to evacuate to a `st_table`
sooner, hence ending up using more memory than 160B, but also no longer being fully embedded, hence requiring some
extra work to be reclaimed by the garbage collector.

For instance, in the following case:

```ruby
hash = {a: 1, b: 2, c: 3}       # 80B
hash[:d] = 4
p ObjectSpace.memsize_of(hash)  # 176B
```

The hash starts in an `80B` slot, but then needs to transition and end up allocating an `entries` array of capacity 4,
for a total of `176B`.
So, as it's often the case, this optimization saves memory in places, but increases memory usage in others.
The hard question is whether it's a gain or a loss overall.
And that's the sort of question that is very hard, if not impossible to answer with certainty, given that it really depends on what
the code you are running looks like.
Even if it's positive for the vast majority of users, there will certainly be some pathological cases in someone's codebase
that causes it to be negative overall.

That being said, even in the worst case, the degradation really isn't that bad, given that this mostly concerns small hashes.

Usually, we answer this type of question by measuring the impact on [the `ruby-bench` suite](https://github.com/ruby/ruby-bench/),
which does include a couple of real-world applications, so that gives us some confidence in the results,
but aside from the rare cases where the results are really strikingly good, this sort of work always end up on a leap of faith.

### Takeaways

I still have roughly 4 months to  polish, measure, and merge that last patch, so hopefully it ships with Ruby 4.1.0 in December,
and it may yield some decent memory savings for Ruby users.

But regardless, I also think this post was a good occasion to showcase how expensive hashes actually are.
As Ruby developers, we tend to overuse them a bit.
There are these convenient sort-of schema-less structs at your fingertips with a dedicated syntax.
But when the structure of the data is known, it's generally preferable to bother defining a Struct or a class with
instance variables as they are way more compact, and way more efficient to acess.

Similarly, the "array of hashes" pattern is really best avoided in performance-sensitive areas,
instead you can often use flat arrays with a mapping of indexes, like in [this Active Record patch](https://github.com/rails/rails/pull/51744) I'm quite proud of.

[^1]: Or `Data.define` which is basically a `Struct` in a trenchcoat.
[^2]: It was first introduced in 3.2, but many types, like hashes, only started benefiting from it in 3.3.