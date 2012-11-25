---
layout: post
title: "What's a bug worth, a case for continuous integration"
date: 2012-11-07 19:48
comments: true
categories: 
- software enginnering
- continuous integration
- business
---
It's almost amazing that being the year 2012, on the break of Mayan Apocalypse, and there's still some people pushing code out the door without stopping for a minute to think how much a bug costs.

I'll save you the thinking, it's costing you customers.

See the following chart I've crafted for you(emphasis on _crafted_), please hit the `play` button.

<script type="text/javascript" src="https://ajax.googleapis.com/ajax/static/modules/gviz/1.0/chart.js"> {"dataSourceUrl":"https://docs.google.com/spreadsheet/tq?key=0AgXlknkCPjl9dFpMT0pjRlNFZ1p4TkFDRWljY1AwYXc&transpose=0&headers=1&range=A1%3AF10&gid=0&pub=1","options":{"titleTextStyle":{"fontSize":16},"vAxes":[{"useFormatFromData":true,"title":"Left vertical axis title","minValue":null,"viewWindowMode":"pretty","viewWindow":{"min":null,"max":null},"maxValue":null},{"useFormatFromData":true,"minValue":null,"viewWindowMode":"pretty","viewWindow":{"min":null,"max":null},"maxValue":null}],"booleanRole":"certainty","title":"Chart title","height":321,"animation":{"duration":500},"width":667,"hAxis":{"useFormatFromData":true,"title":"Horizontal axis title","minValue":null,"viewWindowMode":"pretty","viewWindow":{"min":null,"max":null},"maxValue":null}},"state":{},"view":{},"chartType":"MotionChart","chartName":"Chart 1"} </script>

<br>
<br>

There's an obvious relationship between the cost of fixing a bug and how much customers your company can effectively take.

It has an easy explanation, if you have only one customer and your solution has a bug, what do you do?
You call her, you explain the bug, you go to her office, you hack a fix, you drink some coffee, and you move on. 
Maybe if you have one big fat customer based on a personal relationship, you can live with that.

I hope it's clear for you this delivery process does not scale, is it?

When you have hundreds or thousands of customers you can't clone yourself and explain to everyone why your product is failing, 
you won't drink a hundred coffees to build rapport and talk your way out of the mess.

I think there's still two big misconceptions about this relationship between your bugs and your customers, 
and it may affect how you decide on your development and delivery process.

**Bug fixing cost and quality are not the same thing**

It's widely known, I hope, that the earlier you find a bug, the cheaper it is to fix it. [This guy][1] even fixes his bugs in the hammock, before writing any code. 
Take a look at the chart of the relative cost of fixing defects, [this is the source][8]

{% img center /images/blog/costofbugfixing.jpg 480 580 'Whats a bug worth' %} 

Obviously you should be investing in good Engineering, peer reviewing even your documents and designs, and testing your components early an often. 
(Quality is not about testing either, but that's material for another rant). What's not so clear, is given that some bugs _will always_ reach your customers, 
how do you reduce the cost of fixing your on-the-wild bugs?

You should do everything in your reach to produce quality products, because it's cheaper in the long run. 
But what will make or break your ability to grow your customer base, is how fast and cheap you move when a bug is found. 
Your maintenance cost, if you want.

**Bug fixing cost is like performance in the browser**

You should watch [this talk][2] from the last Strangeloop, besides being great, Lars Bak makes a great point about performance in the browser, 
when a new level of performance was reached on the Javascript VM, all new kinds of applications started to pop up taking advantage of that performance.

It's _not_ the other way around.

[Correlation does not imply causation][3], until it does, just be sure to understand what causes what.

Speed in the browser did not improve because Gmail was running too slow, first speed improved, then we have Gmail.

It's the same with your customers.

If you wait till having lots of customers to start thinking about improving your maintenance costs, you will never have them. 
Having low support and maintenance costs will make you find a way to acquire more customers, just because you can.

**What to do?**

This is not by any means a complete nor bulletproof list, but some strategies I've found from personal experience that help.

**You do [continuous integration and deployment][7]**

Have you ever been involved in a delivery process having to test thousands of test cases, run dozens of performance and stress tests, 
do it in multiple platforms, all of it, just because you patched 3 lines of code, and you must be absolutely sure everything is still working as intended?

I have, and it's not fun

It's not fun for your customer either, because you end up batching even your hot-fixes, and they're not so hot anymore. 
And your customer has to wait, and you will eventually lose your customer.

Continuous integration is not about some geeks with shiny tools, it's about customers.

**You develop with operations in mind**

There's a [great talk by Theo Schlossnagle][4] about what it means to have a career in web operations, walking the path, and becoming a craftsman, you must watch it, seriously, because it's _that_ good. 

One of the remarkable points is that you must build systems that are observable. Developers cannot separate themselves from the fact that software has to operate, 
actually run. And developers shouldn't be trying to reproduce a bug in a controlled environment in order to understand if there's really a bug. 
You should be able to diagnose the problem in the running system, so it must be observable. How much elements in that queue? is it stalled? you must know, now.

And you don't build observable systems if you start thinking about it after you've shipped, using an entirely different team(hello DevOps).

Software with operations in mind is like software with security in mind, or quality in mind, it's a state of being, and it's about your development process.

**You use the right tools**

How long does it take you to _see_ that a function is returning the wrong value? 
How long does it take you to find the 3 lines of log that point you to the exact spot the problem is?
How long does it take you to analyze a crash dump and get to the cause of the crash?

Being able to debug and diagnose a problem fast, is almost as important as being able to fix it fast, and deploying the fix fast.

This is an area where I personally think there's a lot of room for improvement regarding the tools we daily use, 
but you should know [DTrace][9] exists and how to use it, idealistically.

**Conclusion**

If you're hacking your brains out and life's good, all the power to you. I like that too.

But if you're really thinking about scaling your business, you should be taking a look at your bug fixing and maintenance costs, now.

There's also a [great book][5] about scaling companies, you should read that one too.

_And don't forget to follow me on [twitter][6]_

[1]: http://blip.tv/clojure/hammock-driven-development-4475586
[2]: http://www.infoq.com/presentations/Performance-V8-Dart
[3]: http://en.wikipedia.org/wiki/Correlation_does_not_imply_causation
[4]: http://www.youtube.com/watch?v=LAP1zaXUvAE
[5]: http://www.amazon.com/Art-Scalability-Architecture-Organizations-Enterprise/dp/0137030428
[6]: http://www.twitter.com/guilespi
[7]: https://speakerdeck.com/sebastianmoreno/continuous-improvement-o-como-poner-los-robots-de-tu-lado
[8]: http://www.riceconsulting.com/public_pdf/STBC-WM.pdf
[9]: http://dtrace.org/blogs/

