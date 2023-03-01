# Loading JavaScript

Webpack processes ES2015 module definitions by default and transforms them into code. It does **not** transform specific syntax, such as `const`, though. The resulting code can be problematic especially in the older browsers.

To get a better idea of the default transform, we can generate a build while setting webpack's `mode` to `none` to avoid any transformation. Change the `build` target to use `none` temporarily (`{ mode: "none" }` in webpack configuration) and run `npm run build`:

**dist/main.js**

```javascript
...
/***/ ((__unused_webpack_module, __webpack_exports__, __webpack_require__) => {

__webpack_require__.r(__webpack_exports__);
/* harmony export */ __webpack_require__.d(__webpack_exports__, {
/* harmony export */   "default": () => __WEBPACK_DEFAULT_EXPORT__
/* harmony export */ });
/* harmony default export */ const __WEBPACK_DEFAULT_EXPORT__ = ((text = "Hello world") => {
  const element = document.createElement("div");
  element.className = "rounded bg-red-100 border max-w-md m-4 p-4";
  element.innerHTML = text;
  return element;
});
...
```

The problem can be worked around by processing the code through [Babel](https://babeljs.io/), a JavaScript transpiler that supports ES2015+ features and more. It resembles ESLint in that it's built on top of presets and plugins. Presets are collections of plugins, and you can define your own as well.

T> Babel isn't the only option, although it's the most popular one. [esbuild-loader](https://www.npmjs.com/package/esbuild-loader) and [swc-loader](https://www.npmjs.com/package/swc-loader) are worth checking out if you don't need any specific Babel presets or plugins and want more performance.

## Using Babel with webpack configuration

Even though Babel can be used standalone, as you can see in the _SurviveJS - Maintenance_ book, you can hook it up with webpack as well. During development, it can make sense to skip processing if you are using language features supported by your browser.

Skipping processing is a good option, primarily if you don't rely on any custom language features and work using a modern browser. Processing through Babel becomes almost a necessity when you compile your code for production, though.

You can use Babel with webpack through [babel-loader](https://www.npmjs.com/package/babel-loader). It can pick up project-level Babel configuration, or you can configure it at the webpack loader itself.

Connecting Babel with a project allows you to process webpack configuration through it. Name your webpack configuration as **webpack.config.babel.js** to achieve this. [interpret](https://www.npmjs.com/package/interpret) package enables this, and it supports other tools as well.

T> Given that [Node supports the ES2015 specification well](http://node.green/) these days, you can use a lot of ES2015 features without having to process configuration through Babel.

W> If you use **webpack.config.babel.js**, take care with the `"modules": false,` setting. If you want to use ES2015 modules, you could skip the setting in your global Babel configuration and then configure it per environment, as discussed below.

### Setting up **babel-loader**

The first step towards configuring Babel to work with webpack is to set up [babel-loader](https://www.npmjs.com/package/babel-loader). It takes the code and turns it into a format older browsers can understand. Install **babel-loader** and include its peer dependency **@babel/core**:

```bash
npm add babel-loader @babel/core -D
```

As usual, let's define a function for Babel:

**webpack.parts.js**

```javascript
const APP_SOURCE = path.join(__dirname, "src");

exports.loadJavaScript = () => ({
  module: {
    rules: [
      // Consider extracting include as a parameter
      { test: /\.js$/, include: APP_SOURCE, use: "babel-loader" },
    ],
  },
});
```

Next, you need to connect this to the main configuration. If you are using a modern browser for development, you can consider processing only the production code through Babel. It's used for both production and development environments in this case. Also, only application code is processed through Babel.

{pagebreak}

Adjust as below:

**webpack.config.js**

```javascript
const commonConfig = merge([
  ...
leanpub-start-insert
  parts.loadJavaScript(),
leanpub-end-insert
]);
```

Even though you have Babel installed and set up, you are still missing one bit: Babel configuration. The configuration can be set up using a `.babelrc` dotfile as then other tooling can use the same.

### Setting up `.babelrc`

At a minimum, you need [@babel/preset-env](https://www.npmjs.com/package/@babel/preset-env). It's a Babel preset that enables the required plugins based on [browserslist](https://www.npmjs.com/package/browserslist) definition.

Install the preset first:

```bash
npm add @babel/preset-env -D
```

To make Babel aware of the preset, you need to write a `.babelrc`. Given webpack supports ES2015 modules out of the box, you should tell Babel to skip processing them.

T> See the _Autoprefixing_ chapter for an expanded discussion of browserslist.

{pagebreak}

Here's a sample configuration:

**.babelrc**

```json
{
  "presets": [["@babel/preset-env", { "modules": false }]]
}
```

If you execute `npm run build --mode none` and examine `dist/main.js`, you will see something different based on your `.browserslistrc` file. Try to include only a definition like `IE 8` there, and the code should change accordingly:

**dist/main.js**

```javascript
...
/***/ ((__unused_webpack_module, __webpack_exports__, __webpack_require__) => {

__webpack_require__.r(__webpack_exports__);
/* harmony export */ __webpack_require__.d(__webpack_exports__, {
/* harmony export */   "default": () => __WEBPACK_DEFAULT_EXPORT__
/* harmony export */ });
/* harmony default export */ const __WEBPACK_DEFAULT_EXPORT__ = (function () {
  var text = arguments.length > 0 && arguments[0] !== undefined ? arguments[0] : "Hello world";
  var element = document.createElement("div");
  element.className = "rounded bg-red-100 border max-w-md m-4 p-4";
  element.innerHTML = text;
  return element;
});
...
```

Note especially how the function was transformed. You can try out different browser definitions and language features to see how the output changes based on the selection.

T> **@babel/preset-env** comes with a `bugfixes` option that, when enabled, writes modern syntax to one that works also in older browsers without compiling down to ES5.

## Polyfilling features

**@babel/preset-env** allows you to polyfill certain language features for older browsers. For this to work, you should enable its `useBuiltIns` option and install [core-js](https://www.npmjs.com/package/core-js). If you are using `async` functions and want to support older browsers, then [regenerator-runtime](https://www.npmjs.com/package/regenerator-runtime) is required as well.

You have to include **core-js** to your project either through an import or an entry (`app: ["core-js", "./src"]`), except if you're using `useBuiltIns: 'usage'` to configure `@babel/preset-env`. **@babel/preset-env** rewrites the import based on your browser definition and loads only the polyfills that are needed. To learn more about **core-js** and why it's needed, [read core-js 3 release post](https://github.com/zloirock/core-js/blob/master/docs/2019-03-19-core-js-3-babel-and-a-look-into-the-future.md).

T> [corejs-upgrade-webpack-plugin](https://www.npmjs.com/package/corejs-upgrade-webpack-plugin) makes sure you are using the newest **core-js** polyfills. Using it can help to reduce the size of the output.

W> **core-js** pollutes the global scope with objects like `Promise`. Given this can be problematic for library authors, there's [@babel/plugin-transform-runtime](https://babeljs.io/docs/plugins/transform-runtime/) option. It can be enabled as a Babel plugin, and it avoids the problem of globals by rewriting the code in a such way that they aren't be needed.

W> Certain webpack features, such as _Code Splitting_, write `Promise` based code to webpack's bootstrap after webpack has processed loaders. The problem can be solved by applying a shim before your application code is executed. Example: `entry: { app: ["core-js/es/promise", "./src"] }`.

## Babel tips

There are other possible [`.babelrc` options](https://babeljs.io/docs/en/options) beyond the ones covered here. Like ESLint, `.babelrc` supports [JSON5](https://www.npmjs.com/package/json5) as its configuration format meaning you can include comments in your source, use single quoted strings, and so on.

Sometimes you want to use experimental features that fit your project. If you expect your project to live a long time, it's better to document the features you are using well.

## Babel plugins

Perhaps the greatest thing about Babel is that it's possible to extend with plugins. Consider the following examples:

- [babel-plugin-import](https://www.npmjs.com/package/babel-plugin-import) rewrites module imports so that you can use a form such as `import { Button } from "antd";` instead of pointing to the module through an exact path.
- [babel-plugin-transform-react-remove-prop-types](https://www.npmjs.com/package/babel-plugin-transform-react-remove-prop-types) removes `propType` related code from your production build. It also allows component authors to generate code that's wrapped so that setting environment at `DefinePlugin` can kick in as discussed in the _Environment Variables_ chapter.
- [babel-plugin-macros](https://www.npmjs.com/package/babel-plugin-macros) provides a runtime environment for small Babel modifications without requiring additional plugin setup.

T> It's possible to connect Babel with Node through [babel-register](https://www.npmjs.com/package/babel-register) or [babel-cli](https://www.npmjs.com/package/babel-cli). These packages can be handy if you want to execute your code through Babel without using webpack.

## Generating differential builds

To benefit from the support for modern language features and to support legacy browsers, it's possible to use webpack to generate two bundles and then write bootstrapping code that's detected by the browsers so that they use the correct ones. Doing this gives smaller bundles for modern browsers while improving JavaScript parsing time. Legacy browsers will still work as well.

As [discussed by Philip Walton](https://philipwalton.com/articles/deploying-es2015-code-in-production-today/), on browser-side you should use HTML like this:

```html
<!-- Browsers with ES module support load this file. -->
<script type="module" src="main.mjs"></script>

<!-- Older browsers load this file (and module-supporting -->
<!-- browsers know *not* to load this file). -->
<script nomodule src="main.es5.js"></script>
```

The fallback isn't without problems as in the worst case, it can force the browser to load the module **twice**. Therefore relying on a user agent may be a better option as [highlighted by John Stewart in his example](https://github.com/johnstew/differential-serving). To solve the issue, [Andrea Giammarchi has developed a universal bundle loader](https://medium.com/@WebReflection/a-universal-bundle-loader-6d7f3e628f93).

On webpack side, you will have to take care to generate two builds with differing browserslist definitions and names. In addition, you have to make sure the HTML template receives the `script` tags as above so it's able to load them.

{pagebreak}

To give you a better idea on how to implement the technique, consider the following and set up a browserslist as below:

**.browserslistrc**

```
# Let's support old IE
[legacy]
IE 8

# Make this more specific if you want
[modern]
> 1% # Browser usage over 1%
```

The idea is to then write webpack configuration to control which target is chosen like this:

**webpack.config.js**

```javascript
// Remember to set "mode": "production" in productionConfig
// so webpack knows to compile for the production target.
const getConfig = (mode) => {
  switch (mode) {
    case "prod:legacy":
      process.env.BROWSERSLIST_ENV = "legacy";
      return merge(commonConfig, productionConfig);
    case "prod:modern":
      process.env.BROWSERSLIST_ENV = "modern";
      return merge(commonConfig, productionConfig);
    ...
    default:
      throw new Error(`Trying to use an unknown mode, ${mode}`);
  }
};
```

{pagebreak}

Above would expect the following target:

**package.json**

```json
{
  "scripts": {
leanpub-start-insert
    "build": "wp --mode prod:legacy && wp --mode prod:modern"
leanpub-end-insert
  }
}
```

To complete the setup, you have to write a script reference to your HTML using one of the techniques outlined above. The webpack builds can run parallel and you could use for example use the [concurrently](https://www.npmjs.com/package/concurrently) package to speed up the execution.

T> These days it's possible to go one step further and [use native JavaScript modules directly in the browser](https://philipwalton.com/articles/using-native-javascript-modules-in-production-today/).

## TypeScript

Microsoft's [TypeScript](http://www.typescriptlang.org/) is a compiled language that follows a similar setup as Babel. The neat thing is that in addition to JavaScript, it can emit type definitions. A good editor can pick those up and provide enhanced editing experience. Stronger typing is valuable for development as it becomes easier to state your type contracts.

Compared to Facebook's type checker Flow, TypeScript is a safer option in terms of ecosystem. As a result, you find more premade type definitions for it, and overall, the quality of support should be better.

[ts-loader](https://www.npmjs.com/package/ts-loader) is the recommended option for TypeScript. One option is to leave only compilation to it and then handle type checking either outside of webpack or to use [fork-ts-checker-webpack-plugin](https://www.npmjs.com/package/fork-ts-checker-webpack-plugin) to handle checking in a separate process.

You can also compile TypeScript with Babel through [@babel/plugin-transform-typescript](https://www.npmjs.com/package/@babel/plugin-transform-typescript) although this comes with small [caveats](https://babeljs.io/docs/en/next/babel-plugin-transform-typescript.html#caveats).

W> Webpack 5 includes TypeScript support out of the box. Make sure you don't have `@types/webpack` installed in your project as it will conflict. [@types/webpack-env](https://www.npmjs.com/package/@types/webpack-env) contains webpack types related to the environment. If you use features like `require.context`, then you should install this one.

T> To split TypeScript configuration, use the `extends` property (`"extends": "./tsconfig.common"`) and then use **ts-loader** `configFile` to control which file to use through webpack.

### Using TypeScript to write webpack configuration

If you have set up TypeScript to your project, you can write your configuration in TypeScript by naming the configuration file as **webpack.config.ts**. Webpack is able to detect this automatically and run it correctly.

For this to work, you need to have [ts-node](https://www.npmjs.com/package/ts-node) or [ts-node-dev](https://www.npmjs.com/package/ts-node-dev) installed to your project as webpack uses it to execute the configuration.

If you run webpack in watch mode or through **webpack-dev-server**, by default compilation errors can cause the build to fail. To avoid this, use the following configuration:

**tsconfig.json**

```json
{ "ts-node": { "logError": true, "transpileOnly": true } }
```

Especially the `logError` portion is important as without this **ts-node** would crash the build on error. `transpileOnly` is useful to set if you want to handle type-checking outside of the process. For example, you could run `tsc` using a separate script. Often editor tooling can catch type issues as you are developing as well eliminating the need to check through **ts-node**.

## WebAssembly

[WebAssembly](https://developer.mozilla.org/en-US/docs/WebAssembly) allows developers to compile to a low-level representation of code that runs within the browser. It complements JavaScript and provides one path of potential optimization. The technology can also be useful when you want to run an old application without porting it entirely to JavaScript.

Starting from webpack 5, the tool supports new style asynchronous WebAssembly. The official examples, [wasm-simple](https://github.com/webpack/webpack/tree/master/examples/wasm-simple) and [wasm-complex](https://github.com/webpack/webpack/tree/master/examples/wasm-complex), illustrate the experimental functionality well. [wasmpack's webpack tutorial](https://rustwasm.github.io/docs/wasm-pack/tutorials/hybrid-applications-with-webpack/index.html) shows how to package Rust code using WebAssembly to be consumed through webpack.

## Conclusion

Webpack loads JavaScript out of the box. Tools like Babel let you target specific browsers and have more control over the output.

To recap:

- Babel gives you control over what browsers to support. It can compile ES2015+ features to a form the older browser understand. **@babel/preset-env** is valuable as it can choose which features to compile and which polyfills to enable based on your browser definition.
- Babel allows you to use experimental language features. Babel ecosystem has numerous presets and plugins to customize it.
- Babel functionality can be enabled per development target. This way you can be sure you are using the correct plugins at the right place.
