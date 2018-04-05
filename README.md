# package.json `"ecma"` version field spec

## Motivation

The adoption of ES2015 is growing. Node supports most of it natively and evergreen browsers can handle most of it. And it's going even further with Node.js and browsers natively supporting features from ES2017 and even a small portion of ES2018.

But we still need to transpile things to ES5 for some older but still relevant browsers. And even for evergreen browsers with full ES2017 support you may need to transpile some ES2018 code supported by Node if packages adopt it before 100% of evergreen browsers support it (e.g. native object rest/spread properties).

This leaves many users who bundle code for the browser using WebPack, Rollup, or another module bundler in a problematic situation.

- They need to output ES5 code to maintain the level of browser support they promise.
- Packages designed for Node.js may be using ES2015+ syntax which will break their code if bundled.
- Packages from portions of even the browser ecosystem such as the Polymer 3.0 ecosystem and some parts of the React ecosystem may be shipping ES2016+ code in their packages.
- Some packages offer multiple variants of their code such as an ES5, ES5 with ES2015 modules, and/or ES2015 code. Or ES2017 to the degree that it is supported by their target versions of Node.js and an ES5 bundle for browsers. So exactly what version of ECMAScript is in use is dependent on what part of the package is imported.

This leaves these users with only three options:

1. Don't transpile anything in node_modules: This of eventually course breaks because the user wants to start using a package that exports non-ES5 code.
2. Transpile everything in node_modules: This works, if you make sure to ignore `.babelrc`, but it is extremely inefficient and hurts performance as you are running the transpiler over all of the code, much of which is already pre-transpiled to ES5.
3. Manually maintain a list of packages to transpile. This is the approach used by many, however it is undesirable as this is a list that needs to be maintained as more and more packages begin shipping non-ES5 code in their packages.

## Overview

This spec defines a `"ecma"` field for `package.json`.

The value of the `ecma` field is an object with glob patterns as its keys and an ECMAScript edition number as its values.

Globs are used instead of extending the `engines` field to declare a single ECMAScript version as globs allow us to define the ECMAScript version for all variants within a package that bundles different variants for different versions of ECMAScript.

## Usage

A simple package that only uses ES5 code may simply declare all JS files as ES5.

```json
"ecma": {"*.js": 5}
```

If a package only targets node and the package maintainers write using ES2016 features but don't use any of the ES2017+ features Node supports they can declare all the JS files as ES2016.

```json
"ecma": {"*.js": 6}
```

If a package is written in ES2016 but defines an ES5 `browser.js` module to be delivered to browsers with `"browser"` that file can be declared separately:

```json
"ecma": {
  "*.js": 6,
  "browser.js": 5
}
```

If a package has two variants, CommonJS modules transpiled to ES5 in `lib/` and ES2015 modules in `modules/` for `"module"` that are otherwise transpiled to ES2015 both should be declared as ES5. For the purposes of the `ecma` field ES2015 modules are not considered to be ES2015 on their own if they are delivered through `module` with the intent of being used by WebPack/Rollup.

```json
"ecma": {"*.js": 5}
```

If a package has two variants like before, except the author has chosen that their ES2015 modules should retain ES2015 code other than the module syntax instead of being transpiled to ES5, then that code should be declared as ES2015.

```json
"ecma": {
  "lib/**/*.js": 5,
  "modules/**/*.js": 6
}
```

If a package delivers its source code alongside the ES5 bundled variant it should declare what version of ECMAScript the source uses.

```json
"ecma": {
  "*.js": 5,
  "src/**/*.js": 8
}
```

If features that have made it to Stage 4 and are planned to be included in the upcoming edition of the ECMAScript standard but are not yet standardized are used. Then the edition number of the ECMAScript standard they are expected to come out in may be used.

For example if source code is written using ES2017 code plus the currently Stage 4 "Rest/Spread Properties" that will be included in ES2018, the author may use `9` as the ECMA version. As given ES2017 is edition `8` of ECMAScript the ES2018 features will be part of edition `9`. This will tell the transpiler that transpiling ES2017 alone is not enough, but should be forward compatible such that in a couple of years when ES2018 is supported natively the transpiler does not think it still needs to transpile the package.

```json
"ecma": {
  "*.js": 5,
  "src/**/*.js": 9
}
```

