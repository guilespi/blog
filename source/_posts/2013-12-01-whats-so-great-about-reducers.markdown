---
layout: post
title: "What's so great about Reducers?"
date: 2013-12-01 13:25
comments: true
categories: 
- clojure
---
This post is about Clojure reducers, but what makes them great are the ideas behind the implementation, which may be portable to other languages.

So if you're interested in performance don't leave just yet.

One of the primary motivators for the reducers library is [Guy Steele's ICFP '09][1] talk, 
and since I assume you don't have one hour to spend verifying I'm telling you it's worth watching, 
I'll do my best to summarize it here, which is a post you will probably scan in less than 15 seconds.

One of the main points of the talk is that **the way we've been thinking about programming for the last 50 years isn't serving us anymore**.

Why? 

Because good sequential code is different from good parallel code.

Parallel vs. Sequential
=====

```
+------------+--------------------------------------+-------------------------------------------------------------+
|            |              Sequential              |                          Parallel                           |
+------------+--------------------------------------+-------------------------------------------------------------+
| Operations | Minimizes total number of operations | Often performs redundant operations to reduce communication |
| Space      | Minimize space usage                 | Extra space to permit temporal decoupling                   |
| Problem    | Linear problem decomposition         | Multiway aggregation of results                             |
+------------+--------------------------------------+-------------------------------------------------------------+
```

The accumulator loop
===========
How would you sum all the elements of an array?

```
SUM = 0 
DO I = 1, 1000000
   SUM = SUM + X(I)
END DO
```

That's right, the accumulator loop, you initialize the accumulator and update the thingy in each iteration step.

But you're complecting how you update your sum with how you iterate your collection, ain't it?

There's a difference between _what_ you do with _how_ you do it. If you say `SUM(X)` it doesn't make promises on the strategy, 
it's when you actually implement that SUM that the sequential promise is made.

{% img center /images/blog/sequential_tree.png 480 580 'Sequential Tree' %}

The problem is the computation tree for the sequential strategy, if we remove the looping machinery and leave only the sums, 
there's a one million steps delay to get to the final result.

So what's the tree you would like to see?

{% img center /images/blog/parallel_tree.png 480 580 'Parallel Tree' %}

And what code you would need to write in order to get that tree? Functional code? 

Think again.

Functional code is not the complete answer, since you can write functional code and still have the same problem.

Since linear linked lists are inherently sequential you may be using a reducer and be on the same spot.

```clojure
(reduce + (range 1 1000000))
```

We need _multiway decomposition_.

Divide and conquer
=============
Rationale behind multiway decomposition is that we need a list representation that allows for binary decomposition of the list.

{% img center /images/blog/tree_list_representation.png 480 580 'List as Tree' %}

You can obviously have many redundant trees representing the same conceptual list, and there's value in redundancy since different trees have different properties.

Summary
========

* Don't split a problem between first and rest, split in equal pieces.
* Don't create a null solution an successively update it, map inputs to singleton solutions and merge tree-wise.
* Combining solutions is trickier than incremental updates.
* Use sequential techniques near the leaves.
* Programs organized for parallelism can be processed in parallel or sequentially.
* Get rid of cons.

Clojure Reducers
=============

So what are Clojure reducers?

In short, it's a library providing a new function `fold`, which is a parallel `reduce+combine` 
that shares the same shape with the old sequence based code, main difference been you get to 
provide a `combiner` function.

Go and read [this][3] and [this][4] great posts by Rich Hickey.

Back? Ok...

As Rich says in his article the accumulator style is not absent but the single initial value 
and the serial execution promises of `foldl/r` have been abandoned.

For what it's worth, I've written in Clojure the **Split a string into words** parallel 
algorithm suggested by Steele [here][5], performance sucks compared against 
`clojure.string/split` but it's a nice algorithm none the less.

```clojure
(defn parallel-words 
  [w]
  (to-word-list 
   (r/fold 100
           combine-states
           (fn [state char] 
             (append-state state (process-char char))) 
           (vec (seq w)))))
```

There are a couple interesting things in the code.

* `combine-states` is the new combiner function, decides how to combine different splits
* `100` is the size when to stop splitting and do a sequential processing (calls `reduce` afterwards). Defaults to `512`.
* The `fn` is the standard reducing function
* The list is transformed into a `vector` before processing.

Last step is just for the sake of experimentation, and has all to do with the underlying structure for vectors.

Both vectors and maps in Clojure are implemented as trees, which as we saw above, is one of the requirements for multiway decomposition. 
There's a [great article here][7] about Clojure vectors, but key interest point is that it provides 
practically `O(1)` runtime for `subvec`, which is how the vector folder `foldvec` successively splits the input vector before reaching the 
sequential processing size.

So if you look at [the source code][6] only for vectors and maps actual fork/join parallelism happens, and standard reduce is called for linear lists.

```clojure

 clojure.lang.IPersistentVector
 (coll-fold
  [v n combinef reducef]
  (foldvec v n combinef reducef))

 Object
 (coll-fold
  [coll n combinef reducef]
  ;;can't fold, single reduce
  (reduce reducef (combinef) coll))

```

What I like the most about reducers is that reducer functions are curried, so you can compose them together as in:

```clojure
(def red (comp (r/filter even?) (r/map inc))) 
(reduce + (red [1 1 1 2]))
;=> 6
```

It's like the utmost example of the [simple made easy][8] Hickey's talk, where decomplecting the system, results in a much simpler but powerful 
design at the same time.

_I'm [guilespi][9] at Twitter_

[1]: http://vimeo.com/6624203
[2]: http://stevereads.com/papers_to_read/ICFPAugust2009Steele.pdf
[3]: http://clojure.com/blog/2012/05/08/reducers-a-library-and-model-for-collection-processing.html
[4]: http://clojure.com/blog/2012/05/15/anatomy-of-reducer.html
[5]: https://gist.github.com/guilespi/7458410
[6]: https://github.com/clojure/clojure/blob/master/src/clj/clojure/core/reducers.clj#L347-L367
[7]: http://hypirion.com/musings/understanding-persistent-vector-pt-1
[8]: http://www.infoq.com/presentations/Simple-Made-Easy
[9]: http://www.twitter.com/guilespi

