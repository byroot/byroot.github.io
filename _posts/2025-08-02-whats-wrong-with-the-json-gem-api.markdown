---
layout: post
title:  "What's wrong with the JSON gem API?"
date:   2025-08-02 10:03:51 +0100
categories: ruby json
---

As I mentioned at the start of my [Optimizing Ruby's JSON](/ruby/json/2024/12/15/optimizing-ruby-json-part-1.html) series of posts,
performance isn't why I candidated to be the new gem's maintainer.

The actual reason is that the gem has many APIs that I think aren't very good, and some that are outright dangerous.

As a gem user, it's easy to be annoyed at deprecations and breaking changes.
It's noisy and creates extra work, so I entirely understand that people may suffer from deprecation fatigue.
But while it occasionally happens to run into mostly cosmetic deprecations that aren't really worth the churn they cause (and that annoys me a lot too),
most of the time there's a good reason for them, it just is very rarely conveyed to the users, and even more rarely discussed,
so let's do that for once.

So I'd like to go over some of the API changes and deprecations I already implemented or will likely implement soon,
given it's a good occasion to explain why the change is valuable, and to talk about API design more broadly.

## Dealing With Deprecations in Ruby

But before I delve into deprecated API, I'd like to mention how to effectively deal with deprecations in modern Ruby.

Since Ruby 2.7, warning messages emitted with `Kernel#warn` are categorized, and one of the available categories is `:deprecated`.
By default, deprecation warnings are silenced; to display them, you must enable the `:deprecated` category like so:

```ruby
Warning[:deprecated] = true
```

It is very highly recommended to do so in your test suite, so much so that Rails and Minitest will do it by default.

However, if you are using RSpec, you'll have to do it yourself in your `spec_helper.rb` file, because we've tried to get
[RSpec to do it too for over four years now, but without success](https://github.com/rspec/rspec/issues/37).
But I'm still hopeful [it will eventually happen](https://github.com/rspec/rspec/pull/161).

Another useful thing to know about Ruby's `Kernel#warn` method is that under the hood, it calls the `Warning.warn` method,
allowing you to redefine it and customize its behavior.

For instance, you could turn warnings into errors like this:

```ruby
module Warning
  def warn(message, ...)
    raise message
  end
end
```

Doing so both ensures warnings aren't missed, and helps tracking them down as you'll get an exception with a full backtrace
rather than a warning that points at a single call-site that may not necessarily help you find the problem.

This is a pattern I use in most of my own projects, and that [I also included into Rails' own test suite](https://github.com/rails/rails/blob/add5a73b26e78d6b13945525874749ae40af21c7/tools/strict_warnings.rb).
For larger projects, where being deprecation-free all the time may be complicated, there's also the more sophisticated [`deprecation_toolkit` gem](https://github.com/Shopify/deprecation_toolkit).

## The create_additions Option

Now, let's start with the API that convinced me to request maintainership.

Do you know the difference between `JSON.load` and `JSON.parse`?

There's more than one, but the main difference is that it has a different set of options enabled by default, and notably
one that is a massive footgun: `create_additions: true`.

This option is so bad that [Rubocop's default set of rules bans `JSON.load` outright for security reasons](https://github.com/rubocop/rubocop/pull/3448),
and it has been involved in more than one [security vulnerabilities](https://discuss.rubyonrails.org/t/cve-2023-27531-possible-deserialization-of-untrusted-data-vulnerability-in-kredis-json/82467).

Let's dig into what it does:

```ruby
require "json"

class Point
  class << self
    def json_create(data)
      new(data["x"], data["y"])
    end
  end

  def initialize(x, y)
    @x = x
    @y = y
  end
end

document = <<~'JSON'
  {
    "json_class": "Point",
    "x": 123.456,
    "y": 789.321
  }
JSON

p JSON.parse(document)
# => {"json_class" => "Point", "x" => 123.456, "y" => 789.321}

p JSON.load(document)
# => #<Point:0x00000001007f6d08 @x=123.456, @y=789.321>
```

So what the `create_additions: true` parsing option does is that when it notices an object with the special key `"json_class"`,
It resolves the constant and calls `#json_create` on it with the object.

By itself, this isn't really a security vulnerability, as only classes with a `.json_create` method can be instantiated this way.
But if you've been using Ruby for a long time, this may remind you of similar issues with gems like `YAML` where similar capabilities
were exploited.

That's the problem with these sorts of duck-typed APIs: they are way too global.

You can have a piece of code using `JSON.load` that is perfectly safe on its own, but then if it's embedded in an application
that also loads some other piece of code that defines some `.json_create` methods you weren't expecting, you may end up with
an unforeseen vulnerability.

But even if you don't define any `json_create` methods, the gem will always define one on `String`:

```ruby
>> require "json"
>> JSON.load('{"json_class": "String", "raw": [112, 119, 110, 101, 100]}')
=> "pwned"
```

Here again, you probably need to find some specific circumstances to exploit that, but you can probably see how this
trick can be used to bypass a validation check of some sort.

So what do I plan to do about it? Several things.

First, I deprecated the implicit `create_additions: true` option. If you use `JSON.load` for that feature, a deprecation
warning will be emitted, asking to use `JSON.unsafe_load` instead:

```ruby
require "json"
Warning[:deprecated] = true
JSON.load('{"json_class": "String", "raw": [112, 119, 110, 101, 100]}')
# /tmp/j.rb:3: warning: JSON.load implicit support for `create_additions: true`
# is deprecated and will be removed in 3.0,
# use JSON.unsafe_load or explicitly pass `create_additions: true`
```

That being said, considering how wonky this feature is, I'm also considering extracting it into another gem.

This used to be impossible, as it was baked deep into the both the C and the Java parsers,
but [I recently refactored it to be pure Ruby code using a callback exposed by the parsers](https://github.com/ruby/json/pull/774).

Now you can provide a `Proc` to `JSON.load`, the parser will invoke it for every parsed value, allowing you to substitute
a value by another:

```ruby
cb = ->(obj) do
  case obj
  when String
    obj.upcase
  else
    obj
  end
end

p JSON.load('["a", {"b": 1}]', cb)
# => ["A", {"B" => 1}]
```

Prior to that change, `JSON.load` already accepted a Proc, but its return value was ignored.

The nice thing is that this callback also now serves as a much safer and flexible way to handle the serialization of rich objects.
For instance, you could implement something like this:

```ruby
types = {
  "range" => MyRangeType
}
cb = ->(obj) do
  case obj
  when Hash
    if type = types[obj["__type"]]
      type.load(obj)
    else
      obj
    end
  else
    obj
  end
end
```

While this requires more code from the user, it gives much tighter control over the deserialization,
but more importantly, it isn't global anymore.
If a library uses this feature to deserialize trusted data, its callback is never going to be invoked by another library
like it's the case with the old `Class#json_create` API.

The obvious solution would have been to follow the same route as `YAML`, with its `permitted_classes` argument, but
in my opinion, it wouldn't have addressed the root of the problem, and it makes for a very unpleasant API to use.

Instead, I believe this Proc interface provides the same functionality as before, but in a way that is both more
flexible and safer.

I think this is a clear case for deprecation, given it is very rarely needed, has security implications, and surprises users.

## Parsing of Duplicate Keys

Another behavior of the parser I recently deprecated is the treatment of duplicate keys.
Consider the following code:

```ruby
p JSON.parse('{"a": 1, "a": 2}')["a"]
```

What do you think it should return? You could argue that the first key or the last key should win, or that this should
result in a parse error.

Unfortunately, JSON is a bit of a "post-specified" format, as in it started as [an extremely simple document](https://www.json.org/json-en.html).
All it says about "objects" is:

> An object is an unordered set of name/value pairs.
> An object begins with `{` and ends with `}`.
> Each name is followed by `:` and the name/value pairs are separated by `,`.

That's it, that's the extent of the specification, as you can see, there is no mention of what a parser should do if it encounters a duplicate key.

Later on, various standardisation bodies tried to specify JSON based on the implementations out there.

Hence, we now have IETF's STD 90, also known as [RFC 8259](https://datatracker.ietf.org/doc/html/rfc8259), which states:

> Many implementations report the last name/value pair only.
> Other implementations report an error or fail to parse the object,
> and some implementations report all of the name/value pairs, including duplicates.

In other words, it acknowledges most implementations return the last seen pair, but doesn't prescribe any particular behavior.

There's also the [ECMA-404 standard](https://ecma-international.org/wp-content/uploads/ECMA-404_2nd_edition_december_2017.pdf)

> The JSON syntax does not impose any restrictions on the strings used as names,
> does not require that name strings be unique, and does not assign any significance to the ordering of name/value pairs.
> These are all semantic considerations that may be defined by
> JSON processors or in specifications defining specific uses of JSON for data interchange.

Which is pretty much the specification language equivalent of: 🤷‍♂️.

The problem with under-specified formats is that they can sometimes be exploited, the classic example being
[HTTP request smuggling](https://en.wikipedia.org/wiki/HTTP_request_smuggling).

And while it wasn't an exploitation per se, [a security issue happened to Hacker One](https://hackerone.com/reports/3000510#activity-32819479),
in part because of that behavior.
Technically, the bug was on the JSON generation side, but if the JSON's gem parser didn't silently accept duplicated keys,
they would have caught it early in development.

That's why starting from version `2.13.0`, `JSON.parse` [now accepts a new `allow_duplicate_key:` keyword argument](https://github.com/ruby/json/pull/818),
and if not explicitly allowed, a deprecation warning is emitted if a duplicate key is encountered:

```ruby
require "json"
Warning[:deprecated] = true

p JSON.parse('{"a": 1, "a": 2}')
# => {"a" => 2}

# /tmp/j.rb:4: warning: detected duplicate key "a" in JSON object.
# This will raise an error in json 3.0 unless enabled via `allow_duplicate_key: true`
#at line 1 column 1
```

As mentioned in the warning message, I plan to change the default behavior to be an error in the next major version, but of course
it will always be possible to explicitly allow for duplicate keys, for the rare cases where it's needed.

Here again, I think this deprecation is justified because duplicated keys are rare, but also almost always a mistake,
hence I expect few people to need to change anything, and the ones who do will likely learn about a previously unnoticed
mistake in their application.

## The to_json And to_s Methods

Before you gasp in horror, don't worry, I don't plan on deprecating the `Object#to_json` method, ever.
It is way too widespread for this to ever be acceptable.

But that doesn't mean this API is good, nor that nothing should be done about it.

At the center of the `json` gem API, there's the notion that objects can define themselves how they should be
serialized into JSON by responding to the `to_json` method.

At first sight, it seems like a perfectly fine API, it's an interface that objects can implement, fairly classic object-oriented design.

Here's an example that changes how `Time` objects are serialized.

By default, `json` will call `#to_s` on objects it doesn't know how to handle:
```ruby
>> puts JSON.generate({ created_at: Time.now })
{"created_at":"2025-08-02 13:03:32 +0200"}
```

But we can instruct it to instead serialize `Time` using the ISO8601 / [RFC 3339](https://datatracker.ietf.org/doc/html/rfc3339)
format:

```ruby
class Time
  def to_json(...)
    iso8601(3).to_json(...)
  end
end

>> puts JSON.generate({ created_at: Time.now })
{"created_at":"2025-08-02T13:05:04.160+02:00"}
```

This seems all well and good, but the problem, like for the `.json_create` method, is that this is a global behavior.
An application may very well need to serialize dates in different ways in different contexts.

Worse, in the context of a library, say an API client that needs to serialize `Time` in a specific way, it's not really
possible to use this API, you can't assume it's acceptable to change such a global behavior, given you know nothing about the application in which you'll run.

So to me, there are two problems here. First, using `#to_s` as a fallback works for a few types, like date, but it is really not helpful
for the overwhelming majority of other objects:

```ruby
>> puts JSON.generate(Object.new)
"#<Object:0x000000011ce214a0>"
```

I really can't think of a situation in which this is the behavior that you want. If `JSON.generate` ends up calling `to_s` on an object, I'm willing to bet that in 99% of the time, the developer didn't intend for that object to be serialized, or forgot to implement a `#to_json` on it.
 
Either way, it would be way more useful to raise an error, and requires that an explicit method to serialize that unknown object be provided.

The second is that it should be possible to customize a given type serialization locally, instead of globally.

In addition, returning a String as a JSON fragment is also not great, because it means recursively calling generators, and
allows to generate invalid documents:

```ruby
class Broken
  def to_json
    to_s
  end
end

>> Broken.new.to_json
=> "#<Broken:0x0000000123054050>"
>> JSON.parse(Broken.new.to_json)
#> JSON::ParserError: unexpected character: '#<Broken:0x000000011c9377a0>'
# > at line 1 column 1 
```

That's the problems the new `JSON::Coder` API is meant to solve.

By default, `JSON::Coder` only accepts to serialize types that have a direct JSON equivalent, so `Hash`, `Array`, `String` / `Symbol`,
`Integer`, `Float`, `true`, `false` and `nil`. Any type that doesn't have a direct JSON equivalent produces an error:

```ruby
>> MY_JSON = JSON::Coder.new
>> MY_JSON.dump({a: 1})
=> "{\"a\":1}"
>> MY_JSON.dump({a: Time.new})
#> JSON::GeneratorError: Time not allowed in JSON
```

But it does allow you to provide a `Proc` to define the serialization of all other types:

```ruby
MY_JSON = JSON::Coder.new do |obj|
  case obj
  when Time
    obj.iso8601(3)
  else
    obj # return `obj` to fail serialization
  end
end

>> MY_JSON.dump({a: Time.new})
=> "{\"a\":\"2025-08-02T14:03:15.091+02:00\"}"
```

Contrary to the `#to_json` method, here the Proc is expected to return a JSON primitive object, so you don't have to
concern yourself with JSON escaping rules and such, which is much safer.

But if for some reason you do need to, you still can using `JSON::Fragment`:

```ruby
MY_JSON = JSON::Coder.new do |obj|
  case obj
  when SomeRecord
    JSON::Fragment.new(obj.json_blob)
  else
    obj # return `obj` to fail serialization
  end
end
```

With this new API, it's now much easier for a gem to customize JSON generation in a local way.

Now, as I said before, I absolutely don't plan to deprecate `#to_json`, nor even the behavior that calls `#to_s` on unknown objects.
Even though I think it's a bad API, and that its replacement is way superior, the `#to_json` method has been at the center of the `json`
gem from the beginning and would require a massive amount of work from the community to migrate out of.

The decision to deprecate an API should always weigh the benefits against the costs.
Here, the cost is so massive that it is unimaginable for me to even consider it.

## load_default_options / dump_default_options

Another set of APIs I've marked as deprecated are the various `_default_options` accessors.

```ruby
>> puts JSON.dump("http://example.com")
"http://example.com"
>> JSON.dump_default_options[:script_safe] = true
>> puts JSON.dump("http://example.com")
"http:\/\/example.com"
```

The concept is simple: you can globally change the default options received by certain methods.

At first sight, this might seem like a convenience, it allows you to set some option without having to pass it around
at potentially dozens of different call sites.

But just like `#to_json` and other APIs, this change applies to the entire application, including some dependencies that may
not expect standard JSON methods to behave differently.

And that's not a hypothetical, I personally ran into a gem that was using JSON to fingerprint some object graphs, e.g.

```ruby
def fingerprint
  Digest::SHA1.hexdigest(JSON.dump(some_object_graph))
end
```

That fingerprinting method was well tested in the gem, and was working well in a few dozen applications until one
day someone reported a bug in the gem. After some investigation, I figured the host application in question
had modified `JSON.dump_default_options`, causing the fingerprints to be different.

If you think about it, these sorts of global settings aren't very different from monkey patching:

```ruby
JSON.singleton_class.prepend(Module.new {
  def dump(obj, proc = nil, opts = {})
    opts = opts.merge(script_safe: true)
    super
  end
})
```

The overwhelming majority of Rubyists are very aware of the potential pitfalls of monkey patching, and some absolutely loathe it,
yet, these sorts of global configuration APIs don't get frowned upon as much for some reason.

In some cases, they make sense. e.g. if the configuration is for an application, or a framework (a framework essentially being an application skeleton),
there's not really a need for local configuration, and a global one is simpler and easier to reason about.
But in a library, that may in turn be used by multiple other libraries with different configuration needs, they're a problem.

Amusingly, [this sort of API was one of the justifications for the currently experimental namespace feature in Ruby 3.5.0dev](https://bugs.ruby-lang.org/issues/21311#Avoiding-unexpected-globally-shared-modulesobjects),
which shows the `json` gem is not the only one with this problem.

Here again, a better solution is the `JSON::Coder` API, if you want to centralize your JSON generation configuration across
your codebase, you can allocate a singleton with your desired options:

```ruby
module MyLibrary
  JSON_CODER = JSON::Coder.new(script_safe: true)

  def do_things
    JSON_CODER.dump(...)
  end
end
```

As a library author, you can even allow your users to substitute the configuration for one of their choosing:

```ruby
module MyLibrary
  class << self
    attr_accessor :json_coder
  end
  @json_coder = JSON::Coder.new(script_safe: true)

  def do_things
    MyLibrary.json_coder.dump(...)
  end
end
```

Thankfully, from what I can see of the gem's usage, these API were very rarely used, so while they're not a major hindrance,
I figured the cost vs benefit is positive. And if someone really needs to set an option globally, they can monkey-patch JSON,
the effect is the same, and at least it's more honest.

## Conclusion

As mentioned previously, the decision to deprecate shouldn't be taken lightly.
It's important to have empathy for the users who will have to deal with the fallout,
and there are a few things more annoying than cosmetic deprecations.

Yet it is also important to recognize when an API is error-prone or even outright dangerous,
and deprecations are sometimes a necessary evil to correct course.

Also, as you probably noticed, a common theme in most of the APIs I don't like in the `json` gem, is global behavior and configuration.
I'm not certain why that is. A part of it might be that as Rubyists we value simplicity and conciseness, and that historically
the community has built its ethos as a reaction against overly verbose and ceremonial enterprise Java APIs, with their dependency injection frameworks and whatnot.

A bit of global state or behavior can sometimes bring a lot of simplicity, but it's a very sharp tool that needs to be handled with extreme care.
