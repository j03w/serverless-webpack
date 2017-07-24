# Serverless Webpack

[![Serverless][ico-serverless]][link-serverless]
[![License][ico-license]][link-license]
[![CircleCI][ico-circleci]][link-circleci]
[![NPM][ico-npm]][link-npm]
[![Contributors][ico-contributors]][link-contributors]

A Serverless v1.x plugin to build your lambda functions with [Webpack][link-webpack].

This plugin is for you if you want to use the latest Javascript version with [Babel][link-babel];
use custom [resource loaders][link-webpack-loaders];
try your lambda functions locally and much more!

> **BREAKING CHANGE IN v2**: `webpack` must now be installed alongside `serverless-webpack` as a peer dependency. This allows more control over which version of Webpack to run.

## Install

```bash
$ npm install serverless-webpack --save-dev
```

Add the plugin to your `serverless.yml` file:

```yaml
plugins:
  - serverless-webpack
```

## Configure

By default the plugin will look for a `webpack.config.js` in the service directory.
Alternatively, you can specify a different file or configuration in `serverless.yml`:

```yaml
custom:
  webpack: ./folder/my-webpack.config.js
```

An base Webpack configuration might look like this:

```js
// webpack.config.js

module.exports = {
  entry: './handler.js',
  target: 'node',
  module: {
    loaders: [ ... ]
  }
};
```

Note that, if the `output` configuration is not set, it will automatically be
generated to write bundles in the `.webpack` directory. If you set your own `output`
configuration make sure to add a [`libraryTarget`][link-webpack-libtarget]
for best compatibility with external dependencies:

```js
// webpack.config.js

module.exports = {
  // ...
  output: {
    libraryTarget: 'commonjs',
    path: '.webpack',
    filename: 'handler.js', // this should match the first part of function handler in `serverless.yml`
  },
  // ...
};
```

By default, the plugin will try to bundle all dependencies. However, you don't
want to include all modules in some cases such as selectively import, excluding
builtin package (ie: `aws-sdk`) and handling webpack-incompatible modules.

