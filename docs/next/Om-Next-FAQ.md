### How can I implement optimistic commits?

Write a mutation that both changes the local state and includes a
remote key.

```clj
(defmethod my-mutate 'change/it!
  [{:keys [state]} key params]
  {:remote true
   :action
   (fn []
     (swap! state [:count] inc))})
```

### How can I delay loads?

Write some conditional logic in your read function.

```clj
(defmethod my-read :delayed/key
  [{:keys [state]} key params]
  (let [[k v :as e] (find @state key)]
    (when-not (nil? e)
      (if (= v :loading)
        {:remote true}
        {:value v}))))
```

### How do I communicate with the server?

Provide a function to the `:send` key of the reconciler's parameter map (along with `:state` and `:parser`). This will be a function which takes two parameters: the EDN of the query expression fragment that will be passed to the server, and a callback to handle the response. Om will provide the callback, your function just needs to make the request and ensure that the callback receives, as its one argument, data in the format of an EDN result of a query expression -- for example by simply reading a transit-json response from the server back into EDN. Example:

```clojure
(defn transit-post [url]
  (fn [{:keys [remote]} cb]
    (.send XhrIo url
      (fn [e]
        (this-as this
          (cb (transit/read (om/reader) (.getResponseText this)))))
      "POST" (transit/write (om/writer) remote)
      #js {"Content-Type" "application/transit+json"})))
```

### Why is my component not rerendered after `transact!`?

`transact!` takes an initiating component and a query expression consisting of mutations and any number of reads. If the initiating component (or any other component) is not rerendered with the mutated data, the first question to ask yourself is: does the transaction affect keys that are not part of the initiating component's query? After the transaction, Om implicitly updates the initiating component based on its query. However, if keys outside the component are affected, they need to be added to the `transact!` query expression explicitly. Assume a component `Foo` has a query `[:foo/bar :foo/baz]` and calls ```(om/transact this `[(app/do-something)])```. If `app/do-something` affects `:foo/bar` or `:foo/baz`, the component will be rerendered with the new data automatically. If it changes`:something/else` however, the call has to be changed to ```(om/transact this `[(app/do-something) :something/else])```.
