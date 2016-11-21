* Date: November 18th, 2016
* Title: A minimal production loader

# Summary

Minimize the amount of Loader code that runs in production to only what is necessary to support module resolution, circular dependencies, and dynamic loading.

# Motivation

I did a [performance audit](https://github.com/stealjs/stealjs/issues/20) on Steal to see what shortcomings we currently have. The goal is to achieve 3 second load times on 3G with a mid-range phone.

Based on this audit there are the falling problems with *steal.production.js*:

* The total budget of JavaScript is 100k. steal.production.js accounts for 21k, so a little over 1/5th of the total JavaScript budget. This doesn't include the npm plugin which is also packed into the main bundle.
* steal.production.js takes 176ms to execute on a mid-range phone.

The goal of this change will be to:

* Limit the amount of Steal code that needs to run in production to under 10k. I need to compare to other bundlers to see if this is achievable or not, but my guess is that it is.
* Support circular dependencies.
* Support progressive loading.
* Support loading as async script tags so that multiple bundles can be loaded in parallel:

```html
<script async src="./path/to/prod.js"></script>
<script async src="./path/to/bundle-a.js"></script>
<script async src="./path/to/bundle-b.js"></script>
```

# Impact

This will be small change to the affecting repositories and most work will be done in a new project. This would reflect a *minor* version change as it would introduce a new feature with no backwards compatibility concerns.  In the future this loader could become the default that steal-tools uses, which would require a *major* version change but there are no plans to do that until it has been used in the wild for a bit behind a flag.

The following projects *will* be affected:

* **transpile** - In the below design I lay out a new bundling format that this new loader will use. Transpile will need to be updated in order to transpile to this format.
* **steal-tools** - Will need to be updated to accept a flag that will use this new loader.

# Design

> Note that API names here aren't important and can change, I'm using steal as the example.

Since the goal of this RFC is a loader that is *as small as possible* it will only support the minimal amount of configuration needed to for progressive loading bundles.

## Module ids replace module names

**transpile** will transpile to a different format, let's call it **_steal_** for now, and will omit something like:

```js
steal([0, 7, 12, 34], function(exports, bar, baz, qux){

});
```

Notice that instead of module names there are integers. This is to save space by omitting the long strings. During the build each module name will be given an integer id.

Also notice that the id of **0** is a special `exports` module. This mirrors the `exports` in AMD and allows for compatibility with ES module circular dependencies.

## bundles configuration

Steal has [bundles](http://stealjs.com/docs/System.bundles.html) configuration that looks something like this:

```js
System.bundles = {
  "bundles/donejs-chat/home": [
    "can-map@3.0.0#bubble",
    "foo-bar@1.0.0#baz"
  ]
};
```

To resolve a moduleName to a bundle Steal needs to loop over each of these keys and perform an `indexOf` to see if the moduleName is part of the bundle.

We can make this more efficient by having the map be module ids to bundle ids. 

```js
steal.bundles = {
  3: 144,
  34: 144,
  22: 145
};
steal.paths = {
  144: "bundles/donejs-chat/home.js"
};
```

## dynamic module name configuration

To support progressive loading we need to be able to map module identifiers to the module id. We can support config like:

```js
steal.map = {
  "./components/tabs": 63
};
steal.bundles = {
  63: 145
};
steal.paths = {
  145: "bundles/app/main.js"
};
```

## baseURL configuration

To make loading the bundles work we need some type of baseURL configuration.

```js
steal.baseURL = "./dist";
```

## async scripts

A key goal is allowing `<script async>` with multiple script tags for each bundle (and for the loader itself). This will look something like:

```js
<script src="dist/loader.js" async></script>
<script src="dist/bundles/app/main.js" async></script>
<script src="dist/bundles/app/bundle-a.js" async></script>
<script src="dist/bundles/app/bundle-b.js" async></script>
```

Since any of the bundles might be executed before the loader they need to be able to handle that. I propose that they include a small shim which is essentially this:

```js
if(typeof steal === 'undefined') {
  steal = function(deps, cb) { steal.waiting.push([deps, cb]); });
  steal.waiting = [];
}
```

When the loader loads it will check for this `steal.waiting` array and register each module in its registry.

When the loader itself executes it will be configured to automatically load the `main`. It will resolve the bundle it belongs to using the `bundles` config and add a `<script async>` tag, if needed, using the `paths` config. It will check if there already is a script tag on the page for that address, and if so it will know not to do anything.
