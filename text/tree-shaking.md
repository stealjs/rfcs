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

1. Traverse the graph and find all `import` declarations and `export` declarations.
1. Remove `export` declarations from the AST *if*
  1. there is no corresponding `import` declaration in another module.
1. *Do not* remove export declarations if:
  1. the module is dynamically imported but another module (`steal.import("foo")`).
  1. is fully imported by another module `import "foo";`.
1. Within the module, remove any variables that are not referenced.
1. Within the module, remove any `import` declarations that are not referenced.
1. Start back at step (__2__).

This algorithm should continue until there is no more code that can be removed.

### Algorithm by example:

Consider this program:

__main.js__

```js
import {first} from './other';
```

__other.js__

```js
import {anotherOne, anotherTwo} from './and-another';

function callAnotherTwo() {
  return anotherTwo();
}

export function first() {
  return anotherOne();
};

export function second() {
  return callAnotherTwo();
}
```

__and-another.js__

```js
export function anotherOne() {
  return 'another';
};

export function anotherTwo() {
  return 'two';
};
```

The steps would be:

1. Traverse the graph and collect the import and export declarations.
1. Remove `export` declarations that do not have a corresponding `import`.
  * The `second()` export in __other.js__ isn't imported else. This function can be remove.
1. Remove unreferenced variables.
  * `callAnotherTwo()` which was used within `second()` is no longer referenced. Remove it.
1. Remove any `import` declarations that are not referenced.
  * Since `callAnotherTwo()` was removed, now `anotherTwo` import declaration is no longer referenced. Remove it.
1. Start back at step __2__.
2. Remove `export` declarations that do not have a corresponding `import`.
  * The `anotherTwo()` export in __and-another.js__ is no longer imported anywhere and can be removed.
1. Remove unused references: there are none now.
1. Remove unused imports: there are none now.
1. Break from the algorithm as nothing more can be removed.
