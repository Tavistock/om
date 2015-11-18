> **DO NOT EDIT THIS PAGE**: This page is under heavy active development. 

## Introduction

The [[Quick Start (om.next)]] introduced the fundamentals, now we'll
explore more closely how Om Next simplifies coordination of state
across the component tree without introducing boilerplate or additional
complexity.

Before we begin we need to look at **normalization** and
**identity**. Note the following two subsections are *not* intended to
be entered into a REPL.

### Normalization

Consider the following bit of data. Let's assume for a moment that
this is the data returned from a remote service based on a root
query. Can you identify what is problematic about this representation
in the context of a user interface?

```clj
(def init-data
  {:list/one [{:name "John" :points 0}
              {:name "Mary" :points 0}
              {:name "Bob"  :points 0}]
   :list/two [{:name "Mary" :points 0 :age 27}
              {:name "Gwen" :points 0}
              {:name "Jeff" :points 0}]})
```

The issue is that the value `"Mary"` appears *twice*. In this case
this value represents the same logical entity. While this is fine for
rendering, this representation is problematic for updates. You would
need to track all the places where `"Mary"` occurs and update them by
hand.

If you give an Om Next reconciler data without wrapping it in an atom,
the reconciler assumes the data has not yet been normalized. It will
use the root query to normalize the data. For example assume the
following bit of code:

```clj
(def reconciler
  (om/reconciler
    {:state  init-data
     :parser (om/parser {:read read :mutate mutate})}))
```

If you deref'ed the `reconciler` you would see the following:

```clj
{:list/one
 [[:person/by-name "John"]
  [:person/by-name "Mary"]
  [:person/by-name "Bob"]],
 :list/two
 [[:person/by-name "Mary"]
  [:person/by-name "Gwen"]
  [:person/by-name "Jeff"]],
 :person/by-name
 {"John" {:name "John", :points 0},
  "Mary" {:name "Mary", :points 0, :age 27},
  "Bob" {:name "Bob", :points 0},
  "Gwen" {:name "Gwen", :points 0}, 
  "Jeff" {:name "Jeff", :points 0}}}
```

Notice that all the data has been de-duplicated. In place of the
original values in `:line/one` and `:list/two` we instead have a
vector that can get be used with `get-in` to get the actual
data. `"Mary"` now appears only once and all the fields are
preserved. We can now update `"Mary"` in one location and expect that
all parts of our user interface that need it will update accordingly.

But how was Om Next able to automatically normalize the data?

Surprise, surprise ... this falls out of colocated component queries
with a small bit of extra help to determine **identity**.

### Identity

Colocated queries actually give us an incredible amount of
information with regards to intent.

```clj
(defui Person
  static om/Ident
  (ident [this {:keys [name]}]
    [:person/by-name name])
  static om/IQuery
  (query [this]
    '[:name :points :age])
  Object
  (render [this]
    ;; ... elided ...
    ))
```

For example if you get a query from a component via
`om.next/get-query` you'll see that the query has some interesting
metadata:

```clj
(-> Person om.next/get-query meta)
;; => {:component om-tutorial.core/Person}
```

This means we always know what component is associated with what data
in the denormalized response.

However this isn't enough to normalize. We need to know what unique
identity value should replace the original one. So we implement the
protocol `om.next/Ident`. `om.next/Ident` takes props and returns a
unique key (unique to the client, not globally). This key has an
additional related purpose beyond normalization - we can also use this
key to know which components are backed by the same data.

So **normalization** simplifies updates. Providing an **identity**
operation allows us to automate **normalization** based on the
colocated queries. The **identity** operation also makes UI
reconciliation trivial since we now know which UI elements map to which
data.

> **Note on DataScript**: the previous sections *do not* apply to a
> custom store like DataScript which already stores all data in a
> normalized form. Normalization is necessary only in the cases
> where you wish to represent client state with simple EDN data or
> where the data is being supplied by a remote service that responds
> to **query expressions**. These results will be in tree form ready
> to render, but again this form isn't ideal for updates.

Lets see how this works in practice. You can skip the following three
sections if you've already done the setup from the Quick Start.

## Setting Up

