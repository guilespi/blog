---
layout: post
title: "Verifying state machine behavior using test.check"
date: 2015-04-12 17:21
comments: true
categories: 
- clojure
- testing
- quickcheck
---
My [previous post][0] was an introduction to the motivations and benefits of a `property-based` testing approach.

I also mentioned the [John Hughes][2] talk, which is great.

But there's a catch.

Until now, we've been considering `property-based` testing in a _functional way_, where properties for functions depend only on the function input, assuming no _state_ between function invocations.

But that's not always the case, sometimes our function inserts data into some database, sends an email, or sends a message to the car anti-lock braking system.

The examples John mentions in his talk are not straightforward to solve without Erlang's QuickCheck, since they're verifying _state machines_ behavior.

Since I'm a Clojurist, I was [a little confused][3] about why I couldn't find a way to do that state machine magic using `test.check`, so bugging Reid and not reading the fine manual was the evident answer.
 
Thing is, test.check has this strategy of composing generators, particularly using `bind` you can generate a value based on a previously generated value by another generator.

For instance, this example shows how to generate a `vector` and then select an element from it:

```clojure
(def keyword-vector (gen/such-that not-empty (gen/vector gen/keyword)))
(def vec-and-elem
  (gen/bind keyword-vector
            (fn [v] (gen/tuple (gen/elements v) (gen/return v)))))

(gen/sample vec-and-elem 4)
;; => ([:va [:va :b4]] [:Zu1 [:w :Zu1]] [:2 [:2]] [:27X [:27X :KW]])
```

But it doesn't have a declarative or simple way to model expected system state.
 
## What is state?

First thing we should think about is, when we do have state? Opposed to a situation when we're testing a [referentially transparent][4] function.

The function `sort` from the previous post is referentially transparent, since for the same input vector, always return the same sorted output.

But what happens when we have a situation like the one described in the talk about this circular buffer?

{% img center /images/blog/circularbuffer.png 680 780 'Circular Buffer' %}

If you want to test the behavior of the `put` and `remove` API calls, it depends on the _state_ of the system, meaning what elements you already have on your buffer.

The properties `put` must comply with, depend on the system state.

So you have this slide from John's presentation:

{% img center /images/blog/fsmmodel.png 680 780 'API Modelling state' %}

With the strategy proposed by QuickCheck to model this testing problem:

* API under test is seen as a sequence of commands.
* Model the state of the system you expect after each command execution.
* Execute the commands and validate the system state is what you expect it to be.

So we need to generate a `sequence` of commands and validate system state, instead of generating input for a single function.

This situation is more common than you may think, even if you're doing functional programming state is everywhere, you have state on your databases and you also have state on your UI.

You would never think about `deleting an email`, and after than `sending the email`, that's an invalid generated sequence of commands(relative to the "email composing state").

This last example is exactly the one described by [Ashton Kemerling][1] from Pivotal, they're using `test.check` to randomly generate test scenarios for the UI. And because test.check doesn't have the _finite state machine modeling thing_, they ended up generating impossible or invalid action sequences, and having to discard them as `NO-OPs` when run.

## FSM Behavior

The problem with Ashton's approach for my situation, was that I had this possibly long sequence of commands or transactions, where each transaction modifies the state of the system, so the last one probable makes no sense at all without some of the _in-the-middle_ ocurring transactions.

The problem is not only discarding invalid sequences, but how to generate something valid _at all_.

Say you have 3 possible actions:

* add `[id, name]`
* edit `[id, name]`
* delete `[id]`

If the command sequence you generate is:

```clojure
[{:type :add :name "John" :id 1} 
 {:type :add :name "Ted" :id 84}
 {:type :delete :id 1}]
```

The `delete` action depends on two things:

1. Someone to be already added (as in the circular buffer described above).
2. Selecting a valid `id` for deletion from the ones already added.

It looks like something you would do using `bind`, but there's something more, there's a `state` that changes when each transaction is applied, and affects each potential command that may be generated afterwards.

Searching around, I found a document titled [Verifying Finite State Machine Behavior Using QuickCheck Eqc_fsm][5] by Ida Lindgren and Robin Malmros, it's an evaluation from _Uppsala Universitet_ to understand whether QuickCheck is suitable for testing mobile telephone communications with base transceiver stations.

Besides the evaluation itself which is worth reading, there's a chapter on Finite State Machines I used as guide to implement something similar with `test.check`.

## The Command protocol

There's a nice diagram in the paper representing the Erlang's finite state machine flow

{% img center /images/blog/fsmflow.png 680 780 'QuickCheck FSM Flow' %}

