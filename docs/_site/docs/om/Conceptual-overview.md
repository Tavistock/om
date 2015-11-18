What follows is an Om conceptual guide.

## Tangible Data

Object Oriented programming as a paradigm has many real benefits but
one of the worst plagues it has inflicted on programming culture is
obscuring data. Functional programming is not a silver bullet but its
emphasis on unadorned data is a guiding light.

No models.

Rather than introducing a middle man, Om allows programmers to build
user interfaces over unadorned data. Still, structuring UI components
benefits greatly from the good bits of Object Oriented thinking and Om
leaves these be.

## Components

Om offers an approach more or less analogous to
React's, but we do not actually pass raw React props or states to
implementers of the life cycle protocols. Instead we pass immutable
values in both cases.

While it may not seem so at first, it's useful to preserve something
like React's component local state for two reasons. Often you do not
want to pollute the original data with transient application state
information. Editing a text field is a good example of this. The other
useful aspect of being able to set component local state is that it's
always guaranteed to be up-to-date. This is not true for application
state since Om renders on `requestAnimationFrame`, and application
state information is only guaranteed to be consistent during the
render phase. Thus event handlers must ask for an up-to-date
view of the application state.

## Application State

`om.core/transact!` is used to transition the application
state. The transition function should only rely on information
obtained by `deref`-ing a cursor, `om.core/get-state`, `om.core/transact!`,
or `om.core/update!`.

## [[Cursors]]

Om components, like React components, take props. In Om components the
props are actually a *cursor* into the app state. Cursors are
conceptually related to functional lenses and zippers. Don't be
afraid–it just means that Om component props *internally* maintain
path information to determine their location in the app state. You can
interact with them with many of the standard Clojure APIs. You can
even make cursors out of JavaScript natives like numbers and strings!
Cursors means Om components are freed from knowing or caring where in the
app state their data comes from while still having the ability to update it
thus preserving component modularity.
