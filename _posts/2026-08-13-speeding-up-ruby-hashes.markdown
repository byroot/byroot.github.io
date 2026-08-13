---
layout: post
title:  "Speeding Up (small) Ruby Hashes"
date:   2026-08-13 09:30:00 +0200
categories: ruby performance
---

Something I must confess is that I absolutely hate writing these blog posts.
It's not quite as bad as having to give a conference talk, but it's up there on the list of activities that feel like pulling teeth to me.
Not that I'm not proud of the result.
I absolutely am.
But the process of writing them is very painful for me.
It's particularly true of the very first sentence, as the post progresses, it gets a bit easier

Yet, I force myself to do it, because it helps me think about problems, and "compile" knowledge in my head.
I'm so terrified of posting something wrong or inaccurate that I tend to double-check some long-held assumptions,
dig into more details about how some things are implemented, etc.

And very often, quickly after publishing the post, I think of new ideas I previously missed.
This post is about one such idea I had right after publishing the previous one on [shrinking Ruby hashes](/ruby/performance/2026/08/05/shrinking-ruby-hashes.html).
If you haven't read it yet, please do, as this one is a direct continuation.

### AR Tables Aren't Hash Tables

One of the main takeaways from the previous post is that, up to 8 entries, Ruby's `Hash` class isn't truly a Hash Table
as its name would make you think.
Instead, it's literally an array of pairs. 
Let's look at its data structure:

```c
#define RHASH_AR_TABLE_MAX_SIZE SIZEOF_VALUE

typedef unsigned char ar_hint_t;

typedef struct ar_table_pair_struct {
    VALUE key;
    VALUE val;
} ar_table_pair;

typedef struct ar_table_struct {
    union {
        ar_hint_t ary[RHASH_AR_TABLE_MAX_SIZE];
        VALUE word;
    } ar_hint;
    /* 64bit CPU: 8B * 2 * 8 = 128B */
    ar_table_pair pairs[RHASH_AR_TABLE_MAX_SIZE];
} ar_table;
```

C can be a little cryptic to the uninitiated, so let me unpack it:

  - `VALUE` is the Ruby object reference, basically a pointer, so 8 bytes[^1]
  - `ar_hint` is 8 bytes long, and can be interpreted as either an array of 8 bytes, or as a single 8-byte (64-bit) integer.
  - `pairs` is the array containing our key-value pairs.

As I mentioned in the previous post, a `hint` is essentially a single-byte hash-code.
In Ruby, hash-codes are 8 bytes long, and when backed by an `st_table` (the real hash-table implementation),
the entire hash-code is stored and compared.

But to save memory, `ar_table` only stores the lower byte of the hash-code.
Fundamentally, that doesn't change anything, except make hash collisions more likely, but that's an acceptable
tradeoff when we know we never have any more than 8 keys.

If we were to implement `ar_table` in Ruby, the structure for `{a: 1, b: 2, c: 3}` could look like this:

```ruby
class ARTable
  def initialize
    @ar_hint = [0x34, 0x65, 0x72]
    @pairs = [:a, 1, :b, 2, :c, 3]
  end
end
```

Now let's look at the core of the `ar_table` lookup routine, the one I looked at closely while writing the previous post,
but that I never really thought of deeply before then:

```c
// Returns the bin index if found, RHASH_AR_TABLE_MAX_BOUND if not found,
// or RHASH_AR_TABLE_CONVERTED_TO_ST_TABLE if #eql? or a Thread converted the hash to st_table.
static unsigned
ar_find_entry_hint(VALUE hash, ar_hint_t hint, st_data_t key)
{
    for (unsigned i = 0; i < RHASH_AR_TABLE_BOUND(hash); i++) {
        const ar_hint_t *hints = RHASH_AR_TABLE(hash)->ar_hint.ary;
        if (hints[i] == hint) {
            ar_table_pair *pair = RHASH_AR_TABLE_REF(hash, i);
            int eq = ar_equal(key, pair->key);
            if (UNLIKELY(!RHASH_AR_TABLE_P(hash))) {
                return RHASH_AR_TABLE_CONVERTED_TO_ST_TABLE;
            }
            if (eq) {
                return i;
            }
        }
    }
    return RHASH_AR_TABLE_MAX_BOUND;
}
```

