---
layout: post
title:  "The Missing Bundler Features"
date:   2026-04-20 09:03:51 +0900
categories: ruby bundler
---

Over the last few months, there has been a lot of talk about making Bundler faster,
both by improving it directly, or by reimplementing it in another language, and while it may surprise some, that didn't excite me much.

Don't get me wrong, all other things being equal, faster is better, so if Bundler gets faster without me having to change my toolchain one bit,
I'll happily take it.
But I certainly would not bother migrating to something else just for speed.

Instead, there are a number of features I believe Bundler is missing, and that over the years,
I tried to convince Bundler's maintainers to consider them, but without any success.

## Why Bundler Is Fast Enough For Me

Given how much I talk about performance on this blog, it might surprise you to hear I don't care about Bundler performance.

The reality is that I very rarely, if ever, have to wait for Bundler.
Most of the time, all gems are already installed locally, so it completes instantly. A few times, a couple of gems are missing, and it completes in mere seconds, this is nowhere near a bottleneck in my workflow.

It's only once every few months, when I upgrade Ruby or checkout a new big project, that I have to wait a couple of minutes
for the installation to complete, so it's not like it's driving me up the wall.

Similarly, for CI and deploys, the gems installed by Bundler are heavily cached, so `bundle install` is very fast.

And even if Bundler was magically able to install everything instantly without a cache, I'd argue that caching gems on CI
would remain absolutely essential simply for resilience purposes.

