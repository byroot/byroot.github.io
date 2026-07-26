---
layout: post
title:  "Optimizing Ruby's JSON, Part 8"
date:   2026-07-26 10:28:51 +0200
categories: ruby json
---

It has now been about 18 months since I concluded my post series on optimising Ruby's json.

Back then, I covered interesting performance patches that happened between version `2.7.2`,
the last version published before I took over maintenance, and version `2.9.0`, the latest release at that time.

During that span, both the parser and the generator became twice as fast on the infamous `twitter.json` benchmark.

```
== Parsing twitter.json (466906 bytes)
ruby 4.0.6 (2026-07-14 revision 03b6d3f889) +YJIT +PRISM [arm64-darwin25]
 2.7.2    575.498  (± 2.1%) i/s    (1.74 ms/i) -      2.880k in   5.004363s
 2.9.0      1.109k (± 0.7%) i/s  (901.86 μs/i) -      5.564k in   5.017923s

Comparison:
2.7.2:      575.5 i/s
2.9.0:     1108.8 i/s - 1.93x  faster
```

```
== Encoding twitter.json (466906 bytes)
ruby 4.0.6 (2026-07-14 revision 03b6d3f889) +YJIT +PRISM [arm64-darwin25]
 2.7.2      1.108k (± 0.9%) i/s  (902.19 μs/i) -      5.635k in   5.083851s
 2.9.0      2.219k (± 0.2%) i/s  (450.69 μs/i) -     11.220k in   5.056759s

Comparison:
2.7.2:     1108.4 i/s
2.9.0:     2218.8 i/s - 2.00x  faster
```

All this happened over about 6 weeks, and back then, performance was the main focus by far,
and there were numerous low-hanging fruits.

Since then, I've released almost 40 new versions, and even though the focus was mostly on deprecating some dangerous parts of the API,
and introducing a few more convenient and efficient APIs,
a number of interesting performance optimizations were implemented.
Most were by myself, but it also appears that the post series did motivate a handful of other people into contributing.

Now the parser is another `1.4x` faster:

```
== Parsing twitter.json (466906 bytes)
ruby 4.0.6 (2026-07-14 revision 03b6d3f889) +YJIT +PRISM [arm64-darwin25]
               2.7.2    586.494  (± 0.7%) i/s    (1.71 ms/i) -      2.950k in   5.029886s
              master      1.570k (± 0.4%) i/s  (637.13 μs/i) -      7.956k in   5.069045s

Comparison:
 2.7.2:      586.5 i/s
master:     1569.5 i/s - 2.68x  faster
```

And the generator `1.8x` faster:

```
== Encoding twitter.json (466906 bytes)
ruby 4.0.6 (2026-07-14 revision 03b6d3f889) +YJIT +PRISM [arm64-darwin25]
Warming up --------------------------------------
               2.7.2   109.000 i/100ms
Calculating -------------------------------------
               2.7.2      1.103k (± 0.5%) i/s  (906.46 μs/i) -      5.559k in   5.039002s

== Encoding twitter.json (466906 bytes)
ruby 4.0.6 (2026-07-14 revision 03b6d3f889) +YJIT +PRISM [arm64-darwin25]
Warming up --------------------------------------
              master   393.000 i/100ms
Calculating -------------------------------------
              master      3.966k (± 0.8%) i/s  (252.17 μs/i) -     20.043k in   5.054161s

Comparison:
 2.7.2:     1103.2 i/s
master:     3965.6 i/s - 3.59x  faster
```

But that's just for the main `twitter.json` benchmark.
Even though the JSON grammar is small, the cost of parsing a document can vary widely.

For instance, the `canada.json` benchmark contains a lot of floating-point numbers,
which are surprisingly hard to parse both efficiently and correctly.
That benchmark is now almost `10x` faster:

```
== Parsing canada.json (2090234 bytes)
ruby 4.0.6 (2026-07-14 revision 03b6d3f889) +YJIT +PRISM [arm64-darwin25]
   2.7.2     32.352 (± 0.0%) i/s   (30.91 ms/i) -    162.000  in   5.007433s
   2.9.0     37.698 (± 0.0%) i/s   (26.53 ms/i) -    189.000  in   5.013513s
  master    312.939 (± 3.2%) i/s    (3.20 ms/i) -      1.568k in   5.010569s

Comparison:
 2.7.2:       32.4 i/s
 2.9.0:       37.7 i/s - 1.17x  faster
master:      312.9 i/s - 9.67x  faster
```

