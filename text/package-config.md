* Date: April 4, 2017
* Title: Package configuration

# Summary

This RFC proposes a new **package** configuration. This configuration mirrors [the AMD package config](https://github.com/amdjs/amdjs-api/wiki/Common-Config#packages-) but adds the ability to define common configuration options like *map*, and *meta* configuration inside of a package.

# Motivation

Defining configuration within an npm package is difficult in Steal when using the npm plugin (the majority of new projects). Some use cases that are difficult include:

* Defining map configuration within a package that are not applied globally (contextual maps). This is useful for providing shortcut names for loading code without having to use long relative paths.
* Specifying an alternative file type, ex. defining that `package/main` is actually `package/main.css`. In SystemJS configuration this is called *defaultExtension*.

Additionally, package configuration would help generalize some features that are implemented solely as part of the npm plugin and make them generally useful. An example is package configuration's **location** configuration. Currently a custom locate hook is used for finding an npm module's location, converting this into package configuration would let this extension handle alternative locations in a global way.

# Impact

This change can be achieved in a backwards compatible way. The ideal solution would require changing the npm module name definition to include a `/` rather than a `#` to add the modulePath. This change would cause contextual maps to work using the map extension.

However, we can make a small additionally normalize hook that prevents the need for the backwards incompatible npm module name change. This change should occur in the next major version, however.

Given that, the first phase of this change could be done in a minor release of Steal, and only require a new extension to be added (and the npm extension to be modified).

# Design

The design of this change should follow the [AMD specification](https://github.com/amdjs/amdjs-api/wiki/Common-Config#packages-) and support name, location, and main properties.

In addition, this change will add support for **map** and **meta** properties that define contextual configuration from within a package.

## Config hook

The bulk of the change will consist of adding a a hook for the **packages** config property within [steal/src/config.js](https://github.com/stealjs/steal/blob/master/src/config.js). This hook will:

1. Turn the configuration into a map (stored as something like `loader.packagesConfig`) with the package **name** as the key. This will allow for faster lookup.
2. Convert any **map** or **meta** configuration into `loader.map` and `loader.meta` configurations. For example for map it would take:

```json
{
  "foo": "bar"
}
```

And turn this into a contextual definition of:

```json
{
  "packageName": {
    "foo": "bar"
  }
}
```

This way the regular map extension can do its normal thing to apply map configuration.

## Locate hook

The extension would include a locate hook that uses the `name`, `location`, and `main` properties to find the url of a module.

## normalize hook

The *map configuration* normalize hook should be changed so that it considers `#` to be treated the same as `/`. This will allow this type of configuration to work:

```json
{
  "map": {
    "foo@1.0.0#": {
      "bar": "baz"
    }
  }
}
```

Where importing `bar` within the `foo` package gives you `baz` instead. This would work with no additional changes of the moduleName where `foo@1.0.0/parent` instead of `@foo@1.0.0#parent`. Since this fix can't be applied until Steal 2.0, it will be worked around in this manner for now.

## npm extension changes

The npm extension should be changed so that it also hooks config for the **packages** property. It loops over the packages array and changes names like `package` to `package@0.0.0` where 0.0.0 is the package's version number.

### Locate hook

The current locate hook should no longer be necessary. Instead the npm plugin should create packages configuration for each npm package, and the existence of this configuration should cause the package extension's own locate hook to be applied.

> *__Note__*: Regular configuration should be continued to be applied globally.