In this case you might add external modules in
[Webpack's `externals` configuration][link-webpack-externals].
Those modules can be included in the Serverless bundle with the `webpackIncludeModules`
option in `serverless.yml`:

```js
// webpack.config.js
var nodeExternals = require('webpack-node-externals')

modules.export = {
  // we use webpack-node-externals to excludes all node deps.
  // You can manually set the externals too.
  externals: [nodeExternals()],
}
```

```yaml
# serverless.yml
custom:
  webpackIncludeModules: true # enable auto-packing of external modules
```

All modules stated in `externals` will be excluded from bundled files. If an excluded module
is stated as `dependencies` in `package.json`, it will be packed into the Serverless
artifact under the `node_modules` directory.

By default, the plugin will use the `package.json` file in working directory, If you want to
use a different package file, set `packagePath` to your custom `package.json`:

```yaml
# serverless.yml
custom:
  webpackIncludeModules:
    packagePath: '../package.json' # relative path to custom package.json file.
```
> Note that only relative path is supported at the moment.

You can find an example setups in the [`examples`][link-examples] folder.

## Usage

### Automatic bundling

The normal Serverless deploy procedure will automatically bundle with Webpack:

- Create the Serverless project with `serverless create -t aws-nodejs`
- Install Serverless Webpack as above
- Deploy with `serverless deploy`

### Run a function locally

The plugin fully integrates with `serverless invoke local`. To run your bundled functions
locally you can:

```bash
$ serverless invoke local --function <function-name>
```

All options that are supported by invoke local can be used as usual:

- `--function` or `-f` (required) is the name of the function to run
- `--path` or `-p` (optional) is a JSON file path used as the function input event
- `--data` or `-d` (optional) inline JSON data used as the function input event

> :exclamation: The old `webpack invoke` command has been disabled.

### Run a function locally on source changes

Or to run a function every time the source files change use the `--watch` option
together with `serverless invoke local`:

```bash
$ serverless invoke local --function <function-name> --path event.json --watch
```

Everytime the sources are changed, the function will be executed again with the
changed sources. The command will watch until the process is terminated.

All options that are supported by invoke local can be used as usual:

- `--function` or `-f` (required) is the name of the function to run
- `--path` or `-p` (optional) is a JSON file path used as the function input event
- `--data` or `-d` (optional) inline JSON data used as the function input event

> :exclamation: The old `webpack watch` command has been disabled.

### Usage with serverless-offline

The plugin integrates very well with [serverless-offline][link-serverless-offline] to
simulate AWS Lambda and AWS API Gateway locally.

Add the plugins to your `serverless.yml` file and make sure that `serverless-webpack`
precedes `serverless-offline` as the order is important:
```yaml
  plugins:
    ...
    - serverless-webpack
    ...
    - serverless-offline
    ...
```

Run `serverless offline` or `serverless offline start` to start the Lambda/API simulation.

In comparison to `serverless offline`, the `start` command will fire an `init` and a `end` lifecycle hook which is needed for `serverless-offline` and e.g. `serverless-dynamodb-local` to switch off resources (see below).

You can find an example setup in the [`examples`][link-examples] folder.

#### Custom paths

If you do not use the default path and override it in your Webpack configuration,
you have use the `--location` option.

#### serverless-dynamodb-local

Configure your service the same as mentioned above, but additionally add the `serverless-dynamodb-local`
plugin as follows:
```yaml
  plugins:
    - serverless-webpack
    - serverless-dynamodb-local
    - serverless-offline
```

Run `serverless offline start`.

#### Other useful options

You can reduce the clutter generated by `serverless-offline` with `--dontPrintOutput` and
disable timeouts with `--noTimeout`.

### Bundle with webpack

To just bundle and see the output result use:

```bash
$ serverless webpack --out dist
```

Options are:

- `--out` or `-o` (optional) The output directory. Defaults to `.webpack`.

### Simulate API Gateway locally

_There are plans to remove the integrated simulation functionality in favor of
using `serverless-offline` (see [#135][link-135]) which already does the job
perfectly and fully integrates with `serverless-webpack`.
Please consider switching to `serverless-offline` if you do not use it already._

To start a local server that will act like API Gateway, use the following command.
Your code will be reloaded upon change so that every request to your local server
will serve the latest code.

```bash
$ serverless webpack serve
```

Options are:

- `--port` or `-p` (optional) The local server port. Defaults to `8000`

The `serve` command will automatically look for the local `serverless.yml` and serve
all the `http` events. For example this configuration will generate a GET endpoint:

```yaml
functions:
  hello:
    handler: handler.hello
    events:
      - http:
          method: get
          path: hello
```

## Example with Babel

In the [`examples`][link-examples] folder there is a Serverless project using this
plugin with Babel. To try it, from inside the example folder:

- `npm install` to install dependencies
- `serverless webpack run -f hello` to run the example function

## Provider Support

Plugin commands are supported by the following providers. ⁇ indicates that command has not been tested with that provider.

|                       | AWS Lambda | Apache OpenWhisk | Azure Functions | Google Cloud Functions |
|-----------------------|------------|------------------|-----------------|------------------------|
| webpack               |      ✔︎     |         ✔︎        |        ⁇        |            ⁇           |
| invoke local          |      ✔︎     |         ⁇        |        ⁇        |            ⁇           |
| invoke local --watch  |      ✔︎     |         ⁇        |        ⁇        |            ⁇           |
| webpack serve         |      ✔︎     |         ✘        |        ⁇        |            ⁇           |

## Release Notes

* 2.0.0
  * Support arbitrary Webpack versions as peer dependency [#83][link-83]
  * Support `serverless offline start` invocation [#131][link-131]
  * Documentation updates [#88][link-88], [#132][link-132], [#140][link-140], [#141][link-141], [#144][link-144]
  * Print Webpack stats on recompile [#127][link-127]

[ico-serverless]: http://public.serverless.com/badges/v3.svg
[ico-license]: https://img.shields.io/github/license/elastic-coders/serverless-webpack.svg
[ico-circleci]: https://img.shields.io/circleci/project/github/elastic-coders/serverless-webpack.svg
[ico-npm]: https://img.shields.io/npm/v/serverless-webpack.svg
[ico-contributors]: https://img.shields.io/github/contributors/elastic-coders/serverless-webpack.svg

[link-serverless]: http://www.serverless.com/
[link-license]: ./blob/master/LICENSE
[link-circleci]: https://circleci.com/gh/elastic-coders/serverless-webpack/
[link-npm]: https://www.npmjs.com/package/serverless-webpack
[link-contributors]: https://github.com/elastic-coders/serverless-webpack/graphs/contributors

[link-webpack]: https://webpack.github.io/
[link-babel]: https://babeljs.io/
[link-webpack-loaders]: https://webpack.github.io/docs/loaders.html
[link-webpack-libtarget]: https://webpack.github.io/docs/configuration.html#output-librarytarget
[link-webpack-externals]: https://webpack.github.io/docs/configuration.html#externals
[link-examples]: ./examples
[link-serverless-offline]: https://www.npmjs.com/package/serverless-offline
[link-serverless-dynamodb-local]: https://www.npmjs.com/package/serverless-dynamodb-local

[comment]: # (Referenced issues)

[link-135]: https://github.com/elastic-coders/serverless-webpack/issues/135

[link-83]: https://github.com/elastic-coders/serverless-webpack/pull/83
[link-88]: https://github.com/elastic-coders/serverless-webpack/pull/88
[link-127]: https://github.com/elastic-coders/serverless-webpack/pull/127
[link-131]: https://github.com/elastic-coders/serverless-webpack/pull/131
[link-132]: https://github.com/elastic-coders/serverless-webpack/pull/132
[link-140]: https://github.com/elastic-coders/serverless-webpack/pull/140
[link-141]: https://github.com/elastic-coders/serverless-webpack/issues/141
[link-144]: https://github.com/elastic-coders/serverless-webpack/issues/144