This tutorial uses [Leiningen](http://leiningen.org),
[Figwheel](https://www.youtube.com/watch?v=j-kj2qwJa_E), and
[Google Chrome](http://www.google.com/chrome/). You should install
Leiningen and Google Chrome before proceeding. Leiningen is a
standard tool for managing Clojure and ClojureScript library
dependencies. Figwheel is a ClojureScript build tool and REPL that
enables an expressive live programming model well suited for
interactive application development. Figwheel also plays well with
text editors that make traditional REPL integration more challenging.

You can of course use any web browser, but this tutorial only includes
relevant instructions for Chrome to avoid tangential material.

Create a new project and switch into it:

```shell
mkdir om-tutorial
cd om-tutorial
```

Inside your project directory create a `project.clj`:

```clj
touch project.clj
```

Make it look like the following:

```clj
(defproject om-tutorial "0.1.0-SNAPSHOT"
  :description "My first Om program!"
  :dependencies [[org.clojure/clojure "1.7.0"]
                 [org.clojure/clojurescript "1.7.170"]
                 [org.omcljs/om "1.0.0-alpha21"]
                 [figwheel-sidecar "0.5.0-SNAPSHOT" :scope "test"]])
```

A Leiningen `project.clj` file simply allows you to declare a variety
of properties about your project. In our the case the most important
is the list of `:dependencies`.

Now create a file `script/figwheel.clj`.

```shell
mkdir script
touch script/figwheel.clj
```

Change `script/figwheel.clj` to look like the following:

```clj
(require '[figwheel-sidecar.repl :as r]
         '[figwheel-sidecar.repl-api :as ra])

(ra/start-figwheel!
  {:figwheel-options {}
   :build-ids ["dev"]
   :all-builds
   [{:id "dev"
     :figwheel true
     :source-paths ["src"]
     :compiler {:main 'om-tutorial.core
                :asset-path "js"
                :output-to "resources/public/js/main.js"
                :output-dir "resources/public/js"
                :verbose true}}]})

(ra/cljs-repl)
```

This file describes how to build your ClojureScript project and starts
a REPL. If you are new to ClojureScript you may find this file a bit
overwhelming. If you would like to know more, after this tutorial you
may want to work through the ClojureScript
[Quick Start](https://github.com/clojure/clojurescript/wiki/Quick-Start)
to re-inforce fundamental ClojureScript concepts encountered in this
tutorial.

## Markup

We now need to provide some basic markup to host our ClojureScript
application.

Make a file `resources/public/index.html`:

```shell
mkdir -p resources/public
touch resources/public/index.html
```

Change the contents of this file to the following:

```html
<!DOCTYPE html>
<html>
    <head lang="en">
        <meta charset="UTF-8">
        <title>Om Tutorial!</title>
    </head>
    <body>
        <div id="app"></div>
        <script src="js/main.js"></script>
    </body>
</html>
```

## Checkpoint

Create a file `src/om_tutorial/core.cljs`:

```shell
mkdir -p src/om_tutorial
touch src/om_tutorial/core.cljs
```

Edit its contents to look like the following:

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]))

(enable-console-print!)

(println "Hello world!")
```

Start Figwheel:

```clj
lein run -m clojure.main script/figwheel.clj
```

For enhanced REPL behavior it's recommended that you install
[rlwrap](http://utopia.knoware.nl/~hlub/uck/rlwrap/). Under OS X this
can be easily done with [brew](http://brew.sh).

If you have `rlwrap` installed you can then launch with:

```clj
rlwrap lein run -m clojure.main script/figwheel.clj
```

Point your browser at
[http://localhost:3449](http://localhost:3449). You should see a blank
page with the title "Om Tutorial!" visible on your browser tab.

Open the Chrome Developer Tools with the **View > Developer >
JavaScript Console** menu. In the JavaScript Console you should see
`Hello, world!` printed out.

## Studying Identity & Normalization

Change `src/om_tutorial/core.cljs` to look like the following:

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]))

(enable-console-print!)

(def init-data
  {:list/one [{:name "John" :points 0}
              {:name "Mary" :points 0}
              {:name "Bob"  :points 0}]
   :list/two [{:name "Mary" :points 0 :age 27}
              {:name "Gwen" :points 0}
              {:name "Jeff" :points 0}]})
```

This is the same bit of data we saw above.

### Adding Reads

Let's write our parsing code next and test as we go along. First lets
deal with reading:

```clj
(defmulti read om/dispatch)

(defn get-people [state key]
  (let [st @state]
    (into [] (map #(get-in st %)) (get st key))))

(defmethod read :list/one
  [{:keys [state] :as env} key params]
  {:value (get-people state key)})

(defmethod read :list/two
  [{:keys [state] :as env} key params]
  {:value (get-people state key)})
```

Note that in this case we *must* supply read functions - our data will
be normalized so we have to build the original tree form. Fortunately
doing this is usually a one liner. However even one liners can have
bugs so we'll want to do some interactive testing with the REPL
momentarily.

But before we do that we need some help from our components that will
map this data into the UI. Fortunately we can write components without
render functions to simplify design and testing. We know that we'll be
rendering a `RootView` with two lists and a `Person` component for
each logical person in our test data:

```clj
(defui Person
  static om/Ident
  (ident [this {:keys [name]}]
    [:person/by-name name])
  static om/IQuery
  (query [this]
    '[:name :points]))

(defui RootView
  static om/IQuery
  (query [this]
    (let [subquery (om/get-query Person)]
     `[{:list/one ~subquery} {:list/two ~subquery}])))
