---
layout: post
title: "Group-by considered harmful"
date: 2012-10-19 23:01
comments: true
categories: 
- NoSQL
- Cassandra
- Big Data
---

Main reason big data is so trendy is because it's closely related with web scale, and that of course means money. It's the kind of problem you would _love_ to have. Everybody thinks big data is about being [Facebook][1], or [Twitter][19], or [Tumblr][2]

But big data is not only about [moving your humongous petabytes][3] of user pictures from one place to another, big data is also a state of mind, and it can help you solve problems better and faster even if you're not on the petabyte scale.

And if you have ever seen a report taking ages to render, this post is for you. Because we're in the Google age, so if you tell me it's acceptable for you to take fucking _seconds_ to spit that information out, you are delusional. In 2008, [Google index size][4] was over a trillion pages, and the typical search returns [in less than 0.2 seconds][5]. 

And one of the main reasons Google is so fast and engineers still deliver applications with such a poor performance, is because we're still modeling data as if disk space is an issue.

Let me explain.

Let's think about a telephony system that needs to keep track of every call entering a company or made by employees. A typical simplified cdr -call detail report- table will look like this:

```
+------------+----------+----------+-----------+---------------+-----------+----------+
| Call Start | Call End | Duration | Direction |    Number     | Extension |   User   |
+------------+----------+----------+-----------+---------------+-----------+----------+
| 20:05      | 20:15    |      600 | OUT       | +130512319292 |      1010 | John.Doe |
+------------+----------+----------+-----------+---------------+-----------+----------+
```

That's perfectly fine, and if someone wants to know who the hell did that 40 minute call to the hot-line, it's all in there.

But let's suppose you're a contact center organization with 2000 extensions, you could easily be dialing 150K calls an hour. That's billion of rows in a year's operation and the table being hit pretty hard. Then someone asks, how much money have we spent in calls to Morocco? Or better yet, I want an alarm to be triggered when a deviation from standard operation occurs to detect fraud.

So to avoid [locking and contention][13] over your call table, you may do something like this:

```
+------------+-----------------+----------+--------------+-------------+
|   Slice    | Number of calls | Duration |     User     | Destination |
+------------+-----------------+----------+--------------+-------------+
| 2012-10-10 |             121 |    49239 | John.Doe     | Germany     |
| 2012-10-10 |              90 |    28711 | John.Doe     | Australia   |
| 2012-09-02 |              78 |    12111 | Richard.Read | Uruguay     |
+------------+-----------------+----------+--------------+-------------+
```

So you separate your [realtime transactions][18] hitting the cdr, from the table used to query about statistical stuff. In datawarehousing there's a name for that, it's called [ETL][14]. The process of offloading your data to a system designed to be queried against. A real pain in the ass.

Of course you have an advantage, you can answer many questions using the same stored information.

+ How many calls has made Ricky this month?
+ How many minutes were spent this week calling Bangladesh, by all users?
+ What's the day of the month with the most number of calls

And we all know how, we use [GROUP BY][6], the most overlooked clause in the performance analysis.

``` sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITED;
SELECT User, SUM(Duration) as Duration
FROM CallSummary 
WHERE Slice = '2012-10-10'
GROUP BY User;
COMMIT;
```

Then you face a problem again, whether to be fast, or to be up-to-date. Because if you update your summary every night you cannot detect a fraud deviation in the last 30 minutes, but if you update your summary with every call you have again the contention issue.

But why is it that you need to have all the information in the same table? 

It's the relational way of storing information, which gives you flexibility when you don't know what you're going to do with your data, but can be the cause of performance degradation. If you're using group-by, the engine is iterating over each value in order to sum it up, there's no free way out of that. I will show you a way that costs disk space and a loss in query flexibility.

Before continuing with data modeling, let me do a little digression on data visualization.

There's a paper by [Stephen Few][7], who divides analytical audiences into three different types:

1) Exploratory Analytics

_Strong need for data exploration. The analyst approaches data with usually open ended questions "What may be hiding here that's interesting?". Tools such as [Panopticon Explorer][15] or [Spotfire][16] are in this category._

2) Custom Analytics 

_People analyzing information in routine ways, performing the same analytical task as part of their job. User only needs to understand the information presented and business processes._

3) Customizable Analytics

_Provides ready-made libraries for developers that need to create an application, such as [QlikView][17]._

And for some reason that escapes me, engineers almost always get stuck in a mix of options 1 and 2. We think our users to be creative geniuses with a need to derive hidden insights from our data, so we put this wonderful flexible tools in place for them.

Bullshit. People want to get their job done and go home with the kids, and it's _your_ job to know better.

Most of the time reporting or "analytics" as it's usually called, is not analytics, because the user is not doing analysis, he's just looking at data translated into information. And what data you're putting on the screen, is the result of proper program management and understanding of your user's needs. Or is it that it took you months to push that product out of the door and you don't really know which information is relevant to your users?

And if you do, why is it that you're modeling your data as if you don't? 

Hey, we'll just let them query the information using all this dimensions, cool, and you can have this pivot table here, you see it ...pivoting... give me fucking a break, the last time I started dragging and dropping dimensions in a pivot table I had a surmenage, who _are_ your users?

Digression ended, how do you escape the group-by trap?

Well, you just store it grouped according to your query needs. I'll show you first the idea, and then a specific Cassandra modeling of it.

Let's suppose that after talking to our users, we come to the conclusion they need the following questions answered:

+ How many minutes spent in outgoing calls to different destinations, by day, week and month.
+ How many calls made by users by day, week and month.