As you may be able to see, it's essentially a linear, AKA `O(N)`, search.
We receive the `hint` of the key we're searching for, and linearly search for a match in the table list.

When a match is found, since we have to worry about collisions, we invoke `Object#eql?` (`ar_equal`), and if it returns false,
we continue our search until we reach the end of the array.

This `O(N)` performance can be verified experimentally:

```ruby
require 'benchmark/ips'

ar = {a:1, b:2, c:3, d:4, e:5, f:6, g:7, h:8}.freeze

Benchmark.ips do |x|
  x.report("ar-hit-0") { ar[:a]; ar[:a]; ar[:a]; ar[:a]; ar[:a]; ar[:a]; ar[:a]; ar[:a]; ar[:a]; ar[:a] }
  x.report("ar-hit-1") { ar[:b]; ar[:b]; ar[:b]; ar[:b]; ar[:b]; ar[:b]; ar[:b]; ar[:b]; ar[:b]; ar[:b] }
  x.report("ar-hit-2") { ar[:c]; ar[:c]; ar[:c]; ar[:c]; ar[:c]; ar[:c]; ar[:c]; ar[:c]; ar[:c]; ar[:c] }
  x.report("ar-hit-3") { ar[:d]; ar[:d]; ar[:d]; ar[:d]; ar[:d]; ar[:d]; ar[:d]; ar[:d]; ar[:d]; ar[:d] }
  x.report("ar-hit-4") { ar[:e]; ar[:e]; ar[:e]; ar[:e]; ar[:e]; ar[:e]; ar[:e]; ar[:e]; ar[:e]; ar[:e] }
  x.report("ar-hit-5") { ar[:f]; ar[:f]; ar[:f]; ar[:f]; ar[:f]; ar[:f]; ar[:f]; ar[:f]; ar[:f]; ar[:f] }
  x.report("ar-hit-6") { ar[:g]; ar[:g]; ar[:g]; ar[:g]; ar[:g]; ar[:g]; ar[:g]; ar[:g]; ar[:g]; ar[:g] }
  x.report("ar-hit-7") { ar[:h]; ar[:h]; ar[:h]; ar[:h]; ar[:h]; ar[:h]; ar[:h]; ar[:h]; ar[:h]; ar[:h] }
  x.report("ar-miss ") { ar[:X]; ar[:X]; ar[:X]; ar[:X]; ar[:X]; ar[:X]; ar[:X]; ar[:X]; ar[:X]; ar[:X] }

  x.compare!(order: :baseline)
end
```

```
ruby 4.1.0dev (2026-08-11T14:41:10Z c-api-shareable-co.. ab6b8ceaac) +YJIT +PRISM [arm64-darwin25]

ar-hit-0     14.188M (± 0.8%) i/s   (70.48 ns/i) -     71.928M in   5.069797s
ar-hit-1     13.247M (± 0.3%) i/s   (75.49 ns/i) -     67.139M in   5.068423s
ar-hit-2     11.890M (± 0.4%) i/s   (84.11 ns/i) -     60.027M in   5.048655s
ar-hit-3     11.119M (± 1.7%) i/s   (89.94 ns/i) -     56.290M in   5.062484s
ar-hit-4     10.456M (± 1.5%) i/s   (95.64 ns/i) -     53.056M in   5.074381s
ar-hit-5     10.018M (± 0.4%) i/s   (99.82 ns/i) -     50.464M in   5.037463s
ar-hit-6      9.380M (± 2.9%) i/s  (106.61 ns/i) -     47.215M in   5.033578s
ar-hit-7      8.983M (± 0.6%) i/s  (111.32 ns/i) -     45.158M in   5.026746s
ar-miss       9.163M (± 1.9%) i/s  (109.14 ns/i) -     46.161M in   5.037831s

Comparison:
ar-hit-0: 14187571.6 i/s
ar-hit-1: 13246581.8 i/s - 1.07x  slower
ar-hit-2: 11889678.7 i/s - 1.19x  slower
ar-hit-3: 11118989.8 i/s - 1.28x  slower
ar-hit-4: 10455642.8 i/s - 1.36x  slower
ar-hit-5: 10017678.5 i/s - 1.42x  slower
ar-hit-6:  9379948.0 i/s - 1.51x  slower
ar-miss :  9162845.7 i/s - 1.55x  slower
ar-hit-7:  8983492.7 i/s - 1.58x  slower
```

