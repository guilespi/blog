---
layout: post
title: "Zipkin distributed tracing using Clojure"
date: 2013-11-28 13:05
comments: true
categories: 
- Clojure
- Zipkin
- Distributed systems
---
When you have a system with many moving parts it's usually difficult trying to understand which one of those pieces is the culprit, 
say for instance your home page is taking 3 seconds to render and you're losing customers, what the hell is going on?

Whether you're using Memcache, Redis, RabbitMQ or a custom distributed service, if you're trying to scale your shit up, you probably have many pieces or boxes involved.

At least that's what happens [at Twitter][1], so they've come up with [a solution called Zipkin][2] to trace distributed operations, 
that is, an operation that is potentially solved using many different nodes.

_Twitter Architecture_
{% img center /images/blog/twitter_arch.png 680 780 'Twitter Architecture' %}

Having dealt with distributed logging in the past, reconstructing a distributed operation from logs, 
it's like trying to build a giant jigsaw puzzle in the middle of a Tornado.

The standard strategy is to propagate some `operation id` and use it anywhere you want 
to track what happened, and that is the essence of what Zipkin does, but in a structured kind of way.

Zipkin
=====

Zipkin was modelled after [Google Dapper][3] paper on distributed tracing and basically gives you two things:

* Trace Collection
* Trace Querying

_Zipkin Architecture_
{% img center /images/blog/zipkin_arch.png 680 780 'Zipkin Architecture' %}

The architecture looks complex but it ain't that much, since you can avoid using `Scribe`, `Cassandra`, `Zookeeper` 
and pretty much everything related to scaling the tracing platform itself.

Since the trace collector _speaks_ the Scribe protocol you can trace directly to the collector, and you can also use 
local disk storage for tracing and avoid a distributed database like Cassandra, it's an easy way 
to get your feet wet without having to setup a cluster to peek a few traces.

Tracing
=====

There are a couple entities involved in Zipkin tracing which you should know before moving forward:

**Trace**

A trace is a particular operation which may occur in many different nodes and be composed on many different Spans.

**Span**

A span represents a sub-operation for the Trace, it can be a different service or a different stage in the operation process.
Also, spans have a hierarchy, so a span can be a child of another span.

**Annotation**

The annotation is how you tag your Spans to actually know what happened, there are two type of spans:

* Timestamp
* Binary

Timestamp spans are used for tracing time related stuff, and Binary annotations are used to tag your operation with a particular context, 
which is useful for filtering later.

For instance you can have a new _Trace_ for each home page request, which decomposes in the _Memcache Span_ 
the _Postgres Span_ and the _Computation Span_, each of those 
with their particular _Start Annotation_ and _Finish Annotation_.

API
=====

Zipkin is programmed in Scala and uses thrift, since it's assumed you're going to have distributed operations, 
the _official client_ is [Finagle][4], which is kind of a RPC system for the JVM, but at least for me, it's quite ugly.

Main reason is that it makes you feel that if you want to use Zipkin you must use a _Distributed Framework_, which is not at all necessary.
For a moment I almost felt like [Corba][12] and [DCOM][13] were coming back from the grave trying to lure me into the abyss.

There's also libraries for [Ruby][5] and [Python][6] but none of them felt quite right to me, 
for Ruby you either use Finagle or you use Thrift, but there's no actual Zipkin library, 
for Python you have [Tryfer][7] which is good and [Restkin][6] which is a REST API on top of it.

Clojure
=====

In the process of understanding what Zipkin can do for you (that means _me_) I [hacked a client][10] for 
Clojure using [clj-scribe][8] and [clj-thrift][9] which made the process almost painless.

It comes with a ring handler so you can trace your incoming requests out of the box.

```clojure
 (require '[clj-zipkin.middleware :as m])

   (defroutes routes
     (GET "/" [] "<h1>Hello World</h1>")
     (route/not-found "<h1>Page not found</h1>"))

   (def app
       (-> routes
       (m/request-tracer {:scribe {:host "localhost" :port 9410}
                          :service "WebServer"})))
```

_Zipkin Web Analyzer_
{% img center /images/blog/clj-zipkin-sample.png 580 680 'Zipkin Sample' %}

It's far from perfect, undocumented and incomplete, but at least it's free :)

Give it a try and let me know what you think.

_I'm [guilespi][11] at Twitter_

[1]: http://highscalability.com/blog/2013/7/8/the-architecture-twitter-uses-to-deal-with-150m-active-users.html
[2]: http://twitter.github.io/zipkin/
[3]: http://research.google.com/pubs/pub36356.html
[4]: http://twitter.github.io/finagle/
[5]: https://rubygems.org/gems/finagle-thrift
[6]: https://github.com/racker/restkin
[7]: https://github.com/racker/tryfer
[8]: https://github.com/livingsocial/clj-scribe/
[9]: https://github.com/xsc/thrift-clj
[10]: https://github.com/guilespi/clj-zipkin
[11]: http://www.twitter.com/guilespi
[12]: http://en.wikipedia.org/wiki/Common_Object_Request_Broker_Architecture
[13]: http://en.wikipedia.org/wiki/Distributed_Component_Object_Model
