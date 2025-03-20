---
layout: post
title:  "Database Protocols Are Underwhelming"
date:   2025-03-21 09:03:51 +0100
categories: performance
---

If you've been in this trade for a while, you have probably seen dozens of debates on the merits and problems of SQL
as a relational database query language.
As an ORM maintainer, I have a few gripes with SQL, but overall it is workable, and anyway, it has so much inertia
that there's no point fantasizing about a replacement.

However one database-adjacent topic I don't think I've ever seen any discussions about, and that I think could be improved,
is the protocols exposed by these databases to execute queries.
Relational databases are very impressive pieces of technology, but their client protocol makes me wonder if they ever
considered being used by anything other than a human typing commands in a CLI interface.

I also happen to maintain the Redis client for Ruby, and while the Redis protocol is far from perfect, I think
there are some things it does better than PostgreSQL and MySQL protocols, which are the two I am somewhat familiar with.

## Mutable State, Lot Of Mutable State

You've probably never seen them, because they're not logged by default, but when Active Record connects to your database
it starts by executing several database-specific queries, which I generally call the "prelude".

Which queries are sent exactly depends on how you configured Active Record, but for most people, it will be the default.

In the case of MySQL it will look like this:

```sql
SET  @@SESSION.sql_mode = CONCAT(@@sql_mode, ',STRICT_ALL_TABLES,NO_AUTO_VALUE_ON_ZERO'),
     @@SESSION.wait_timeout = 2147483
```

For PostgreSQL, there's a bit more:

```sql
SET client_min_messages TO 'warning';
SET standard_conforming_strings = on;
SET intervalstyle = iso_8601;
SET SESSION timezone TO 'UTC'
```

In both cases the idea is the same, we're configuring the connection, making it behave differently.
And there's nothing wrong with the general idea of that, as a database gets older, new modes and features get introduced
so for backward compatibility reasons you have to opt-in to them.

My issue with this however is that you can set these at any point.
They're not restricted to an initial authentication and configuration step, so when as a framework or library you hand
over a connection to user code and later get it back, you can't know for sure they haven't changed any of these settings.
Similarly, it means you have both configured and unconfigured connections and must be careful to never use an unconfigured one.
It's not the end of the world but noticeably complexifies the connection management code.

This statefulness also makes it hard if not impossible to recover from errors. If for some reason a query fails, it's hard
to tell which state the connection is in, and the only reasonable thing to do is to close it and start from scratch with a new connection.

If these protocols had an explicit initial configuration phase, it would make it easier to have some sort of "reset state"
message you could send after an error (or after letting user code run unknown queries) to get the connection back to a known clean state.

From a Ruby client perspective, it would look like this:
```ruby
connection = MyDB.new_connection
connection.authenticate(user, password)
connection.configure("SET ...")
connection.query("INSERT INTO ...")
connection.reset
```

You could even cheaply reset the state whenever a connection is checked back into a connection pool.

I'm not particularly knowledgeable about all the constraints database servers face, but I can't think of a reason why such
protocol feature would be particularly tricky to implement.

## Safe Retries

One of the most important jobs of a database client, or network clients in general, is to deal with network errors.

Under the hood, most if not all clients will look like this:

```ruby
def query(command)
  packet = serialize(command)
  @socket.write(command)
  response = @socket.read
  deserialize(response)
end
```

It's fairly trivial, you send the query to the server and read the server response.
The difficulty however is that both the `write` and the `read` operations can fail in dozens of different ways.

Perhaps the server is temporarily unreachable and will work again in a second or two.
Or perhaps it's reachable but was temporarily overloaded and didn't answer fast enough so the client timeout was reached.

These errors should hopefully be rare, but can't be fully avoided.
Whenever you are sending something through the network, there is a chance it might not work, it's a fact of life.
Hence a client should try to gracefully handle such errors as much as possible, and there aren't many ways to do so.

The most obvious way to handle such an error is to retry the query, the problem is that most of the time, from the point
of view of the database client, it isn't clear whether it is safe to retry or not.

