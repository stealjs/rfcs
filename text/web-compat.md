* Date: March 1, 2017
* Title: Transpiling projects to browser-compat ES modules

# Summary

Provide a tool that will transpile 3rd party packages so that they can be used with `<script type="module">`.

# Motivation

The primary reason is to provide a way for developers to take advantage of `<script type="module">` while still remaining within the Steal ecosystem and taking advantage of Steal's tooling (especially steal-tools and its multi-build).

## Speed

Loading a Steal application in development can be slow. This is for a number of reasons:

* Hundreds of files can be loaded if using a few libraries.
* Steal doesn't start loading your main module until it's loaded configuration. This causes a waterfall for loading.
* Some of the code is complex (and possibly inefficient) such as the node_modules algorithm.
* All files with import/export are transpiled to ES5, even for features the browser supports (such as `class`).
* All loading happens in the main thread. Expensive operations like transpiling block other modules from loading.

Some of this problems can be remedied easily, some would require a significant amount of work, but probably it would never be possible to match the browser's loading speed.

## Compatibility

Since the native `<script type="module">` cannot be extended (except through ServiceWorker `fetch` hook), Steal is essentially incompatible with the use of `<script type="module">`. This forces developers to choose either Steal (ands its features) or choose the native loader.

# Impact

This will be an alternative to using **steal.js** in development. steal-tools should still work for production builds.

# Design

> **Note**: I'm calling this tool `webify` below but that is just a code name, open to changing it.

I'm proposing a new tool in the Steal ecosystem that makes a project web-compatible (compatible with `<script type="module">`).

This tool can be used a couple of different ways:

```shell
webify can-stache
```

This will look for the `can-stache` package and:

* Transpile it to ES module syntax.
* Changes its import specifiers to be relative (instead of `import "jquery"` it becomes `import "../path/to/jquery.js"`.
* Moves all of the code to a different folder, probably `web_modules/`.

You can also webify all of your project's dependencies by running:

```shell
webify
```

This will load your package.json **main** and webify any child packages.

## Plugins

Since `<script type="module">` does not have any hooks, it's not possible to load non-JavaScript modules with it. There are a couple of possible workarounds:

* Transpile plugins to JavaScript using a Node.js tool (maybe webify).
* Use Service Workers to provide the ability to create plugins on-the-fly.

These options have strengths and weaknesses. For this reason I would consider plugins to be out-of-scope for the first version of this project, and something we add afterwards.
