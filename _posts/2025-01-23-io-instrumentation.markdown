---
layout: post
title:  "Instrumenting Rails Application"
date:   2025-01-23 14:50:51 +0100
categories: ruby performance
excerpt: "Recent Ruby releases added some nice tools to better instrument applications"
published: true
---

In [my previous post about how IO-bound Rails applications really are](https://byroot.github.io/ruby/performance/2025/01/23/the-mythical-io-bound-rails-app.html),
I pointed at a common pitfall, how CPU starvation can look like slow IOs.

```ruby
start = Time.now
database_connection.execute("SELECT ...")
query_duration = (Time.now - start) * 1000.0
puts "Query took: #{query_duration.round(2)}ms"
```

In the above example, the instrumentation tells you how long it took for the database to answer your query, but may
also include the time needed for the Ruby thread to re-acquire the GVL, or perhaps the Ruby GC to run, or even the
operating system's scheduler to resume the process.

Thankfully, in recent years Ruby added some new APIs that help measure these things.

## GC.total_time

Database queries and other IOs can often result in lots of allocations. For instance, if you select 100 database rows,
with a dozen columns each, you can probably expect a thousand or more allocations and any of these might trigger a GC
run inside the code block you are timing.

As such it can be a good idea to keep an eye on how much time Ruby is spending in GC. To help with that, [back in 2021
I resurrected an old feature request on the Ruby bug tracker](https://bugs.ruby-lang.org/issues/10917#note-4) and
convinced [Koichi Sadasa to implement the new `GC.total_time` API](https://github.com/ruby/ruby/pull/4757).

This accessor is a monotonic counter, that represents the number of nanoseconds spent in the GC. So to tell how much
time a particular block of code spent in GC, you can do a simple subtraction:

```ruby
def time_in_gc
  before = GC.total_time
  yield
  diff_ms = (GC.total_time - before) / 1_000_000.0

  puts "gc_time: #{diff_ms.round(2)}ms"
end

time_in_gc do
  2_000_000.times { Object.new }
end
```

```
gc_time: 24.18ms
```

Now of course, if you are running a multi-threaded application, you can't just subtract the time spent in GC from the
measured IO time, because another thread might be responsible for it. But it's still a good idea to instrument it
and display it next to the IO duration.

That's why starting from Rails 7.2, [I added this measurement into Rails instrumentation API](https://github.com/rails/rails/pull/51770).
Every `ActiveSupport::Notifications` event now has an associated `gc_time`, and Rails request logs include the overall time spent in GC.

## GVL Instrumentation API

Even more common than GC, is GVL contention. If you configured your application to use too many threads, it can cause
long delays for a thread to resume after finishing some IOs.

That's why [in Ruby 3.2 I added a new C API to allow instrumenting the GVL](https://bugs.ruby-lang.org/issues/18339).

This is quite a low-level API, and you need a C extension to integrate with it, but I wrote
[`gvltools`](https://github.com/Shopify/gvltools for that, and John Hawthorn wrote the
[`gvl_timing` gem](https://github.com/jhawthorn/gvl_timing), and there's
also [`gvl-tracing`](https://github.com/ivoanjo/gvl-tracing) from Ivo Anjo.

Here's how `gvltools` can be used to distinguish actual IO time from GVL wait time:

```ruby
require "bundler/inline"

gemfile do
  gem "bigdecimal" # for trilogy
  gem "trilogy"
  gem "gvltools"
end

GVLTools::LocalTimer.enable

def measure_time
  realtime_start = Process.clock_gettime(Process::CLOCK_MONOTONIC, :float_millisecond)
  gvl_time_start = GVLTools::LocalTimer.monotonic_time
  yield

  realtime = Process.clock_gettime(Process::CLOCK_MONOTONIC, :float_millisecond) - realtime_start
  gvl_time = GVLTools::LocalTimer.monotonic_time - gvl_time_start
  gvl_time_ms = gvl_time / 1_000_000.0
  io_time = realtime - gvl_time_ms
  puts "io: #{io_time.round(1)}ms, gvl_wait: #{gvl_time_ms.round(2)}ms"
end

trilogy = Trilogy.new

# Measure a first time with just the main thread
measure_time do
  trilogy.query("SELECT 1")
end

def fibonacci( n )
  return  n  if ( 0..1 ).include? n
  ( fibonacci( n - 1 ) + fibonacci( n - 2 ) )
end

# Spawn 5 CPU-heavy threads
threads = 5.times.map do
  Thread.new do
    loop do
      fibonacci(25)
    end
  end
end

# Measure again with the background threads
measure_time do
  trilogy.query("SELECT 1")
end
```

If you run this example, you can see that on the first measurement, the GVL wait time is pretty much zero,
but on the second, it adds a massive half-a-second overhead:

```
realtime: 0.22ms, gvl_wait: 0.0ms, io: 0.2ms
realtime: 549.29ms, gvl_wait: 549.22ms, io: 0.1ms
```

The downside of this API however, is that it adds some overhead to Ruby's thread scheduler. I never really managed to
come up with a precise figure of how much overhead, perhaps it's negligible, but until then, it's a bit hard to justify
integrating it as a Rails default.

That being said, recently [Yuki Nishijima from Speedshop open-sourced a middleware](https://github.com/speedshop/gvl_metrics_middleware/tree/main)
to hook this new instrumentation API into various APM services, so it might progressively see broader usage.

## Operating System Scheduler

The one remaining thing that could cause IO operations to appear longer than they really are is the operating scheduler.
Unless you are running your application on dedicated hardware, and spawn no more than one Ruby process per core, then
it can happen that the operating system doesn't immediately resume a process after it is done blocking on some IO.

I'm unfortunately not aware of a really good way to measure this.

The best I've found is `/proc/<pid>/schedstat` on Linux:

```bash
# cat /proc/1/schedstat
40933713 1717706 178
```

The second number in that list is the amount of nanosecond the given process spent in the "runqueue", in other words,
waiting to be assigned a CPU core so it can resume work.

But reading `/proc` around every IO would be a bit heavy-handed, so it's not something I've ever integrated into an
application monitoring. Instead, we monitor it more globally on a per-machine basis as an indication that we're running
too many processes in our containers.


