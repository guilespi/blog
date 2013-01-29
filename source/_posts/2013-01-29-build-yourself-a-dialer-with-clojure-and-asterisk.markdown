---
layout: post
title: "Build yourself a dialer with Clojure and Asterisk"
date: 2013-01-29 17:05
comments: true
categories: 
- Clojure
- Asterisk
- Programming
- DIY
---
There are some really [nice][1] [alternatives][2] out there if you want your application to be able to make a call or send a SMS.

But the truth is sometimes you don't want to rely on `the cloud` for your latency-sensitive communications, you already have communications infrastructure 
you want to reuse, or you have such a volume of calls to make that it's cheaper for you to roll your own solution.

So I will show you a DIY guide to roll your own dialer using Clojure and [Asterisk][3], the self proclaimed PBX & Telephony Toolkit.

** What is a Dialer **

If you ever received a spam call from someone trying to sell you something, it was probably made by an automated dialer. 
The purpose is to reach the most possible people in the least time, optimizing resources.

Sometimes it's someone selling Viagra, but hopefully it's used for higher purposes such as massive notification of upcoming emergencies. 

** Integrating with Asterisk **

Asterisk has a lot if integration alternatives, custom dial-plans, AGI scripting, outgoing call spooling, or you can write your own low-level C module, 
each strategy serves its purpose.

For this scenario I've decided to show you an integration with Asterisk using the [Asterisk Manager API][4], 
which allows for remote command execution and event-handling.

I've written a binding for Clojure called [clj-asterisk][5] to sit on top of the low-level text based protocol.

** Making a Call **

The `clj-asterisk` binding map against the Asterisk API is straightforward, so checking against the [Originate Action][6] which is the 
one we need to create an outgoing call.

```
Action: Originate
Channel: SIP/101test
Context: default
Exten: 8135551212
Priority: 1
Callerid: 3125551212
Timeout: 30000
Variable: var1=23|var2=24|var3=25
ActionID: ABC45678901234567890
```
The corresponding `clj-asterisk` invocation is:

``` clojure
(ns call.test
  (:require [clj-asterisk.manager :as manager]
               [clj-asterisk.events :as events]))

(manager/action :Originate {:Channel "SIP/101test"
                            :Context "default"
                            :Exten 8135551212
                            :Priority 1
                            :Timeout 30000
                            :CallerID 3125551212
                            :Variables [var1=23
                                        var2=24
                                        var3=25]})
```

The `ActionID` attribute is not specified since it's internally handled by the `clj-asterisk` library in order to track async responses from Asterisk.

** Receiving events **

For most telephony related actions blocking is not desirable, since most of the time the PBX is handling a conversation and waiting for 
something to happen, using a blocking scheme is far from the best. You need a strategy to wait for events that tell you when something 
you may be interested in, happens.

In this case we will be interested in the `Hangup` event in order to know when the call has ended, so the dialing port is free, 
so we can issue a new call. If you're interested in the complete list of events, [it's available on the Asterisk Wiki][7]

To receive an event using `clj-asterisk` you only need to declare the method with the event name you need to handle:

``` clojure
(ns call.test
  (:require [clj-asterisk.manager :as manager]
            [clj-asterisk.events :as events]))

(defmethod events/handle-event "Hangup"
  [event context]
  (println event))

```
The method passes as parameter the received event and the connection context where the event happened. 

** The Main Loop **

In order to have a proper dialer you will need a main-loop, which life-fulfillment-purpose is:

* Decide on which contacts are to be called
* How many ports are free so how many I can dial now
* Handle retrying and error rules
* Dispatching the calls

I'm assuming you have some data storage to retrieve the contacts to be dialed and will share those details in a later post, 
I will focus now only in the dialing strategy.

``` clojure
(defn process
  "Loops until all contacts for a notification are reached or finally
   cancelled"
  [notification context]
  (let [total-ports (get-available-ports notification)
        contact-list (model/expand-rcpt notification)]
    (loop [remaining contact-list pending-contacts []]
      (or (seq remaining)
           (seq pending-contacts))
             (do
               (let [pending (filter (comp not realized?) pending-contacts)
                     finished (filter realized? pending-contacts)
                     failed (filter
                             (fn [r] (not (contains? #{"CONNECTED" "CANCELLED"} (:status @r))))
                             finished)
                     free-ports (- total-ports (count pending))
                     contacts (take free-ports remaining)
                     dialing (dispatch-calls context notification contacts)]
                 (println (format "Pending %s Finished %s Failed %s Free Ports %s Dispatched %s"
                                  (count pending) (count finished) (count failed) free-ports
                                  (count dialing)))
                 (Thread/sleep 100)
                 (recur (concat (drop free-ports remaining) (map :contact failed))
                        (concat pending dialing)))))))
```

Lets go piece by piece...

You wanna know how many ports are available to dial, for instance you may have only 10 outgoing lines to be used.