We observe a few things:

* You have a set of possible commands to generate.
* Each command has a precondition to check validity against current state.
* Each command affects state in some way, generating a new-state.

Translating that ideas into Clojure we can model a command protocol

```clojure
(defprotocol Command
  (precondition [this state] "Returns true if command can be applied in current system state")
  (exec [this state cmd] "Applies generated command to the specified system state, returns new state")
  (generate [this state] "Generates command given the current system state, returns command generator"))
```

So far, we've assumed nothing about:

* How state is going to be modeled.
* What structure each generated command has.

**Since we're using `test.check` we need a particular protocol function `generate` returning the command generator.**

## A Command generator

Having a protocol, lets define our `add`, `edit` and `delete` transactions.

Questions to answer are:

1. What will our model of the world look like? 
2. What's the initial and expected states after applying each transaction?

Our expected state will be something like this:

```clojure
{:people [{:name "John" :id 1}
          {:name "Ted" :id 84}
          {:name "Tess" :id 22}]}
```                                                                    

So our `add` transaction will be:

```clojure
(def add-cmd
  (reify
    Command
    (precondition [_ state]
      (vector? (:people state)))

    (exec [_ state cmd]
      (update-in state [:people] (fn [people]
                                   (conj people
                                         (dissoc cmd :type)))))
    (generate [_ state]
      (gen/fmap (partial zipmap [:type :name :id])
                (gen/tuple (gen/return :add)
                           (gen/not-empty gen/string-alphanumeric)
                           gen/int))))))
```                                                                    

The highlights:

* Our only precondition is to have a vector to `conj` the transaction into.
* The `generate` function returns a standard `test.check` generator for the command.
* The `exec` function applies the generated command to the current system state and returns a new state.

Now, what's interesting is the `delete` transaction:

```clojure
(def delete-cmd
  (reify
    Command
    (precondition [_ state]
      (seq (:people state)))

    (exec [_ state cmd]
      (update-in state [:people] (fn [people]
                                   (vec (filter #(not= (:id %)
                                                       (:id cmd))
                                                people)))))

    (generate [_ state]
      (gen/fmap (partial zipmap [:type :id])
                (gen/tuple (gen/return :delete)
                           (gen/elements (mapv :id (:people state))))))))
```

Note the differences:

* `delete` can only be executed if the people list actually has _someone_ inside
* The generator selects to delete an `id` from the people in the current state (using `gen/elements` selector)
* Applying the command implies removing the selected person from the next state.

## Valid sequence generator

So how do we generate a sequence of commands giving a command list?

This is a recursive approach, that receives the available commands and the sequence size to generate:

```clojure
(defn command-seq
  [state commands size]
  (gen/bind (gen/one-of (->> (map second commands)
                             (filter #(precondition % state))
                             (map #(generate % state))))
            (fn [cmd]
              (if (zero? size)
                (gen/return [cmd])
                (gen/fmap
                 (partial concat [cmd])
                 (command-seq (exec (get commands (:type cmd)) state cmd)
                              commands
                              (dec size)))))))
```

The important parts being:

* Selects only one valid command to generate with `one-of` after filtering preconditions.
* If command sequence size is `0` just finish, otherwise recursively concat the rest of the sequence.
* The new state is updated in the step `(exec (get commands (:type cmd)) state cmd)`, where we need to retrieve the original command object.

If you would like to generate random sequence sizes, just bind it with `gen/choose`.

```clojure
(gen/bind (gen/choose 0 10)
          (fn [num-elements]
            (command-seq {:people []} 
                         {:add-cmd add-cmd
                          :delete-cmd delete-cmd}  
                         num-elements)))
```

Note the initial state is set to `{:people []}` for the `add` command precondition to succeed.

If we generate 3 samples now, it looks good, but there's still a problem....

```clojure
(({:id 0, :name "C", :type :add-cmd}
  {:id 0, :name "2", :type :add-cmd}
  {:id 0, :name "xi", :type :add-cmd}
  {:id 0, :name "p", :type :add-cmd}
  {:id 0, :type :delete-cmd}
  {:id 0, :name "3Q", :type :add-cmd}
  {:id 0, :name "9", :type :add-cmd}
  {:id 0, :type :delete-cmd})
 ({:id -1, :name "H", :type :add-cmd}
  {:id -1, :type :delete-cmd}
  {:id 1, :name "q", :type :add-cmd}
  {:id 0, :name "F", :type :add-cmd}
  {:id 1, :type :delete-cmd})
 ({:id -1, :name "fY", :type :add-cmd}
  {:id 0, :name "a", :type :add-cmd}
  {:id -1, :type :delete-cmd}
  {:id 2, :name "u", :type :add-cmd}
  {:id 2, :type :delete-cmd}
  {:id 1, :name "7", :type :add-cmd}
  {:id -1, :name "E", :type :add-cmd}
  {:id 1, :type :delete-cmd}
  {:id 0, :type :delete-cmd}
  {:id -1, :type :delete-cmd}))
```

