---
layout: post
title: "Better Errors"
description: "Exception-based control flow sucks. And I know how to do it better."
date: 2015-10-16
tags:
 - :en
 - :code-design
---

Why and when try/catch sucks
============================

Exceptions were invented to handle unexpected situations. And here is
where they are good. When your app:
* needs to die silently 
* or panic on top-level about syntax error
* or catch errors that caused by really unexpected 
situation (read: when you fucked up writing the code)

then it's fine, app should panic and you should fix it.

But problem is that people use exceptions to manage control flow.
And it looks like fucking goto.  Really, see the following example
(stolen from [niwi.nz][https://www.niwi.nz/2015/03/08/error-handling/]):

```
def read_report_file(name:str, owner:User) -> Report:
    try:
        path = os.path.join(REPORTS_DIR, name)
        file = io.open(path, "rt")
        return process_file_and_get_report(file, owner)

    except PermissionDeniend as e:
        send_notification_about_unauthorized_request(e, owner)
        raise e

    except FileNotFound as e:
        raise ReportNotFound("Report file does not exists") from e

    finally:
        file.close()
```

Let's explain, what's so bad there.

* It's a violation of the Single Responsibility Principle. The function
that use try/catch is doing *at least* two things by definition: it does
some (hopefully) usefull work and handles some error.
* It's a violation of open/closed principle. When you finished the function
you shouldn't touch it. But if you need to add more catch clauses you
have to change the function without changing main logic. What if you
break something by this? About open/closed principle you can read more in
[wiki][https://www.wikiwand.com/en/Open/closed_principle]
* It brings unnecessary complexity. In particular, incidental complexity:
the sequence that you list which exception types you're going to handle
before others. Watch [simple made easy][http://www.infoq.com/presentations/Simple-Made-Easy]
to realize why simplicity matters.

Go developers in their documentation explain this problem too:

```
A number of designs for exceptions have been proposed but each adds significant
complexity to the language and run-time. By their very nature, exceptions span
functions and perhaps even goroutines; they have wide-ranging implications. 
There is also concern about the effect they would have on the libraries. They
are, by definition, exceptional yet experience with other languages that
support them show they have profound effect on library and interface
specification. It would be nice to find a design that allows them to be truly 
exceptional without encouraging common errors to turn into special control 
flow that requires every programmer to compensate.
```

I always knew they are smart guys. They are telling the same that I wanted to
from the beginning. Exceptions are good in *truly exceptional situations*, and
in other cases we need a better solution.

A better solution
=================

In go they forced the following syntax: each function should return a tuple
of two values where the first one is the result and the second one is error.
When error is nil then it's fine, moving on. When it's not then we have to
process it somehow.

```golang
def get_soundcard_usb_version(computer):
    sound_card, err = get_sound_card(computer)
    if err:
       return None, err
    usb, err = get_usb(sound_card)
    if err:
       return None, err
    version, err = get_version(usb)
    if err:
       return None, err
    return version, None
```

The idea to return error is value is good, but the code looks a little awkward.
In clojure we can add a couple of macros to reduce boilerplate
(see [adambard][http://adambard.com/blog/acceptable-error-handling-in-clojure/])
but we still have to return a vector in each function.

What if we had a way to distinguish errors from results? Wait, we have it:

```clojure
(defrecord Failure [message])

(defn fail [message] (Failure. message))

(defprotocol ComputationFailed
  "A protocol that determines if a computation has resulted in a failure.
   This allows the definition of what constitutes a failure to be extended
   to new types by the consumer."
  (failed? [self]))

(extend-protocol ComputationFailed
  Object    (failed? [self] false)
  Failure   (failed? [self] true)
  Exception (failed? [self] true))
```

Nice! Thank's Rich for that. Now we can check if our function call failed
or not:

```clojure
(if (failed? (somefun 42)) "oh that's bad" "oh that's great!")
```

Great. I think you already got that we're not stopping here and not
going to do the checking in every single function we use.
Let's automate that:

```clojure
(defn failed-arg [args]
  (first (filter failed? args)))

(defn maybe [f & args]
  (if-let [x (failed-arg args)] x (apply f args)))
```

What have we just did? We've created a simple way to pass
errors through functions. If any argument of the function is Failure
then nothing will be done and the argument will be returned. Else
we will just call the function. Les't demonstrate:

```clojure
(maybe + 2 3) ;= 5
(maybe + 2 (fail "oops!")) ;= #Failure{:message "oops!"}
```

Yahoo! Aren't we cool already? Definitely we are. Let's go further:

```clojure
(defmacro maybe->> [val & fns]
  (let [fns (for [f fns] `(maybe ~f))]
    `(->> ~val ~@fns)))
```

Now we chain functions that can potentially fail just as simple as any others:

```clojure
(maybe->> 3 #(fail (str "arg: " %)) #(/ 5 %)) ;= #Failure{:message "arg: 3"}
(maybe->> 3 #(+ 2 %) #(/ 5 %)) ;= 1
```

Last thing is to implement the bingings syntax. And we will use
m...khm-khm...onads. Scared already? Relax, I'll show how it all works.

```clojure
(require '[clojure.algo.monads :refer [defmonad domonad]])

(defmonad error-m 
  [m-result identity
   m-bind   (fn [x f] (maybe f x))])

(defmacro attempt-all 
  ([bindings return] `(domonad error-m ~bindings ~return))
  ([bindings return else]
     `(let [result# (attempt-all ~bindings ~return)]
        (if (failed? result#) ~else result#))))
```

Yes, looks scary. Moreover, I stole it 
from [brehaut][http://brehaut.net/blog/2011/error_monads]. But here is how
it works:

```clojure
(attempt-all [a 1
              b (inc a)] b)) ;= 2

(attempt-all [a (fail "oops")
              b (inc a)] b)) ;= #Failure{:message "oops"}

(attempt-all [a (fail "oops")
              b (inc a)] b ":(")) ;= ":("
```

Summary
=======

In my opinion, Failure type provides sane way to handle errors that
can be predicted. You can isolate error handling from domain logic and
write more composable and more functional code.

Learn More
==========

* <http://adambard.com/blog/acceptable-error-handling-in-clojure/>
* <http://brehaut.net/blog/2011/error_monads>
* <http://michaeldrogalis.tumblr.com/post/40181639419/trycatch-complects-we-can-do-so-much-better>
* <https://www.niwi.nz/2015/03/08/error-handling/>
* <http://www.infoq.com/presentations/Simple-Made-Easy>
