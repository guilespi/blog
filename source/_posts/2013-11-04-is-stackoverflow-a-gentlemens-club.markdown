---
layout: post
title: "Is StackOverflow a gentlemen's club?"
date: 2013-11-04 14:53
comments: true
categories: 
- conferences
- stats
- stackoverflow
- women in tech
---
For some time now, tech industry has taken an active role in trying to solve the gender imbalance problem.

* Organizing [conferences][4] for women or by women
* Trying to bring more women into math or [cs][5]
* Offering [grants and scholarships][2] for [Hacker School][3]
* [Public scorn][6] [for offenders][11] when deserved
* [Anti-Harassment][7] policies for conferences

There's even [gender studies in mathematics and other sciences][1].

But even if the issue has been on the table for a while, I've attended a few conferences where I live and the usual attendance looks like this:

{% img center /images/blog/meetup-gender.jpg 680 780 'What a conference looks like' %} 

I don't think you'll find more than 5 women in the picture.

And I can tell you **for sure**, that picture does not represent _at all_ the _women in tech_ in the city. 
There may be an imbalance, but women are not by any means, 0,5% of computer science graduates here.

So women are not participating, but why? 

The Data
=========

Since StackExchange has [open data][9] you can [derive some insights from][8], I decided to take a look at the problem from a different perspective, 
and address the question of the underrepresentation using StackOverflow data.

I started with a few questions in mind:

* What % of users are women?
* What's the question/answer rate for men and women?
* How the reputation score for men and women compares?
* How it compares to ambiguous/unisex names?
* Is in fact SO a gentleman's club?

Since SO has no gender data, gender needs to be inferred from  _user names_, which is obviously not 100% accurate, 
many names are unisex or depend on the country of origin. I decided to use [this database and program][10] which 
has names from all over the world curated by local people. In many cases you will get a statistical result: _mostly male_ 
or _mostly female_, and that makes sense.

_Be warned_ this is not a scientific study nor tries to be, just trying to see some patterns here.

First Glimpse
==============

First I wanted to get a glimpse of the general trends, so I did a random draw of 50k users, more than enough for what I need.

``` sql
Select * From users 
Order By newid()
```
StackExchange limits the number of rows returned by the online database browser to 50k, so that's it.

{% img center /images/blog/random-share.png 680 780 'Random share' %} 

```
  > table(users$gender)/nrow(users)
  anonymous    error in name        is female          is male     is mostly female   is mostly male   is unisex name   name not found 
   0.25662       0.00006              0.02910          0.23640          0.00946          0.04654          0.02942          0.39240 
```

As you see there's a 27% of confirmed males and only a 4% of confirmed females.  `Anonymous` users are the usual numerical users like `user395607`, 
and `name not found` refers to things like `ppkt`, `HeTsH`,  `holden321` and `ITGronk`, you get the idea.

Then I wanted to see how reputation was distributed among those users, and how that compared against how long the user was using the site.

{% img center /images/blog/random-reputation.png 680 780 'Random reputation' %} 

There you go, an image is worth a thousand words, reputation difference among genders is _huge_, it doesn't seem to be related to how long you've been around either.

Fresh users
===========

To confirm that, I drew randomly 50k fresh users, who joined the site after `2012-10-10`, just to see if trends were any different considering only last year data.

``` sql
Select * From users 
Where CreationDate > '2012-10-10'
Order By newid()
```

{% img center /images/blog/fresh-share.png 680 780 'Fresh share' %} 

```
    > table(users$gender)/nrow(users)
       anonymous    error in name        is female          is male     is mostly female   is mostly male   is unisex name   name not found 
         0.35620      0.00002             0.03178          0.20320          0.01014            0.04076          0.02544         0.33246 
```

Here women seem to be a little bit closer, but still a great difference.

{% img center /images/blog/fresh-reputation.png 680 780 'Fresh reputation' %} 

The best of the best
====================