Each `add-cmd` is repeating the `id`, since it's generating it without checking the current state, let's change our `add` transaction generator

```clojure
(generate [_ state]
          (gen/fmap (partial zipmap [:type :name :id])
                    (gen/tuple (gen/return :add-cmd)
                               (gen/not-empty gen/string-alphanumeric)
                               (gen/such-that #(-> (mapv :id (:people state))
                                                   (contains? %)
                                                   not)
                                              gen/int))))

```

Now the `id` field generator checks that the generated `int` doesn't belong to the current ids in the state (we could have returned a `uuid` or something else, but it wouldn't make the case for state-dependent generation)

To complete the example we need:

* To apply the commands to the system under test.
* A way to retrieve the system state.
* Comparing the final system state with our final generated expected state.
 
Which is pretty straightforward, so we'll talk about shrinking first.

## Shrinking the sequence

If you were to found a failing command sequence using the code above, you would quickly realize it doesn't shrink properly.

Since we're generating the sequence composing `bind`, `fmap` and `concat` and not using the internal `gen/vector` or `gen/list` generators, the generated sequence doesn't know how to shrink itself.

If you read [Reid's account][6] on writing `test.check`, there's a glimpse of the problem we face, shrinking depends on the generated data type. So a generated `int` knows how to shrink itself, which is different on how a `vector` shrinks itself.

If you combine existing generators, you get shrinking for free, but since we're generating our sequence recursively with `concat` we've lost the `vector` type shrinking capability.

And there's a good reason it is so, but let's first see how shrinking works in `test.check`.

### Rose Trees

`test.check` complects the data generation step with the shrinking of that data. So when you generate some value, behind the scenes all the alternative shrinked scenarios are also generated.

Let's sample an int vector generator:

```clojure
(gen/sample (gen/not-empty (gen/vector gen/int)) 5)
=> ([0] [1] [-1 1] [1 1 -3] [2 3])
```

The `sample` function is hiding from you the alternatives and showing only the actual generated value.

But, if we call the generator using `call-gen`, we have a completely different structure:

```clojure
(gen/call-gen (gen/not-empty (gen/vector gen/int)) (gen/random) 1)
=> [[1] ([[0] ()])]
```

What we have is a [rose tree][7], which is a `n-ary` tree, where each tree node may have any number of childs.

`test.check` uses a very simple modeling approach for the tree, in the form of `[parent childs]`.

So this tree

{% img center /images/blog/rosetree.png 340 390 'Rose Tree' %}

Is represented as

```clojure
[1 [[2 []] 
    [3 []]]]
```

Everytime you get a generated value, what you're looking at is at the root of the tree obtained with `rose/root`, which is exactly what `gen/sample` is doing.

```clojure
(require '[clojure.test.check.rose-tree :as rose])

(rose/root (gen/call-gen (gen/not-empty (gen/vector gen/int)) (gen/random) 1))
=> [0]
```

The shrinking tree you would expect for a generated vector is:

{% img center /images/blog/vectorshrinktree.png 680 780 'Vector rose tree' %}

The more deep inside the tree, the more shrunk the value is. So for instance `integers` shrink up to zero, and `vectors` randomly remove elements until nothing is left.

**If we were to actually look inside the shrunk vector tree, it would also include the shrunked integers, but you get the idea.**

## Shrinking a valid command sequence

I said before our sequence doesn't shrink since it's generated recursively, so this is how our sequence tree looks like so far.

{% img center /images/blog/singlenodecmdtree.png 340 390 'Single node command tree' %}

But even if we were using the vector shrinker we would end up with something like this:

{% img center /images/blog/invalidcmdtree.png 680 780 'Invalid command tree' %}

Since the vector shrinker doesn't really know what a valid command sequence looks like, it will just do a random permutation of commands, ending up with many invalid sequences (like `[{:add 1} {:delete 2}]`).

We will need a custom shrinker, that shrinks only valid command sequences, with a resulting tree like this one:

{% img center /images/blog/validcmdtree.png 680 780 'Valid command tree' %}

To do that, we will modify our protocol to add a new function `postcondition`.

```clojure
(defprotocol Command
  (precondition [this state] "Returns true if command can be applied in current system state")
  (exec [this state cmd] "Applies generated command to the specified system state, returns new state")
  (generate [this state] "Generates command given the current system state, returns command generator"))
  (postcondition [this state cmd] "Returns true if cmd can be applied on specified state"))
```

`postcondition` will be called while shrinking, in order to validate if a shirking sequence is valid for the hipotetical state generated by the previous commands.

Another important function is `gen/pure`, which allows to return our custom rose tree as generator result.

So this is how our command generator looks like now:

```clojure
(defn cmd-seq
  [state commands]
  (gen/bind (gen/choose 0 10)
            (fn [num-elements]
              (gen/bind (cmd-seq-helper state commands num-elements)
                        (fn [cmd-seq]
                          (let [shrinked (shrink-sequence (mapv first cmd-seq)
                                                          (mapv second cmd-seq))]
                            (gen/gen-pure shrinked)))))))


(defn cmd-seq-helper
  [state commands size]
  (gen/bind (gen/one-of (->> (map second commands)
                             (filter #(precondition % state))
                             (map #(generate % state))))
            (fn [cmd]
              (if (zero? size)
                (gen/return [[cmd state]])
                (gen/fmap
                 (partial concat [[cmd state]])
                 (cmd-seq-helper (exec (get commands (:type cmd)) state cmd)
                                 (map second commands)
                                 (dec size)))))))
```

We see two different things here:

1. Generator also returns the state for that particular command.
2. There's a call to `shrink-sequence` that generates the rose tree given the command sequence and intermediate states.

The `shrink-sequence` function being:

```clojure
(defn shrink-sequence
  [cmd-seq state-seq]
  (letfn [(shrink-subseq [s]
            (when (seq s)
              [(map #(get cmd-seq %) s)
               (->> (remove-seq s)
                    (filter (partial valid-sequence? state-seq cmd-seq))
                    (mapv shrink-subseq))]))]
    (shrink-subseq (range 0 (count cmd-seq)))))
```

Highlights:

* Returns a rose tree in the form `[parent childs]`.
* `remove-seq` generates a sequence of subsequences with only one element removed.
* `valid-sequence?` uses `postcondition` to validate the shrinked seq.
* Recursively shrinks the shrunk childs until nothing's left.

## Putting all together

I've put together a running sample for you to check out [here][8].

There's only one property defined: _applying all the generated transactions should return true_, but it fails when there are two delete commands present.

```clojure
(defn apply-tx
  "Apply transactions fails when there are two delete commands"
  [tx-log]
  (->> tx-log
       (filter #(= :delete-cmd (:type %)))
       count
       (> 2)))

(def commands-consistent-apply
  (prop/for-all [tx-log (cmd-seq {:people []} {:add-cmd add-cmd :delete-cmd delete-cmd})]
                (true? (apply-tx tx-log))))
```

```
(tc/quick-check 10 commands-consistent-apply)
=>

{:result false, :seed 1428695347616, :failing-size 7, :num-tests 8, 
 :fail [({:id 6, :name "8", :type :add-cmd} 
         {:id -4, :name "KvoOq", :type :add-cmd} 
         {:id -6, :name "hWn", :type :add-cmd} 
         {:id 6, :type :delete-cmd} 
         {:id -4, :type :delete-cmd})], 
 :shrunk {:total-nodes-visited 55, :depth 16, :result false, 
          :smallest [({:id 0, :name "0", :type :add-cmd} 
                      {:id -2, :name "2", :type :add-cmd} 
                      {:id 0, :type :delete-cmd} 
                      {:id -2, :type :delete-cmd})]}}
```

If you look closely the failing test case has three `add` commands, but when shrunk only two needed in order to fail appear.

Have fun!

I'm [guilespi][15] on Twitter.

[0]: /blog/2015/04/12/property-based-testing-using-quickcheck/
[1]: https://www.youtube.com/watch?v=HXGpBrmR70U
[2]: https://www.youtube.com/watch?v=zi0rHwfiX1Q
[3]: https://twitter.com/guilespi/status/566315813268111360
[4]: http://en.wikipedia.org/wiki/Referential_transparency_%28computer_science%29
[5]: http://www.diva-portal.org/smash/get/diva2:343744/FULLTEXT01.pdf
[6]: http://reiddraper.com/writing-simple-check/
[7]: http://en.wikipedia.org/wiki/Rose_tree
[8]: https://github.com/guilespi/fsm-test-check/blob/master/src/fsm_test_check/core.clj
[15]: https://twitter.com/guilespi



