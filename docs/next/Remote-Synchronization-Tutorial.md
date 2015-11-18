> **DO NOT EDIT THIS PAGE**: This page is under heavy active
> development.

## Introduction

The [[Quick Start (om.next)]] introduces Om Next
fundamentals. [[Components, Identity & Normalization]] covers
intermediate concepts. Both of these are necessary reading before
proceeding.

### HTTP Caching

While Relay & Falcor both provide good overall caching stories,
neither make any attempt to solve the HTTP caching problem. While this
may be fine when operating at scale with a large pool of performance
experts to draw from, smaller teams must be able leverage standard
performance advice. HTTP caching is one of the most tried and true
techniques for enhancing web application performance and Om Next
supports it out of the box.

### Remotes

Om Next parsing works in two modes. The first which we've already seen
takes **query expressions** and turns them into a data tree ready to
be fed into component props. However we haven't yet demonstrated that
Om Next *also* supports a parsing mode to derive new **query
expressions** that cannot be resolved on the client. This feature
enables transparent synchronization with a remote service. We'll see
this synchronization component in detail later, but first we'll examine how
this architecture permits us to recover HTTP caching.

At this point we assume you are now comfortable with Figwheel
configuration and will not cover setup. However note that this
tutorial requires some different dependencies, your `project.clj`
should look like the following:

```clj
(defproject om-tutorial "0.1.0-SNAPSHOT"
  :description "My first Om program!"
  :dependencies [[org.clojure/clojure "1.7.0"]
                 [org.clojure/clojurescript "1.7.170"]
                 [org.omcljs/om "1.0.0-alpha21"]
                 [com.cognitect/transit-cljs "0.8.225"]
                 [figwheel-sidecar "0.5.0-SNAPSHOT" :scope "test"]])
```

## Message Forwarding

> *The big idea is "messaging" - Alan Kay*

Imagine we are designing a live dashboard. This live dashboard will
attract a large number of users. Most of these users will simply read
the stream and will not interact with it. For this reason we would
like to serve these users with a cached response. In a traditional
application we could imagine serving these users via a JSON payload
from the following URL:

```
/dashboard/items
```

If a user is logged in, after rendering the unadorned dashboard we may
make a second request for a user's dynamic modifications to the stream with
a different end point:

```
/dashboard/user/id/favorites
```

We would then modify the UI views to reflect this merged state.

The problem is that in Om Next we represent the query as a recursive data
structure not a simple URL. So how can we recover the benefits of HTTP
caching?

As it turns out we can easily filter out the **static** part of the
message from the **dynamic** part of the message, the **static** part
of the message can be trivially hashed, and we can make two requests
as we did before.

Let's see how! We'll be using the same data as
[[Queries With Unions]].

### Remotes

In this part of the tutorial we're not at all concerned with what the
UI looks like so we're not providing those parts:

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [goog.crypt :as gcrypt]
            [cognitect.transit :as t]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]
            [cljs.pprint :as pprint])
  (:import [goog.crypt Sha256]))

(enable-console-print!)

(def init-data
  {:dashboard/items
   [{:id 0 :type :dashboard/post
     :author "Laura Smith"
     :title "A Post!"
     :content "Lorem ipsum dolor sit amet, quem atomorum te quo"}
    {:id 1 :type :dashboard/photo
     :title "A Photo!"
     :image "photo.jpg"
     :caption "Lorem ipsum"}
    {:id 2 :type :dashboard/post
     :author "Jim Jacobs"
     :title "Another Post!"
     :content "Lorem ipsum dolor sit amet, quem atomorum te quo"}
    {:id 3 :type :dashboard/graphic
     :title "Charts and Stufff!"
     :image "chart.jpg"}
    {:id 4 :type :dashboard/post
     :author "May Fields"
     :title "Yet Another Post!"
     :content "Lorem ipsum dolor sit amet, quem atomorum te quo"}]})

(defui Post
  static om/IQuery
  (query [this]
    [:id :type :title :author :content]))

(defui Photo
  static om/IQuery
  (query [this]
    [:id :type :title :image :caption]))

(defui Graphic
  static om/IQuery
  (query [this]
    [:id :type :image]))

(defui DashboardItem
  static om/Ident
  (ident [this {:keys [id type]}]
    [type id])
  static om/IQuery
  (query [this]
    (zipmap
      [:dashboard/post :dashboard/photo :dashboard/graphic]
      (map #(conj % :favorites)
        [(om/get-query Post)
         (om/get-query Photo)
         (om/get-query Graphic)]))))

(defui Dashboard
  static om/IQuery
  (query [this]
    [{:dashboard/items (om/get-query DashboardItem)}]))

(defmulti read om/dispatch)
```

So far so good. Let's take a look at our read method. It does a bit
more than we've encountered before. We've also added some simple
helpers.

The result of this read function returns not only the value but also
the query for a variety of **remotes**. Later we'll see that when
constructing the reconciler we'll pass along a list of **remotes**.
In this case we have our local state provided by `:value`, the
`:dynamic` portion of the query will be passed along to a remote
service as well as the `:static` portion. The only difference is that
we'll use the `:static` portion to compute a specific URL.

*There is nothing special about these remotes, you can call them
 whatever you want and list as many as you like*.

```clj
(defmethod read :dashboard/items
  [{:keys [state ast]} k _]
  (let [st @state]
    {:value   (into [] (map #(get-in st %)) (get st k))
     :dynamic (update-in ast [:query]
                #(->> (for [[k _] %]
                        [k [:favorites]])
                  (into {})))
     :static  (update-in ast [:query]
                #(->> (for [[k v] %]
                        [k (into [] (remove #{:favorites}) v)])
                  (into {})))}))

(defn sha-256 [s]
  (let [sha (Sha256.)
        _   (.update sha s)]
    (gcrypt/byteArrayToHex (.digest sha))))

(def p (om/parser {:read read}))
(def w (t/writer :json))
(def app-state (atom (om/tree->db Dashboard init-data true)))
```

Notice that all read (and mutation) functions receive a simple AST
(Abstract Syntax Tree) representing the current portion of the
query. In our map besides returning `:value` we can also return
modified query fragments to produce different queries.

Let's see this in action at the REPL:

```clj
(p {:state app-state} (om/get-query Dashboard) :static)
```

The result should be familiar except we're missing the `:favorites`
key. If you examine the `:static` key in the read function above it
should now be clear what's going on. We're changing each query
expression in the union query.

Try the following:

```clj
(let [query (p {:state app-state} (om/get-query Dashboard) :static)
      json  (t/write w query)
      hash  (.substring (sha-256 json) 0 16)]
    (str "/api/" hash))
;; "/api/02e397cc1447d688"
```

You should see the exact same URL on your machine.

When we write our `send` function and we request the `:static` portion
of the message we'll use this convenient URL.

Now what about the dynamic query?

```clj
(p {:state app-state} (om/get-query Dashboard) :dynamic)
;; [{:dashboard/items
;;   {:dashboard/post    [:favorites],
;;    :dashboard/photo   [:favorites], 
;;    :dashboard/graphic [:favorites]}}]
```

There's our dynamic query.

Using the same parsing infrastructure we can present a merged view of
local state, HTTP cached state, and the user's dynamic query without
issue.
