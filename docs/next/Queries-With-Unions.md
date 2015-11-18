> **DO NOT EDIT THIS PAGE**: This page is under heavy active
> development.

## Introduction

The [[Quick Start (om.next)]] introduces Om Next
fundamentals. [[Components, Identity & Normalization]] covers
intermediate concepts. Both of these are necessary reading before
proceeding.

### Queries With Unions

Many useful user interfaces present heterogeneous content in the form
of a scrollable list - "Dashboard" or "Stream" view. However, so far the joins
we have represented have only shown homogeneous content. In the
following section we'll show how Om Next easily supports this
popular form of presentation.

At this point we assume you are now comfortable with Figwheel
configuration and will not cover setup.

## Heterogeneous Data

Lets assume we are building a live dashboard which will host many
different kinds of widgets. We'll see that this case does
not actually present any challenges.

Assume our data looks something like the following:

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]
            [cljs.pprint :as pprint]))

(enable-console-print!)

(def init-data
  {:dashboard/items
   [{:id 0 :type :dashboard/post
     :author "Laura Smith"
     :title "A Post!"
     :content "Lorem ipsum dolor sit amet, quem atomorum te quo"
     :favorites 0}
    {:id 1 :type :dashboard/photo
     :title "A Photo!"
     :image "photo.jpg"
     :caption "Lorem ipsum"
     :favorites 0}
    {:id 2 :type :dashboard/post
     :author "Jim Jacobs"
     :title "Another Post!"
     :content "Lorem ipsum dolor sit amet, quem atomorum te quo"
     :favorites 0}
    {:id 3 :type :dashboard/graphic
     :title "Charts and Stufff!"
     :image "chart.jpg"
     :favorites 0}
    {:id 4 :type :dashboard/post
     :author "May Fields"
     :title "Yet Another Post!"
     :content "Lorem ipsum dolor sit amet, quem atomorum te quo"
     :favorites 0}]})