``` clojure
total-ports (get-available-ports notification)
```
You wanna know the recipients to be reached.

``` clojure
contact-list (model/expand-rcpt notification)
```

Then you wanna know the status of the contacts you're already dialing and waiting for an answer or for the call to finish.

``` clojure
let [pending (filter (comp not realized?) pending-contacts)
     finished (filter realized? pending-contacts)
     failed (filter
                  (fn [r] (not (contains? #{"CONNECTED" "CANCELLED"} (:status @r))))
                  finished)
     free-ports (- total-ports (count pending))
```
Here pending-contacts is a list of [futures][10], the contacts being currently dialed. Since we don't wanna block waiting for the answer the `realized?` 
function is used in order to count how many of them are finished and filter them. If the finish status is not `CONNECTED` or `CANCELLED` 
we assume the contact has failed and we need to issue a retry for those, typically the `BUSY` and `NO ANSWER` statuses.

Then, given the total available ports minus the already being dialed contacts, a new batch of contacts is dialed

``` clojure
contacts (take free-ports remaining)
dialing (dispatch-calls context notification contacts)
```

The `dispatch-calls` function is pretty straightforward, it just async calls each contact of the list.

``` clojure
(defn dispatch-calls
  "Returns the list of futures of each call thread (one p/contact)"
  [context notification contacts]
  (map #(future (call context notification %)) contacts))
```

Finally the call function issues the request against the Asterisk PBX and saves the result for further tracking or analytics.

``` clojure

(defn call
  "Call a contact and wait till the call ends.
   Function returns the hangup event or nil if timedout"
  [context notification contact]
  (manager/with-connection context
    (let [trunk (model/get-trunk notification)
          call-id (.toString (java.util.UUID/randomUUID))
          prom (manager/set-user-data! call-id (promise))
          response (manager/action :Originate
                                   {:Channel (format "%s/%s/%s"
                                                     (:technology trunk)
                                                     (:number trunk)
                                                     (:address contact))
                                    :Context (:context trunk)
                                    :Exten (:extension trunk)
                                    :Priority (:priority trunk)
                                    :Timeout 60000
                                    :CallerID (:callerid trunk)
                                    :Variables [(format "MESSAGE=%s" (:message notification))
                                                (format "CALLID=%s" call-id)]})]
      (model/save-result notification
                         contact
                         (deref prom 200000 {:error ::timeout})))))
```

The tricky part here is that it's impossible to know before-hand the call-id Asterisk is going to use for our newly created call, 
so we need a way to *mark* our call and relate to it later when an event is received, we do that using the call variable `CALLID` 
which is a `guid` created for each new call.

Our call creating function will wait on a `promise` until the call ends, something we will `deliver` in the Hangup event as shown here:

```clojure

;; Signal the end of the call to the waiting promise in order to
;; release the channel
(defmethod events/handle-event "Hangup"
  [event context]
  (println event)
  (manager/with-connection context
    (let [unique-id (:Uniqueid event)
          call-id (manager/get-user-data unique-id)
          prom (manager/get-user-data call-id)]
      (println (format "Hanging up call %s with unique id %s" call-id unique-id))
      (deliver prom event)
      (manager/remove-user-data! call-id) ;;FIX: this should be done
      ;;on the waiting side or promise may get lost
      (manager/remove-user-data! unique-id))))

;; When CALLID is set, relate it to the call unique-id
;; to be used later in hangup detection
;;
;; The context has the following info inside:
;;   callid => promise
;;   Uniqueid => callid
;;
;; so it's possible to deliver a response to someone waiting
;; on the callid promise
(defmethod events/handle-event "VarSet"
  [event context]
  (when (= (:Variable event) "CALLID")
    (manager/with-connection context
      (println (format "Setting data %s match %s" (:Uniqueid event) (:Value event)))
      (manager/set-user-data! (:Uniqueid event) (:Value event)))))
```

It seems more convoluted than what it actually is, when the `CALLID` variable is set we receive an event that allows the mapping between call-id and 
Asterisk UniqueId to be done. Then when the `Hangup` occurs we can find the promise to be delivered and let the `call` function happily end.

Keep tuned for part II, when I will publish the data model and the complete running Dialer.

[Here][11] is the gist with the code of the current post.

_While you wait, you can follow me on [Twitter][8]!_

[1]: http://www.twilio.com
[2]: http://plivo.com/
[3]: http://www.asterisk.org/ 
[4]: http://www.voip-info.org/wiki/view/Asterisk+manager+API
[5]: https://github.com/guilespi/clj-asterisk
[6]: http://www.voip-info.org/wiki/view/Asterisk+Manager+API+Action+Originate
[7]: http://www.voip-info.org/wiki/view/asterisk+manager+events
[8]: http://www.twitter.com/guilespi
[10]: http://clojuredocs.org/clojure_core/clojure.core/future
[11]: https://gist.github.com/4666926

