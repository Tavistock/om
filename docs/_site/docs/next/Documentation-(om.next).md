> WARNING: This page documents alpha software.

### Table of Contents

#### om.dom

* [node](#node)
* [render-to-str](#render-to-str)

#### om.next

* **Macros**
  * [defui](#defui)
* **Query & Identity**
  * [Ident](#ident)
  * [IQuery](#iquery)
  * [IQueryParams](#iqueryparams)
  * [get-query](#get-query)
  * [get-unbound-query](#get-unbound-query)
  * [get-params](#get-params)
* **Components**
  * [factory](#factory)
  * [component?](#component)
  * [react-key](#react-key)
  * [react-type](#react-type)
  * [props](#props)
  * [computed](#computed)
  * [get-computed](#get-computed)
  * [get-ident](#get-ident)
  * [set-state!](#set-state)
  * [update-state!](#update-state)
  * [get-state](#get-state)
  * [get-rendered-state](#get-rendered-state)
  * [set-query!](#set-query)
  * [set-params!](#set-params)
  * [mounted?](#mounted)
  * [react-ref](#react-ref)
  * [subquery](#subquery)
* **Parsing & Mutation**
  * [parser](#parser)
  * [dispatch](#dispatch)
  * [transact!](#transact)
* **Indexing**
  * [ref->components](#ref-components)
  * [class->any](#class-any)
  * [ref->any](#ref-any)
  * [class->any](#class-any)
* **Normalization / Denormalization (Default Database)**
  * [tree->db](#tree-db)
  * [db->tree](#db-tree)
* **Reconciler**
  * [reconciler](#reconciler)
  * [reconciler?](#reconciler-2)
  * [add-root!](#add-root)
  * [remove-root!](#remove-root)
  * [merge!](#merge)
  * [app-state](#app-state)
  * [from-history](#from-history)

# om.dom

### node

```clj
(dom/node some-component)
```

Return the DOM node associated with a component.

### render-to-str

```clj
(defn render-to-str [c]
  ...)
```

Equivalent to [`React.renderComponentToString`](http://facebook.github.io/react/docs/top-level-api.html#react.rendercomponenttostring). For example:

```clj
(dom/render-to-str (om/build some-widget data))
```

# om.next

## Macros

### defui

```clj
(defui MyComponent
  Object
  (componentDidMount [this]
                     (.log js/console "did mount"))
  (render [this]
          (div nil "Hello, world!")))

```

Macro for defining components. `defui` creates a JavaScript class that
inherits from `React.Component`. `defui` is like `deftype` but there
is no support for defining fields. In addition there is special
handling of the "static" protocols `om.next/Ident`, `om.next/IQuery`
and `om.next/IQueryParams`. The React component specs and lifecycle methods are available at the [react docs](https://facebook.github.io/react/docs/component-specs.html). 

## Query & Identity

### Ident

```clj
(defui MyComponent
  static om/Ident
  (ident [this props]
    [:some/key (:some/id props)])
```

A protocol for identity resolution. This protocol is used to solve two
problems. First, in the case where initial state or state novelty is
supplied in a denormalized form. Second, when making an association
from a logical entity to multiple component instances. The first case
simplifies the problem of updating data while the latter simplifies
keeping multiple views of the same data in sync.

### IQuery

```clj
(defui MyComponent
  static om/IQuery
  (query [this]
    [:prop-a :prop-b])
  Object
  (render [this]
    (div nil "Hello, world!")))
```

A protocol for declaring queries. This method should always return
a vector. The query may include quoted symbols that start with `?`. If
bindings for these query variables are supplied via `IQueryParams`
they will replace the symbols.

### IQueryParams

```clj
(defui MyComponent
  static om/IQueryParams
  (params [this]
    {:start 0 :end 10})
  static om/IQuery
  (query [this]
    '[(:list/items {:start ?start :end ?end})])
  Object
  (render [this]
    (div nil "Hello, world!")))
```

Define params to bind to the query. Should be a map of keywords that
match query variables present in the query.

### get-query

```clj
(om.next/get-query MyComponent)
```

Returns the bound query for the component.

### get-unbound-query

### get-params

## Components

### factory

```clj
(om.next/factory MyComponent
  {:keyfn :id :validator my-validator})
```

Create a factory function from an Om component. Can optionally supply
`:keyfn` - this should produce the React `key` property from the
component props. Can also supply `:validator`, a function which should
`assert` that the props are valid.

### component?

```clj
(om.next/component? 1) ;; false
```

Returns true if the argument is a component.

### react-key

```clj
(om.next/react-key some-component)
```

Returns the React `key`.

### react-type

```clj
(om.next/react-type some-component)
```

Returns the component constructor. Works even if the component has not
been mounted.

### props

```clj
(defui MyComponent
  Object
  (render [this]
    (let [{:keys [foo bar]} (om/props this)]
      ;; ...
      )))
```

Get the immutable props for an Om component.

### computed

```clj
(some-widget
  (om/computed props
    {:delete (fn [e] ...)
     :update (fn [e] ...)}))
```

Add computed information to props. Useful for passing down computed
information like event handling callbacks and client only (non-remote)
state. The first argument should be props, the second argument a map.

### get-computed

```clj
(let [{:keys [delete update]} (om/get-computed this)]
  ...)
```

Return the computed properties on props. The first argument can be
props or a component. The second argument an be a keyword or
sequential collection of keys for deeply accessing the computed
properties.

### set-state!

```clj
(om.next/set-state! some-component {:foo :bar})
```

Set component local state. Will schedule the component for re-render.

### get-state

```clj
(om.next/get-state some-component)
```

Return the component local state. Will be the latest state, not the
rendered component state.

### get-rendered-state

```clj
(om.next/get-rendered-state some-component)
```

Return the rendered component local state.

### set-query!

```clj
(om.next/set-query! some-component [:foo :baz]})
```

Mutate the query of a component. Schedules the component for re-render.

### set-params!

```clj
(om.next/set-params! some-component {:start 5 :end 10})
```

Mutate the query params of a component. Schedules the component for re-render.

### mounted?

```clj
(om.next/mounted? some-component)
```

Returns true if the component is mounted.

### react-ref

```clj
(om.next/react-ref some-component :foo/widget)
```

Return a component associated with the supplied name - can be a
keyword or string.

### subquery

```clj
(om.next/subquery x subquery-ref subquery-class)
```

Once a component has mounted it may wish to use a instantiated child
as the source of the subquery. `subquery` will return the query
provided by the component associated with `subquery-ref` when
mounted. Otherwise it will fallback on the query provided statically
by `subquery-class`.

## Parsing & Mutations

### parser

```clj
(om.next/parser {:read read-fn :mutate mutate-fn})
```

Construct a parser from the supplied configuration map. The map should
only have two keys:

* `:read` - a function of three arguments `[env key params]` that
  should return a valid parse result map. This map can contain a
  `:value` entry along with remote entries. If `:value` is supplied it
  will be used to rewrite a value in the resulting tree. Remote
  query entries must be query expression AST fragments that correspond
  to the `:remotes` specified to the reconciler.
* `:mutate` - a function of three arguments `[env key params]` that
  should return a valid parse mutation result map. This map should
  contain a `:value` and an `:action` entry. `:value` is an optional
  hint at keys affected by the mutation; it has no effect on rerendering
  and should only contain keys valid for `:read` functions. The value
  of `:action` should be a function of zero arguments that applies the
  requested mutation.

Returns a function of up to three arguments. The first argument should
be the `env` map. The second argument should be the **query
expression**. The final optional argument sets the parse mode to
remote, defaults to `false`.

### dispatch

A helper function for writing `read` and `mutate` multimethods that
dispatch on `key`.

```clj
(defmethod mutate om/dispatch)

(defmethod mutate `do/something ...)
```

### transact!

```clj
(om.next/transact! some-component
  `[(todo/update `{:title "Get Milk!"})])
```

Transition the application state. `transact!` takes two arguments.
The first argument may be either a component instance or a reconciler.
The second argument is a **query expression** that includes mutation.

## Indexing

### ref->components

```clj
(om.next/ref->components reconciler [todo/by-id 0])
```

A development time helper. Given a reference return all the components
that match. The reference can be a keyword or an ident. If keyword
will return all components with the given property.

### ref->any

```clj
(om.next/ref->any reconciler [todo/by-id 0])
```

A development time helper. Given a reference return the first component
that matches. The reference can be a keyword or an ident. If keyword
will return any component with the given property.

### class->any

```clj
(om.next/class->any reconciler SomeClass)
```

A development time helper. Given a class return a matching component.

## Normalization / Denormalization (Default Database)

The following operations are only needed if you intend to leverage the
default database format. If you are using a custom client side
in-memory database like DataScript, the following operations are
unnecessary.

### tree->db

```clj
(om.next/tree->db SomeComponent some-data)
```

Given a component (or query expression) and some data, normalize the
data into the default databaes format according to the query. All
nodes that can be mapped via Ident implementations will be replaced
with the ident as link. The original node data will be merged into
tables indexed by ident.

Can pass optional third argument to merge the ident indexed tables back
into the result. Otherwise must be accessed via `meta` on the result.

### db->tree

```clj
(om.next/db->tree query some-data app-state-db)
```

Given a query expression, some data in the default database format,
some application state data in the default database format,
denormalize it. This will replace all ident link nodes with their
actual data recursively. This is useful in parse in order to avoid
manually joining in nested relationships.

## Reconciler

### reconciler

```clj
(om.next/reconciler
  {:state app-state
   :parser my-parser})
```

Construct a reconciler based on the supplied configuration. The
configuration can be a map with the following keys:

* `:state` - the application state. If not an atom the reconciler will
normalize the data with the query supplied by the root component.
* `:parser` - a parser
* `:normalize` - whether to normalize the data provided by the
  user. If reconciler is given `IAtom` for `:state` this defaults to
  `false`.
* `:remotes` - a vector logical remotes present in the system. Remotes
  are simply user specified keywords.
* `:send` - a function of two arguments. The first argument will be a
  map of remotes and the pending message to be sent. The second
  argument is a callback that can be invoked as many times as
  necessary with the results from the various remotes. It's up to the
  user to specify the remote resolution order and whether data will be
  loaded piecemeal or all at once.
* `:root-render` - the root render function. Defaults to
  `ReactDOM.render`. Can be switched out according to context, i.e. React
  Native.
* `:root-unmount` - the root unmount function. Defaults to
  `ReactDOM.unmountComponentAtNode`. Can be switched out according to
  context, i.e. React Native.

### reconciler?

```clj
(om.next/reconciler? x)
```

Returns true if the argument is a reconciler instance, false otherwise.

### add-root!

```clj
(om.next/add-root! reconciler
  MyRootComponent (goog.dom/getElement "app"))
```

Add a DOM root for the reconciler to control. The first argument is a
reconciler, the second a root component, and the last, a DOM node target.

### remove-root!

```clj
(om.next/remove-root! reconciler (goog.dom/getElement "app"))
```

Given a reconciler and DOM target node, remove the DOM target from the
reconciler's control.

### merge!

```clj
(om.next/merge! reconciler novelty)
```

Merge in novelty into the reconciler. The novelty should be in the
correct shape (normalized or denormalized) depending the reconciler
configuration. Affected component will be queued for re-render.

### app-state

```clj
(om.next/app-state reconciler)
```

Return a reference to the atom managed by the reconciler. Useful when
reconciler state was initialized with plain data.

### from-history

```clj
(om.next/from-history reconciler
  #uuid "894e7a30-a5b8-4751-8bd6-a51aa122a919")
```

When mutating the application state via transactions or query
modifications, Om will log a UUID associated with an application state
before the change was applied. You can use the UUID to recover a
previous application state. Useful for "Fix & Continue" development
workflows.