As expected, looking up the 8th key is noticeably slower than looking up the first one.
When measured from the Ruby side, since there is a fixed cost overhead in the virtual machine dispatch, etc,
so the measured difference is only `~1.5x`, but that's still significant.

Again, given we're only ever dealing with at most 8 entries, an `O(N)` algorithm is fine.
In this specific case, the linear search performance isn't that far from what it would be if the Hash was backed by an `st_table`:

```ruby
require 'benchmark/ips'

ar = {a:1, b:2, c:3, d:4, e:5, f:6, g:7, h:8}.freeze
# Creating  a hash with a capacity > 8 gives us an `st_table`
st = Hash.new(capacity: 9).merge(ar).freeze

Benchmark.ips do |x|
  x.report("ar-hit-7") { ar[:h]; ar[:h]; ar[:h]; ar[:h]; ar[:h]; ar[:h]; ar[:h]; ar[:h]; ar[:h]; ar[:h] }
  x.report("st-hit-7") { st[:h]; st[:h]; st[:h]; st[:h]; st[:h]; st[:h]; st[:h]; st[:h]; st[:h]; st[:h] }

  x.compare!(order: :baseline)
end
```

```
ruby 4.1.0dev (2026-08-11T14:41:10Z c-api-shareable-co.. ab6b8ceaac) +YJIT +PRISM [arm64-darwin25]

ar-hit-7      8.967M (± 1.9%) i/s  (111.52 ns/i) -     45.030M in   5.021777s
st-hit-7     10.574M (± 0.1%) i/s   (94.57 ns/i) -     53.527M in   5.061952s

Comparison:
ar-hit-7:  8966929.4 i/s
st-hit-7: 10574312.0 i/s - 1.18x  faster
```

So using `ar_table` versus `st_table` is your classic space vs time trade-off.

Yet, when most people think about hash tables, they think about `O(1)` access, so it feels slightly wrong.
But what if `ar_table` lookups could be made `O(1)` too?

### SWAR Search

The core of `ar_find_entry_hint` is a loop that searches for a specific integer in an array of 8 such integers.
But if you look at it from another angle, it's searching for a specific byte, AKA character, inside an array of bytes, AKA string, of length 8.

Efficiently searching for characters in strings is something I've done a lot in the `json` gem, and that I've touched on
previously on this blog.
Searching byte by byte in a string is quite wasteful, because the cost of iterating over each byte tends to dwarf the cost of comparing said bytes.

And in our case, we're looking at 8 bytes, so exactly the size of our CPU registers, which makes it a perfect fit for [SWAR](https://en.wikipedia.org/wiki/SWAR), which
stands for SIMD within a register.

