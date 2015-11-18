In this short tutorial we'll see how easy it is to integrate custom
data sources into Om Next applications.

## Setting up

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
mkdir om-datascript
cd om-datascript
```

Inside your project directory create a `project.clj`:

```clj
touch project.clj
```

Make it look like the following:

```clj
(defproject om-datascript "0.1.0-SNAPSHOT"
  :description "My first Om program!"
  :dependencies [[org.clojure/clojure "1.7.0"]
                 [org.clojure/clojurescript "1.7.170"]
                 [org.omcljs/om "1.0.0-alpha21"]
                 [datascript "0.13.1"]
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
     :compiler {:main 'om-datascript.core
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
        <title>Om + DataScript!</title>
    </head>
    <body>
        <div id="app"></div>
        <script src="js/main.js"></script>
    </body>
</html>
```

## Checkpoint

Create a file `src/om_datascript/core.cljs`:

```shell
mkdir -p src/om_datascript
touch src/om_datascript/core.cljs
```

Edit its contents to look like the following:

```clj
(ns om-datascript.core
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
page with the title "Om + DataScript!" visible on your browser tab.

Open the Chrome Developer Tools with the **View > Developer >
JavaScript Console** menu. In the JavaScript Console you should see
`Hello, world!` printed out.

Let's begin!

### Integrating DataScript

Make `src/om_datascript/core.cljs` look like the following:

```clj
(ns om-datascript.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]
            [datascript.core :as d]))

(enable-console-print!)

(def conn (d/create-conn {}))

(d/transact! conn
  [{:db/id -1
    :app/title "Hello, DataScript!"
    :app/count 0}])

(defmulti read om/dispatch)

(defmethod read :app/counter
  [{:keys [state query]} _ _]
  {:value (d/q '[:find [(pull ?e ?selector) ...]
                 :in $ ?selector
                 :where [?e :app/title]]
            (d/db state) query)})

(defmulti mutate om/dispatch)

(defmethod mutate 'app/increment
  [{:keys [state]} _ entity]
  {:value {:keys [:app/counter]}
   :action (fn [] (d/transact! state
                    [(update-in entity [:app/count] inc)]))})

(defui Counter
  static om/IQuery
  (query [this]
    [{:app/counter [:db/id :app/title :app/count]}])
  Object
  (render [this]
    (let [{:keys [app/title app/count] :as entity}
          (get-in (om/props this) [:app/counter 0])]
      (dom/div nil
        (dom/h2 nil title)
        (dom/span nil (str "Count: " count))
        (dom/button
          #js {:onClick
               (fn [e]
                 (om/transact! this
                   `[(app/increment ~entity)]))}
          "Click me!")))))

(def reconciler
  (om/reconciler
    {:state conn
     :parser (om/parser {:read read :mutate mutate})}))

(om/add-root! reconciler
  Counter (gdom/getElement "app"))
```

By now large portions of the program should look familiar to you. By
putting DataScript reads and mutations behind the parser, the `Counter`
component is freed from the precise details of the data source.

Instead of an atom the DataScript database is now our state
source. This is what we receive as the `:state` key in our `env`
parameter in our read and mutation functions.

Try clicking the button a few times. Once again you'll see logs
dropped into the Chrome JavaScript Console. Grab one of the UUIDs
and try the following at the Figwheel REPL (your UUID and DataScript
state will not be the same):

```clj
(in-ns 'om-datascript.core)
(om/from-history reconciler
  #uuid "c362be68-2867-46da-a8e5-5c107398e49d")
;; =>
;; #datascript/DB {:schema {},
;;                 :datoms [[1 :app/count 6 536870919]
;;                          [1 :app/title "Hello, DataScript!" 536870913]]}
```

That's all there is to it! Due to it's support for Datomic Pull
Syntax, DataScript is a natural fit for Om Next applications
especially if they do not or cannot have significant remote service
integration.
