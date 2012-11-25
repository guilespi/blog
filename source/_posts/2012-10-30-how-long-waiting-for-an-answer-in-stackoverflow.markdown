---
layout: post
title: "How long waiting for an answer in StackOverflow"
date: 2012-10-30 23:52
comments: true
categories: 
- programming languages
- stackoverflow
- statistics

---
I'm not a [StackOverflow][0] active contributor, something I recently decided should start to change.

I think it's amazing the `speed` an answer is given for any asked question, like *freaking* fast. If you are using Google Reader to peek new questions filtered by tag, when you see a question, almost for sure it's already answered.

Fortunately all StackExchange data is [open][1], so we can see exactly _how_ fast is that. I used the online [data browser][2], more than enough for the task.
 
I decided to consider only the questions having an accepted answer, since questions with many bogus answers should not be treated as having an answer at all.

**tl;dr**

The average answer time seems to be dependent on a mix of the maturity of the language and how many people is using it. 

{% img center /images/blog/averageanswerbylanguage5.png 550 650 'How long waiting for an answer by language, easy questions' %} 

Hey, Haskell has pretty good answer times, at least considering its [33th][5] position in the [TIOBE Index][7].

**Not all questions are the same**

Of course not all questions are the same, this is from the first query I ran.

{% img center /images/blog/averageanswerbylanguageunfiltered.png 550 650 'How long waiting for an answer by language, all questions' %} 

This is an unfiltered query using all the questions from year 2012, you see the average answer time is much higher than the previous chart, around 1000 minutes, looking at the data:

    Language	Ans. Time         Stdev
    c               934       7630.98957971267
    c++            1036	      7258.13498426685
    clojure        1078	      7485.94721484444
    haskell        1199       9059.91937846459
    php            1210	      8588.58929278208
    lua            1386	      6569.08356022594
    c#             1452	      8875.00837073432
    scala          1472	     10707.9191188056
    javascript     1490	      9756.64151519177
    java           1755	     10541.6111024572
    ruby           2124      11850.4353701107

The standard deviation is huge, we have a lot of questions that took ages to get answered, making the average answer time meaningless. 

So I decided to take out questions with an answer time greater than 24 hours, as 92% of the questions have an approved answer in less than 5 hours.
([here][8] you can see the query used to get this table)

    DifficultyGroup	   Total	 Average	          StandardDev
    Easy                47099      	 27	          44.7263057563449 
    Medium                344       691          339.312936469053
    Hard                 1926      3769	        2004.75027395979
    Hell                 1623     66865	       96822.8840748525
    

It started to look like something:

{% img center /images/blog/averageanswerbylanguage24.png 550 650 'How long waiting for an answer by language, less than 24 hours' %} 
[This][4] is the query.

You see there, PHP running at front with 68 minutes average accepted answer time, either it's too easy or there're too many of them.

If you wanna see how the distribution goes when considering accepted answers in less than 5 hours, is the first picture of the page, the trend is also there.

**What about the time?**

Something unexpected, average answer time is almost unaffected by the time of the day the question was asked. 
The only thing I see here is that Ruby programmers are being killed by the lunch break and c++ programmers slowly fade out with the day, ain't it?

{% img center /images/blog/averageanswerhourbyhour.png 550 650 'How long waiting for an answer by time of day, ruby and c++' %} 

[This][9] is the query.

There goes my idea of catching unanswered questions at night. It would be interesting to see how many cross-timezone answering is happening.

**Conclusion**

It should work better running a regression against the complete dataset using more features than only programming language and time of day
 to automatically guess which questions have more chance of have a long life unanswered. Maybe next time.

_Follow me on [Twitter][3]_

[0]: http://www.stackoverflow.com
[1]: http://media10.simplex.tv/content/xtendx/stu/stackoverflow/
[2]: http://data.stackexchange.com/faq
[3]: http://www.twitter.com/guilespi
[4]: https://gist.github.com/3984333
[5]: http://www.tiobe.com/index.php/content/paperinfo/tpci/index.html
[7]: http://en.wikipedia.org/wiki/TIOBE_index
[8]: https://gist.github.com/3984320
[9]: https://gist.github.com/3984329