In my view, the best feature of `HTTP` by far is its explicit verb specification.
The HTTP spec clearly states that clients, and even proxies, are allowed to retry some specific verbs such as `GET` or `DELETE`
because they are [idempotent](https://en.wikipedia.org/wiki/Idempotence#Computer_science_meaning).

The reason this is important is that whenever the `write` or the `read` fails, in the overwhelming majority of cases,
you don't know whether the query was executed on the server or not.
That is why idempotency is such a valuable property, by definition an idempotent operation can safely be executed twice,
hence when you are in doubt whether it was executed, you can retry.

But knowing whether a query is idempotent or not with SQL isn't easy.
For instance, a simple `DELETE` query is idempotent:

```sql
DELETE
FROM articles
WHERE id = 42;
```

But one can perfectly write a `DELETE` query that isn't:

```sql
DELETE
FROM articles
WHERE id IN (
  SELECT id
  FROM articles
  LIMIT 10
);
```

So in practice, database clients can't safely retry on errors, unless the caller instructs them that it is safe to do so.
You could attempt to write a client that parses the queries to figure out whether they are idempotent, but it is fraught with peril,
hence it's generally preferable to rely on the caller to tell us.

That's one of the reasons why I've been slowly refactoring Active Record lately, to progressively make it easier to retry
more queries in case of network errors.
But even once I'll be done with that refactoring, numerous non-idempotent queries will remain, and whenever they fail,
there is still nothing Active Record will be able to do about it.

## Idempotency Keys

However, there are solutions to turn non-idempotent operations into idempotent ones, using what is sometimes called "Idempotency Keys".
If you've used [the Stripe API](https://docs.stripe.com/api/idempotent_requests), perhaps you are already familiar with them.
I suspect they're not the first ones to come up with such a solution, but that's where I was first exposed to it.

Conceptually it's rather simple, when performing a non-idempotent operation, say creating a new customer record, you can
add an `Idempotency-Key` `HTTP` header containing a randomly generated string.
If for some reason you need to retry that request, you do it with the same idempotency key, allowing the Stripe API to
check if the initial request succeeded or not, and either perform or discard the retry.

They even go a bit further, when a request with an idempotency key succeeds, they record the response so that in case of
a retry, they return you exactly the original response. Thanks to this feature, it is safe to retry all API calls to their
API, regardless of whether they are idempotent or not.

This is such a great feature that last year, at Rails World 2024, when I saw there was a ValKey booth, hosted by
[Kyle Davis](https://fosstodon.org/@linux_mclinuxface), I decided to go have a chat with him, to see if perhaps ValKey
was interested in tackling this fairly common problem.

Because everything I said about SQL and idempotency also applies to Redis (hence to ValKey).
It is also hard for a Redis client to know if a query can safely be retried, and for decades, long before I became the
maintainer, the Redis client would [retry all queries by default](https://github.com/avgerin0s/redis-rb/blob/f17a33f05146d29256622e7736abe00870aed6ef/lib/redis.rb#L139-L152).

At first, it would only do so in case of `ECONNRESET` errors, but over time more errors were added to the retry list.
I must admit I'm not the most knowledgeable person about `TCP`, so perhaps it is indeed safe to assume the server never
received the query when such an error is returned, but over time more and more errors were added to the list, and I highly
doubt all of them are safe to retry.

That's why when I later wrote `redis-client`, a much simpler and lower-level client for Redis, I made sure not to retry by
default, as well as a way to distinguish idempotent queries by having both a `call` and a `call_once` method.

But from the feedback I got when Mike Perham replaced the `redis` gem with `redis-client` in Sidekiq, lots of users
started noticing reports of errors they wouldn't experience before, showing how unreliable remote data stores can be in practice,
especially in cloud environments.

So even though these retries were potentially unsafe, and may have occasionally caused data loss, they were desired by users.

That's why I tried to pitch an idempotency key kind of feature to Kyle, and he encouraged me to open [a feature request
in the ValKey repo](https://github.com/valkey-io/valkey/issues/1087). After a few rounds of discussion, the ValKey core
team accepted the feature, and while as far as I know it hasn't been implemented yet, the next version of ValKey will likely
have it.

It is again pretty simple conceptually:

```SQL
MULTISTORE 699accd1-c7fa-4c40-bc85-5cfcd4d3d344 EX 10
INC counter
LPOP queue
EXEC
```

Just like with Stripe's API, you start a transaction with a randomly generated key, in this case, a UUID, as well as an expiry.

In the example above we ask ValKey to remember this transaction for the next 10 seconds, that's for how long we can safely
retry, after that ValKey can discard the response.

Assuming the next version of ValKey ships with the feature, that should finally offer a solution to safely retry all possible queries.

I fully understand that relational databases are much bigger beasts than an in-memory key-value store, hence it likely is harder
to implement, but if I was ever asked what feature MySQL or PostgreSQL could add to make them nicer to work with, it certainly would be this one.

In the case of ValKey, given it's a text protocol that meant introducing a new command, but MySQL and PostgreSQL both have
binary protocols, with distinct packet types, so I think it would be possible to introduce at the protocol level with
no change to their respective SQL syntax, and no backward compatibility concerns.

## Prepared Statements

Another essential part of database protocols that I think isn't pleasant to work with is prepared statements.

Prepared statements mostly serve two functions, the most important one is to provide a query and its parameters separately,
as to eliminate the risk of SQL injections.
In addition to that, it can in some cases help with performance, because it saves on having to parse the query every time,
as well as to send it down the wire. Some databases will also cache the associated query plan.

Here's how you use prepared statements using the MySQL protocol:

  - First send a `COM_STMT_PREPARE` packet with the parametized query (`SELECT * FROM users WHERE id = ?`).
  - Read the returned `COM_STMT_PREPARE_OK` packet and extract the `statement_id`.
  - Then send a `COM_STMT_EXECUTE` with the `statement_id` and the parameters.
  - Read the `OK_Packet` response.
  - Whenever you no longer need that prepared statement, send a `COM_STMT_CLOSE` packet with the `statement_id`.

Now ideally, you execute the same statements relatively often, so you keep track of them, and in the happy path you
can perform a parameterized query in a single roundtrip by directly sending a `COM_STMT_EXECUTE` with the known `statement_id`.

But one major annoyance is that these `statement_id` are session-scoped, meaning they're only valid with the connection
that was used to create them.
In a modern web application, you don't just have one connection, but a pool of them, and that's per process, so you need
to keep track of the same thing many times.

Worse, as explained previously, since closing and reopening the connection is often the only safe way to recover from errors,
whenever that happens, all prepared statements are lost.

These statements also have a cost on the server side. Each statement requires some amount of memory in the database server.
So you have to be careful not to create an unbounded amount of them, which for an ORM isn't easy to enforce.

It's not rare for applications to dynamically generate queries based on user input, typically some advanced search or filtering form.

In addition, Active Record allows you to provide SQL fragments, and it can't know whether they are static strings or dynamically
generated ones. For example, it's not good practice, but users can perfectly do something like this:

```ruby
Article.where("published_at > '#{Time.now.to_s(db)}'")
```

Also, if you have [Active Record query logs](https://api.rubyonrails.org/classes/ActiveRecord/QueryLogs.html), then most
queries will be unique.

All this means that a library like Active Record has to have lots of logic to keep track of prepared statements and their
lifetime. You might even need some form of Least Recently Used logic to prune unused statements and free resources on the server.

In many cases, when you have no reason to believe a particular query will be executed again soon, it is actually advantageous
not to use prepared statements.
Ideally, you'd still use a parameterized query, but then it means doing 2-3 rountrips[^1] to the database instead of just one.

So for MySQL at least, when you use Active Record with a SQL fragment provided as a string, Active Record fallback to
not use prepared statements, and instead interpolate the parameters inside the query.

Ideally, we'd still use a parameterised query, just not a prepared one, but the MySQL protocol doesn't offer such functionality.
If you want to use parameterized queries, you have to use prepared statements and in many cases, that will mean an extra roundtrip.

I'm much less familiar with the PostgreSQL protocol, but from glancing at its specification I believe it works largely in the same way.

So how could it be improved?

First I think it should be possible to perform parameterized queries without a prepared statement, I can't think of a reason
why this isn't a possibility yet.

Then I think that here again, some inspiration could be taken from Redis.

## EVALSHA

Redis doesn't have prepared statements, that wouldn't make much sense, but it does have something rather similar in
the form of [Lua scripts](https://valkey.io/topics/eval-intro/).

```
> EVAL "return ARGV[1] .. ARGV[2]" 0 "hello" "world!"
"helloworld!"
```

But just like SQL queries, Lua code needs to be parsed and can be relatively large, so caching that operation is preferable for
performance.
But rather than a `PREPARE` command that returns you a connection-specific identifier for your given script, Redis
instead use SHA1 digests.

You can first load a script with the `SCRIPT LOAD` command:

```
> SCRIPT LOAD "return ARGV[1] .. ARGV[2]"
"702b19e4aa19aaa9858b9343630276d13af5822e"
```

Then you can execute the script as many times as desired by only referring its digest:

```
> EVALSHA "702b19e4aa19aaa9858b9343630276d13af5822e" 0 "hello" "world!"
"helloworld!"
```

And that script registry is global, so even if you have 5000 connections, they can all share the same script, and you can
even assume scripts have been loaded already, and load them on a retry if they weren't:

```ruby
require "redis-client"
require "digest/sha1"

class RedisScript
  def initialize(src)
    @src = src
    @digest = Digest::SHA1.hexdigest(src)
  end

  def execute(connection, *args)
    connection.call("EVALSHA", @digest, *args)
  rescue RedisClient::CommandError
    connection.call("SCRIPT", "LOAD", @src)
    connection.call("EVALSHA", @digest, *args)
  end
end

CONCAT_SCRIPT = RedisScript.new(<<~LUA)
  return ARGV[1] .. " " .. ARGV[2]
LUA

redis = RedisClient.new
p CONCAT_SCRIPT.execute(redis, 0, "Hello", "World!")
```

I'm not a database engineer, so perhaps there's some big constraint I'm missing, but I think it would make a lot of sense
for prepared statement identifiers to be some sort of predictable digests, so that they are much more easily shared
across connection, and let the server deal with garbage-collecting prepared statements that haven't been seen in a long
time, or use some sort of reference counting strategy.

## Conclusion

I could probably find a few more examples of things that are impractical in MySQL and PostgreSQL protocols, but I think
I've shown enough to share my feelings about them.

Relational databases are extremely impressive projects, clearly built by very smart people, but It feels like the developer
experience isn't very high on their priority list, if it's even considered.
And that perhaps explains part of the NoSQL appeal in the early 2010's.
However, I think it would be possible to significantly improve their usability without changing the query language, just by improving the query
protocol.

[^1]: 3 roundtrips in total, but you theoretically can do the `COM_STMT_CLOSE` asynchronously.