Stage 0 - Stage 3 features that have not been standardized are not handled by the "ecma" field. The majority of these features are not available natively in browsers or any version of Node.js. A small number of Stage 3 candidates have some support in a few browsers, but not enough to consider publishing packages that require them. As such package maintainers that made use of Stage 0 – Stage 3 code in their packages are expected to at-minimum transpile the non-src Stage 0 – Stage 3 code in their packages.

Special syntaxes requiring transpiling such as JSX and Flow are not handled by the `ecma` field and must be handled in some other way.

As a more advanced example, here is the `ecma` field for a contrived package that has a `src/` written in ES2018 included in the package, an ES5 `index.js` and `lib/`, ES modules transpiled to ES5 in a `index.modules.js` and a `modules/` folder, an ES6 `index.es.js` and `es/` folder, and ESM `.mjs` code targeted at the level of ECMAScript supported by Node 10.

```json
"ecma": {
  "index.js": 5,
  "lib/**/*.js": 5,
  "index.modules.js": 5,
  "modules/**/*.js": 5,
  "index.es.js": 6,
  "es/**/*.js": 6,
  "*.mjs": 9,
  "src/**/*.js": 9
}
```

## Detailed design

### Versions

The version number must be the incrementing "Edition" number, not the year based numbering of the standard name. The intermediate `5.1` is not supported.

i.e. `5` for ES5, `6` for es6/ES2015, `7` for es7/ES2016, `8` for es8/ES2017, `9` for the upcoming es9/ES2018.

As a special case, ES2015 modules are **not** considered to be es6/ES2015 code **if** they are referred to using `"module"` instead of `"main"` and do not contain any other ES2016 syntax.

This allows for packages to contain code that is entirely transpiled to ES5 but offers a ES2016 modules variant using `"module"` that WebPack and Rollup can use to bundle and tree shake code, without requiring that the modules be transpiled.

### Globs

The glob implementation used must conform to the following rules:

- It must support the following Glob syntax:

  `*`, `**`, `?`, `{a,b}`, `[a-c]`, `!(a|b)`, `?(a|b)`, `+(a|b)`, `*(a|b)`, `@(a|b)`
- It must not treat `#` as the start of a comment
- It must not treat `!` as the start of a negation
- It must use `/` as the directory separator

For reference, both the popular `minimatch`/`node-glob` and `micromatch` packages should be capable of conforming.

- Both libraries offer a `nonegate` option to disable negation behaviour
- `minimatch` has a `nocomment` option to desable comment behaviour, and `micromatch` does not appear to support comments

When evaluating the glob pattern, the search path of the glob should be relative to the root of the package.

When multiple glob patterns match a file, the glob pattern with the longest string length is used. The simplest way to handle this is to reverse sort by string length and test the longer glob matches first.

The behaviour when two globs are the same length and both match a file, such as `*.js` and `a.js` matching the file `a.js` is not defined. Single character file names should be rare and in the cases they exist generally will not require one to have a different ECMAScript version than the others. In the very rare case that this is necessary authors are expected to take advantage of the extra glob syntax that support is mandated for to craft a glob hack that will make the string the author wishes to have a higher specificity be longer. For example `[a].js`.

### Responsibilities of WebPack and other bundlers

Bundlers must read the `ecma` field and test files against the globs to check for an ECMAScript version. If an `ecma` field is present and a match is found for a file, the bundler must add this ECMAScript version to the metadata about the file that the bundler exposes to plugins/loaders and configuration.

As a special note for WebPack. The functional versions of the rule conditions in `modules.loaders.rules.*.use.*.{exclude,include}` should be given a second argument, this argument can either be a simple `{ecma: version}` object or can be the `NormalModule` WebPack exposes to other functions, which WebPack should expose an `.ecma = version` property on. This will allow users to write loader rules that include/exclude based on the `ecma` version of a file instead of just based on paths.

### Responsibilities of users and configuration

In a bundler like WebPack and Rollup where the user is responsible for configuring transpilers it is the user's responsibility to use the metadata exposed by the bundler to setup a transpiler that will transpile the code in node_modules for their environment.

For example, here is an example WebPack configuration for someone who transpiles their own code with babel-loader, outputs an ES5 bundle, and wants to transpile all ES2015+ package code with preset-env.

