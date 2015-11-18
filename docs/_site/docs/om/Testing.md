### Testing with clojurescript.test

#### Getting started

If you're planning to run tests with `phantomjs`, it's important to include a polyfill for `Function.prototype.bind` otherwise your tests may fail to run with an error such as:

```
Running ClojureScript test: phantom
TypeError: 'undefined' is not a function (evaluating 'RegExp.prototype.test.bind(
    /^(data|aria)-[a-z_][a-z\d_.\-]*$/
  )')
```

Mozilla's developer website has code available [here](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind#Compatibility) which you copy and save to a location in your project directory. You may also want check this file into source control so others can run your tests without issues.

After you've added this code to your project, you'll need to update your `:test-commands` configuration to look something like the following.

```clj
:test-commands {"phantom" ["phantomjs" :runner "path/to/polyfill.js" "target/generated/js/test.js"]
```

Alternatively you can use [`slimerjs`](http://slimerjs.org/) which uses a more modern JavaScript runtime.