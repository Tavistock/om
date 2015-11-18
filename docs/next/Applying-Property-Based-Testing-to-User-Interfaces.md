## Introduction

Traditional user interface development methodologies present
incredible challenges for automated testing. Mutable object graphs
coupled to mutable view graphs coupled to unmanaged programmatic
mutation are a never-ending siege upon test automation. As a result it
is not uncommon that Quality Assurance (QA) teams must enumerate limited
paths into a vast state space they cannot hope to cover. It
should thus come as little surprise that end users encounter so many
*simple* bugs in their interactions.

The sad state of affairs is further exacerbated when a product must
target multiple platforms. Automated UI testing frameworks between
platforms vary in their capability (and certainly in their APIs)
requiring an incredible amount of effort duplication. And such testing
frameworks cannot fix the broken foundations and thus still require a
considerable amount of human intervention. This level of human
intervention is rarely something many smaller teams can even afford.

By correcting the foundations we can provide radically enhanced
automated testing of user interfaces.

### Better Foundations

In Om Next you do not construct user interfaces on quicksand - there
are no mutable objects graphs and using local mutation is discouraged
for anything beyond platform specific transient styling and
animations. Thus every critical state of an Om Next application is a
complete and consistent snapshot. Not only that, React's approach to
rendering means that the data should closely correspond to the shape
of the UI. This correspondence is further strengthened when embracing
a component co-located query model - the UI data tree must precisely
match the component tree. And because mutations are reified it is a
simple matter to fabricate a series of mutation expressions to test
some invariant.

*This means we can confidently test UI state transitions without
rendering anything at all.*

The implications are enormous for applications that intend to leverage
React Native - *shared* test automation can now cover the web, iOS,
and Android.

As we shall see Om Next is ready for this kind of test automation
without doing anything more than being constructed on simple, good
foundations.

### Property Based Testing