```js
module: {
  loaders: [
    // Transpile all my files according to my .babelrc
    {
      test: /\.jsx?$|\.mjs$/,
      exclude: s => /node_modules/.test(s),
      loader: 'babel-loader',
    },
    // Transpile node_modules only using preset-env,
    // but exclude any files marked as es5
    {
      test: /\.jsx?$|\.mjs$/,
      include: s => /node_modules/.test(s),
      // Exclude files that do not have an `ecma` glob,
      // or are defined as being ES5 or lower
      exclude: (s, f) => !f.ecma || f.ecma <= 5,
      loader: 'babel-loader',
      options: {
        // Do not get confused by any stray
        // .babelrc files incorrectly left in packages
        babelrc: false
        // Only use the env preset
        presets: ['@babel/preset-env'],
      },
    }
  ],
},
```

However for other bundlers that pride themselves on being pre-configured and ready to use out of the box. The bundler itself should be responsible for setting up a transpiler configuration that will transpile code in node_modules if it has a `ecma` field declaring the file a being a version that the bundler needs to have transpiled.

### Responsibilities of loaders and transpilers

Loaders and transpilers do not have any special responsibilities that are required of them here. However if it is possible it may be beneficial for loaders such as `babel-loader` to pass on the edition number extracted from `ecma` to presets. This way `@babel/preset-env` may have the option of skipping unnecessary transforms in the future.

i.e. If a file is defined as being written in ES2016 and `env` is targeting ES2015 the `env` can decide to exclude all the ES2017+ transforms and only have the cost of running transforms to transpile ES2016 to ES2015.

## Links

- [WebPack issue](https://github.com/webpack/webpack/issues/6918)
- [Rollup issue](https://github.com/rollup/rollup/issues/2104)

## *"Just transpile to ES5"*

There are a number of reasons why it does not make sense to require all npm packages transpile their default `main`'s code to ES5.

- Many packages are designed primarily for use in Node.js. At this point Node 4 is the lowest supported version, and it has partial ES2015 support. Very soon even that will be EOL and Node 6 will be the new minimum for many packages, which has [support for 99% of ES2015 features](http://node.green/). In one year Node 8 will be the minimum, and that has over [support for over 65% of ES2017](http://node.green/#ES2017) (that 65% effectively covers all the useful features of ES2017, most of the stuff that version does not support are less commonly used features like shared memory and atomics). It does not make sense to demand that packages designed primarily for use in Node.js all transpile their code to ES5 and forgo the advantages of native support just to support a small subset of the package's users that want to use the package in the browser.
- Using pre-transpiled ES5 code voids your ability to choose to ship ES2015 code. 80-90% of browsers currently in use have nearly complete support for ES2015. For some sites their target market may be nearly 100% composed of evergreen browsers with effectively complete ES2015 support. And for other sites, creating 2 bundles of their code and switching between (one for evergreen browsers with ES6 support and another for older browsers) may be beneficial enough to them to be worth the extra steps.
- Sometimes transpiling the ES6 code of a package to *ES5* yourself with env targets and other Babel optimizations specific to your environment can yield significant savings to the resulting code delivered to the browser. If all packages are pre-compiled to ES5 you lose this option.
- Some parts of ES2015+ such as generators and async functions require the regenerator runtime which is a significant size and unlike a polyfill cannot be omitted in environments with native support if it is used. If a package uses these features and we demand it is transpiled to ES5, the regenerator runtime will be a requirement in Node.js and evergreen browsers where there is enough native support of generators (and sometimes async functions) to not need regenerator.
- Some browser features such as natively supported Custom Elements require ES6 classes and do not work with ES5 transpiled code; unless you include an ES5 adapter that cannot be transpiled as part of your ES5 bundle and has some quirks when you include it. Parts of the ecosystem using these like Polymer 3.x **must** distribute ES6 code to avoid breakage in current browsers.
- Browsers are beginning to support ES2015 modules natively with `<script type="module">`. These modules are full ES2015 and do not require transpilation to ES5. `nomodule` is also available which begins to open up a browser supported method of having build tools generate an ES5 script bundle and an ES2015 module bundle and serve them separately to the browser without any special loader scripts.

## License

<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons Licence" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/88x31.png" /></a>

This document is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.