I [previously posted a quick explanation of what SIMD](/ruby/json/2025/01/12/optimizing-ruby-json-part-6.html#a-note-on-simd)
(and incidentally SWAR) is, so I'm not gonna repeat it here.

But the crux of the idea is that instead of interpreting `ar_hint` as a list of 8 1-byte long numbers, we can interpret it
as a single 8-byte number, then, as long as we make sure not to overflow, we can perform the same operations on all these bytes all at once.

This is a common enough trick that the Wikipedia article directly showcases how to find a NULL byte inside an 8-byte number:

```c
#include <stdio.h>
#include <stdint.h>

static void has_null_byte(uint64_t word)
{
    uint64_t x7 = (word & 0x7f7f7f7f7f7f7f7f) + 0x7f7f7f7f7f7f7f7f;
    uint64_t x8 = x7 | word;
    uint64_t matches = x8 | 0x7f7f7f7f7f7f7f7f;
    if (~matches) {
        printf("0x%llx has a NULL byte\n", word);
    }
    else {
        printf("0x%0llx does not have a NULL byte\n", word);
    }
}

int main(int argc, char **argv)
{
    has_null_byte(0x1020304050607080);
    has_null_byte(0x1020304000607080);
    return 0;
}
```

```
0x1020304050607080 does not have a NULL byte
0x1020304000607080 has a NULL byte
```

The above example might sound a bit like magic, so let's try to unpack it.

The very first step is `word & 0x7f7f7f7f7f7f7f7f`, or scoped to a byte, `byte & 0x7f` (`127`), or in binary form `byte & 0b01111111`.
In other words, we get rid of the most significant bit of each byte, which is necessary to prevent the next operation from ever overflowing.

Then for each byte we add that same `0x7f` value. The idea is that since `0x7f` is `0b01111111` if the byte contained anything but `0`,
the addition carry will cause the most significant bit to be set to `1`.

e.g.
  - `0b00000000 + 0b01111111 = 0b01111111`
  - `0b00000001 + 0b01111111 = 0b10000000`
  - `0b00000010 + 0b01111111 = 0b10000001`
  - ...

So all the bytes that had any of their 7 lower bits set now have their 8th bit set too.
However, we need to handle `0x70` / `0b10000000` specifically, as it got its most significant bit discarded by the first
bitwise `AND`.

To restore that most significant bit, we do a bitwise `OR` with the original value (`x7 | word`), so if the original byte
was `0x70`, its lifetime would look like this:

```c
0x70 | 0b10000000 // start
0x00 | 0b00000000 // & 0x7f
0x7f | 0b01111111 // + 0x7f
0x8f | 0b11111111 // | 0x70 aka 0b10000000
```

At that point, all bytes except the one that was fully zero now have their most significant bit set,
so to answer the question of whether any of the bytes were originally zero, we can only keep that most significant bit (`x8 | 0x7f7f7f7f7f7f7f7f`),
and then invert (`NOT`, `~matches`) all the bits.

As a result, `0x00` bytes become `0x70` / `0b10000000` and all other bytes become `0x00`, meaning that if none of the 8 bytes were zero
in the first place, then our resulting number is `0`, that's our boolean condition.

Even better, since we know all bits but the most significant ones are always `0`, beyond answering the question of whether at least
one byte was NULL, we can even derive the byte index from the bit index.

### Counting Zeros

Since bitmaps are quite common, CPUs tend to have instructions dedicated to them, such as `ffs` ([Find First Set](https://en.wikipedia.org/wiki/Find_first_set)),
or `ctz` (Count Trailing Zeros) or even `nlz` (Number of Trailing Zeros).

However, here we have to care about [endianness](https://en.wikipedia.org/wiki/Endianness).

If we consider the number `0x1020304050607080`, you might think that when interpreted as an array of bytes, it would be equivalent to:

```c
char ary[8] = { 0x10, 0x20, 0x30, 0x40, 0x50, 0x60, 0x70, 0x80 };
``` 

Which is somewhat natural for a human using a left-to-right language.
After all, that's how we write numbers.

But actually, that would only be true on *big-endian* architectures, which today are very rare.

Most modern computers are *little-endian*, which essentially means that they store and read numbers right-to-left,
so the array above would, in reality, be:

```c
char ary[8] = { 0x80, 0x70, 0x60, 0x50, 0x40, 0x30, 0x20, 0x10 };
```

Hence, the index we want must start from the least significant byte.

So we need to use a different function depending on the CPU endianness.
On most architectures, we want to start counting from the least significant byte, so we want the number of trailing zeros (`ntz`),
while on the few big-endian architectures that Ruby supports, we want to count the number of leading zeros (`nlz`).

Also, these are counting bits, so we need to essentially divide the result by `8`, or actually by `CHAR_BIT`, which is almost
always `8`, but is theoretically allowed to be about anything.

If you add that we also need to support 32-bit architectures, we end up with a macro like this one:

```c
#if SIZEOF_VALUE == 8
  #ifdef WORDS_BIGENDIAN
    #define AR_HINT_FIND_FIRST_ZERO_BYTE(x) (nlz_int64(x) / CHAR_BIT)
  #else
    #define AR_HINT_FIND_FIRST_ZERO_BYTE(x) (ntz_int64(x) / CHAR_BIT)
  #endif
#else
  #ifdef WORDS_BIGENDIAN
    #define AR_HINT_FIND_FIRST_ZERO_BYTE(x) (nlz_int32(x) / CHAR_BIT)
  #else
    #define AR_HINT_FIND_FIRST_ZERO_BYTE(x) (ntz_int32(x) / CHAR_BIT)
  #endif
#endif
```

This relies on a lot of other `define` from Ruby itself, but it's not super interesting, so I won't dig into it.

The one thing I'll mention is that, in general, integer division is a relatively slow operation[^2], but in this case we know
that we will almost certainly always divide by the constant 8, which any self-respecting C compiler will optimize into
a right-shift operation: `x / 8 == x >> 3`.
And right shifting is very fast.

So now, in addition to being able to tell if any byte is null, we can also tell its position:

```c
static void first_null_byte(uint64_t word)
{
    uint64_t x7 = (word & 0x7f7f7f7f7f7f7f7f) + 0x7f7f7f7f7f7f7f7f;
    uint64_t x8 = x7 | word;
    uint64_t matches = x8 | 0x7f7f7f7f7f7f7f7f;
    uint64_t indexes = ~matches;

    if (indexes) {
        printf("0x%llx has a NULL byte at index %u\n", word, __builtin_ctzll(indexes) / CHAR_BIT);
    }
    else {
        printf("0x%0llx does not have a NULL byte\n", word);
    }
}

int main(int argc, char **argv)
{
    first_null_byte(0x1020304050607080);
    first_null_byte(0x1020304000607080);
    return 0;
}
```

```
0x1020304050607080 does not have a NULL byte
0x1020304000607080 has a NULL byte at index 3
```

But even better, the `indexes` variable, as its name suggests, doesn't just contain the position of the first NULL byte,
but the position of all NULL bytes.
To access the next position, all we need to do is clear the least significant bit, the usual trick to do this is:

```c
x & (x - 1)
```

This works because `-1` in binary will ensure we turn the least significant bit into a zero, but we'll likely turn another
lower one up. So if we use that result as an `AND` bitmask:

```ruby
>> puts 42.to_s(2)
101010
=> nil
>> puts (42 - 1).to_s(2)
101001
=> nil
>> puts (42 & (42 - 1)).to_s(2)
101000
```

```c
static void find_null_bytes(uint64_t word)
{
    uint64_t x7 = (word & 0x7f7f7f7f7f7f7f7f) + 0x7f7f7f7f7f7f7f7f;
    uint64_t x8 = x7 | word;
    uint64_t matches = x8 | 0x7f7f7f7f7f7f7f7f;
    uint64_t indexes = ~matches;

    printf("0x%0llx ------------------------\n", word);

    if (!indexes) {
        printf("does not have a NULL byte\n");
        return;
    }

    while (indexes) {
        printf("NULL byte at index %u\n", __builtin_ctzll(indexes) / CHAR_BIT);
        indexes &= indexes - 1;
    }
}

int main(int argc, char **argv)
{
    find_null_bytes(0x1020304050607080);
    find_null_bytes(0x1000304000607000);
    return 0;
}
```

```
0x1020304050607080 ------------------------
does not have a NULL byte
0x1000304000607000 ------------------------
NULL byte at index 0
NULL byte at index 3
NULL byte at index 6
```

### Mask Making

At this point, you might be thinking that I skipped a step.
It's great that we can find NULL bytes quickly in an array, but the `ar_table` hint is an arbitrary number, not `0`.

So for this little function to help us, we'd first need to turn the matching bytes we care about into zeros, and the other
ones into non-zeros.

If you are familiar with logic gates, you may have recognized `XOR`.
All we need to do is to `XOR` all the hint bytes against the value we're looking for, e.g., if we're looking for, say, `0x42`:

```ruby
>> 0x42 ^ 0x42
=> 0
>> 0x11 ^ 0x42
=> 83
>> 0x00 ^ 0x42
=> 66
```

But since we got 8 bytes to `XOR` we need to build the `0x4242424242424242` mask first, which we can easily do with a multiplication:

```ruby
>> puts (0x0101010101010101 * 0x42).to_s(16)
4242424242424242
```

Again, putting it all together:

```c
static void find_bytes(uint8_t needle, uint64_t haystack)
{
    uint64_t search_mask = 0x0101010101010101 * needle;
    uint64_t word = haystack ^ search_mask;
    uint64_t x7 = (word & 0x7f7f7f7f7f7f7f7f) + 0x7f7f7f7f7f7f7f7f;
    uint64_t x8 = x7 | word;
    uint64_t matches = x8 | 0x7f7f7f7f7f7f7f7f;
    uint64_t indexes = ~matches;

    printf("0x%x in 0x%0llx ------------------------\n", needle, haystack);

    if (!indexes) {
        printf("miss\n");
        return;
    }

    while (indexes) {
        printf("hit at index %u\n", __builtin_ctzll(indexes) / CHAR_BIT);
        indexes &= indexes - 1;
    }
}

int main(int argc, char **argv)
{
    find_bytes(0x42, 0x1020304050607080);
    find_bytes(0x42, 0x1042304200607000);
    return 0;
}
```

```
0x42 in 0x1020304050607080 ------------------------
miss
0x42 in 0x1042304200607000 ------------------------
hit at index 4
hit at index 6
```

### Integrating Into Ruby

Since I had a working algorithm, I now had to integrate it into Ruby.

My very first attempt looked a bit like that:

```c
static unsigned
ar_find_entry_hint(VALUE hash, ar_hint_t hint, st_data_t key)
{
    VALUE matches = ar_search_hint(hint, RHASH_AR_TABLE(hash)->ar_hint.word);
    unsigned i;
    while ((i = ar_search_position(matches)) < RHASH_AR_TABLE_BOUND(hash)) {
      ar_table_pair *pair = RHASH_AR_TABLE_REF(hash, i);
      int eq = ar_equal(key, pair->key);
      if (UNLIKELY(!RHASH_AR_TABLE_P(hash))) {
          return RHASH_AR_TABLE_CONVERTED_TO_ST_TABLE;
      }
      if (eq) {
          return i;
      }
      matches &= matches - 1;
    }
    return RHASH_AR_TABLE_MAX_BOUND;
}
```

Essentially, the `find_bytes` function is described above, replacing the for-loop.
Unfortunately, that didn't work.
Worse Ruby wouldn't even build because Ruby somewhat bootstraps itself
by first compiling a minimal version of Ruby called `miniruby` which is then used to run a bunch of build scripts,
and since I had somehow broken the Hash implementation, some build scripts would fail.

What I had overlooked is that, as I mentioned before, `ar_equal` calls `Hash#eql?` on arbitrary objects, and since
objects can define their own `eql?` method, it's calling into arbitrary code, which can very well do weird things,
such as mutating the Hash we're currently looking up...

So the mistake here is that whenever we call `ar_equal`, we can no longer trust the result of `matches`.
We'd have to either revalidate it by checking if `ar_hint` changed, or recompute it entirely.

That's why the `for` condition is `i < RHASH_AR_TABLE_BOUND(hash)`.
You'd naturally think that it would be better for performance to store that `bound` in a local variable,
but we can't do that because the table bounds may change as we iterate over it.

But in the end, I thought it wasn't really worth the complexity. `ar_table` is small enough that we don't really
have to concern ourselves with collisions, and even when they happen, the cost of the linear search really isn't so bad.

So I later changed it to just skip to the first matching hint:

```c
// Returns the bin index if found, RHASH_AR_TABLE_MAX_BOUND if not found,
// or RHASH_AR_TABLE_CONVERTED_TO_ST_TABLE if #eql? or a Thread converted the hash to st_table.
static unsigned
ar_find_entry_hint(VALUE hash, ar_hint_t hint, st_data_t key)
{
    for (unsigned i = ar_hint_first_match(hint, RHASH_AR_TABLE(hash)->ar_hint.word); i < RHASH_AR_TABLE_BOUND(hash); i++) {
        const ar_hint_t *hints = RHASH_AR_TABLE(hash)->ar_hint.ary;
        if (hints[i] == hint) {
            ar_table_pair *pair = RHASH_AR_TABLE_REF(hash, i);
            int eq = ar_equal(key, pair->key);
            if (UNLIKELY(!RHASH_AR_TABLE_P(hash))) {
                return RHASH_AR_TABLE_CONVERTED_TO_ST_TABLE;
            }
            if (eq) {
                return i;
            }
        }
    }
    return RHASH_AR_TABLE_MAX_BOUND;
}
```

And the benchmarking result was relatevely nice:

```
|            |compare-ruby|built-ruby|
|:-----------|-----------:|---------:|
|hit_first   |     14.783M|   13.635M|
|            |       1.08x|         -|
|hit_fourth  |     13.139M|   13.877M|
|            |           -|     1.06x|
|hit_last    |     11.343M|   13.801M|
|            |           -|     1.22x|
|miss        |     11.096M|   16.892M|
|            |           -|     1.52x|

```

What I loved to see here was that the performance was now identical whether we'd hit the first or the last key.
It was also very noticeably faster on a miss.

But I was a bit bothered by the loss of performance on hitting the very first key.
As mentioned in the previous post, very small hashes are common, so even though ~8% slower isn't too terrible, I'd rather avoid it.

And as always, I focused on optimizing the happy-path.
If you look at the code, on the first iteration, the `if (hints[i] == hint) {` condition is entirely redundant.
Not only have we already established that `hints[i] == hint`, so it's needless overhead, but it being in a condition messes with
the branch predictor.

So I decided to specialize the first match:

```c
static unsigned
ar_find_entry_hint(VALUE hash, ar_hint_t hint, st_data_t key)
{
    unsigned first_match = ar_hint_first_match(hint, RHASH_AR_TABLE(hash)->ar_hint.word);

    if (LIKELY(first_match >= RHASH_AR_TABLE_BOUND(hash))) {
        return RHASH_AR_TABLE_MAX_BOUND;
    }

    RUBY_ASSERT(RHASH_AR_TABLE(hash)->ar_hint.ary[first_match] == hint);
    int eq = ar_equal(key, RHASH_AR_TABLE_REF(hash, first_match)->key);
    if (UNLIKELY(!RHASH_AR_TABLE_P(hash))) {
        return RHASH_AR_TABLE_CONVERTED_TO_ST_TABLE;
    }
    if (LIKELY(eq)) {
        return first_match;
    }
    else {
        for (unsigned i = first_match + 1; i < RHASH_AR_TABLE_BOUND(hash); i++) {
            const ar_hint_t *hints = RHASH_AR_TABLE(hash)->ar_hint.ary;
            if (UNLIKELY(hints[i] == hint)) {
                eq = ar_equal(key, RHASH_AR_TABLE_REF(hash, i)->key);
                if (UNLIKELY(!RHASH_AR_TABLE_P(hash))) {
                    return RHASH_AR_TABLE_CONVERTED_TO_ST_TABLE;
                }
                if (eq) {
                    return i;
                }
            }
        }
    }

    return RHASH_AR_TABLE_MAX_BOUND;
}
```

It's noticeably more code, but now we're much more friendly to the branch predictor, and it results in faster lookup.
Even for the first key:

```
|            |compare-ruby|built-ruby|
|:-----------|-----------:|---------:|
|hit_first   |     13.891M|   15.085M|
|            |           -|     1.09x|
|hit_third   |     13.322M|   15.065M|
|            |           -|     1.13x|
|hit_fourth  |     12.759M|   14.410M|
|            |           -|     1.13x|
|hit_last    |     11.545M|   15.275M|
|            |           -|     1.32x|
|miss        |     10.872M|   16.286M|
|            |           -|     1.50x|
```

Now, when I run the first benchmark with my patch applied, we can see some nice, constant-time performance:

```
ruby 4.1.0dev (2026-08-13T12:08:01Z ar-find-entry-swar-2 a62abe3f4c) +YJIT +PRISM [arm64-darwin25]
ar-hit-0     13.973M (± 0.3%) i/s   (71.57 ns/i) -     70.048M in   5.013198s
ar-hit-1     14.029M (± 0.1%) i/s   (71.28 ns/i) -     70.743M in   5.042555s
ar-hit-2     14.013M (± 0.3%) i/s   (71.36 ns/i) -     70.121M in   5.003860s
ar-hit-3     13.981M (± 0.1%) i/s   (71.53 ns/i) -     70.910M in   5.071933s
ar-hit-4     14.011M (± 0.5%) i/s   (71.37 ns/i) -     71.328M in   5.090874s
ar-hit-5     14.081M (± 0.2%) i/s   (71.02 ns/i) -     70.663M in   5.018430s
ar-hit-6     14.010M (± 0.6%) i/s   (71.38 ns/i) -     70.308M in   5.018301s
ar-hit-7     13.998M (± 0.3%) i/s   (71.44 ns/i) -     70.268M in   5.019812s
ar-miss      16.053M (± 1.0%) i/s   (62.30 ns/i) -     80.447M in   5.011457s

Comparison:
ar-hit-0: 13972808.8 i/s
ar-miss : 16052554.8 i/s - 1.15x  faster
ar-hit-5: 14080711.5 i/s - 1.01x  faster
ar-hit-1: 14029245.1 i/s - same-ish: difference falls within error
ar-hit-2: 14013320.9 i/s - same-ish: difference falls within error
ar-hit-4: 14011018.5 i/s - same-ish: difference falls within error
ar-hit-6: 14010319.4 i/s - same-ish: difference falls within error
ar-hit-7: 13998111.9 i/s - same-ish: difference falls within error
ar-hit-3: 13980914.6 i/s - same-ish: difference falls within error
```

And the cherry on top is that now `ar_table` is always faster than `st_table`:

```
ruby 4.1.0dev (2026-08-13T12:08:01Z ar-find-entry-swar-2 a62abe3f4c) +YJIT +PRISM [arm64-darwin25]
ar-hit-7     13.975M (± 0.2%) i/s   (71.55 ns/i) -     69.920M in   5.003078s
st-hit-7     10.884M (± 0.5%) i/s   (91.88 ns/i) -     54.857M in   5.040316s

Comparison:
ar-hit-7: 13975351.4 i/s
st-hit-7: 10883687.8 i/s - 1.28x  slower
```

I haven't merged [my patch](https://github.com/ruby/ruby/pull/18222) yet, but at that point I don't really see any reason not to,
so I kinda just need to clean it up.

[^1]: As usual, I will assume 64-bit architecture through the whole post, as to make things simpler.
[^2]: If you've played [Turing Complete](https://store.steampowered.com/app/1444480/Turing_Complete/), you may have a feel for it.