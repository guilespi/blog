---
layout: post
title: "Property-based testing using QuickCheck"
date: 2015-04-12 17:05
comments: true
categories: 
- clojure
- testing
- quickcheck
---
Last year I attended [Clojure/West][16] in San Francisco, and was lucky enough to be at the talk by [John Hughes][0], called [Testing the hard stuff and staying sane][1].

I had been previously exposed to some of the concepts of generative testing, particularly Haskell's own QuickCheck, but never took the time to do something with it, this talk by John Hughes really stroke a chord on the usefulness of generative -or property based- testing, and how much effort you can save by knowing when and _how_ to use it.

I've been using Clojure `test.check` for a while, and since I'm preparing a [conference talk][17] on the subject, I decided to write something about it.

So bear with me, in this two-entry blog post I'll try to convince you why down the road, it may save your ass too.

## What generative testing is not

Probably the reason I've always looked down upon generative testing, was thinking it was just about random/junk data generation, for the too-lazy-to-think-your-own-test-cases kind of attitude.

Well, that's **not** what generative testing is about.

You _will_ have data generators for some input domain values, but trying to generate random noise to make the program fail is just [fuzzy testing][2], and generative testing is more than that.

How?

## On Types vs. Tests

[I've written before][3] about the difficulty of using types to prove your program is correct. Some people will always say you can do it with type systems(and types even more complex than the program under proof), and you can [always use Coq][4].

But for everyday programming languages and type systems it's not that easy, say for instance this `Java` function (assuming such thing exists).

```java
int f1(int x) {
    return 1/x;
}
```

You can say just by looking at the function, that any integer except zero will succeed.

In this other case:

```java
int f2(String x) {
    return x.length();
}
```

The function will succeed except when `x` is `null`.

So assuming that's expected behavior, you can write some tests to check on those special failure cases.

```java
@Test(expected=ArithmeticException.class)
public void testDivideByZero() {
    f1(null);
}

//this is just a unit test
@Test
public void testUnity() {
    assertEquals(1, f1(1));
}
    
@Test(expected=NullPointerException.class)
public void testNullPointerException() {
    f2(null);
}
```

But for the sake of making and argument, assume you're testing `openssl` and this is the function you have...

```c
int dtls1_process_heartbeat(SSL *s)
	{
    ...	
	/* Read type and payload length first */
	if (1 + 2 + 16 > s->s3->rrec.length)
		return 0; /* silently discard */
	hbtype = *p++;
	n2s(p, payload);
	if (1 + 2 + payload + 16 > s->s3->rrec.length)
		return 0; /* silently discard per RFC 6520 sec. 4 */
	pl = p;

	if (hbtype == TLS1_HB_REQUEST)
		{
		unsigned char *buffer, *bp;
		unsigned int write_length = 1 /* heartbeat type */ +
					    2 /* heartbeat length */ +
					    payload + padding;
		int r;

		if (write_length > SSL3_RT_MAX_PLAIN_LENGTH)
			return 0;

		/* Allocate memory for the response, size is 1 byte
		 * message type, plus 2 bytes payload length, plus
		 * payload, plus padding
		 */
		buffer = OPENSSL_malloc(write_length);
		bp = buffer;

		/* Enter response type, length and copy payload */
		*bp++ = TLS1_HB_RESPONSE;
		s2n(payload, bp);
		memcpy(bp, pl, payload);
		bp += payload;
		/* Random padding */
		RAND_pseudo_bytes(bp, padding);
    ...
```

Unless you've been living under a rock, you should have heard about [the heartbleed openssl bug][5], and it's just what you think, the bug was in the heartbeat processing function above, and [this is the patch with the fix][6].

{% img center /images/blog/opensslpatch.png 680 780 'Openssl heartbleed patch' %}

Who was the motherfucker that missed that unit test, huh?

When the function logic is more complex, it's exponentially more difficult to define both types and tests that make us feel more confident about pushing our nasty bits of code to a production environment.

And that's because the possible states our system or function can be, expand like hell when new variables and conditional branches are added (more on this later).

## Code Coverage vs. Domain Coverage

Looking at the function above you can see the problem is not on some untested code path, but on some _values_ used on function invocation.

Some people aim for 100% code coverage, according to [Wikipedia][8]

_In computer science, code coverage is a measure used to describe the degree to which the source code of a program is tested by a particular test suite. A program with high code coverage has been more thoroughly tested and has a lower chance of containing software bugs than a program with low code coverage._

Which is great, but since you can have 100% code coverage of the `1/x` function, but regarding domain coverage (for which values of `x` the function works as expected) you have nothing.

**Code coverage without domain coverage is just half the picture.**

Even unit tests prove _almost_ nothing.

## Tests do not prove correctness

There's a great quote by [Edsger Dijkstra][7] from Notes on Structured Programming that says

_Program testing can be used to show the presence of bugs, but never to show their absence!_

Which is to say, no matter how many unit tests you write, you're only proving that your program works (or fails) for the set of inputs you have selected when writing your tests.

It doesn't say a thing about the generalities or about a general property of the system or function under test.

## What is generative testing?

So what _is_ generative testing?

In generative testing you describe some properties your system or function must comply with, and the test runner provides randomized data to check if the property holds for that data, that's why it's also known as `property-based` testing.

A `property` is a high-level specification of behavior that should hold for a range of data points.

So a property works somewhat like a _domain iterator_, bringing a little bit closer types and tests.

Since you're defining how the system should behave for a particular _domain_ of values, not when the program is compiled, but when it's **run**.

## Why random data generation is important?

In the StrangeLoop 2014 conference, Joe Armstrong gave a talk called [The mess we're in][12], where he discussed system's complexity, go watch it since it's real fun.

He says that a `C` program with only six `32 bit` integers, has the same number of states that atoms exist on the planet, so testing your program by computing all combinations it's going to take a _really long_ time.

And if it's almost impossible to find the number of states computationally, imagine trying to find the number of possible failing states _manually_.

I've been in the position of having to hunt a bug that occurs only once a year in a system processing millions of transactions daily, and it's not fun at all. Pray to the logging gods the proper piece of information revealing the culprit is logged, so you don't have to wait another year for the bug to show up.

If your software runs inside a car, would you wait for the next deadly crash to analyze that dead-driver log file? Maybe that's why [Volvo uses QuickCheck][13] to test embedded systems.

Generative testing helps you put and test your system in so many different states it would be impossible to do manually.

## What's in a property

So, should we throw away all of our type systems and unit tests?

Not so fast, property based testing is not a **replacement** for types nor for unit tests.

Haskell and Scala both have their frameworks for property based testing (QuickCheck and ScalaTest) and are strongly typed languages.

Property based testing helps us define considerations for our programs where type systems do not reach, and where dynamically typed languages have a void.

So what does a property look like?

All concepts so far hold true for any language with a generative testing framework, many re-implementations exist from the [original QuickCheck version][9], from `C`, `C++`, `Ruby`, `Clojure`, `Javascript`, `Java`, `Scala`, etc. So now I will show you a couple of examples in different languages, just for you to grasp the basic property definition semantics, which is quite similar along the implementations.

These examples are not meant to show how powerful generative testing can be, yet.

### Sorting in Javascript

Let's say you want to test a `sort` function of yours, and instead of specifying individual test cases for particular arrays of integers, you define a property, which says that after sorting the array, the last element should always be greater than the first one.

This is what the property looks like in Javascript's [JSCheck][11]

```javascript
JSC.reps(10)
JSC.test(
    "First is lower than last after sort",
    function (verdict, v) {
        var sorted = v.sort();
        return verdict(sorted[0] < sorted[sorted.length - 1]);
    },
    [
        JSC.array([JSC.integer()]])
    ]
);
```
You don't say which particular arrays, just any array of integers must comply with the property, the framework will generate values for you (in this case 10 repetitions will be run). 

This is the result:

```
First is lower than last after sort: 10 cases tested, 0 pass, 10 fail
 FAIL [1] ([3])
 FAIL [2] ([5])
 FAIL [3] ([7])
```

Did you spot when the property doesn't hold?

### Sorting in Clojure

This is what the same property looks like in Clojure's [test.check][10].

```clojure
(def prop-sorted-first-less-than-last
  (prop/for-all [v (gen/not-empty (gen/vector gen/int))]
    (let [s (sort v)]
      (< (first s) (last s)))))

(tc/quick-check 200 prop-sorted-first-less-than-last)
```

With the following result:

```clojure
   => {:result false, :failing-size 0, :num-tests 1, :fail [[3]],
       :shrunk {:total-nodes-visited 5, :depth 2, :result false,
                :smallest [[0]]}}
```

As you see, **both fail**, since they doesn't hold for single element arrays.

The basic semantic for both languages is the same, you need:

* A property name (or claim in JSCheck)
* Some data generator for your input values
* A verdict or testing function who validates the property

{% blockquote http://book.realworldhaskell.org/read/testing-and-quality-assurance.html %}
This encourages a higher level approach to testing in the form of abstract invariant functions should satisfy universally.
{% endblockquote %}


## Shrinking

One of the best features of QuickCheck is the ability to shrink your failure cases to the minimum failing case (not all the implementations have it by the way).

When generating random data, you may end up with a failing case too big to rationalize (for instance a thousand elements vector), but it doesn't necessarily means that all the 1000 elements are needed for the function under test to fail.

When QuickCheck finds a failing case, it tries to shrink the input data to the _smallest_ failing case.

This is a powerful feature if you don't want to repeat many unnecessary steps in order to reproduce a problem.

A simple example to illustrate the feature comes from `test.check` samples.

Here a property must hold for all integer vectors, and it is that no vector should have the element `42` in it.

```clojure
(def prop-no-42
  (prop/for-all [v (gen/vector gen/int)]
    (not (some #{42} v))))
```

When the tests are run, `test.check` find a failing case being the vector `[10 1 28 40 11 -33 42 -42 39 -13 13 -44 -36 11 27 -42 4 21 -39]`, which is **not** the minimum failing case.

```clojure
(tc/quick-check 100 prop-no-42)
;; => {:result false,
       :failing-size 45,
       :num-tests 46,
       :fail [[10 1 28 40 11 -33 42 -42 39 -13 13 -44 -36 11 27 -42 4 21 -39]],
       :shrunk {:total-nodes-visited 38,
                :depth 18,
                :result false,
                :smallest [[42]]}}
```

So it starts shrinking the failing case until it reaches the smallest vector for which the property doesn't hold, which is `[42]`.

Unfortunately `JSCheck` doesn't shrink the failure cases, but [jsverify][14] does, so if you want some shrinking on Javascript give it a try.

## Final thoughts

Since QuickCheck depends on generators to cover the domain, we need to consider those domains may be infinite or very large, so it may be impossible to find the offending failure cases. None the less, we know that by running long enough or a large enough number of tests, we have better odds of finding a problem.

Regarding the name, `property-based` testing is a much better name than `generative` testing, since the later gives the idea that it's about generating data, when it's truly about function and system properties.

The higher level approach of property definition, coupled with the data generation and shrinking features provided by QuickCheck, really helps the case of having something more closer to _proofs_ about how your system behaves.

In the next post I'll write about finite state machine testing using `test.check` and show more complex examples, stay tuned.

I'm [guilespi][15] on Twitter, reach out!

[0]: http://www.cse.chalmers.se/~rjmh/
[1]: https://www.youtube.com/watch?v=zi0rHwfiX1Q
[2]: http://en.wikipedia.org/wiki/Fuzz_testing
[3]: http://blog.guillermowinkler.com/blog/2012/11/14/neither-types-nor-tests-will-solve-your-data-coverage-problem/
[4]: https://coq.inria.fr/a-short-introduction-to-coq
[5]: http://heartbleed.com/
[6]: http://git.openssl.org/gitweb/?p=openssl.git;a=commitdiff;h=96db902
[7]: http://en.wikiquote.org/wiki/Edsger_W._Dijkstra
[8]: http://en.wikipedia.org/wiki/Code_coverage
[9]: http://en.wikipedia.org/wiki/QuickCheck
[10]: https://github.com/clojure/test.check
[11]: www.jscheck.org
[12]: https://www.youtube.com/watch?v=lKXe3HUG2l4
[13]: http://www.quviq.com/volvo-quickcheck/
[14]: http://jsverify.github.io/
[15]: https://twitter.com/guilespi
[16]: https://www.youtube.com/playlist?list=PLZdCLR02grLp__wRg5OTavVj4wefg69hM
[17]: http://testing.uy