```

As before the very first step we're interested in is writing some
queries. We won't proceed until we can get the data into a desirable
form.

### The Queries

Add the following code to your file:

```clj
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
    [:id :type :title :image]))

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
```

Of these the only one that should be at all surprising is
`DashboardItem`. Instead of returning a vector it returns a map.

Lets use `om.next/get-query` to examine what the total query looks
like via the Figwheel REPL:

```clj
(in-ns 'om.tutorial.core)
(om/get-query Dashboard)
;; =>
;; [{:dashboard/items
;;   {:dashboard/post    [:id :type :title :author :content :favorites],
;;    :dashboard/photo   [:id :type :title :image :caption :favorites],
;;    :dashboard/graphic [:id :type :title :image :favorites]}}]
```

It should now be clear that joins that contain heterogeneous information
are represented as maps with entries for each possible subquery.

Also note that `DashboardItem`'s `Ident` implementation is computed from
the props of the subitem it will present.

Finally notice that because queries are just regular ClojureScript
data we can easily we get some extra bit of information. We tack on
`:favorites` to each subquery. We will be writing shared favoriting
logic shortly.

Before we do that let's verify that we can normalize the data.

### Normalization

At the REPL try the following:

```clj
(om/tree->db Dashboard init-data true)
```

You should see the following data printed back out:

```clj
{:dashboard/items
 [[:dashboard/post 0]
  [:dashboard/photo 1]
  [:dashboard/post 2]
  [:dashboard/graphic 3]
  [:dashboard/post 4]],
 :dashboard/post
 {0
  {:id 0,
   :type :dashboard/post,
   :title "A Post!",
   :author "Laura Smith",
   :content "Lorem ipsum dolor sit amet, quem atomorum te quo",
   :favorites 0},
  2
  {:id 2,
   :type :dashboard/post,
   :title "Another Post!",
   :author "Jim Jacobs",
   :content "Lorem ipsum dolor sit amet, quem atomorum te quo",
   :favorites 0},
  4
  {:id 4,
   :type :dashboard/post,
   :title "Yet Another Post!",
   :author "May Fields",
   :content "Lorem ipsum dolor sit amet, quem atomorum te quo",
   :favorites 0}},
 :dashboard/photo
 {1
  {:id 1,
   :type :dashboard/photo,
   :title "A Photo!",
   :image "photo.jpg",
   :caption "Lorem ipsum",
   :favorites 0}},
 :dashboard/graphic
 {3
  {:id 3,
   :type :dashboard/graphic,
   :title "Charts and Stufff!", 
   :image "chart.jpg", 
   :favorites 0}}}
```

Looks great!

There really isn't anything more to do that we haven't already seen
before. We add a transaction so we can favorite things in the
timeline. But that's about it!

## Putting It All Together

The complete code follows:

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]
            [cljs.pprint :as pprint]))

(enable-console-print!)

(def init-data
  {:dashboard/items
   [{:id 0 :type :dashboard/post
     :author "Laura Smith"
     :title "A Post!"
     :content "Lorem ipsum dolor sit amet, quem atomorum te quo"
     :favorites 0}
    {:id 1 :type :dashboard/photo
     :title "A Photo!"
     :image "photo.jpg"
     :caption "Lorem ipsum"
     :favorites 0}
    {:id 2 :type :dashboard/post
     :author "Jim Jacobs"
     :title "Another Post!"
     :content "Lorem ipsum dolor sit amet, quem atomorum te quo"
     :favorites 0}
    {:id 3 :type :dashboard/graphic
     :title "Charts and Stufff!"
     :image "chart.jpg"
     :favorites 0}
    {:id 4 :type :dashboard/post
     :author "May Fields"
     :title "Yet Another Post!"
     :content "Lorem ipsum dolor sit amet, quem atomorum te quo"
     :favorites 0}]})

(defui Post
  static om/IQuery
  (query [this]
    [:id :type :title :author :content])
  Object
  (render [this]
    (let [{:keys [title author content] :as props} (om/props this)]
      (dom/div nil
        (dom/h3 nil title)
        (dom/h4 nil author)
        (dom/p nil content)))))

(def post (om/factory Post))

(defui Photo
  static om/IQuery
  (query [this]
    [:id :type :title :image :caption])
  Object
  (render [this]
    (let [{:keys [title image caption]} (om/props this)]
      (dom/div nil
        (dom/h3 nil (str "Photo: " title))
        (dom/div nil image)
        (dom/p nil "Caption: ")))))

(def photo (om/factory Photo))

(defui Graphic
  static om/IQuery
  (query [this]
    [:id :type :title :image])
  Object
  (render [this]
    (let [{:keys [title image]} (om/props this)]
      (dom/div nil
        (dom/h3 nil (str "Graphic: " title))
        (dom/div nil image)))))

(def graphic (om/factory Graphic))

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
         (om/get-query Graphic)])))
  Object
  (render [this]
    (let [{:keys [id type favorites] :as props} (om/props this)]
      (dom/li
        #js {:style #js {:padding 10 :borderBottom "1px solid black"}}
        (dom/div nil
          (({:dashboard/post    post
             :dashboard/photo   photo
             :dashboard/graphic graphic} type)
            (om/props this)))
        (dom/div nil
          (dom/p nil (str "Favorites: " favorites))
          (dom/button
            #js {:onClick
                 (fn [e]
                   (om/transact! this
                     `[(dashboard/favorite {:ref [~type ~id]})]))}
            "Favorite!"))))))

(def dashboard-item (om/factory DashboardItem))

(defui Dashboard
  static om/IQuery
  (query [this]
    [{:dashboard/items (om/get-query DashboardItem)}])
  Object
  (render [this]
    (let [{:keys [dashboard/items]} (om/props this)]
      (apply dom/ul
        #js {:style #js {:padding 0}}
        (map dashboard-item items)))))

(defmulti read om/dispatch)

(defmethod read :dashboard/items
  [{:keys [state]} k _]
  (let [st @state]
    {:value (into [] (map #(get-in st %)) (get st k))}))

(defmulti mutate om/dispatch)

(defmethod mutate 'dashboard/favorite
  [{:keys [state]} k {:keys [ref]}]
  {:action
   (fn []
     (swap! state update-in (conj ref :favorites) inc))})

(def reconciler
  (om/reconciler
    {:state  init-data
     :parser (om/parser {:read read :mutate mutate})}))

(om/add-root! reconciler Dashboard (gdom/getElement "app"))
```

