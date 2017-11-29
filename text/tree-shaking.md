* Date: 11/29/2017
* Title: Tree Shaking

# Summary

Implement tree-shaking in steal-tools. Tree-shaking is a type of dead code elimination that removes code by recursively removing (shaking) unused `export` declarations from a bundle.

I've spent 3 days examining how it could be implemented and created a repository [tree-shaking-spike](https://github.com/stealjs/tree-shaking-spike) which has all of the information.

# Motivation

Tree-shaking would reduce bundle size of applications by removing unused code. Minifiers can remove a lot of dead code, but when something is exported by a module it doesn't *appear* to be dead. Tree-shaking makes those exports no longer be exported, so the minifier will remove them.

# Impact

Most of the changes for this feature would occur in __steal-tools__. It is possible that some minor changes would be included in __transpile__. This would most likely only be to add some metadata, like what `export`s a module contains.

# Design

## API

This would be included by default as there isn't any good reason to make it enabled through an option. Likely there would be a way to turn it off just as minification can be turned off.

```js
stealTools.build({}, {
  shake: false
})
```

## Core algorithm

Tree-shaking should be implemented as a new step in both the *multibuild* and *optimize* build paths. It should be implemented as a step that occurs *after* transpiling. So it would go something like:

1. Transpile.
1. Tree-shaking.
1. Minify. // Dead leaves are removed here
1. Bundling and other stuff

The algorithm for implementing tree-shaking is a recursive process of walking through each Node in the graph. It goes, roughly:

1. Starting at the `main`, find unused exports by looking at the `import` declarations and removing the difference.
1. Remove those `export` declarations and any functions they call that is not used by other functions.
1. Find `import` declarations that are not used in each module and remove them.
1. Start by at step __1__.

This algorithm should continue until there is no more code that can be removed.