```

The only new idea here is `om.next/Ident`. Like `om.next/IQuery` it's
a static method so that we can invoke regardless of whether we have
instantiated any components or not. This method takes the props the
component will receive (or has received) and return a unique
identifier. This identifier will be used for normalization (to dedupe)
as well as to identify which mounted components depend on the same
data and therefore should change together.

In anycase this is all we need to do to view a normalized version of our data,
try the following at the Figwheel REPL:

```clj
(in-ns 'om-tutorial.core)
(require '[cljs.pprint :as pp])
(def norm-data (om/tree->db RootView init-data true))
(pp/pprint norm-data)
```

You should see a normalized version of the data pretty printed.

Now let's verify that we can reconstruct the data:

```clj
(def parser (om/parser {:read read}))
(parser {:state (atom norm-data)} '[:list/one])
```

You should see denormalized data.

### Adding Mutations

Lets add some simple mutations to increment and decrement the `:points`
field of the various characters:

```clj
(defmulti mutate om/dispatch)

(defmethod mutate 'points/increment
  [{:keys [state]} _ {:keys [name]}]
  {:action
   (fn []
     (swap! state update-in
       [:person/by-name name :points]
       inc))})

(defmethod mutate 'points/decrement
  [{:keys [state]} _ {:keys [name]}]
  {:action
   (fn []
     (swap! state update-in
       [:person/by-name name :points]
       #(let [n (dec %)] (if (neg? n) 0 n))))})
```

By now this should look pretty straightforward.

Let's verify that we can mutate and get the expected denormalized
view in the Figwheel REPL:

```clj
(def parser (om/parser {:read read :mutate mutate}))
(def st (atom norm-data))
(parser {:state st} '[(points/increment {:name "Mary"})])
(parser {:state st} '[:list/one])
```

You should see that `"Mary"` has her points incremented.

But wait! She also appears in `:list/two`, we better check that
as well:

```clj
(parser {:state st} '[:list/two])
```

You should see that her score is correct in both places.

This is a lot of power for little effort. Normalization is a strategy
directly lifted from Relay and Falcor.

### Something to look at

Modify `Person` to the following:

```clj
(defui Person
  static om/Ident
  (ident [this {:keys [name]}]
    [:person/by-name name])
  static om/IQuery
  (query [this]
    '[:name :points :age])
  Object
  (render [this]
    (println "Render Person" (-> this om/props :name))
    (let [{:keys [points name foo] :as props} (om/props this)]
      (dom/li nil
        (dom/label nil (str name ", points: " points))
        (dom/button
          #js {:onClick
               (fn [e]
                 (om/transact! this
                   `[(points/increment ~props)]))}
          "+")
        (dom/button
          #js {:onClick
               (fn [e]
                 (om/transact! this
                   `[(points/decrement ~props)]))}
          "-")))))

(def person (om/factory Person {:keyfn :name}))
```
          
This should look pretty straighforward.

After `Person` add `ListView`:

```clj
(defui ListView
  Object
  (render [this]
    (println "Render ListView" (-> this om/path first))
    (let [list (om/props this)]
      (apply dom/ul nil
        (map person list)))))

(def list-view (om/factory ListView))
```

After `ListView` add `RootView`, the reconciler construction and kick off:

```clj
(defui RootView
  static om/IQuery
  (query [this]
    (let [subquery (om/get-query Person)]
      `[{:list/one ~subquery} {:list/two ~subquery}]))
  Object
  (render [this]
    (println "Render RootView")
    (let [{:keys [list/one list/two]} (om/props this)]
      (apply dom/div nil
        [(dom/h2 nil "List A")
         (list-view one)
         (dom/h2 nil "List B")
         (list-view two)]))))

(def reconciler
  (om/reconciler
    {:state  init-data
     :parser (om/parser {:read read :mutate mutate})}))

(om/add-root! reconciler
  RootView (gdom/getElement "app"))
```

You should now see a UI that you can interact with. You'll see that if
you change Mary's points in one list her points will change in the
other. But you already knew that since you tested this in the REPL.

## Minimal Updates

Open the Chrome Developer Console if it isn't already open. Notice
that every component prints when it renders. Notice that after the
initial render, components only re-render themselves on data changes -
this is a significant render optimization.

At the same time we haven't had to resort to caching via local state
or other shenanigans that would break time travel.

So you *can* have your cake and eat it too!

The full code for this tutorial follows.

## Appendix

The complete source for this tutorial.

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]))

(enable-console-print!)

(def init-data
  {:list/one [{:name "John" :points 0}
              {:name "Mary" :points 0}
              {:name "Bob"  :points 0}]
   :list/two [{:name "Mary" :points 0 :age 27}
              {:name "Gwen" :points 0}
              {:name "Jeff" :points 0}]})

;; -----------------------------------------------------------------------------
;; Parsing

(defmulti read om/dispatch)

(defn get-people [state key]
  (let [st @state]
    (into [] (map #(get-in st %)) (get st key))))

(defmethod read :list/one
  [{:keys [state] :as env} key params]
  {:value (get-people state key)})

(defmethod read :list/two
  [{:keys [state] :as env} key params]
  {:value (get-people state key)})

(defmulti mutate om/dispatch)

(defmethod mutate 'points/increment
  [{:keys [state]} _ {:keys [name]}]
  {:action
   (fn []
     (swap! state update-in
       [:person/by-name name :points]
       inc))})

(defmethod mutate 'points/decrement
  [{:keys [state]} _ {:keys [name]}]
  {:action
   (fn []
     (swap! state update-in
       [:person/by-name name :points]
       #(let [n (dec %)] (if (neg? n) 0 n))))})

;; -----------------------------------------------------------------------------
;; Components

(defui Person
  static om/Ident
  (ident [this {:keys [name]}]
    [:person/by-name name])
  static om/IQuery
  (query [this]
    '[:name :points :age])
  Object
  (render [this]
    (println "Render Person" (-> this om/props :name))
    (let [{:keys [points name foo] :as props} (om/props this)]
      (dom/li nil
        (dom/label nil (str name ", points: " points))
        (dom/button
          #js {:onClick
               (fn [e]
                 (om/transact! this
                   `[(points/increment ~props)]))}
          "+")
        (dom/button
          #js {:onClick
               (fn [e]
                 (om/transact! this
                   `[(points/decrement ~props)]))}
          "-")))))

(def person (om/factory Person {:keyfn :name}))

(defui ListView
  Object
  (render [this]
    (println "Render ListView" (-> this om/path first))
    (let [list (om/props this)]
      (apply dom/ul nil
        (map person list)))))

(def list-view (om/factory ListView))

(defui RootView
  static om/IQuery
  (query [this]
    (let [subquery (om/get-query Person)]
      `[{:list/one ~subquery} {:list/two ~subquery}]))
  Object
  (render [this]
    (println "Render RootView")
    (let [{:keys [list/one list/two]} (om/props this)]
      (apply dom/div nil
        [(dom/h2 nil "List A")
         (list-view one)
         (dom/h2 nil "List B")
         (list-view two)]))))

(def reconciler
  (om/reconciler
    {:state  init-data
     :parser (om/parser {:read read :mutate mutate})}))

(om/add-root! reconciler
  RootView (gdom/getElement "app"))
```
