---
layout: post
title: "Real Life Clojure Application"
date: 2013-04-20 17:56
comments: true
categories: 
- Clojure
- Programming
---
When a language is going through its maturity process, there's always the need for sample code and applications to look at, design patterns, 
coding guidelines, best libraries, you name it, you usually get those looking at the source code of others. 

That is what really builds community, having a common language, besides the _language_.

Clojure's been going through this phenomenon for the last years, as you see [in this question][1] from 2008 and [this question][2] still being answered in 2012.

This year I made a real life full application in Clojure so I've spent some time deciding on many strategies, 
where to put which code, how to test, best libraries to use and what not.

So I decided to put [the source code][3] online, not only because I think it may help someone, 
but hopefully somebody will come up and help me improve what it's done in one way or another, it's certainly _far_ from perfect.

In case you wonder, it's an application for automatic call and sms dispatching.

{% img center /images/blog/notifyme.png 480 580 'Notify Me Blaster Application' %} 

Among other things, you'll find inside:

* Web development using [ring][4], [compojure][5] and [hiccup][13].
* Client side _almost_ entirely done in [clojurescript][14].
* Authentication and authorization using [friend][6].
* Database access using both [jdbc][7] and [korma][8].
* Async jobs using [quartz][9] for call and sms dispatching.
* Unit tests using [midje][10].
* Chart drawing using [incanter][12].
* Asterisk telephony integration using my own [clj-asterisk][15] bindings.
* Deploy configuration using [pallet][11] _(not yet finished)_.

_If you like it say hi! I'm [guilespi][16]_


[1]: http://stackoverflow.com/questions/329221/medium-size-clojure-sample-application
[2]: http://stackoverflow.com/questions/3628958/good-clojure-code-examples
[3]: https://github.com/guilespi/notify-me
[4]: https://github.com/ring-clojure/ring
[5]: https://github.com/weavejester/compojure
[13]: https://github.com/weavejester/hiccup
[14]: https://github.com/clojure/clojurescript
[6]: https://github.com/cemerick/friend
[7]: https://github.com/clojure/java.jdbc
[8]: https://github.com/korma/Korma
[9]: https://github.com/michaelklishin/quartzite
[10]: https://github.com/marick/Midje
[12]: https://github.com/liebke/incanter
[15]: https://github.com/guilespi/clj-asterisk
[11]: https://github.com/pallet
[16]: http://www.twitter.com/guilespi
