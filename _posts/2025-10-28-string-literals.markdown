---
layout: post
title:  "Frozen String Literals: Past, Present, Future?"
date:   2025-10-28 09:03:51 +0100
categories: ruby performance
---

If you are a Rubyist, you've likely been writing `# frozen_string_literal: true` at the top of most of your Ruby
source code files, or at the very least, that you've seen it in some other projects.

Based on informal discussions at conferences and online, it seems that what this magic comment really is about is not always well understood,
so I figured it would be worth talking about why it's there, what it does exactly, and what its future might look like.

## Ruby Strings Are Mutable

Before we can delve into what makes frozen string literals special, we first need to talk about the Ruby String type,
because it's quite different from the equivalent type in other popular languages.

In the overwhelming majority of popular languages, strings are immutable.
That's the case in Java, JavaScript, Python, PHP, Go, etc.

There are a few exceptions, though, like Perl, C++, and of course Ruby:

```ruby
>> str = String.new
=> ""
>> str.object_id
=> 24952
>> str << "foo"
=> "foo"
>> str
=> "foo"
>> str.capitalize!
=> "Foo"
>> str.upcase!
=> "FOO"
>> str
=> "FOO"
>> str.object_id
=> 24952
```

Implementation-wise, they're just an array of bytes, with an associated encoding to know how these bytes should be interpreted:

```ruby
class String
  attr_reader :encoding

  def initialize
    @bytes = []
    @encoding = Encoding::UTF_8
  end
end
```

That too is quite unusual.

## String Encoding

Most languages, especially the ones I listed above, instead have chosen a specific internal encoding, and all strings are encoded that way.
For instance, in Java and JavaScript, strings are encoded in UTF-16 because they were created somewhat at the same time as the first Unicode specification, and at that time, many people thought that surely 16 bits should be enough to encode all possible characters, but that later turned out to be wrong.
Most other languages, like Python 3, use UTF-8.

In these languages, whenever you have to handle text in another encoding, you start by re-encoding it into the internal representation.
In Ruby however, strings with different internal encodings can exist in the same program, and Ruby supports over a hundred different encodings:

```ruby
>> Encoding.list.size
=> 103
```

