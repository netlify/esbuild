# @netlify/esbuild

This is a fork of esbuild maintained by Netlify. If you're interested in using or contributing to this fork, please read this document carefully.

## Objectives and guiding principles

We're big fans of esbuild at Netlify! ❤️

[zip-it-and-ship-it](https://github.com/netlify/zip-it-and-ship-it), our serverless functions bundler, uses esbuild to prepare JavaScript and TypeScript functions for deployment. In that context, there are certain features that we require in order to provide a smooth experience for our customers, some of which aren't provided by esbuild out-of-the-box just yet.

We've implemented those changes in this fork, which we maintain with the following principles:

1. **Minimize changes**: We don't have a different vision for esbuild, so we'll keep our changes to the mininum necessary. We won't introduce breaking changes to existing APIs.

2. **Build in public**: Everything about our fork will always be public — we'll publish every new release to npm as a public package and we'll use issues and pull requests in our public repository to discuss any changes we're planning or working on.

3. **Give back**: We'll take every change we introduce and contribute it back to the upstream repository via a pull request.

4. **Aim for unification**: We plan every feature so that it benefits the esbuild community and not our specific needs (which is made possible by the fact that esbuild is extendable via a plugin API). We do this because our end goal is to eventually decommission this fork and go back to using the main package.

As such, we prioritize the API designs from upstream over ours, which means that features in the fork are more volatile and subject to change based on what happens upstream.

## Changes

The Netlify fork diverges from upstream on the changes below.

### `onDynamicImport` plugins

There is an additional [plugin type](https://esbuild.github.io/plugins/#using-plugins) called `onDynamicImport`. These plugins are executed for every `require()` or `import()` call that contains an expression that can't be statically analyzed at build time.

_Example:_

- This `require()` can be resolved at build time, as esbuild can load the file at the path it finds and include it in the bundle:

  ```js
  const language = require("./locale/en.json");
  ```

- This `require()` can't be resolved at build time, since `req.params` only exists at runtime, meaning that esbuild can't load a file from a path it can't yet compute:

  ```js
  const language = require(`./locale/${req.params.lang}.json`);
  ```

  The resulting bundle will generate a runtime error, since it'll be requiring a file that doesn't exist.

The `onDynamicImport` plugin type can help mitigate this, since it allows developers to intercept these statements and provide a replacement module (or a shim) for handling the expressions at runtime.

For the example above, one could write a plugin like so:

```js
plugins: [
	{
		name: "dynamic-import-plugin",
		setup: (build) => {
			const filesWithDynamicImports = new Set();

			build.onDynamicImport({}, (args) => {
				// Add the importer file to a list of files with dynamic imports.
				// It can be used to show a warning to users, letting them know
				// which files could potentially lead to a broken build, prompting
				// them to flag them as external if we're not able to resolve them.
				filesWithDynamicImports.add(args.importer);

				// Parse the expression (e.g. "require(`./locale/${language}.json`)") with
				// a JavaScript parser, like https://www.npmjs.com/package/acorn.
				const parsedExpression = acorn.parse(args.expression);

				// We're able to infer that the expression is dynamically loading
				// files from the "locale/" directory, so we can make the choice to
				// include that directory with the bundle and make the require work
				// at runtime.
				assert(
					parsedExpression.body[0].expression.quasis[0].value === "./locale/"
				);

				// Because "locale/" will now be relative to the entire bundle and not
				// namedspaced to a given file or npm module, we can create our own
				// namespace and modify the require() at runtime.
				const filesNamespace = generateRandomString();

				// This replacement module will resolve the require at runtime. It will
				// take an expression (e.g. "./locale/en.json") and return the value of
				// require(`./${filesNamespace}/locale/en.json`), which is where we'll place
				// the files from "locale/".
				const replacementModule = `
          module.exports = expr => require(path.join('${filesNamespace}', expr))
        `;

				return {
					contents: replacementModule,
				};
			});
		},
	},
];
```

With this plugin, two things will happen:

1. The value of `contents` will be written to a file created by esbuild, with a path generated from hashing the file (e.g. `input-1q2w3e4r.js`)

2. The original `require()` call will be replaced with `require("./input-1q2w3e4r.js")(\`./locale/${req.params.lang}.json\`)` so that the shim is placed in charge of resolving the expression at runtime.
