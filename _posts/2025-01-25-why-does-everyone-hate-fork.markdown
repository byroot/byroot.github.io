---
layout: post
title:  "Why Does Everyone Hate fork(2)?"
date:   2025-01-25 10:47:51 +0100
categories: ruby performance
---

I want to write a post about [Pitchfork](https://rubygems.org/gems/pitchfork), explaining where it comes from, why it
is like it is, and how I see its future.
But before I can get to that, I think I need to explain a few things, namely why in many circles `fork` is seen as a
relic of the past, if not outright the devil's creation.
And yet, it's ubiquitous in the Ruby ecosystem.

Note that if you have some system programming experience, you probably won't learn much here.

If you've ever deployed a Ruby application to production, it is almost certain you've interacted with
[`fork(2)`](https://man7.org/linux/man-pages/man2/fork.2.html) whether you realize it or not.
Have you configured Puma's `worker` setting? Well, Puma uses `fork(2)` to spawn these workers, more accurately the Ruby
[`Process.fork`](https://docs.ruby-lang.org/en/3.4/Process.html#method-c-fork) method, which is the Ruby API for
the underlying `fork(2)` syscall.

And even if you're not a Rubyist, if you've used PHP, Nginx, Apache HTTPd, Redis, and many others you've used a system
that is heavily relient on `fork(2)`, if not entirely architectured around it.

Yet, [many people would argue that `fork(2)` is evil and shouldn't be used](https://www.microsoft.com/en-us/research/uploads/prod/2019/04/fork-hotos19.pdf).
Personally I kinda both agree and disagree with that point of view, and I'll try to explain why.

## A Bit Of History

According to Wikipedia, the first occurrence of the fork concept dates all the way back to 1962 by the same guy who
coined [Conway's law](https://en.wikipedia.org/wiki/Conway%27s_law), and was later introduced in the first versions of UNIX.

Initially, it was meant as a primitive to create a new process. You'd call `fork(2)` to make a copy of the current process
and from there would mutate that new process into what you want it to be, quickly ending up with an `exec(2)`.
You can still do this today in Ruby:

```ruby
if (child_pid = Process.fork)
  # We're in the parent process, and we know the child process ID.
  # We can wait for the child to exit or send signals etc.
  Process.wait(child_pid)
else
  # We're in the child process.
  # We can change the current user and other attributes.
  Process.uid = 1
  # And then we can replace the current program with another.
  Process.exec("echo", "hello")
end
```

In a way that design was quite elegant. You have a handful of simple primitives you can compose together to get
exactly the behavior you need, instead of one huge function that takes a myriad of arguments.

But it is also very inefficient, as entirely duplicating a process to create a new one is generally overkill.
In the example above, if you imagine that our parent program has gigabytes of addressable memory, it's a huge waste
to copy all of that just to throw it all out almost immediately to replace it with an extremely small program like `/bin/echo`.

Of course, modern operating systems don't actually copy all that, and instead use [Copy-on-Write](https://en.wikipedia.org/wiki/Copy-on-write#In_virtual_memory_management),
but that's still very costly, and can easily take hundreds of milliseconds if the parent process is big.

That's why this historical usage of `fork(2)` to spawn other programs is mostly considered deprecated today, and most
newer software will use more modern APIs such as `posix_spawn(3)` or `vfork(2)+exec(2)`.
 
But that's not the only use of `fork(2)`. I have no idea if this was envisioned right from the start, or if it just became
a thing, but all the software I listed in the introduction uses `fork(2)` without ever following it with an `exec(2)` call.

## Fork as a Parallelism Primitive

Again, I wasn't even born in the early seventies, so I'm not too sure when this practice really started but at some
point `fork(2)` started being used as a parallelism primitive, particularly for servers.

Let's say you want to implement a simple "echo" server from scratch, in Ruby it might look like this:

```ruby
require 'socket'

server = TCPServer.new('localhost', 8000)

while socket = server.accept
  while line = socket.gets
    socket.write(line)
  end
  socket.close
end
```

This script first opens a listening socket on port `8000`, then blocks on the `accept(2)` syscall to wait for a client
to connect. When that method returns, it gives us a bidirectional socket, from which we can read, in this case with `#gets`,
and also write back to the client.

While this is using modern Ruby, that's very similar to how various servers would be written back then, but overly simplified.

If you want to play with it, you can use `telnet localhost 8000` and start writing things.

But there's one big issue with that server: it only supports a single concurrent user.
If you try to have two `telnet` sessions active, you'll see the second one can't connect.

So what people started doing, was to leverage `fork(2)` to be able to support more users:

```ruby
require 'socket'

server = TCPServer.new('localhost', 8000)
children = []

while socket = server.accept
  # prune exited children
  children.reject! { |pid| Process.wait(pid, Process::WNOHANG)}

  if (child_pid = Process.fork)
    children << child_pid
    socket.close
  else
    while line = socket.gets
      socket.write(line)
    end
    socket.close
    Process.exit(0)
  end
end
```

The logic is the same as before, but now once `accept(2)` returns us a socket, instead of blocking on it,
we `fork(2)` a new child process, and let that child do the blocking operations until the client closes the connection.

If you are an astute reader (or simply already knowledgeable about `fork(2)` semantics), you may have noticed that after
the call to `fork`, both the parent and the new children have access to the socket. That is because, in UNIX, sockets are
"files", hence represented by a "file descriptor", and part of the `fork(2)` semantic is that all file descriptors are
also inherited.

That's why it is important that the parent close the socket, otherwise, it will stay open forever in the parent process[^1],
And this is one of the first reasons why many people hate `fork(2)`.

## A Double-Edged Sword

As showcased above, the fact that child processes inherit all open file descriptors allows to implement some very useful things,
but it can also cause catastrophic bugs if you forget to close a file descriptor you didn't mean to share.

For instance, if you are forking a process that has an active connection to a SQL database, and you keep using that
connection in both processes, weird things will happen:

```ruby
require "bundler/inline"
gemfile do
  gem "trilogy"
  gem "bigdecimal" # for trilogy
end

client = Trilogy.new
client.ping

if child_pid = Process.fork
  sleep 0.1 # Give some time to the child

  5.times do |i|
    p client.query("SELECT #{i}").first[0]
  end
  Process.kill(:KILL, child_pid)
  Process.wait(child_pid)
else
  loop do
    client.query('SELECT "oops"')
  end
end
```

Here the script establishes a connection to MySQL, using the `trilogy` client, then forks a child
that queries `SELECT "oops"` indefinitely in a loop. Once the child is spawned, the parent issues 5 queries,
each one supposed to return a single number from 0 to 4, and print their result.

If you run this script, you'll get a somewhat random output, similar to this:

```
"oops"
1
"oops"
"oops"
3
```

What's happening here is that both processes are writing inside the same socket. For the MySQL server, it's not a big
deal because our queries are small, so they're somewhat "atomically" written into the socket if we were to issue larger
queries, two queries might end up interleaved, which would cause the server to close the connection with some form of
protocol error.

But for the client, it's really bad. Because the responses of both processes are sent back in the same socket, and
each client is issuing `read(2)` and might be getting the response to the query it just issued, but the response of
another unrelated query issued by the other process.

When two processes try to `read(2)` on the same socket, they each get part of the data, but you don't have proper control
over which process gets what, and it's unrealistic to try to synchronize the two processes so they each get the response
they expect.

With this in mind, you can imagine how much of a hassle it can be to properly close all the sockets and other open files
of an application before you call `fork(2)`. Perhaps you can be diligent in your own code, but you likely are using some
libraries that may not expect `fork(2)` to be called and don't allow you to close their file descriptors.

For the `fork+exec` use case, there's a nice feature that makes this much easier, you can mark a file descriptor as needing
to be closed when `exec` is called, and the operating system takes care of that for you, `O_CLOEXEC` (for close on exec),
which in Ruby is conveniently exposed as a method on the `IO` class:

```ruby
STDIN.close_on_exec = true
```

But there's no such flag for the `fork` system call when it's not followed by an `exec`. Or more accurately
there is one, `O_CLOFORK`, which has existed on a few UNIX systems, mostly IBM ones, and [was added to the POSIX spec in 2020](https://austingroupbugs.net/view.php?id=1318).
But it isn't widely supported today, most importantly Linux doesn't support it.
[Someone submitted a patch to add it to Linux in 2011](https://lore.kernel.org/all/1304749754.2821.712.camel@edumazet-laptop/T/#m2a7dbb0f0f6106b3d9ecc8c485a683ea6b2e02ee),
but it seems there wasn't much appetite for it, and [someone else made another attempt in 2020](https://lore.kernel.org/lkml/20200525081626.GA16796@amd/T/#m4ef81228aba3f9524329f83a124d0322ed53f834),
but it encountered some strong opposition, which is a shame, because it would be tremendously useful.

Instead, what most code that wants to be fork-safe does, it either trying to detect a fork happened by continuously checking
the current process ID:

```ruby
def query
  if Process.pid != @old_pid
    @connection.close
    @connection = nil
    @old_pid = Process.pid
  end

  @connection ||= connect
  @connection.query
end
```

Or alternatively rely on some `at_fork` callback, in C land usually it is [`pthread_atfork`](https://man7.org/linux/man-pages/man3/pthread_atfork.3.html),
and [since Ruby since 3.1, you can decorate `Process._fork`](https://bugs.ruby-lang.org/issues/17795) (note the `_`):

```ruby
module MyLibraryAtFork
  def _fork
    pid = super
    if pid == 0
      # in child
    else
      # in parent
      MyLibrary.close_all_ios
    end
    pid
  end
end
Process.singleton_class.prepend(MyLibraryAtFork)
```

Since `fork(2)` is quite ubiquitous in Ruby, many popular libraries that deal with sockets, such as Active Record, or
the `redis` gem, do their best to take care of this transparently, so you don't have to think about it.
Hence in most Ruby programs it just works.

But with native languages, it can be quite tedious and that's one of the reasons why many people absolutely hate `fork(2)`.
Any code that makes use of files or sockets might be utterly broken after `fork(2)` has been called, unless special attention
was paid to fork safety, which is rarely the case.

## Some of Your Threads May Die

Going back to our small echo server, you may wonder why one would use `fork(2)` instead of a thread here.
Again, I wasn't there at the time, but my understanding is that threads became a thing much later (late eighties?),
and even once they existed, they took quite a while to be standardized and ironed out, hence usable across platforms.

There is also probably an argument that multi-processing with `fork(2)` is easier to reason about. Each process has its
own memory space, so you don't have to concern yourself as much with race conditions and other thread pitfalls, so I can
see why even when threads became an option, some may have preferred to stick with `fork(2)`.

But since threads became a thing long after `fork(2)`, it seems that the people in charge of implementing and
standardizing them ran into a bit of a pickle, and didn't find a way to make them both play well together.

Here's what [the POSIX standard fork entry](https://pubs.opengroup.org/onlinepubs/009696799/functions/fork.html) says about that:

> A process shall be created with a single thread.
> If a multi-threaded process calls fork(), the new process shall contain a replica of the calling thread and its entire
> address space, possibly including the states of mutexes and other resources.
> Consequently, to avoid errors, the child process may only execute async-signal-safe operations until such time as one
> of the exec functions is called.

In other words, the standard acknowledges that the classic `fork+exec` dance can be done from a multi-threaded process,
but kind of wash its hands about the use of `fork` not followed by `exec`. They recommend only using async-signal-safe
operations, which is really just a very small subset of things. So really, according to the standard, if you call
`fork(2)` after some threads have been spawned, without the intention to call `exec` quickly, then here be dragons.

The reason is that only the thread which called `fork(2)` remains alive in the children, all the other threads
are present but dead. If another thread had locked a mutex or something like that, it would stay locked forever,
which might lead to a deadlock if a new thread tries to acquire it.

The standard also includes a rationale section about why it is this way, which is a bit long but interesting:

> The general problem with making fork() work in a multi-threaded world is what to do with all of the threads.
> There are two alternatives. 
> One is to copy all of the threads into the new process.
> This causes the programmer or implementation to deal with threads that are suspended on system calls or that might be
> about to execute system calls that should not be executed in the new process.
> The other alternative is to copy only the thread that calls fork().
> This creates the difficulty that the state of process-local resources is usually held in process memory.
> If a thread that is not calling fork() holds a resource, that resource is never released in the child process because
> the thread whose job it is to release the resource does not exist in the child process.
>
> When a programmer is writing a multi-threaded program, [...]
> **The fork() function is thus used only to run new programs**, and the effects of calling functions that require certain
> resources between the call to fork() and the call to an exec function are undefined.
>
> The addition of the forkall() function to the standard was considered and rejected.

So they did consider the possibility of having another version of `fork(2)`, called `forkall()` which would also have
copied other threads, but they couldn't come up with a clear semantic on what happens in some cases.

Instead, they gave users a way to have a callback invoked around `fork` to restore state, for instance, re-initialize mutexes.
However, if you go look at [that callback man page `pthread_atfork(3)`](https://man7.org/linux/man-pages/man3/pthread_atfork.3.html),
you can read:

> The original intention of pthread_atfork() was to allow the child process to be returned to a consistent state. [...]
> In practice, this task is generally too difficult to be practicable.

So while `pthread_atfork` is still there and you can use it, the standard acknowledges that it is very hard to use correctly.

That's why many system programmers will tell you to never mix `fork(2)` with multi-threaded programs, or at least never
to call `fork(2)` ever after a thread was spawned, because then, all bets are off. Hence, you somewhat had to choose your
camp, and it seems threads clearly won.

But that's for C or C++ programmers.

In the case of today's Ruby programmers, however, the reason to use `fork(2)` over threads, is that it's the only way
to get true parallelism [^2]. Because of the infamous GVL, Ruby threads only really allow to parallelize IO operations,
and can't parallelize Ruby code execution, hence pretty much all Ruby application servers integrate with `fork(2)` in
some way so they can exploit more than a single CPU core.

Luckily, some of the pitfalls of mixing threads with `fork(2)` are alleviated by Ruby.
For instance, Ruby mutexes are automatically released when their owner dies, due to how they are implemented.
In pseudo Ruby code they'd look like this:

```ruby
class Mutex
  def lock
    if @owner == Fiber.current
      raise ThreadError, "deadlock; recursive locking"
    end

    while @owner&.alive?
      sleep(1)
    end

    @owner = Fiber.current
  end
end
```

Of course in reality they're not sleeping in a loop to wait, they use a much more efficient way to block, but it's to
give you the general idea.
The important point is that Ruby mutexes keep a reference to the fiber (hence thread) that acquired the lock,
and automatically ignore it if it's dead.
Hence upon fork, all mutexes held by the background thread are immediately released, which avoids most
deadlock scenarios.

It's not perfect of course, if a thread died while holding a mutex, it's very possible that it left the resource that was protected
by the mutex in an inconsistent state, in practice however I've never experienced something like that, granted it's likely
because the existence of the GVL somewhat reduces the need for mutexes.

Now, Ruby threads aren't fully exempt from these pitfalls, because ultimately on MRI, Ruby threads are backed by native threads,
so you can end up with a nasty deadlock after forking if another thread released the GVL and called a C API that locks a mutex.

While I never got hard proof of it, I suspect this was happening to some Ruby users because from my understanding,
glibc's [`getaddrinfo(3)`](https://man7.org/linux/man-pages/man3/getaddrinfo.3.html),
which Ruby uses to resolve host names, does use a global mutex, and Ruby calls it with the GVL released, allowing for a
fork to happen concurrently.

To prevent this, [I added another lock inside MRI](https://bugs.ruby-lang.org/issues/20590), to prevent `Process.fork`
from happening while a `getaddrinfo(3)` call is ongoing.
This is far from perfect, but given how much Ruby relies on `Process.fork`, that seemed like a sensible thing to do.

It's also not rare for Ruby programs that rely on fork to run into crashes on macOS, because numerous macOS system APIs
do implicitly spawn threads or lock mutexes, and [macOS chose to consistenty crash when it happens](https://github.com/rails/rails/issues/38560).

So even with pure Ruby code, you occasionally run into `fork(2)`'s pitfalls, you can't just use it willy-nilly.

## Conclusion

So to answer the question in the title, the reason `fork(2)` is hated is because it doesn't compose well, particularly
in native code.
If you wish to use it, you have to be extremely careful about the code you are writing and linking to.
Whenever you use a library you have to make sure it won't spawn some threads, or hold onto file descriptors,
and given the choice between `fork(2)` and threads, most systems programmers will choose threads. They have their own
pitfalls, but they compose better, and it is likely that you are calling into APIs that are using threads under the
hoods, so the choice is somewhat already made for you.

But the situation isn't nearly as bad for Ruby code, as it makes it much easier to write fork-safe code, and the Ruby
philosophy makes it so libraries like Active Record take it upon themselves to deal with these gnarly details for you.
So problems mostly come up when you want to bind to some native libraries that spawn threads, like `grpc` or `libvips`,
as they generally don't expect `fork(2)` to happen and aren't generally kin in accepting it as a constraint.

Especially since fork is mostly used at the end of the application initialization, even libraries
that are technically not fork-safe, will work because they generally initialize their threads and file descriptors
lazily upon the first request.

Anyway, even if you still think `fork(2)` is evil, until Ruby offers another usable primitive for true parallelism
(which should be the subject of the next post), it will remain a necessary evil.

[^1]: Technically, Ruby will automatically close it once the object is garbage collected, but you get the idea.
[^2]: Yes, there are also Ractors to some extent, but that will be the subject of the next post.
