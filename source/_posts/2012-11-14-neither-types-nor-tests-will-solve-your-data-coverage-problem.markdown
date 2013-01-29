---
layout: post
title: "Neither types nor tests will solve your data coverage problem"
date: 2012-11-14 22:24
comments: true
categories: 
- programming
- testing
- types
- computer science
---
I just watched the Strangeloop talk titled [Types vs. Tests: An Epic Battle][1] from Amanda Laucher and Paul Snively. 
As Amanda says it's a discussion many of us have had in the past, I used to talk about it with [fedesilva][2], 
hardcore Scala advocate, you know what I mean?

For me, I think types and tests are two sides of the same coin, because neither strategy
for proving correctness is computable.

Bear with me.

The purpose of having types or having tests, it's to prove your program to be correct before [your bugs reach your customers][0],
one strategy tries to prove it _at compile time_ and the other _after compile time_. But what it really means your program to be _correct_?

Let's go for ride on [Computability Theory][3].

**Functions and programs**

According to the definition a function `f` is computable if a program `P` exists which can calculate the function given unlimited amounts of time and storage.

    f:N→N is computable ↔ ∃ a program P which computes the function.

Also we must define when a program `P` converges for input `n`.

    Program P with input n converges if ∃ m ∈ N / <Q,n> = m , it's written <Q,n>↓

In computer theory there are well known functions to be non computable, two of them are:

    Θ(n) = 1  if <Ix(n), n>↓
           0  if <Ix(n), n>↑

Which says that the function `Θ` is equal to 1 if the program of index `n` converges on input `n` and 0 if the program of index `n` diverges on input `n`.

There's another very famous function which has been proved to be non computable.

    stop(p, n) = 1 if <Ix(p), n>↓
                 0 if <Ix(p), n>↑

Which pretty much says that given a description of an arbitrary computer program, decide whether the program finishes running or continues to run forever.
As you may have guessed, it's the well known [Halting Problem][4], and it's _not_ computable.

**How the Halting Problem relates to types and tests**

Let's assume our program `P` we're trying to prove correct, computes the function `f:N→N`. We can define our program that proves correctness, a program `T` 
that computes the function

    ft:N→{0,1} 

Which is to say, for every input of the domain, our program `T` decides if the program converges or not. It's starting to sound familiar ain't it?

Let's assume such a program `T` exists to prove correctness, and we have a macro `MT` to find such a program given `P`. We could write the following program `Q`

```
PROGRAM(x0, x1)
x3=MT(x0)
x4=EVAL_PROG(x3, x1)
RESULT(x4)
```

What `Q` does it's having received a program `x0` and an input `x1`, first finds the `T` decider program for the program `x0`, and then evaluates the program
with the input `x1`.

So what do we have here?

    Q(p, n) = 1 ↔ ft(n) = 1 ↔ <P, n> ↓

So we have written a program which computes the `stop` function, which is absurd. It means we cannot have a program that decides on the computability of
a program.

**Show me the code**

In practice, it means that if you have this program

``` c
int f(int x) {
   while (x > 0) {
     x--;
   }
   return 0;
}
```
This program doesn't stop for `x < 0`, and according to theory, there's no program you can write to find out about it.

There are also a few other funny cases regarding the domain of your functions, such as

``` c
int f(int x) {
   return 1 / (x - 3);
}
```
This function fails miserably if `x = 3`. Just think about it when your functions have a more complex domain.

**How to improve your tests for correctness**

Most people I see are worried about having 100% _code coverage_, but it's not that usual to see people worried about _data coverage_.

As seen in the previous example if you forget to test for `x = 3` you may have 100% code coverage but your program will blow up anyway.

Regarding types, I know [Dependent Types][5] exists, but it's the other side of the same coin, you have to provide a constructive
proof that the type is _inhabited_. So if you don't define your type considering the special cases of your function domain, no one is coming
up to save your ass.

But when thinking about correctness you should be thinking about your function domain.

**Conclusion**

Both tests and types are useful ways to validate your program is correct, but not perfect. Even the discussion is meaningless, because it's just a matter
of taste whether you like to specify your correctness rules in types or tests, but it's something you will keep doing as far as I can tell.

As Rich Hickey [said][6], both tests and types are like guard rails, and you must know the cliff is there in order to decide building them.

**Update:** 
Many people wrote to me as if I'm saying you can't prove a program to be correct, that was not what I've tried to say.
It was that you can't have a system that can prove programs to be correct without specifying the rules yourself.
That is, it was a case against `Q` not against `T`.

_And hey! you follow me on [twitter][7]!_

[1]: http://www.infoq.com/presentations/Types-Tests
[2]: http://www.twitter.com/fedesilva
[3]: http://en.wikipedia.org/wiki/Computability_theory
[4]: http://en.wikipedia.org/wiki/Halting_problem
[5]: http://en.wikipedia.org/wiki/Dependent_type
[6]: https://twitter.com/richhickey/status/116490495500357633
[0]: http://blog.guillermowinkler.com/blog/2012/11/07/whats-a-bug-worth/
[7]: http://www.twitter.com/guilespi
