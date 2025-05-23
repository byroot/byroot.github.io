---
layout: post
title:  "There Isn't Much Point to HTTP/2 Past The Load Balancer"
date:   2025-02-24 20:47:51 +0100
categories: ruby performance
---

I want to write a post about [Pitchfork](https://rubygems.org/gems/pitchfork), explaining where it comes from, why it
is like it is, and how I see its future.
But before I can get to that, I think I need to share my mental model on a few things, in this case, HTTP/2.

From time to time, either online or at conferences, I hear people complain about the lack of support for HTTP/2 in
Ruby HTTP servers, generally Puma.
And every time I do the same, I ask them why they want that feature, and so far nobody had an actual use case for it.

Personally, this lack of support doesn't bother me much, because the only use case I can see for it, is wanting to expose
your Ruby HTTP directly to the internet without any sort of load balancer or reverse proxy, which I understand may seem
tempting, as it's "one less moving piece", but not really worth the trouble in my opinion.

If you are not familiar with the HTTP protocol and what's different in version 2 (and even 3 nowadays), you might
be surprised by this take, so let me try to explain what it is all about.

## What Does HTTP/2 Solve?

HTTP/2 started under the name SPDY in 2009, with multiple goals, but mainly to reduce page load latency, by allowing it to
download more resources faster.
A major factor in page load time is that a page isn't just a single HTTP request.
Once your browser has downloaded the HTML page and starts parsing it, it will find other resources it needs to also
download to render the page, be it stylesheets, scripts, or images.

So a page isn't one HTTP request, but a cascade of them, and in the late 2000s, the number of resources on the average
page kept going up.
This bloat was in part offset by broadband getting better, but still, HTTP/1.1 wasn't really adequate to download
many small files quickly for a few reasons.

The first one is that [RFC 2616](https://datatracker.ietf.org/doc/html/rfc2616), which introduced HTTP/1.1
specified that browsers were only allowed *two* concurrent connections to a given domain:

> 8.1.4 Practical Considerations
>
> Clients that use persistent connections SHOULD limit the number of simultaneous connections that they maintain to a
> given server. A single-user client SHOULD NOT maintain more than 2 connections with any server or proxy.

So if you can only request a single resource per connection, and are limited to two connections, even if you have a very
large bandwidth, the latency to the server will have a massive impact on performance whenever you need to download more than
a couple of resources.

Imagine you have an excellent 100Gb connection, but are trying to load a webpage hosted across the Atlantic ocean.
The roundtrip time to that server (your ping), will probably be around 60ms. If you need to download 100 small resources
through just two connections, it will take at least `ping * (resources / connections)`, so 3 seconds, which isn't great.

That's what made many frontend optimization techniques like assets bundling absolutely essentials back then, they
made a major difference in load time[^1].
Similarly, some websites were using a technique called domain sharding, splitting assets into multiple domains to allow
more concurrency.

In theory, even these two connections could have been used much more effectively by pipelining requests,
the RFC 2616 has an entire section about it, and that was one of the big features added in HTTP/1.1 compared to 1.0.
The idea is simple, after sending your request, you don't have to wait for the response before sending more requests.
You can send 10 requests immediately before having received a single response, and the server will send them one by one
in order.

But in practice most browsers ended up disabling that feature by default because they ran into misbehaving servers, dooming
the feature.
It also wasn't perfect, as you could experience *head-of-line blocking*.
Since responses don't have an identifier to map them to the request they're the answer to, they have to be sent in order.
If one resource is slow to generate, all the subsequent resources can't be sent yet.

That's why as early as 2008, browsers stopped respecting the two concurrent connection rule.
Firefox 3 started raising the connection limit to 6 per domain, and most other browsers followed suit shortly after.

However, more concurrent connections isn't an ideal solution, because TCP connections have a *slow start*.
When you connect to a remote address, your computer doesn't know if the link to that other machine can support 10 gbit/s
or only 56 kbit/s.
Hence, to avoid flooding the network with tons of packets that will be dropped on the floor, it starts relatively slow
and periodically increase the throughput until it receives packet loss notifications, at that point it know it has more
or less reached the maximum throuhput the link can sustain.

That's why persistent connections are a big deal, a freshly established connection has a much lower throughput than one
that has seen some use.

So by multiplying the number of connections, you can download more resources faster, but it would be preferable if they
were all downloaded from the same connection to not suffer as much from TCP slow start.

And that's exactly the main thing HTTP/2 solved, by allowing multiplexing of requests inside a single TCP connection,
solving the head-of-line blocking issue[^2].

It also did a few other things, such as mandating the use of encryption[^3] and also compressing request and response headers
with GZip, and "server push", but multiplexing is really the big one.

## Why It Doesn't Matter Over LAN

So the main motivation for HTTP/2 is multiplexing, and over the Internet, especially mobile Internet with somewhat more
spotty connections, it can have a massive impact.

But in the data center, not so much. If you think about it, the very big factor in the computation we did above was the
roundtrip time (ping) with the client.
Unless your infrastructure is terribly designed, that roundtrip time between your server (say Puma) and its client
(your load balancer or reverse proxy) should be extremely small, way under one millisecond, and totally dwarfed by the
actual request render time.

When you are serving mostly static assets over the Internet, latency may be high and HTTP/2 multiplexing is a huge deal.
But when you are serving application-generated responses over LAN (or even a UNIX socket), it won't make a measurable
difference.

In addition to the low roundtrip time, the connections between your load balancer and application server likely have
a very long lifetime, hence don't suffer from TCP slow start as much, and that's assuming your operating system hasn't
been tuned to disable slow start entirely, which is very common on servers.

## Server Push Fail

Another reason people may have wanted HTTP/2 all the way to the Ruby application server at one point was the "server push"
capability.

The idea was relatively simple, servers were allowed to send HTTP resources to the client without being prompted for it.
This way, when you request the landing page of a website, the server can send you all the associated resources up front
so your browser doesn't have to parse the HTML to realize it needs them and start to ask for it.

However, that capability was actually removed from the spec and nowadays all browsers have removed it because was
actually doing more harm than good. It turns out that if the browser already had these resources in its cache, then
pushing them again would slow down the page load time.

People tried to find smart heuristics to know which resources may be in the cache or not, but in the end, none worked
and the feature was abandoned.

Today it has been superseded by [103 Early Hints](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/103), which
is a much simpler and elegant spec, and is retro-compatible with HTTP/1.1.

So there isn't any semantic difference left between HTTP/1.1 and HTTP/2.
From a [`Rack`](https://github.com/rack/rack/blob/main/SPEC.rdoc) application point of view, whether the request was
issued through an HTTP/2 or HTTP/1.1 connection makes no difference.
You can tunnel one into the other just fine.

## Extra Complexity

In addition to not providing much if any benefit over LAN, HTTP/2 adds some extra complexity.

First, the complexity of implementation, as HTTP/2 while not being crazy complicated at all, is still a largely binary
protocol, so it's much harder to debug.

~~But also the complexity of deployment. HTTP/2 is fully encrypted, so you need all your application servers to have a key and
certificate, that's not insurmountable, but is an extra hassle compared to just using HTTP/1.1, unless of course for some
reasons you are required to use only encrypted connections even over LAN.~~ Edit: The HTTP/2 spec doesn't actually require
encryption, only browsers and some libraries, so you can do unencrypted HTTP/2 inside your datacenter.

So unless you are deploying to a single machine, hence don't have a load balancer, bringing HTTP/2 all the way to
the Ruby app server is significantly complexifying your infrastructure for little benefit.

And even if you are on a single machine, it's probably to leave that concern to a reverse proxy, which will also take
care of serving static assets, normalize inbound requests, and also probably fend off at least some malicious actors.

There are numerous battle-tested reverse proxies such as Nginx, Caddy, etc, and they're pretty simple to setup,
might as well use these common middlewares rather than to try to do everything in a single Ruby application.

But if you think a reverse proxy is too much complexity and you'd rather do without, there are now zero config solutions
such as [thruster](https://github.com/basecamp/thruster), I haven't tried it so I can't vouch for it, but at least on
paper it solves that need.

## Conclusion

I think HTTP/2 is better thought of not as an upgrade over HTTP/1.1, but as an alternative protocol to more efficiently
transport the same HTTP resources over the Internet. In a way, it's similar to how HTTPS doesn't change the semantics
of the HTTP protocol, it only changes how it's serialized over the wire.

So I believe handling HTTP/2 is better left to your infrastructure entry point, typically the load balancer or reverse proxy, for the same
reason that TLS has been left to the load balancer or reverse proxy for ages. They have to decrypt and decompress
the request to know what to do with it, why re-encrypt and re-compress it to forward it to the app server?

Hence, in my opinion, HTTP/2 support in Ruby HTTP servers isn't a critically important feature, would be nice to have it for a few
niche use cases, but overall, the lack of it isn't hindering much of anything.

Note that I haven't mentioned HTTP/3, but while the protocol is very different, its goals are largely the same as HTTP2, so I'd apply the same conclusion to it.

[^1]: Minifying and bundling still improve load time with HTTP/2, fewer requests and fewer bytes transferred are still positive, so they're still useful, but it's no longer critical to achieve a decent experience.
[^2]: At the HTTP layer at least, HTTP/2 still suffers from some forms of head-of-line blocking in lower layers, but it is beyond the scope of this post.
[^3]: The RFC doesn't actually requires encryption, but all browser implementations do.
