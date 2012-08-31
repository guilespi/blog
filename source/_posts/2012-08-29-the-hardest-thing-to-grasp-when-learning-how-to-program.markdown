---
layout: post
title: "The hardest thing to grasp when learning how to program"
date: 2012-08-29 19:54
comments: true
categories: 
---
Today I saw an old Quora post asking [what's the hardest concept to understand](http://www.quora.com/What-are-the-most-common-hard-concepts-to-understand-when-learning-how-to-program) when learning how to program.

tl;dr
----

{% blockquote @guilespi, Reflexiones de resaca %}
Programming is hard, because reality is hard
{% endblockquote %}

My answer:

---

Some concepts are better explained by another student who has just grasped a concept than by a teacher who finds almost obvious all of the stuff.

So my list of difficult concepts to learn, is made of the many things most people do wrong, even with many years of experience.

* Edge cases
Almost everybody can get the happy-path-version of an algorithm working with a little bit of work.
Not everybody can get the permutation of possibilities and edge cases right.

* Organizing code
Almost everybody can get an unmaintainable sheet of poorly designed code, spit some coherent output.
Not everybody can even understand when they're looking at a huge mess, why is it that it's a mess

* Avoid repetition
This complexity arises from the need to have a _sometimes_ huge system on your head at the same time.
Designing small libraries and doing a bottom-up design helps, but there will come the time when you need to have this huge monstrosity in your head at once, and you'll fail.

* Concurrency
While there are many strategies to deal with many things happening at once with your reality, having a lot of moving parts is hard. As it's hard having a system with thousands of small pieces talking to each other.

Many of this situations are just particular cases derived from the Brooks paper [There's no silver bullet](http://en.wikipedia.org/wiki/No_Silver_Bullet), you cannot evade reality essential complexity by doing tricks aimed at the accidental complexity of programming.

Programming is hard, because reality is hard.