We will see how Om Next's architecture when paired with
[property based testing](https://en.wikipedia.org/wiki/QuickCheck) can
dramatically increase our confidence in the correctness of user
interface code with minimal effort. We will use
[test.check](https://github.com/clojure/test.check) a robust and
featureful implementation of the property based testing methodology. Even
if you are not familiar with property based testing, the benefits will
become immediately clear.

## Setup

We assume you are now familiar with the previous tutorial setups. We
will not cover that material.

However your `project.clj` should look like the following:

```clj
(defproject om-tutorial "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.7.0"]
                 [org.clojure/clojurescript "1.7.170"]
                 [org.clojure/test.check "0.8.2"]
                 [org.omcljs/om "1.0.0-alpha21"]
                 [figwheel-sidecar "0.5.0-SNAPSHOT" :scope "test"]])
```

## Building the UI Model

We will build the state and state transition model for a simple
application. This simple application has a few concepts, people,
friends, and two friending operations - add and remove.

Your `ns` form should look like the following:

```clj
(ns om-tutorial.core
  (:require [clojure.test.check :as tc]
            [clojure.test.check.generators :as gen]
            [clojure.test.check.properties :as prop]
            [om.next :as om :refer-macros [defui]]))
```

We import the various bits of `test.check` that we'll need and we only
need `defui` from `om.next`.

```clj
(enable-console-print!)

(def init-data
  {:people [{:id 0 :name "Bob" :friends []}
            {:id 1 :name "Laura" :friends []}
            {:id 2 :name "Mary" :friends []}]})

(defui Friend
  static om/Ident
  (ident [this props]
    [:person/by-id (:id props)])
  static om/IQuery
  (query [this]
    [:id :name]))

(defui Person
  static om/Ident
  (ident [this props]
    [:person/by-id (:id props)])
  static om/IQuery
  (query [this]
    [:id :name {:friends (om/get-query Friend)}]))

(defui People
  static om/IQuery
  (query [this]
    [{:people (om/get-query Person)}]))
```

We declare some simple data and the `defui` bits we'll need to
normalize this information.

```clj
(defmulti read om/dispatch)

(defmethod read :people
  [{:keys [state query] :as env} key _]
  (let [st @state]
    {:value (om/db->tree query (get st key) st)}))
```

We use a new helper, `om.next/db->tree` to avoid having to manually
join in references.

Let's define our mutations. First let's do `friend/add`:

```clj
(defmulti mutate om/dispatch)

(defn add-friend [state id friend]
  (letfn [(add* [friends ref]
            (cond-> friends
              (not (some #{ref} friends)) (conj ref)))]
    (-> state
      (update-in [:person/by-id id :friends]
        add* [:person/by-id friend]))))

(defmethod mutate 'friend/add
  [{:keys [state] :as env} key {:keys [id friend] :as params}]
  {:action
   (fn [] (swap! state add-friend id friend))})
```
   
There are two bugs in `add-friend`. The first is that it permits self
friending, the second, it does not change the friend list of the other
person. Don't worry our property based tests will catch this.

Lets write the `friend/remove` mutation:

```clj
(defn remove-friend [state id friend]
  (letfn [(remove* [friends ref]
            (cond->> friends
              (some #{ref} friends) (into [] (remove #{ref}))))]
    (-> state
      (update-in [:person/by-id id :friends]
        remove* [:person/by-id friend]))))

(defmethod mutate 'friend/remove
  [{:keys [state] :as env} key {:keys [id friend] :as params}]
  {:action (fn [] (swap! state remove-friend id friend))})
```

There's only one bug in this one, it's one of the same bugs in the
add friends code.

We now define some useful top levels:

```clj
(def app-state
  (atom (om/tree->db People init-data true)))

(def parser (om/parser {:read read :mutate mutate}))
```

## Property Based Testing

The first thing that we need to be able to do is generate random
transactions. A huge part of property based testing is that by writing
generators property based testing can give us something important back
in return - shrinking. That is property based testing can find the
failure and then automatically shrink it into a minimal case. We don't
have time to cover every aspect of building generators but it should
be clear what the following accomplishes:

```clj
(def gen-tx-add-remove
  (gen/vector
    (gen/fmap seq
      (gen/tuple
        (gen/elements '[friend/add friend/remove])
        (gen/fmap (fn [[n m]] {:id n :friend m})
          (gen/tuple
            (gen/elements [0 1 2])
            (gen/elements [0 1 2])))))))
```
            
Using a REPL try the following:

```clj
(gen/sample gen-tx-add-remove 10)
;; ([]
;;  []
;;  [(friend/add {:id 1, :friend 1}) (friend/remove {:id 1, :friend 2})]
;;  [(friend/add {:id 0, :friend 0})
;;   (friend/remove {:id 0, :friend 1})
;;   (friend/add {:id 0, :friend 1})]
;; [(friend/add {:id 0, :friend 2})
;;  (friend/add {:id 2, :friend 2})
;;  (friend/remove {:id 2, :friend 1})
;;  (friend/add {:id 2, :friend 1})])
```

As you can see all `gen-tx-add-remove` does is generate random
transactions containing friend adds and removes.

### Model the User

We'll focus on testing the *UI data tree*. This allows us to test the
normalized model since the UI data tree is completely derived from it, but
it also gives us confidence that the user will never see
something unexpected. If it's not in the UI data tree it won't be
present in a stateless view derived from it.

The other reason to focus on the *UI data tree* is that
this is what the user actually interacts against. They click buttons,
move mice, type keys, and gesture to generate a stream of transactions
that transition the normalized state that we then denormalize yet
again to present feedback to the user.

The user is the unknown actor in our system and they will inevitably
produce some unexpected series of transactions that could put the UI
into a corrupted state. So we want to model the user as process and
test this.

This bears repeating.

> If you want to increase confidence in the
> correctness of user interface code **model the user**.

Anything less than this is not testing anything of importance.

### Preventing Self Friending

Here's our code to test that self friending should not be possible:

```clj
(defn self-friended? [{:keys [id friends]}]
  (boolean (some #{id} (map :id friends))))

(defn prop-no-self-friending []
  (prop/for-all [tx gen-tx-add-remove]
    (let [parser (om/parser {:read read :mutate mutate})
          state  (atom (om/tree->db People init-data true))]
      (parser {:state state} tx)
      (let [ui (parser {:state state} (om/get-query People))]
        (not (some self-friended? (:people ui)))))))
```

If you've gone through all the previous tutorials this test should be
obvious. We are using `test.check` to generate random transactions and we
then want to know if anyone in the UI data tree has themselves in
the friend list.

Let's try the following at the REPL:

```clj
(tc/quick-check 100 (prop-no-self-friending))
;; ... elided ...
;; {:result false, :seed 1445791557331, :failing-size 1, :num-tests 2,
;;  :fail [[(friend/add {:id 0, :friend 0})]],
;;  :shrunk {:total-nodes-visited 1, :depth 0, :result false,
;;           :smallest [[(friend/add {:id 0, :friend 0})]]}}
```

We quickly determine that the property does not hold and we get the
smallest transaction that proves the property does not hold.

Let's fix our friend adding mutation:

```clj
(defn add-friend [state id friend]
  (letfn [(add* [friends ref]
            (cond-> friends
              (not (some #{ref} friends)) (conj ref)))]
    (if-not (= id friend) ;; FIXED
      (-> state
        (update-in [:person/by-id id :friends]
          add* [:person/by-id friend]))
      state)))
```

Rerun the test. We should see no failures now.

### Friend Consistency

We now want to know with confidence that regardless of how many
friends are added and removed that friend lists always remain
consistent. That is A is a friend of B if and only if B is a friend of
A.

```clj
(defn friends-consistent? [people]
  (let [indexed (zipmap (map :id people) people)]
    (letfn [(consistent? [[id {:keys [friends]}]]
              (let [xs (map (comp :friends indexed :id) friends)]
                (every? #(some #{id} (map :id %)) xs)))]
      (every? consistent? indexed))))

(defn prop-friend-consistency []
  (prop/for-all [tx gen-tx-add-remove]
    (let [parser (om/parser {:read read :mutate mutate})
          state  (atom (om/tree->db People init-data true))]
      (parser {:state state} tx)
      (let [ui (parser {:state state} (om/get-query People))]
        (friends-consistent? (:people ui))))))
```

Again our test should be pretty easy to understand. We're just
checking that in our UI data tree that it's not possible for anyone to
ever be in a situation where a user would see inconsistent friend
lists.

Let's run the following test:

```clj
(tc/quick-check 100 (prop-friend-consistency))
;; ... elided ...
;; {:result false, :seed 1445791900621, :failing-size 3, :num-tests 4,
;;  :fail [[(friend/add {:id 0, :friend 1})]],
;;  :shrunk {:total-nodes-visited 2, :depth 0, :result false,
;;  :smallest [[(friend/add {:id 0, :friend 1})]]}}
```

Oops looks like we forget to do the corresponding operation in add
friend. Let's fix that:

```clj
(defn add-friend [state id friend]
  (letfn [(add* [friends ref]
            (cond-> friends
              (not (some #{ref} friends)) (conj ref)))]
    (if-not (= id friend)
      (-> state
        (update-in [:person/by-id id :friends]
          add* [:person/by-id friend])
        (update-in [:person/by-id friend :friends] ;; FIXED
          add* [:person/by-id id]))
      state)))
```
      
Let's try again:

```clj
(tc/quick-check 100 (prop-friend-consistency))
;; ... elided ...
;; {:result false, :seed 1445792089722, :failing-size 4, :num-tests 5,
;;  :fail [[(friend/add {:id 0, :friend 1})
;;          (friend/remove {:id 1, :friend 0})
;;          (friend/remove {:id 1, :friend 0})]],
;;  :shrunk {:total-nodes-visited 7, :depth 1, :result false,
;;  :smallest [[(friend/add {:id 0, :friend 1}) (friend/remove {:id 1, :friend 0})]]}}
```

Hrm it appears there's still something wrong. The minimal example
gives us a hint that it's probably something with friend removal. And
if we examine friend removal we'll see that we are in fact missing a
corresponding remove.

Let's fix that:

```clj
(defn remove-friend [state id friend]
  (letfn [(remove* [friends ref]
            (cond->> friends
              (some #{ref} friends) (into [] (remove #{ref}))))]
    (-> state
      (update-in [:person/by-id id :friends]
        remove* [:person/by-id friend])
      (update-in [:person/by-id friend :friends] ;; FIXED
        remove* [:person/by-id id]))))
```

Let's try again:

```clj
(tc/quick-check 100 (prop-friend-consistency))
```

This should succeed.

## Conclusion

The fundamental problem with automating testing of user interfaces
reduces to obstacles around modeling the user. By removing all such
obstacles Om Next makes it possible to deliver confidence without
actually providing any specific features around testability.

*Model the user*.

## Appendix

The entire correct final source code follows:

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [clojure.test.check :as tc]
            [clojure.test.check.generators :as gen]
            [clojure.test.check.properties :as prop]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]))