I remember a few years back, when [the `mimemagic` gem was yanked because of licensing issues](https://github.com/rails/rails/issues/41750),
a lot of Rails users complained that their CI pipeline was broken and they couldn't work anymore.
Well, Shopify CI and image builder continued uninterrupted, as it kept re-using the gem from its cache.
It's also entirely resilient to rubygems.org outages. 

That's why I was quite amused a few months back, when some people were floating the idea of making rubygems.org pay by usage,
thinking big players would have to pay for most of the infrastructure.
The CIs of big Ruby companies actually hit rubygems.org much less than even a small project CI with no cache would.

So yes, it's very rare that I find myself staring at the output of `bundle install` for more than a few seconds.

However, **I frequently waste hours, if not days**, dealing with dependency hell in big gemfiles, **because Bundler is missing some important features**.

## Dependency Hell

To better illustrate what I mean, let me describe one of the things I was working on just last week before leaving for Ruby Kaigi.

For various reasons, I wanted to upgrade the `openssl` gem in Intercom's monolith, as it's still running a 2.x version.

Except `bundle update openssl` would silently do nothing, because the Gemfile contained the [`web-push 2.0.0` gem](https://rubygems.org/gems/web-push/versions/2.0.0),
which has an `openssl ~> 2.2` dependency constraint.
So I first tried to upgrade to the latest version of `web-push`, except that the later version has a `jwt ~> 3.0` dependency constraint,
which was a problem for 4 other gems in the Gemfile, which have a `jwt ~> 2.0` constraint, and some of these gems
haven't been released in years, so while [I did submit pull requests](https://github.com/sumoheavy/jira-ruby/pull/482)
to unblock all of that, I have no clue when the situation will be unblocked (if ever).

So now, I can either patiently wait and perhaps try to ping the multiple maintainers every couple of weeks, or I'll have to use
unreleased versions of these gems using `gem 'name', github: '...'`, which works, but is annoying for various reasons.

## Upper Constraints

The root cause of most, if not all, of these dependency hell problems is upper version constraints.
`openssl ~> 2.2` is a shorthand for `openssl >= 2.2, < 3`, and the problem here is the `< 3`.

This `~>` operator (often named the pessimistic operator) essentially assumes that the dependency follows [Semantic Versioning](https://semver.org/),
and that the next major version will not work with the current version of the gem.

But I would argue that upper version constraints are almost always wrong, because when you are writing a gem, that next major version of your dependency doesn't yet exist, hence you don't actually know it won't work.

A SemVer major just tells us that there are some breaking changes, it doesn't tell us that absolutely every caller will be broken.
Dependencies like `openssl` have a very large API, just because they bumped the major to signal that some things were broken doesn't mean they broke your usage of it.

Hence, the "pessimistic operator" name is quite apt, but I'd argue it's wrong to be this pessimistic.

If it was possible to update the dependencies of a gem after it has been released, then yes,
it would make sense to warn your user that your gem doesn't work well with this new major version of OpenSSL,
but doing so in advance is likely to be more harmful than helpful.

But while I have strong opinions on how maintainers should declare their dependencies, ultimately, they are king in their domain,
so they are free to do as they wish.

However, I also consider that my Gemfile is my domain.
Therefore, I shall be empowered to disregard any such constraint in my Gemfile as I see fit, but Bundler won't let me.
That I think is utterly wrong.

So let's talk about the Bundler features I think would save me loads of time.

## Forcing A Version

Using the same `openssl` example, my initial goal was just to upgrade that particular gem, so I believe I should have been
able to tell Bundler to just do that, even if it meant not respecting some other gem's wishes.

In terms of actual feature design, what I'd like would be to be able to just add `force: true` on a gem:

```ruby
gem 'openssl', '>= 3.0', force: true
```

This would cause Bundler to replace all `openssl` dependency constraints from other gems with this one.
Bundler would, of course, be free to print as many warnings as it wishes, but it should install the gems I want.

Perhaps it would turn out that indeed, `web-push 2.0.0` is truly incompatible with `openssl 3.x`, but that's my Gemfile,
that's on me to test this properly, and to deal with the consequences if I didn't.

It's my machine, my project, and I shall be allowed to break it if that's what I want to do.

In practice, they didn't have to change anything when [they relaxed the dependency to allow `openssl 3.x`](https://github.com/pushpad/web-push/commit/b1b60bb2a2ab0f4bd9331b944117b110ef94c3bf)
and [again for `openssl 4.x`](https://github.com/pushpad/web-push/pull/19), which further proves my point
that pessimistic constraints are almost always wrong.

## Subtituting A Gem

Another dependency hell sort of problem is outdated, broken gems.
And the most well-known occurrence of that problem in the last few years was [the `httpclient` gem](https://rubygems.org/gems/httpclient).

This was a relatively popular gem for a long time, even used as a dependency of `google-cloud` gems, but until recently, the
last version was published in 2016.
The problem, however, was that by default the gem would set up SSL using vendored root certificates, which worked fine until they
expired in 2023, and suddenly, many Ruby applications just broke.
Then later on, when Ruby 3.4 was released, the gem started throwing various warnings.

All this time, many people had to fork the gem and point their Gemfile at their own fork.

Ultimately, in 2025, [Yasuo Honda](https://github.com/yahonda/) managed to contact the former maintainer and get the gem ownership, which allowed him to fix all these problems and cut a new release.
But the fact remains that this gem had been a thorn in many Rubyists' sides for several years.

This is just an example.
There are other gems that are no longer actively maintained, and that could cause similar issues at any time if they'd ever hit a backward compatibility issue of some sort.

As mentioned, the ability of Bundler to pull a gem from a Git repository is what unblocked everyone, but I can't help but think it's a big waste of effort, as pointing your Gemfile to a git repo you don't control is a bit risky.
It would have been much better to be able to substitute the vanilla `httpclient` gem with another gem published on rubygems.org.

Since I had to fork `httpclient` to fix its issues for my employer, I would have much preferred to release my fixes
as a gem named `byroot-httpclient`, and for everyone who trusts me to have been able to tell Bundler to use that instead.

That's the feature [I tried to suggest to bundler maintainers over two years ago](https://github.com/rubygems/rfcs/issues/54):

```ruby
gem "byroot-httpclient", as: "httpclient"
```

As a way to tell Bundler that my gem is a substitute for the vanilla one.

Unfortunately, like almost every time I tried to interact with Bundler, I wasn't able to get anything moving.

Two years later, I still think that feature would be tremendously helpful, but it could even be simplified.
In practice, you don't even need Bundler to substitute a gem for another, you just need it not to install a particular
gem:

```
ban 'httpclient' # Broken because ....
gem 'byroot-httpclient' # Replace `httpclient` 
```

Which makes for an even simpler feature, and would also be useful to prune some dependencies that are being pulled, but
that you know you're not actually using.

It would also help the community pick over the maintenance of abandoned gems under different names, as it would allow people
to migrate to alternative implementations.
In a sense, it solves the need for gem namespacing much better than what has been explored so far.

## It's All About Control

Ultimately, the two features I described above serve the same purpose: giving control back to the user.

It's quite apparent in Bundler design that it trusts gem publishers more than the actual Bundler user, which, to me, is wrong.
That impression is reinforced by the concerns raised when I requested that feature:

> The biggest worry that comes immediately to mind for me is "how do we keep a feature like this from harming maintainers?"
> For example, if I maintain left-pad, but someone else is using left-padder, how do we ensure that errors are reported to left-padder and not to left-pad?

Which might come from a good place, but doesn't make much sense to me, as it's already possible to substitute upstream code
via git gems, private gem servers, or even simply monkey patches, yet as the maintainer of numerous gems, this has never been
a significant harm to me.

## Conclusion

I very sincerely believe Bundler is still one of the best, if not the best, package manager out there.
And the fact that it has been used as a model for other package managers is proof of that.

But as a mere user, I have the feeling it has stagnated over the last decade.

Don't get me wrong, it changed a lot internally, probably became more reliable and now faster, but in terms of user
ergonomics, I haven't personally witnessed significant improvements since the early 2010's.

I think these two features (or any feature solving these use cases) could be a major advancement for Ruby's developer experience.