We can storage each counter separately, and only lock when updating a specific counter. Our counters will have this format:

```
OutgoingMinutes:<Destination>:<Period>:<Slice>
CallsMade:<User>:<Period>:<Slice>
```

Where `<Destination>` is any destination we've called, `<User>` is any user in the system, `<Period>` is one of "ByDay", "ByMonth" and "ByWeek", and `<Slice>` is the specific time the value is being counted for.

So an instance of our data may look like this:

```
OutgoingMinutes:Canada:ByDay:2012-10-09 = 432 ; 432 minutes in calls made to Canada on the day 2012-10-09
OutgoingMinutes:USA:ByMonth:2012-11 = 54021 ; 54021 minutes in calls made to USA on November
CallsMade:John.Doe:ByDay:2012-12-20 = 43 ; John made 43 calls today 
```

If you're looking at it and you think it looks like a map, it's pretty much one. With the main difference each key is isolated from each other, you don't need to lock on a page and four indexes in order to update a row. Each value gets updated separately. 

There's two things we need to be sure about:

+ Counter increment or decrement supporting concurrency
+ A way to query for date ranges, and it should be _fast_

So it happens, we can do this with [Cassandra counters][10], and we get the bonus of having our data replicated and distributed, so you can scale out [almost][8] [linearly][9], no matter how many users or destinations you have, or how long you decide to keep track of statistics.

The modeling is pretty straightforward, it looks like this:

``` json
Statistics = {
   OutgoingMinutes:Canada:ByDay = {
        2012-10-09 : 432
        2012-10-08 : 121
        2012-10-07 : 987
        2012-09-11 : 100
        ...
   },
   OutgoingMinutes:USA:ByMonth = {
        2012-11 : 54021
        2012-10 : 43222
        ...
   },
   CallsMade:John.Doe:ByDay = {
        2012-12-20 : 43,
        2012-12-19 : 34,
        ...
   }
}
```

I like modeling Cassandra schemas like JSON, it helps visualizing data layout better. For this particular case each element is:

`Statistics` is the Column Family, `OutgoingMinutes:Canada:ByDay` is the Row Key, `2012-10-09` is the Column name -you're going to be sorting by this value- and 
`432` is the value, this is a counter column type that supports distributed updating.

You can create the column family like this:

```
create column family Statistics with column_type=Standard and default_validation_class=CounterColumnType and key_validation_class=UTF8Type and comparator=UTF8Type;
```

When you've been factorizing things out for years, repeating the key in order to have different values it's nothing less than a deadly sin.

```
OutgoingMinutes:Canada:ByDay = 100
OutgoingMinutes:Canada:ByMonth = 1000
OutgoingMinutes:Canada:ByYear = 10000
```

It's like it's screaming for a normalization, so much bytes being wasted in key names! 

But hey, we're rendering views like this in our cluster with 100+ GB of data in milliseconds, _without_ doing any kind of ETL.

{% img left /images/blog/countervisualization.png 580 680 'i6 analytics visualization' %} 

We're trading disk space for computation _and_ I/O time, in order to improve query performance, but it's I/O time the usual culprit, having to iterate over a more flexible data structure in order to satisfy the rendering of a particular view . Sometimes you can even trade computation time for I/O time to improve performance, as in the case of [Cassandra][11] [data compression][12].

Wrapping up, this is not a case against relational databases, it's a case for modeling things carefully, you don't have to stick to an option because it seems to be flexible enough to accommodate all needs. You can always implement a message queue on top of MySQL, but it would be foolish not to use RabbitMQ, and you can always implement an inverted index on top of MySQL, but you should better use Lucene.

Data storage doesn't need to be _always_ relational.

[1]: http://techcrunch.com/2012/08/22/how-big-is-facebooks-data-2-5-billion-pieces-of-content-and-500-terabytes-ingested-every-day/
[2]: http://highscalability.com/blog/2012/2/13/tumblr-architecture-15-billion-page-views-a-month-and-harder.html
[3]: http://gigaom.com/cloud/facebook-hadoop-cluster/
[4]: http://googleblog.blogspot.com/2008/07/we-knew-web-was-big.html
[5]: http://googleblog.blogspot.com/2009/01/powering-google-search.html
[6]: http://msdn.microsoft.com/en-us/library/b6ddy7ah(v=vs.80).aspx
[7]: http://www.perceptualedge.com/articles/visual_business_intelligence/differences_in_analytical_tools.pdf
[8]: http://stackoverflow.com/questions/8839436/when-does-cassandra-hit-amdahls-law
[9]: http://techblog.netflix.com/2011/11/benchmarking-cassandra-scalability-on.html
[10]: http://wiki.apache.org/cassandra/Counters
[11]: http://www.datastax.com/dev/blog/whats-new-in-cassandra-1-0-compression
[12]: http://www.edwardcapriolo.com/roller/edwardcapriolo/entry/cassandra_compression_is_like_getting
[13]: http://www.toadworld.com/Knowledge/DatabaseKnowledge/GuyHarrisonsResolvingOracleContention/ResolvingOracleContention/Feb2008Article/tabid/304/Default.aspx
[14]: http://en.wikipedia.org/wiki/Extract,_transform,_load
[15]: http://www.panopticon.com/
[16]: http://spotfire.tibco.com/
[17]: http://www.qlikview.com/
[18]: http://datawarehouse4u.info/OLTP-vs-OLAP.html
[19]: http://www.slideshare.net/nkallen/q-con-3770885
