* Date: 11/23/2016
* Title: New web-worker based back-end

# Summary

The goal of this RFC is to design a new architecture for Steal that's built on top of the browser standard of [&lt;script type=module&gt;](https://html.spec.whatwg.org/multipage/webappapis.html#module-script). Steal is currently based on older Loader spec that is never going to be implemented. It will require significant changes to Steal to support the new spec.

Along with supporting native modules, this also gives us the chance to better architect Steal for speedier development and better isolation of plugins.

# Terms

* **module script** will be used to refer to script tags that have a *module* attribute, &lt;script type=module&gt;. These are scripts that will load module code.
* **ES modules** will be used to refer to modules with import/export.
* **Dynamic module** will be used to describe any non-ES modules. These modules will need to be included in Steal's registry, not the brower's native one.

# Motivation

The motivation of this RFC is to make Steal compatible with native module loading. Steal will provide many of the same features it has today, so that it merely extends the capabilities of module scripts. This way you can use plain module scripts when you only need the bare-bones features they provide, and can seamlessly upgrade to use the features Steal provides when it is needed.

For example you might have a page like this:

```js
<script type="module" src="./components/tabs/tabs.js"></script>
```

And then when you need to load a CommonJS module you just add steal which will give you that:

```js
<script src="./node_modules/steal/steal.js"></script>
<script type="module" src="./components/tabs/tabs.js"></script>
```

# Impact

This would be a large change to Steal and would require a *major* version bump, so it is something that could not come until 2.0 of Steal. Given the scope and size of this change, I could see a prolonged period in beta, perhaps 6 months or more.

This change would require changes in:

* steal
* steal-tools (maybe, maybe not)
* steal-npm
* steal-conditional
* steal-less
* steal-qunit
* steal-mocha
* steal-css
* live-reload

# Design

Module scripts have no API that allows hooking the way that Steal currently does through the *normalize*, *locate*, etc. hooks. The only hook that exists is the Service Worker **fetch** event.

For the rest of this design we are going to assume that every browser that supports module scripts also supports service workers. Since no browser currently supports module scripts we won't know if this will be true or not. For browsers without either module scripts or service workers, we will fall back to using polyfills *for both*.

## Steal service worker

The steal service worker will sit in between the window and all requests that go out for modules.

Once the request has been fetched and we have raw source it will go through the **transpile** step.

Once transpiled the source will be parsed for module identifiers. The module identifiers will go through **normalize** and **locate** steps because script modules only understand URLs.

Once we have gone through transpile, normalize, and locate, the AST will then go through codegen to produce a source string. This is what will be returned from the service worker.

## steal.js

Only ES modules can be executed by the script module process. Any other module formats, like CommonJS, need to be executed in a separate registry.

**steal.js** is a module loader that runs in the window context and acts as the registry for dynamic modules like CommonJS. It is like our steal.js today with all normalize, locate, and translate extensions taken out (those run in the service worker context as described in the previous section).

steal.js only handles instantiation. The following diagram shows what will happen when an ES module depends on a CommonJS module.

<img width="542" alt="screen shot 2016-11-23 at 12 32 27 pm" src="https://cloud.githubusercontent.com/assets/361671/20572291/eb0dfc7a-b178-11e6-92f8-1c0c163e1510.png">

* An ES module *page.js* is requested via `<script type="module" src="./page.js"></script>`.
* The Steal service worker will peform translate, normalize, and locate.
* An ES module *tabs.js* is requested via `import './tabs.js';` in *page.js*.
* We know that tabs.js is really a CommonJS module, that module gets sent to the steal.js window context to be instantiated.
* The script module receives back a **wrapper module** for tabs.js that looks something like this: `export default steal.get("tabs.js");`. This wrapper module just pulls the CommonJS module's namespace from the Steal registry.

A crucial part of this is that we need to wait for the entire ES module tree to be "ready to execute" before executing the CommonJS modules. The Steal service worker will keep a trace of module imports so it knows when the CommonJS module tree needs to run.

## Plugin architecture

One possible outcome of this new architecture is that we might break plugins into their own separate workers. I think we need to base this on performance. If there is little or no loss from breaking them out into their own workers I think it makes sense to do so as it would allow a nice consistent API.

```js
onmessage = function(ev){
  var message = ev.data;
  
  if(message.type === "normalize") {
    npmNormalize(message.name).then(function(normalizedName){
      postMessage({
        id: message.id,
        name: normalizedName
      });
    });
    return;
  } else {
    // We're not interested in this event so just posting back.
    postMessage({});
  }
};
```

This is speculative based on whether it is practical to run plugins in a separate worker process or not. I can design this API further along in this RFC if wanted or we can save that for a new RFC when it comes time to do plugins.