Then I drew the 50k users with the highest reputation score.

``` sql
Select * From users 
Order By Reputation Desc
```

Now we're seeing some changes:

{% img center /images/blog/top-share.png 680 780 'Top share' %} 
```
    > table(users$gender)/nrow(users)
       anonymous    error in name        is female          is male is mostly female   is mostly male   is unisex name   name not found 
         0.00794          0.00002          0.01064          0.34130          0.00890          0.06656          0.03524          0.52940 
```

As you expect here there's almost no `anonymous` users, in the _online community_ charity has a name attached to it, ain't it?

And the reputation trend is still there, something you can readily confirm if you scroll the first pages of the [all time user's reputation page][12].

{% img center /images/blog/top-reputation.png 680 780 'Top reputation' %} 

But then I charted the reputation _distribution_ against gender and something interesting arises:

{% img center /images/blog/top-distribution-all.png 680 780 'Top distribution all' %} 

As you see, there's a great deal of outliers there, with 75% of the top 50k users below 4200 points.

```
> quantile(users$reputation)
       0%       25%       50%       75%      100% 
  1024.00   1415.00   2178.00   4214.25 618862.00 
```

So what happens when we look at the distribution considering the 75% of the users, that are in fact below 4215 points?

{% img center /images/blog/first-three-quantiles.png 680 780 'First three quantiles' %} 

Well **that's something!** Now distribution looks pretty much alike.

Seems to me those outliers, that are mostly men, are well beyond everyone else, men or women. 

How do _you_ read that data?

Wrap up
=======

At 4% females, SO seems to be suffering the same phenomena occurring in the _real world_, that is, women being underrepresented, 
but being SO a strongly moderated virtual community the problem can't be related to harassment. 
So, there's something else going on, is it that SO is designed for male competitiveness, 
being the site designed exclusively by males -there were no women on the team AFAIK-?

Isn't it the reason you want diversity on your teams to start with? To provide a different perspective on the world, that enables you to reach _everyone_.

In my opinion, that's why women should be part of the _creation_ process, don't you think?

None the less, a large group of men are acting as if they already know what women need, 
and as patriarchy mandates, providing it, creating programs, making conferences
and burning witches, but not a single soul has asked women what they think, want, or need.

For a change, I've put up [an anonymous survey online][14] with the purpose of understanding how women who are already in tech, 
feel, if you have a female coworker, friend or colleague please share it,
we may even find some interesting insights if we start listening.

_I'm [guilespi][13] at Twitter_

Survey
======

Share the link above or take the survey here:

<iframe src="https://docs.google.com/forms/d/1g3FfFKz70HTuFWug609Zkc_5lZzldPkG_ubidjl2oTA/viewform?embedded=true" width="760" height="500" frameborder="0" marginheight="0" marginwidth="0">Loading...</iframe>

[1]: http://pages.uoregon.edu/wmnmath/Statistics/index.html
[2]: http://women2.com/etsy-hacker-school-scholarships-support-women-in-technology/
[3]: https://www.hackerschool.com/faq
[4]: http://railsgirls.com/
[5]: http://railsbridge.org/
[6]: http://valleywag.gawker.com/business-insider-ctos-is-your-new-tech-bro-nightmare-1280336916
[7]: http://www.wired.com/underwire/2013/07/convention-harassment-comic-con/
[8]: http://blog.guillermowinkler.com/blog/2012/10/30/how-long-waiting-for-an-answer-in-stackoverflow/
[9]: http://data.stackexchange.com/
[10]: http://www.heise.de/ct/ftp/07/17/182/
[11]: https://twitter.com/adriarichards/status/313417655879102464/photo/1
[12]: http://stackoverflow.com/users?tab=Reputation&filter=all
[13]: http://www.twitter.com/guilespi
[14]: https://docs.google.com/forms/d/1g3FfFKz70HTuFWug609Zkc_5lZzldPkG_ubidjl2oTA/viewform