So I figured it would be interesting to look back at some of the optimizations that moved the needle.
Now, please do keep in mind some of these are now 18 months old, so I don't always have as much fresh context
I mind than I had when I wrote the previous parts.
But hopefully it's still interesting.

### More Specialization

The very first performance improvement that landed after `2.9.0` was about more efficiently scanning strings in the generator.
[Back in part 3, I covered how I previously optimized that part of the generator](/ruby/json/2024/12/27/optimizing-ruby-json-part-3.html#dont-ignore-the-not-so-happy-path),
and back then, I noted that there was still some unnecessary overhead:

> I also realize now that I’m writing this, that I could have used something other than 3 in the lookup table for 0xE2 so
> that we don’t do any extra work for the 15 other bytes that mark the start of a 3-byte wide codepoint we’re not
> interested in, and also so that only the script safe version of the escape table would ever enter this branch of the code.

For context, when generating a JSON string, there are three types of characters you need to escape: double-quotes (`"`), backslashes (`\`)
and ASCII control characters (newline, tab, etc).

But given it's relatively common to interpolate JSON documents into JavaScript code, the JSON gem also provides a `script_safe: true`
escaping option, to also escape forward slashes (`/`), line separator (`U+2028`), and paragraph separator (`U+2029`).

The first one is to protect from XSS attacks, as an unescaped `/` could allow the injection of elements in the HTML document, e.g.

```ruby
<script>
  var config = <%= config.to_json %>;
</script>
```

Could allow an attacker to inject arbitrary scripts in the page:

```html
<script>
  var config = { "title": "</script><script>alert('pwned!')</script>" };
</script>
```

As for the two separator characters, they aren't really a security concern, but up until recently, they would cause JavaScript syntax errors,
as the JavaScript spec treats them as newlines, and unescaped newlines are not valid inside strings.
Conceptually, JSON was meant to be a subset of JavaScript, but these two characters broke that promise.

Thankfully, [the ES2019 specification now allows these unescaped characters in Javascript strings](https://github.com/tc39/proposal-json-superset),
so it's no longer really necessary to escape them unless you want to support fairly old JavaScript engines.
We already [stopped doing it by default in Rails](https://github.com/rails/rails/pull/55800), and I'll probably make that change in `JSON` at some point as well.

But anyway, the problem isn't so much that there are a few more characters to escape, that alone has a negligible performance impact.
The issue is that these are multi-byte characters, hence they cause noticeably more branching, and as we've seen in previous parts, branching is bad for performance.

Previously the lookup table was:

```c
static const char escape_table[256] = {
    // ASCII Control Characters
    1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
    1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
    // ASCII Characters
    0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0, // '"'
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0, // '\\'
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    // Continuation byte
    1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
    1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
    1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
    1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
    // First byte of a 2-byte code point
    2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,
    2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,
    // First byte of a 3-byte code point
    3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,
    //First byte of a 4+ byte code point
    4,4,4,4,4,4,4,4,5,5,5,5,6,6,1,1,
};
```

And whenever we'd find the first byte of a 3-byte code point, we'd check whether we have to escape in "script safe" mode
and whether the code point is the one we care about:

```c
// snip...
  case 3: {
      unsigned char b2 = ptr[pos + 1];
      if (RB_UNLIKELY(out_script_safe && ch == 0xE2 && b2 == 0x80)) {
          unsigned char b3 = ptr[pos + 2];
          if (b3 == 0xA8) {
              FLUSH_POS(3);
              fbuffer_append(out_buffer, "\\u2028", 6);
              break;
          } else if (b3 == 0xA9) {
              FLUSH_POS(3);
              fbuffer_append(out_buffer, "\\u2029", 6);
              break;
          }
      }
      // fallthrough
  }
```

So we'd run extra checks every time we run into a 3-byte code point, even when not in script safe-mode.

To speed that up, I packed an extra bit of information in the lookup table.
Now, the lower 3 bits contain the size of the codepoint, and the 4th bit is a boolean telling us whether escaping is required:

```c
// 0 - single byte char that don't need to be escaped.
// (x | 8) - char that needs to be escaped.
static const unsigned char CHAR_LENGTH_MASK = 7;

static const unsigned char ascii_only_escape_table[256] = {
    // ASCII Control Characters
     9, 9, 9, 9, 9, 9, 9, 9, 9, 9, 9, 9, 9, 9, 9, 9,
     9, 9, 9, 9, 9, 9, 9, 9, 9, 9, 9, 9, 9, 9, 9, 9,
    // ASCII Characters
     0, 0, 9, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, // '"'
     0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
     0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
     0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 9, 0, 0, 0, // '\\'
     0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
     0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    // Continuation byte
     1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
     1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
     1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
     1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
    // First byte of a  2-byte code point
     2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2,
     2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2,
    // First byte of a 3-byte code point
     3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3,
    //First byte of a 4+ byte code point
     4, 4, 4, 4, 4, 4, 4, 4, 5, 5, 5, 5, 6, 6, 9, 9,
};
```

So that now, the `if (RB_UNLIKELY(out_script_safe && ch == 0xE2 && b2 == 0x80))` and the subsequent branches only
need to be taken when we encounter `0xE2`, and not for the many other 3-byte code points.

This resulted in a 15% gain on the `twitter.json` benchmark:

```
== Encoding twitter.json (466906 bytes)
ruby 3.4.1 (2024-12-25 revision 48d4efcb85) +YJIT +PRISM [arm64-darwin23]
Warming up --------------------------------------
               after   244.000 i/100ms
Calculating -------------------------------------
               after      2.436k (± 0.9%) i/s  (410.43 μs/i) -     12.200k in   5.007702s

Comparison:
              before:     2125.9 i/s
               after:     2436.5 i/s - 1.15x  faster
```

Which isn't too surprising because it contains a lot of Japanese characters in the 3-byte range, e.g.:

```json
  "description": "元野球部マネージャー❤︎…最高の夏をありがとう…❤︎",
```

You can read [the full patch on GitHub](https://github.com/ruby/json/pull/724).

### Further Escaping Optimization

Not long after that, [Scott Myron](https://github.com/samyron) opened [a pull request to speed up escaping further using SIMD](https://github.com/ruby/json/pull/730).

If you're unfamiliar with what SIMD is, I briefly [explained it in part 6](https://byroot.github.io/ruby/json/2025/01/12/optimizing-ruby-json-part-6.html#a-note-on-simd),
that's also where I explained that even though the `simdjson` project had shown SIMD can be used to accelerate JSON parsing very significantly,
I wasn't very keen on taking the associated maintenance burden of maintaining at least three implementations of the same routines,
two of which are particularly cryptic, because SIMD intrinsics have such eyesore names like `_mm_cmplt_epu8` or `vcltq_u8`.

So Scott's pull request was a bit of a maintainer's dilemma for me.
On one hand, his first prototype was showing some very substantial gains, but on the other, I was reticent about including
hard-to-maintain code, given I'm no SIMD expert myself, and I also don't even have a proper `x86` development environment to debug potential issues.

I was also worried about giving him false expectations, causing him to put a lot of work into something that may never be merged,
which, from my point of view, is a very rude thing for a maintainer to do.

So [I explained to him that his initial pull request was more than I could chew](https://github.com/ruby/json/pull/730#issuecomment-2612031717),
and that I'd try to carefully extract the simplest possible SIMD routine in order to make sure it's not causing issues to some users
before considering any more SIMD usage.
Thankfully, he was very understanding of the situation and was willing to work on this regardless of whether it had any chance of being merged.

After a couple of weeks of back and forth, we settled on just two routines, a NEON one that works on all `arm64` CPUs,
and an SSE2 onethat works on all `x86_64` CPUs, and we left more powerful `x86` implementations such as `SSE4.2` and `AVX2` for the future.

The `SSE2` (Intel) implementation:
```c
static inline TARGET_SSE2 FORCE_INLINE int sse2_update(const char *ptr)
{
    __m128i chunk         = _mm_loadu_si128((__m128i const*)ptr);

    const __m128i lower_bound = _mm_set1_epi8(' '); 
    const __m128i backslash   = _mm_set1_epi8('\\');
    const __m128i dblquote    = _mm_set1_epi8('\"');

    __m128i too_low       = _mm_cmplt_epu8(chunk, lower_bound);
    __m128i has_backslash = _mm_cmpeq_epi8(chunk, backslash);
    __m128i has_dblquote  = _mm_cmpeq_epi8(chunk, dblquote);
    __m128i needs_escape  = _mm_or_si128(too_low, _mm_or_si128(has_backslash, has_dblquote));
    return _mm_movemask_epi8(needs_escape);
}
```

And here's the `NEON` (Arm64) one:

```c
static inline FORCE_INLINE uint64_t neon_rules_update(const char *ptr)
{
    uint8x16_t chunk = vld1q_u8((const unsigned char *)ptr);

    const uint8x16_t lower_bound = vdupq_n_u8(' '); 
    const uint8x16_t backslash   = vdupq_n_u8('\\');
    const uint8x16_t dblquote    = vdupq_n_u8('\"');

    uint8x16_t too_low       = vcltq_u8(chunk, lower_bound);
    uint8x16_t has_backslash = vceqq_u8(chunk, backslash);
    uint8x16_t has_dblquote  = vceqq_u8(chunk, dblquote);
    uint8x16_t needs_escape  = vorrq_u8(too_low, vorrq_u8(has_backslash, has_dblquote));

    return neon_match_mask(needs_escape);
}
```

As you can see, the SIMD intrinsic names make this code quite cryptic for the uninitiated,
yet, what they do is conceptually fairly simple, so as usual, I'll try to explain it using Ruby code.

The first key thing to keep in mind is that SIMD is all about applying transformation onto fixed-size *vectors*,
or simply put, arrays.
So in most cases, the Ruby equivalent would be some `Array` or `Enumerable` method like `#map`,
but you have to imagine that the `map` is applied to all the elements simultaneously.

With that said, let's decompose it line by line, using the `NEON` implementation.

```c
uint8x16_t chunk = vld1q_u8((const unsigned char *)ptr);
```

That first line loads (`vld` -> vector load) `16` 8-bit unsigned integers (`u8`) into a register.
The Ruby equivalent would be something like:

```ruby
def ruby_rules_update(str, offset)
  chunk = str.byteslice(offset, 16).bytes
  # ...
end
```

Then the three `vdupq_n_u8` calls are defining constant vectors with the characters we care about, using Ruby arrays
it would look like:

```ruby
LOWER_BOUND = [" ".ord]   * 16
BACKSLASH   = ["\\".ord]  * 16
DBLQUOTE    = ['"'.ord]   * 16
```

After that, we compare each element of `chunk` against our constants.
First using `<` (`lt` -> lether than), then `==` (`eq` -> equal):

```ruby
too_low       = chunk.zip(LOWER_BOUND).map { |char, bound| char < bound }
has_backslash = chunk.zip(BACKSLASH).map   { |a, b| a == b }
has_dblquote  = chunk.zip(DBLQUOTE).map    { |a, b| a == b }
```

Which produces three arrays of booleans, which we combine with `|` (`vorrq_u8` -> "vector or"):

```ruby
needs_escape = too_low.zip(has_backslash).map { |a, b| a | b }
needs_escape = needs_escape.zip(has_dblquote).map { |a, b| a | b }
```

Putting it all together in a single method:

```ruby
VECTOR_SIZE = 16
LOWER_BOUND = [" ".ord]   * VECTOR_SIZE
BACKSLASH   = ["\\".ord]  * VECTOR_SIZE
DBLQUOTE    = ['"'.ord]   * VECTOR_SIZE

def ruby_rules_update(str, offset)
  chunk = str.byteslice(offset, VECTOR_SIZE).bytes

  too_low       = chunk.zip(LOWER_BOUND).map { |char, bound| char < bound }
  has_backslash = chunk.zip(BACKSLASH).map   { |a, b| a == b }
  has_dblquote  = chunk.zip(DBLQUOTE).map    { |a, b| a == b }

  needs_escape = too_low.zip(has_backslash).map { |a, b| a | b }
  needs_escape = needs_escape.zip(has_dblquote).map { |a, b| a | b }

  needs_escape
end
```

And as you can see, just like the SIMD implementation, that Ruby method returns a vector of booleans, telling us
which characters need to be escaped.

```ruby
>> ruby_rules_update('___\\___""_________', 1)
=> [false, false, true, false, false, false, true, true, false, false, ...]
```

Even though in C we repack that vector into a normal 64-bit number that we use as a bitfield, conceptually it's the same thing as an array of booleans.

The very counter-intuitive part of SIMD is that when you're very used to scalar code, it really looks like SIMD is wasting a lot
of time computing useless things.
By that I mean that the idiomatic scalar equivalent would likely short-circuit to avoid computing `c == '"'` if we previously established
that `c < ' '`.
Yet, since SIMD instructions operate on 16 (or more) bytes at once, the amortized cost of the check on a per-byte basis is so low
that we win anyway.

For me, that's one of the challenging things with writing or reviewing SIMD code, it requires having a fairly different mindset.

You can have a look at [the final pull request](https://github.com/ruby/json/pull/743), it contains a lot of extra boilerplate
required to test for SIMD support, etc, but I explained the core of it just above.

Overall, between the very first version and when the first SIMD code landed, there were over three months of iteration and back-and-forth.
We also included [some code from radiospiel, who happened to also look at introducing SIMD](https://github.com/ruby/json/pull/769), but didn't notice Scott's work.

In the end, the gain on `twitter.json` was quite substantial:

```
== Encoding twitter.json (466906 bytes)
ruby 3.4.2 (2025-02-15 revision d2930f8e7a) +YJIT +PRISM [arm64-darwin24]
               after      3.115k (± 3.1%) i/s  (321.01 μs/i) -     15.759k in   5.063776s

Comparison:
              before:     2508.3 i/s
               after:     3115.2 i/s - 1.24x  faster
```

### A Brand New Parser

Another change that was foreshadowed on this blog was the re-implementation of the parser.
I ended part 7 with this comment:

> I'd like to drop Ragel and replace it with a simpler recursive descent parser.
> The existing one could certainly be improved, but I find it much harder to work with generated parsers than to write them manually.

When I wrote this, I actually meant it, but as a nice-to-have.
Something I'd eventually work on when I'd get the time, but that wasn't pressing by any means.

But I suspect publishing this combination of keywords on the Internet must have lit some sort of bat-signal in
[Kevin Newton](https://github.com/kddnewton)'s office,
because not long after he pinged me with just that, a reimplementation of the JSON parser using recursive descent.

The implementation wasn't quite complete, all sorts of corner cases weren't handled yet, but [the code was clean, barely 200 lines](https://github.com/byroot/json/blob/5694cff0b773ed786da7c0f97521714fef4fee54/ext/json/ext/parser/parser.c) and much, much easier
to work with than the previous 1400 lines of Ragel implementation.
Even though it was mostly just prototype quality, you could really feel that there were lots of lessons he learned while
working on [Prism](https://github.com/ruby/prism/) baked in that prototype.

That being said, after adding the missing parts and refactoring the code quite heavily to familiarize myself with it,
the size grew to a bit less than 600 lines, and [the performance impact was mixed](https://github.com/ruby/json/pull/729#issuecomment-2593780223).
The parser performed about the same on `twitter.json`, and even `6%` faster on `citm_catalog.json`, but `8%` slower on `activitypub.json`.
So, gain some, lose some, however it did still replace 1400 lines of Ragel code with less than 600 lines of pure C, so it was really worth it in my opinion.

But ultimately, a code that is easy to understand and work with is also easier to optimize,
so I was confident that this new implementation would allow me to find optimization faster than with Ragel, so it was worth pursuing.

I just didn't want to merge a measurable regression, so I needed to find enough performance to be on par with Ragel on all benchmarks.

And like several times in the past, I found some nice wins using lookup tables.

In [Kevin's initial prototype](https://github.com/byroot/json/blob/5694cff0b773ed786da7c0f97521714fef4fee54/ext/json/ext/parser/parser.c), there was two fairly hot loop.

One was for skipping whitespaces:

```c
static inline void
j2_eat_whitespace(j2_parser_t *parser) {
    while (parser->cursor < parser->end) {
        switch (*parser->cursor) {
            case ' ':
            case '\t':
            case '\n':
            case '\r':
                parser->cursor++;
                break;
            case '/':
                // snip... deal with comments
                break;
            default:
                return;
        }
    }
}
```

Since JSON allows for whitespace between all tokens, this function is called a lot, making it quite the hotspot.
That being said, it's almost always a noop, given that we benchmark against minified JSON documents, hence there are no comments
not whitespace in any of the benchmarks.

Which somewhat maps the reality, as aside from the occasional configuration file, most JSON documents in the wild
are produced as compactly as possible to save bandwidth.

Hence, checking for 5 different characters, when it will almost never match, is a bit of a waste.

And here again, it looked like a good case for using [a lookup table](/ruby/json/2024/12/15/optimizing-ruby-json-part-1.html#lookup-tables),
so I changed the code:

```c
static const bool whitespace[256] = {
    [' '] = 1,
    ['\t'] = 1,
    ['\n'] = 1,
    ['\r'] = 1,
    ['/'] = 1,
};

static void
json_eat_comments(JSON_ParserState *state)
{
  // snip...
}

static inline void
json_eat_whitespace(JSON_ParserState *state)
{
    while (state->cursor < state->end && RB_UNLIKELY(whitespace[(unsigned char)*state->cursor])) {
        if (RB_LIKELY(*state->cursor != '/')) {
            state->cursor++;
        } else {
            json_eat_comments(state);
        }
    }
}
```

I also extracted the comment handling part into its own function, so that the compiler is less likely to inline it[^1].
This is because comments in JSON documents are rare.
They're mostly just present in configuration files, which really aren't that performance sensitive.
So it's best to discourage the compiler from inlining rarely called functions, because compilers have some sort of soft limit
to inlining.

They try to avoid creating functions that are too big using various heuristics, so you can sometimes witness some really bad
performance regressions by just adding a few lines of code in an inlined function, causing the compiler to stop inlining
some other functions, tanking the performance.

This is a particularly common problem for large dispatch loops, like the one found in recursive descent parsers and interpreter loops.

Similarly, in the initial prototype, the string parsing code was relying on sequential character comparisons to either
the end of the string or escape sequences:

```c
  case '"': {
      // %r{\A"[^"\\\t\n\x00]*(?:\\[bfnrtu\\/"][^"\\]*)*"}
      parser->cursor++;
      const uint8_t *start = parser->cursor;

      while (parser->cursor < parser->end) {
          if (*parser->cursor == '"') {
              VALUE string = rb_enc_str_new((const char *) start, parser->cursor - start, rb_utf8_encoding());
              parser->cursor++;
              return string;
          } else if (*parser->cursor == '\\') {
              // TODO: Parse escape sequence
              parser->cursor++;
          }

          parser->cursor++;
      }

      rb_raise(rb_eRuntimeError, "unexpected end of input");
      break;
  }
```

Which at first performed OK, but was actually incomplete because a conforming JSON parser must also fail to parse if it encounters
an unescaped ASCII escape code.

Once I added an `} else if (*parser->cursor < ' ') {` branch, the performance lowered to be significantly worse than the previous parser.

So there too, I introduced a lookup table:

```c
static const bool string_scan[256] = {
    // ASCII Control Characters
     1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
     1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
    // ASCII Characters
     0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, // '"'
     0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
     0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
     0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, // '\\'
     0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
     0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
};
```

With these two changes and a few more minor tweaks, the new parser was finally [a bit faster than the old one](https://github.com/ruby/json/pull/729#issuecomment-2595328158).
`15%` on `twitter.json`, `7%` on `activitypub.json` and `11%` on `citm_catalog.json`.
Nothing major, but there again, the goal was merely to match the previous performance, with the hope that it would be a better
basis for future gains.

### To Be Continued

I think that's enough content for now.
When I get the time to write part 9, I'll talk about how [radiospiel](https://github.com/radiospiel) sped up number generation.

[^1]: Many versions later, I even marked it with `__attribute__((noinline))` to really discourage inlining.