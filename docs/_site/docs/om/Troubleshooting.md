### Why won't my component unmount?

### Async rendering side effects
Om batches updates and executes them asynchronously on requestAnimationFrame. One consequence of that is that updates will not happen on the same cycle as the state change that triggered them. This may cause certain API that expect that (e.g. event.preventDefault()) to fail. You should be able to handle such edge cases by finding a workaround within Om, or alternatively reaching into React.

### Why do events lose all data when put on a core.async channel or logged to the console?

### React's SyntheticEvents 
React's [SyntheticEvent documentation](http://facebook.github.io/react/docs/events.html) describes SyntheticEvents as "a cross-browser wrapper around the browser's native event". The implementation of SyntheticEvent uses the [object pool pattern](http://en.wikipedia.org/wiki/Object_pool_pattern) to reduce the frequency of garbage collection. When an event is fired in React, an object from the event pool is used to represent the event. After each event loop, the objects backing these events are cleared of their information and released back to the React's object pool to be used again. A SyntheticEvent can be made to persist across event loops by calling persist on the SyntheticEvent object(e.g. event.persist()).

SyntheticEvents that are logged to the console or put into core.async channels without having persist() called on them will have no information associated with them when accessed. Event objects should have persist() called on them within the callback handler or event data should be accessed within the callback handler to avoid this.

### "No protocol method ITransact.-transact! defined for type ..."

### By default, only maps and vectors are automatically turned into cursors.

If you try to call `om/transact!` or `om/update!` on a cursor, but this cursor is not storing a map or a vector (for example, a primitive, set, list or seq), then you will see this error. There are two common cases where this happens:

1. When you have a cursor like `{:foo "foo"}` and pass `(:foo cursor)` to a component, if that component then tries to transact the cursor, it will be operating on a string and not a cursor and this error will occur.
1. When you set a part of a cursor to a seq (rather than a map or vector). A common mistake is forgetting to call vec after manipulating a vector: `(om/update! my-cursor [:path] (map some-fn [1 2 3]))` should instead be `(om/update! my-cursor [:path] (mapv some-fn [1 2 3]))`