(enable-console-print!)

(def init-data
  {:people [{:id 0 :name "Bob" :friends []}
            {:id 1 :name "Laura" :friends []}
            {:id 2 :name "Mary" :friends []}]})

(defui Friend
  static om/Ident
  (ident [this props]
    [:person/by-id (:id props)])
  static om/IQuery
  (query [this]
    [:id :name]))

(defui Person
  static om/Ident
  (ident [this props]
    [:person/by-id (:id props)])
  static om/IQuery
  (query [this]
    [:id :name {:friends (om/get-query Friend)}]))

(defui People
  static om/IQuery
  (query [this]
    [{:people (om/get-query Person)}]))

(defmulti read om/dispatch)

(defmethod read :people
  [{:keys [state query] :as env} key _]
  (let [st @state]
    {:value (om/db->tree query (get st key) st)}))

(defmulti mutate om/dispatch)

(defn add-friend [state id friend]
  (letfn [(add* [friends ref]
            (cond-> friends
              (not (some #{ref} friends)) (conj ref)))]
    (if-not (= id friend)
      (-> state
        (update-in [:person/by-id id :friends]
          add* [:person/by-id friend])
        (update-in [:person/by-id friend :friends]
          add* [:person/by-id id]))
      friend)))

(defmethod mutate 'friend/add
  [{:keys [state] :as env} key {:keys [id friend] :as params}]
  {:action
   (fn [] (swap! state add-friend id friend))})

(defn remove-friend [state id friend]
  (letfn [(remove* [friends ref]
            (cond->> friends
              (some #{ref} friends) (into [] (remove #{ref}))))]
    (if-not (= id friend)
      (-> state
        (update-in [:person/by-id id :friends]
          remove* [:person/by-id friend])
        (update-in [:person/by-id friend :friends]
          remove* [:person/by-id id]))
      state)))

(defmethod mutate 'friend/remove
  [{:keys [state] :as env} key {:keys [id friend] :as params}]
  {:action (fn [] (swap! state remove-friend id friend))})

(def app-state
  (atom (om/tree->db People init-data true)))

(def parser (om/parser {:read read :mutate mutate}))

(def gen-tx-add-remove
  (gen/vector
    (gen/fmap seq
      (gen/tuple
        (gen/elements '[friend/add friend/remove])
        (gen/fmap (fn [[n m]] {:id n :friend m})
          (gen/tuple
            (gen/elements [0 1 2])
            (gen/elements [0 1 2])))))))

(defn self-friended? [{:keys [id friends]}]
  (boolean (some #{id} (map :id friends))))

(defn prop-no-self-friending []
  (prop/for-all [tx gen-tx-add-remove]
    (let [parser (om/parser {:read read :mutate mutate})
          state  (atom (om/tree->db People init-data true))]
      (parser {:state state} tx)
      (let [ui (parser {:state state} (om/get-query People))]
        (not (some self-friended? (:people ui)))))))

(defn friends-consistent? [people]
  (let [indexed (zipmap (map :id people) people)]
    (letfn [(consistent? [[id {:keys [friends]}]]
              (let [xs (map (comp :friends indexed :id) friends)]
                (every? #(some #{id} (map :id %)) xs)))]
      (every? consistent? indexed))))

(defn prop-friend-consistency []
  (prop/for-all [tx gen-tx-add-remove]
    (let [parser (om/parser {:read read :mutate mutate})
          state  (atom (om/tree->db People init-data true))]
      (parser {:state state} tx)
      (let [ui (parser {:state state} (om/get-query People))]
        (friends-consistent? (:people ui))))))
```
