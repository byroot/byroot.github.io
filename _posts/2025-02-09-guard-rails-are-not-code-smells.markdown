---
layout: post
title:  "Guardrails Are Not Code Smells"
date:   2025-02-09 10:47:51 +0100
categories: ruby performance
---

I want to write a post about [Pitchfork](https://rubygems.org/gems/pitchfork), explaining where it comes from, why it
is like it is, and how I see its future.
But before I can get to that, I think I need to share my mental model on a few things, in this case, resiliency.

A few years ago, I read a tweet calling out [`unicorn-worker-killer`](https://github.com/kzk/unicorn-worker-killer) as a
major code smell.
I don't quite remember who or when it was, it doesn't matter anyway, but I think it is an interesting topic because,
depending on how that gem is used, I can either strongly agree with or vehemently oppose this tweet.

## What Does It Do?

`unicorn-worker-killer` provides two optional features for [Unicorn](https://yhbt.net/unicorn/README.html).

The first one allows to set a number of requests after which a Unicorn worker should be recycled, as in shutdown and
replaced by a fresh one, and the second allows to set the memory usage threshold after which a worker should be recycled.

In my opinion, the first setting is indeed a terrible code smell and there's absolutely no valid reason to use it.
However the second one is absolutely essential, and in my opinion, it's borderline irresponsible to deploy code to
production without something akin to it.

Both features are meant as ways to deal with memory leaks or bloat, so you may wonder why I have such radically different
opinions on what seem to be two fairly similar features.
It is because the former sweeps memory leaks under the rug, while the latter gracefully handles them.

## Why MaxRequests is Bad

One of the reasons why memory leaks are such a painful bug to deal with in Ruby applications is that unless they
are really bad, they'll often take hours or days to bloat your process enough for the system OOM killer to trigger.
So contrary to most bugs, they're not easy to correlate with an application deployment.
Assuming you deploy several times a day, you may only notice something went wrong over the weekend after you haven't
deployed for a day or two. In some cases, you may even only notice it months later when most of the team is away for summer
or winter vacations and no deployment happens for a week.

`Unicorn::WorkerKiller::MaxRequests` "solves" that by essentially continuously restarting your application on a regular
schedule, making it impossible to even know you are suffering from a memory leak.

And not only is it sweeping issues under the rug, but it also negatively impacts the performance of your application.
While restarting a Unicorn worker is a relatively cheap operation, Ruby processes need some warmup time.
Even if we ignore JIT warmup, the regular interpreter has lots of inline caches, so the first time a code path is
executed it needs to do some extra work, and that work tends to cause Copy-on-Write invalidations, further impacting latency.
And beyond the VM itself, while it's a bad pattern, it's very common for Ruby application to have some lazily initialized state.

That is why I believe `Unicorn::WorkerKiller::MaxRequests` is a major code smell.
It is an indication that it was acknowledged that a memory leak exists, but that the team has given up on trying to fix it.

Even if you are aware of this, you may think it's an acceptable tradeoff for your use case, but it may only be a matter
of time until you don't have one but two or ten memory leaks and then need to face the problem or keep lowering the
`max_requests` setting.

## Why MaxMemory is Good

On the other hand, having a `max_memory` setting that gracefully restarts workers when they reach a given memory threshold
is an essential resiliency feature.

If your application doesn't enforce a max memory policy by itself, the operating system will do it for you, and it won't
be pretty.
What happens when there is no more available memory depends on the operating system and how it is configured but in the
context of Ruby programs, the most likely case is that you'll run into the Linux OOM Killer.
In short, when Linux can no longer hand memory to a program, it looks at all the running processes on the machine, and
use a few heuristics to find a process to kill to free some memory.
Nothing guarantees that the process that will be killed is the one leaking, and the process will receive a `SIGKILL` so
it won't be able to shut down cleanly and may cause huge problems.

For the anecdote, over a decade ago, I worked for a startup that had a PHP and Backbone.js application.
All of it, including the MySQL server, was running on a single machine.
One day there was a spike of traffic and the number of PHP processes caused the Linux OOM Killer to trigger, and its
heuristics thought that the big `mysqld` process was a much more attractive target than the hundreds of 30 MB PHP processes,
hence it decided to SIGKILL `mysqld` bringing the entire website to the ground.

That's why if you don't enforce a max memory policy yourself, you will be subject to the overall system memory limits
and if you ever reach them it won't be pretty.

That's why I believe `Unicorn::WorkerKiller::Oom` is an essential resiliency feature, even though it needs to be used
correctly.
It is crucial that every OOM event is reported somewhere and that there is alerting in place if it becomes
a frequent occurrence. If no one notices that suddenly OOM events are through the roof, then it's no better than `max_request`.

You can even go farther than that.
Something I implemented in Shopify's monolith, is that for a sample of OOM events before the worker is recycled,
the heap is first dumped using [`ObjectSpace.dump_all`](https://docs.ruby-lang.org/en/3.4/ObjectSpace.html#method-i-dump_all)
and uploaded in an object store, allowing us to investigate the root cause and identify leaks if any.

This is the best of both worlds.
OOM events are gracefully handled, any significant increase in such events is reported, and debugging is facilitated.

It might be that your application legitimately started using more memory and you just need to increase the threshold,
but it is preferable to have to review the policy once in a while than to be paged at night because a hard-to-diagnose
memory leak was introduced and started causing havoc.

On another note, I'm not a big fan of the exact implementation provided by `unicorn-worker-killer`, because it uses
[RSS](https://en.wikipedia.org/wiki/Resident_set_size) as its memory metric, and I believe
[PSS](https://en.wikipedia.org/wiki/Proportional_set_size) is a much better memory metrics for preforking applications[^1].

In any case, cleanly shutting down workers on your own terms outside of the request cycle is much preferable to letting
the operating system sends `SIGKILL` to somewhat random processes.

## Accept It: Bugs Will Happen

Here I used memory as an example because I think it's a good illustration of where to draw the line.
But the point I'm trying to make is much more general.

Another example could be Unicorn's built-in request timeout setting.
Whenever a worker is assigned a request, it is allotted a fixed amount of time to complete, if not, the worker will be
shut down and replaced.

You may think this is bad design and is working around a bug, but that's the key to building resilient systems.
You must accept that bugs and other catastrophic events will happen eventually, hence systems should be designed in a
way that limits the blast radius of such events.

This isn't to say bugs are acceptable.
You absolutely should strive to prevent as many bugs as reasonably possible upfront through testing, code reviews and what not.
Having a resilient system is in no way a license to be complacent and ship crap software.

But it is fanciful to believe you can prevent all bugs via sheer competency or process.

If you are telling me your car doesn't have seatbelts because you are such a good driver that you don't need them,
I will run in the opposite direction as fast as I can.

This attitude can function for a while at a small scale but is untenable whenever a project grows to have
a larger team, hence a faster rate of change.

## The Resilient Nature of Share-Nothing Architecture

This is why as an infrastructure engineer, I'm quite a fan of "share-nothing" architecture: it has inherent resiliency benefits.

Take PHP for instance.
As much as I dislike it as a programming language, I have to admit its classic execution model is really nice.
Every request starts with a fresh new process, making it close to impossible to leak state between requests.

That model wouldn't perform well with Ruby though, because of the necessary warmup I mentioned previously,
but Unicorn's model is somewhat halfway through.
Workers do process more than one request, but never concurrently, hence you still benefit from quite a high level of
isolation, and if something goes really wrong, killing the worker is always an option and the operating system will take
care of reclaiming the resources.

Whereas cleanly interrupting a thread is basically impossible.
You probably heard how [Ruby's `Timeout` module should be avoided like the plague](https://jvns.ca/blog/2015/11/27/why-rubys-timeout-is-dangerous-and-thread-dot-raise-is-terrifying/),
well that's because it is trying to interrupt a thread, and that can go really wrong.

If you look at [`rack-timeout`](https://github.com/zombocom/rack-timeout), which is the Puma equivalent to Unicorn's
request timeout.
You'll see that whenever it has to timeout a request, it shutdowns the entire Puma worker because there's no way to
cleanly shut down a single thread.

## Conclusion

As I tried to explain here, even if [Ruby didn't have a GVL](/ruby/performance/2025/01/29/so-you-want-to-remove-the-gvl.html),
and even considering [fork has its share of problems](/ruby/performance/2025/01/25/why-does-everyone-hate-fork.html),
I strongly believe that this execution model has some very attractive properties that aren't easily replaced.

As with everything, it's of course a tradeoff, you gain some and lose some, but it shouldn't be lazily discarded as some
sort of deprecated model for badly written applications.

It also has its benefits, and resiliency is just one of them.

[^1]: I wrote about the difference between RSS and PSS in [a post on Shopify's engineering blog](https://shopify.engineering/ruby-execution-models).