While I'm not 100% percent certain of why Ruby went that way, I highly suspect it is in big part due to Ruby's Japanese origin.
In the early days of the Unicode specification, there was an attempt at unifying some of the "common" Chinese, Korean, and Japanese characters,
as what is now called the [Han unification](https://en.wikipedia.org/wiki/Han_unification).
Because of that character unification attempt, Unicode had lots of problems for Japanese text, hence the Japanese IT industry didn't adopt Unicode as fast as the Western IT industry did, and for a very long time, Japanese-specific encoding such as [Shift JIS](https://en.wikipedia.org/wiki/Shift_JIS) remained widespread.

As such, being able to work with Japanese text without going through a forced Unicode conversion was an important feature for a large part of Ruby's core contributors.

But let's go back to mutability.

## Pros And Cons

Like most things in engineering, both immutable and mutable strings have pros and cons, so it's not like one choice is inherently superior to the other.

One of the advantages of immutable strings is that you can more easily share them, for instance:

```ruby
sliced_string = very_long_string[1..-1]
```

In the above case, if strings are mutable, you need to copy all but one of the bytes of `very_long_string` into `sliced_string`, which can be costly.
But if strings are immutable, you can instead have `sliced_string` internally be pointing at the content of `very_long_string` with just an offset.
That is what some languages call String Views, or String slices.

Another advantage of immutable strings is that they allow for [interning](https://en.wikipedia.org/wiki/String_interning).
The idea is simple, if strings can't be mutated, whenever you have multiple instances of strings with identical content, you can coalesce them into a single instance.
This deduplication can be done more or less aggressively, as it's always a tradeoff in how much CPU time you want to spend searching for duplicates in the hope of saving some memory.

Some other advantages include not having to worry about mutation in multi-threaded code, as well as dictionary keys.
Strings are used a lot as dictionary keys.
If you mutate a string, you change its hash code, and that basically breaks hash tables.

On the other hand, mutable strings are very handy in some scenarios, like to iteratively build a final string:

```ruby
buffer = ""
10.times do
  buffer << "hello"
end
```

Whereas in a language with immutable strings like Java, concatenating strings in a loop is known as a classic performance gotcha:

```java
String buffer = "";
for (int i = 0; i < 10; i++) {
  buffer += "hello";
}
```

In the above example, on every loop, the `+=` operator causes a new string to be allocated, and the content to be copied, which gets exponentially more expensive as the string grows.
Instead, you are supposed to use a different object as a buffer: `StringBuilder`:

```java
StringBuilder buffer = new StringBuilder();
for (int i = 0; i < 10; i++) {
  buffer.append("hello");
}
buffer.toString();
```

That's the Java equivalent of appending strings to an array and then calling `array.join("")`.
It's a common enough mistake that at some point the Java compiler gained the ability to [detect that pattern and automatically replace it with the equivalent code using `StringBuilder`](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.18.1).

While having to use a different buffer type isn't the end of the world, I do very much like that it's not necessary in Ruby.

But more generally, the advantage of mutable strings is that for some algorithms, being able to modify the string in place saves a lot of memory allocations and copying.

## Ruby Actually Has Both

Earlier in this post, I said Ruby had mutable strings, but it's not quite true.
Ruby actually has both mutable and immutable strings, because in Ruby, every mutable object can be frozen, hence, Ruby has both mutable and immutable strings, and it takes advantage of this.

A fun way to poke at Ruby internals is through [the `ObjectSpace.dump` method](https://docs.ruby-lang.org/en/3.4/ObjectSpace.html#method-i-dump).

```ruby
require "json"
require "objspace"

def dump(obj)
  JSON.pretty_generate(JSON.parse(ObjectSpace.dump(obj)))
end

str = "Hello World" * 80
puts dump(str)
```

The above script will output something like:

```json
{
  "address": "0x105068e10",
  "type": "STRING",
  "slot_size": 40,
  "bytesize": 880,
  "memsize": 921,
  ...
}
```

It tells us the string content is `880B` (`bytesize`) and that Ruby allocated a `40B` wide slot (`slot_size`),
hence the string content is stored in an external buffer for a total of `921B` (`memsize`).

Now, look what happens if we slice that string:

```ruby
require "json"
require "objspace"

def dump(obj)
  JSON.pretty_generate(JSON.parse(ObjectSpace.dump(obj)))
end

str = "Hello World" * 80
puts "initial str: #{dump(str)}\n"

slice = str[40..-1]

puts "str after:\n#{dump(str)}\n"
puts "slice:\n#{dump(slice)}\n"
```

```json
str after:
{
  "address": "0x105178e18",
  "type": "STRING",
  "slot_size": 40,
  "shared": true,
  "references": [ "0x1051786c0" ],
  "memsize": 40,
  ...
}

slice:
{
  "address": "0x1051786e8",
  "type": "STRING",
  "slot_size": 40,
  "shared": true,
  "references": [ "0x1051786c0" ],
  "memsize": 40,
  ...
}
```

Now, both `str` and `slice` have the `shared: true` attribute, which indicates that they're not actually owning their content, they are pointing inside another String object.
You can also see that both `str` and `slice` have a reference to the same object at address: `0x1051786c0`.
So even though it has mutable strings, Ruby is still able to optimize some operations using "string views" like languages with immutable strings.
However, since `str` is mutable, Ruby couldn't directly create a string view that references `str`, it first had to transfer the buffer ownership to a third String object, and that one is immutable.
But if `str` was frozen, Ruby would have been able to directly create `slice` as a view inside `str`.

Similarly, when I was listing some of the pros and cons of mutable strings, I mentioned how mutable strings are a problem when used as hash table keys.
Perhaps you've never noticed it, but to avoid this problem, Ruby automatically freezes string keys in Hash:

```ruby
str = "test"
p str.frozen? # => false
hash = { str => 1 }
p hash.keys.first # => "test"
p hash.keys.first.frozen? # => true
p [str.object_id, hash.keys.first.object_id] # => [16, 24]
```

As you can see, here Ruby couldn't directly use the `str` string as a Hash key, it first had to make a frozen copy of it.
Here, too, if `str` was frozen, Ruby could have saved the extra work of duplicating this string.

I believe that illustrates the common tradeoffs at play with mutable strings.
On one hand, they can be much more efficient, allowing for in-place modifications, but on the other hand, they impose extra allocations and copying to protect yourself from mutations.

## The History Of Frozen String Literal

To avoid this extra copying overhead, it used to be a fairly common optimization technique to store string literals in constants.
For instance, you can see this idiom in [a 17 years old patch to rack](https://github.com/rack/rack/commit/8b8690bcb7762cde729088c2abdacb610ebea1f7):

```ruby
module Rack
  class MethodOverride
    METHOD_OVERRIDE_PARAM_KEY = "_method".freeze
    HTTP_METHOD_OVERRIDE_HEADER = "HTTP_X_HTTP_METHOD_OVERRIDE".freeze

    def call(env)
      # ...
      method = req.POST[METHOD_OVERRIDE_PARAM_KEY] ||
        env[HTTP_METHOD_OVERRIDE_HEADER]
      # ...
    end
  end
end
```

It's this pattern that led Charlie Somerville from GitHub to open [a feature request to propose a new syntax for frozen string literals](https://bugs.ruby-lang.org/issues/8579): `%f`.

```ruby
req.POST[%f(_method)] || env[%f(HTTP_X_HTTP_METHOD_OVERRIDE)]
```

This syntax wasn't accepted, but as a counter proposal, [Yusuke Endoh (mame)](https://github.com/mame) suggested an "f suffix":

```ruby
req.POST["_method"f] || env["HTTP_X_HTTP_METHOD_OVERRIDE"f]
```

This one was accepted and implemented in Ruby `2.1.0dev`.

However, many core developers didn't like this new syntax, so even after its implementation, multiple counterproposals were made.
Notably, [Akira Tanaka (akr), proposed a file-based directive](https://bugs.ruby-lang.org/issues/8976): `# freeze_string: true`, but it didn't catch on.

However before the final 2.1.0 release, [Charles Nutter](https://github.com/headius) [opened another feature request](https://bugs.ruby-lang.org/issues/8992),
and suggested to instead implement a compiler optimization for `String#freeze`, so as to provide the same feature but without introducing a new syntax.

If you aren't familiar with how the Ruby virtual machine works, or virtual machines in general, you may be surprised to hear that Ruby has a compiler, but it absolutely does.

Prior to Ruby 2.1, the program `"Hello World".freeze` would be compiled by Ruby into a sequence of two instructions:

```ruby
>> puts RubyVM::InstructionSequence.compile(%{"Hello World".freeze}).disasm
== disasm: #<ISeq:<compiled>@<compiled>:1 (1,0)-(1,19)>
0000 putstring                              "Hello World"             (   1)[Li]
0002 opt_send_without_block                 <calldata!mid:freeze, argc:0, ARGS_SIMPLE>
0004 leave
```

First, a `putstring` instruction to put `"Hello World"` on the VM stack, followed by an `opt_send_without_block` to call the `#freeze` method on it.

```ruby
def putstring(frozen_string)
  @stack.push(frozen_string.dup)
end
```

When invoked, the instruction receives a reference to a frozen String object that has been created by the Ruby compiler.
But since the semantics is that the string `#freeze` will be called on must be mutable, it has to duplicate it, and it's the mutable copy that is put on the stack.

In my opinion, the `putstring` instruction isn't correctly named, because its name suggests it just puts the frozen string directly on the stack.
This isn't consistent with other `put*` instructions like `putobject`, which directly puts an object on the stack without duping it:

```ruby
def putobject(object)
  @stack.push(object)
end
```

But also inconsistent with some other instructions like `duparray` and `duphash`, which actually behave like `putstring` does:

```ruby
def duparray(array)
  @stack.push(array.dup)
end
```

So it would be much clearer if it had been named `dupstring` instead of `putstring`.

But anyways, Charles' suggestion was to have the compiler generate a different set of VM instructions when the `#freeze` method is called
on a string literal:

```ruby
>> puts RubyVM::InstructionSequence.compile(%{"Hello World".freeze}).disasm
== disasm: #<ISeq:<compiled>@<compiled>:1 (1,0)-(1,20)>
0000 opt_str_freeze                         "Hello World", <calldata!mid:freeze, argc:0, ARGS_SIMPLE>(   1)[Li]
0003 leave
```

As you can see, on more recent rubies, the `putstring` and `opt_send_without_block` instructions have been replaced by a single `opt_str_freeze`.
Its implementation in pseudo-Ruby would be something like:

```ruby
def opt_str_freeze(frozen_string)
  if RubyVM.string_freeze_was_redefined?
    @stack.push(frozen_string.dup.freeze)
  else
    @stack.push(frozen_string)
  end
end
```

As you can see, to not break semantics, the instruction has to check that `String#freeze` hasn't been redefined, but apart from that cheap precondition, the instruction does strictly less work than before.

This is the feature Ruby 2.1.0 ultimately shipped with in December 2013.

## Further Optimizations

To further reduce string allocations, in 2014, Aman Karmani (tmm1) and Charlie Somerville (charliesome) from GitHub submitted [a patch to add two more optimized instructions, `opt_aref_with` and `opt_aset_with`](https://bugs.ruby-lang.org/issues/9382).

Before their patch, accessing a hash with a string key would cause a string allocation:

```ruby
>> puts RubyVM::InstructionSequence.compile(%{some_hash["str"]}).disasm
...
0003 putstring                              "str"
0005 opt_aref                               <calldata!mid:[], argc:1, ARGS_SIMPLE>[CcCr]
0007 leave
```

After the patch, these two instructions were replaced by a single `opt_aref_with`:

```ruby
>> puts RubyVM::InstructionSequence.compile(%{some_hash["str"]}).disasm
...
0003 opt_aref_with                          "str", <calldata!mid:[], argc:1, ARGS_SIMPLE>
0006 leave
```

Similar to `opt_str_freeze`, these instructions would check if the method is being called on a Hash, and if `Hash#[]` hadn't been redefined.
When both conditions are true, the instruction would be able to look up in the hash without first copying the string.

```ruby
def opt_aref_with(frozen_string)
  if RubyVM.hash_aref_was_redefined? || !@stack.last.is_a?(Hash)
    # fallback
    @stack.push(frozen_string.dup)
    value = RubyVM.call_method(:[], 1)
    @stack.push(value)
  else
    # fast path
    hash = @stack.pop
    value = hash[frozen_string]
    @stack.push(value)
  end
end
```

According to Aman Karmani, this reduced allocations in GitHub by 3%, which is quite massive for what is a relatively small patch.

As a sidenote, this optimized instruction has [just been removed by Aaron Paterson](https://bugs.ruby-lang.org/issues/21553) on the Ruby trunk, because given most performance-sensitive code already uses the magic comment, this optimization no longer yields much benefit.

## Ruby 3.0 And Frozen String Literals

Perhaps in part because of that new feature, or perhaps because of other reasons.
The knowledge of the performance impact of all these useless string duplication in Ruby applications started to spread around 2014,
and some community members, notably Richard Scheenman, started to submit [pull requests in Rails](https://github.com/rails/rails/pull/21057),
[rack](https://github.com/rack/rack/pull/737) and a bunch of other gems, with some pretty significant results, such as an 11.9% latency reduction on [codetriage.com](https://www.codetriage.com/).

These performance gains were generally too good to pass up, but regardless, many people felt that the resulting code was much more ugly.
So the question of freezing string by default came back regularly, but was always rejected.

Until [Akira Matsuda (amatsuda) brought the issue again at the Ruby core developer meeting in August 2015](https://github.com/ruby/dev-meeting-log/blob/master/2015/DevMeeting-2015-08-20.md#magic-comment-for-frozen-string-literal-by-default),
and there [Matz decided that Ruby string literals would be frozen in Ruby 3.0](https://xcancel.com/yukihiro_matz/status/634386185507311616).

A number of other features to ease the transition were also decided.
First, the `# frozen_string_literal: true` magic comment was introduced to help gems prepare for Ruby 3.0.

Then, to ensure that any code that wouldn't have been made compatible with Ruby 3.0 would remain usable, two Ruby command line options were added: `--enable-frozen-string-literal` and `--disable-frozen-string-literal`.

This way, once Ruby 3.0 would be released, if your code or one of your dependencies wasn't compatible yet, you could just set
`RUBYOPT="--disable-frozen-string-literal"` and keep going.

And also a `--debug-frozen-string-literal` command line option, to help developers. 

All these new features were released with Ruby 2.3 in December 2015.

What happens when you run Ruby with `--enable-frozen-string-literal` or with the `# frozen_string_literal: true` magic comment is that the compiler generates a different bytecode:

```ruby
>> puts RubyVM::InstructionSequence.compile(%{# frozen_string_literal: true\n"Hello World"}).disasm
== disasm: #<ISeq:<compiled>@<compiled>:2 (2,0)-(2,13)>
0000 putobject                              "Hello World"             (   2)[Li]
0002 leave
```

Now, instead of the `putstring` instruction, the compiler generates a `putobject` instruction.
As I mentioned above, this instruction directly puts the frozen string that was created during compilation on the stack, with no extra duplication.

So it's important to understand that frozen string literals are strictly less work for Ruby than mutable string literals.

## Community Usage

Following the release of Ruby 2.3, the Rubocop project added [a new cop to enforce the use of the `# frozen_string_literal: true` comment](https://github.com/rubocop/rubocop/pull/2542/commits/425b7469f109f2eae0648b600aa3ad24e85f6e21),
with the intent of helping projects be ready for Ruby 3.0 in the future.

Over the following years, many projects migrated to frozen string literals, [including Rails](https://github.com/rails/rails/pull/29506) and [rake](https://github.com/ruby/rake/pull/209) in 2017, [Rack in 2018](https://github.com/rack/rack/pull/1250),
and of course a long tail of other projects.

It's always hard to say with certainty how much a feature is used, but I think it's safe to say that, aside from a few projects that deliberately chose not to follow suit, a large majority of the actively developed gems did migrate to frozen string literals.
However, many of the more stable and less actively developed gems didn't.

There was no indication of when Ruby 3.0 would be released, and the lack of compatibility with it wasn't advertised by warnings or any other methods, hence, few people even knew whether any of their dependencies needed to be updated.

Over time, the magic comment slowly became an incantation most Rubyists follow, in big part because of rubocop, but as far as I know, basically no one was trying to run their application with `--enable-frozen-string-literal`, and few even knew about it.

## Abandoned Plan

However, [in October 2019, just before the release of Ruby 2.7, Matz abandoned the plan to make frozen string literal the default for Ruby 3.0](https://bugs.ruby-lang.org/issues/11473#note-53).

> I consider this for years. I REALLY like the idea but I am sure introducing this could cause HUGE compatibility issue, even bigger than Ruby 1.9.
> So I officially abandon making frozen-string-literals default (for Ruby3).
>
> --
> Matz

I must say this decision did surprise me at the time.
I definitely understand not wanting to cause a Python 3 sort of moment, but I don't think frozen string literals would have caused it,
because ultimately you could always have set `RUBYOPT="--disable-frozen-string-literal"` and kept running your applications unchanged if necessary.

I'm pretty sure if Python 3 had a way of running Python 2 code, the migration would have been much less of a big deal.

It was even more surprising to me because Ruby 2.7 also introduced new deprecation warnings in preparation for the keyword argument change in Ruby 3.0, and from my point of view, this breaking change was way bigger than frozen string literals would ever have been.
It caused so many deprecations that a [Ruby 2.7.2 was later released specifically to turn deprecation warnings off](https://www.ruby-lang.org/en/news/2020/10/02/ruby-2-7-2-released/).
And arguably, updating code to support the new keyword argument logic was way more involved than for frozen string literals.
If you have a look at [the migration guide](https://www.ruby-lang.org/en/news/2019/12/12/separation-of-positional-and-keyword-arguments-in-ruby-3-0/), it's fairly long and complex,
whereas frozen string literals only need a few strategically placed `.dup` there and there.

As a datapoint, I personally handled the migration of Shopify's monolith and roughly 700 gem dependencies for both the Ruby 3.0 keyword arguments and for `--enable-frozen-string-literal`.
For keyword arguments, I had to send pull requests to almost a hundred gems, as well as change a lot of code in the monolith itself, and some of them were really non-trivial to fix.
For frozen string literals, I only had to send pull requests to 12 gems, and it was just a matter of adding a few `.dup` calls.

But anyway, by the time of the Ruby 3.0 release, it had been almost 5 years since the initial plan had been laid out, and most of the performance-sensitive code had migrated to use the magic comment, so this abandonment didn't spark much discussion, and few people noticed.

## New Standards

Until four years later, in January 2024, I started hearing about `standardrb` and how [it doesn't enforce the presence of the frozen string literal magic comment](https://github.com/standardrb/standard/pull/181).
I also saw a few projects starting to remove them, or new projects deliberately not adding them, because this extra comment at the top is seen as cruft.

And I must say I agree.
I hate that comment.

Back when I started with Ruby, in version 1.8, the default encoding of source files was ASCII, so we frequently had to add a magic comment
at the top of the file to tell Ruby they were encoded in UTF-8.

```ruby
# encoding: utf-8
```

I hated that comment back then, because what I always loved about Ruby is that the source code is almost entirely free of boilerplate.
So when Ruby 2.0 made UTF-8 the default encoding, and we could finally get rid of all this cruft, it made me extremely happy.

I would love to do the same with the frozen string literal comment, but once you are aware of all these useless allocations and copies, it's really hard to unsee.
I'm now familiar enough with the VM that when I look at code without the magic comment, I pretty much visualize the implicit `dup` calls.

When I look at code like this:

```ruby
env["HTTPS"] == "on" ? "https" : "http"
```

I can't help but see this:

```ruby
env["HTTPS".dup] == "on".dup ? "https".dup : "http".dup
```

Which drives me nuts.
And yes, these are small strings, and the GC got faster in the last few years, but still, string literals are everywhere, so these allocations add up and cause a death by a thousand cuts.

So seeing that the community was slowly unlearning this lesson pained me, and I decided I'd try to revive the initiative.

## Chilled String Literals

In my opinion, what the initial plan lacked was a proper deprecation path.
Many Ruby users had heard the default would change with Ruby 3.0, but Ruby itself never emitted any deprecation to warn users that code would need to be updated, so very little work happened to prepare for it.

Hence, if I wanted to convince Matz to try again, I needed to come up with a way to emit useful deprecation warnings whenever some code would mutate a literal string.
That's where I came up with [the concept of *chilled strings*](https://bugs.ruby-lang.org/issues/20205).

Starting from Ruby 3.4, when a source file has no `frozen_string_literal` comment (either `true` or `false`), instead of generating `putstring` instructions, the compiler now generates `putchilledstring` instructions:

```ruby
>> puts RubyVM::InstructionSequence.compile(%{puts "Hello World"}).disasm
== disasm: #<ISeq:<compiled>@<compiled>:1 (1,0)-(1,18)>
0000 putself                                                          (   1)[Li]
0001 putchilledstring                       "Hello World"
0003 opt_send_without_block                 <calldata!mid:puts, argc:1, FCALL|ARGS_SIMPLE>
0005 leave
```

This new instruction is identical to `putstring`, except it additionally marks the newly allocated string with the `STR_CHILLED` flag.
Then I modified the `rb_check_frozen` function, which is responsible for raising `FrozenError` when a frozen object is mutated, to also check for that flag.
When a chilled string is mutated, a deprecation warning is emitted, and the flag is removed so that only the very first mutation emits a warning:

```ruby
>> Warning[:deprecated] = true
=> true
>> "test" << "a" << "b"
(irb):3: warning: literal string will be frozen in the future (run with --debug-frozen-string-literal for more information)
=> "testab"
```

The migration plan is that in a yet to be defined future version, these deprecation warnings would be visible by default, and then in a further version, frozen string literals would become the default.

## Mesuring The Performance Impact

Just like in the previous discussions back in 2014, [Yusuke Endoh (mame)](https://github.com/mame) objected to the change, arguing that the performance benefits of frozen string literals were never properly measured because back in 2014, lots of code wasn't compatible so it wasn't possible to measure.

> how much would the performance degrade if we removed `# frozen_string_literal: true` from all code used in yjit-bench?

So I went ahead and built a modified Ruby interpreter on which the magic comment had no effect, and [benchmarked it against mainline Ruby](https://bugs.ruby-lang.org/issues/20205#note-34).

The results were that frozen string literals make Lobsters, an open source discussion board in Rails, 8-9% faster.
It also made `railsbench`, a synthetic Rails application, 4-6% faster, and `liquid-render` 11% faster.

And one thing to note is that the benchmarked codebase and its dependencies, like Rack, still contain lots of code that was hand-optimized from the pre-frozen string literal days to avoid allocations.
So the difference would be certainly larger if mutable string literals weren't already worked around.

Similarly, back then I was surprised to only see a meager 1-2% gain on the `erubi-rails` benchmark, given it's quite string-heavy.
But in retrospect, it's very much expected because one of the biggest performance tricks of erubi is that it works around mutable string literals in its code generation by leveraging `opt_str_freeze` instructions:

```ruby
>> puts Erubi::Engine.new("Hello <% name%>!").src
_buf = ::String.new; _buf << 'Hello '.freeze; name; _buf << '!'.freeze;
_buf.to_s
```

All this makes it hard to come up with a clear measure of the performance benefits of freezing string literals.
At this point, making them the default is more to allow Rubyists to write nicer and less contrived code, not so much about improving performance.

After some more rounds of discussion, Matz [accepted the proposal](https://bugs.ruby-lang.org/issues/20205#note-35) but without committing to any specific timeline,
and I implemented the feature with [Étienne Barrié](https://github.com/etiennebarrie), which shipped with Ruby 3.4.0.

## So It's Done?

So at this point, it may look like a done deal.
The deprecations are in place, it's just a matter of deciding when to flip the switch.

But as we've seen in the past, that doesn't mean much.
Matz may still change his mind at any point, and there are still a few Ruby core members actively campaigning against frozen string literals.

Personally, I'm quite tired of arguing about it.
It might be a personal bias, given the overwhelming majority of the code I interact with has been frozen string literal compatible for a decade, but it seems to me that the Ruby community very largely adopted frozen string literals, so for me it seems obvious to make it the default.

But not everyone in Ruby core has the same view of the community.
Some members like Mame are very involved in [quines](https://en.wikipedia.org/wiki/Quine_(computing)) and other forms of [artistic programming like TRICK](https://github.com/tric/trick2025),
in which mutable string literals are used a lot.
So I understand that for him, switching the default means breaking a number of historical programs he cares about.

Ultimately, as always with Ruby's direction, it will come down to what Matz decides.
For now, he has publicly accepted the migration plan, but not yet committed to any timeline, and I'm not sure Matz really has a vision of what the community at large desires on this topic.
With Ruby 4.0 being likely released this year, it's very possible this migration stays in limbo for years and is ultimately abandoned again.

## Alternatives

At the end of the day, I don't care so much about frozen string literals being the default.
I just want to be able to stop adding this ugly comment at the top of my files, without losing the performance benefit and without having to explicitly freeze my constants.

An alternative to changing the default could be to allow setting compiler options for entire directories.
This would allow Rubyists to enable frozen string literals in a single place, typically the `gemspec` or Rails config.

However, this would fragment Ruby more, because it means a given code snippet may or may not work based on where it is located.
This was already a concern with the magic comment, it would be an even bigger one with directory-based compiler options.
So I'm not sure Matz would be ok with that.

## Conclusion

I can't predict what the future of string literals in Ruby will be.
I do hope they'll be frozen a few years from now, but I'm not holding my breath.

In the meantime I do encourage gem authors to tes[t their gems with `--enable-frozen-string-literal`](https://github.com/asciidoctor/asciimath/pull/78)

What is certain, however, is that performance-wise, they only have upsides, as they're strictly less work for the Ruby VM, but your performance-sensitive dependencies likely already use them, or at least work around mutable string literals in the hot paths.
Hence, you are unlikely to notice a big difference if you were to run your application with `RUBYOPT="--enable-frozen-string-literal"`.
However, if you do measure a negative performance impact, there is no doubt you are measuring incorrectly.

