## Overview

This part of the tutorial is continued from [Part 3 - Basic Front End Application Skeleton with React-Router and React-Bootstrap](/Part-3-%2D-Basic-Front-End-Application-Skeleton-with-React%2DRouter-and-React%2DBootstrap), but it should be applicable to any similarly built application.  Much of the code in this part of the application will be specific to the node server we have configured.

In this part of the tutorial, we will be hooking up and implementing a couple of features to make developing the application easier and faster.

Primary Technologies
 - Webpack
 - Webpack dev middleware
 - Webpack hot module replacement

## Update existing configuration

There are a couple of options that I chose for the earlier parts of this tutorial that turned out to be non-optimal, incorrect, or simply made the application more complex than necessary.  We will go through these options to correct them.

### tsconfig.json

First, we will switch the tsconfig `lib` option to include `es6`.  Since we are using the same tsconfig.json file for both the server, and the client, it is inconvenient to limit ourselves to es5 code on the server.  Additionally, some third party type definitions will reference things like 'Map' and 'Set', which are only available in es6.  For more information about the consequences of changing this option, please review the section about polyfills on [Part 2 - Basic Front End Application Skeleton with React and Redux](/Part-2-%2D-Basic-Front-End-Application-Skeleton-with-React-and-Redux).

_tsconfig.json_
```js
{
  "compilerOptions": {
    //Use strict type checking
    //This makes it so 'null' and 'undefined' are considered explicit values
    //It shows type errors where types could not be inferred, among other things
    "strict": true,

    //Transforms react JSX syntax into React.createElement() calls
    "jsx": "react",

    //skip type checking .d.ts files, particularly from third party libraries
    //this can massively speed up compilation time
    //"skipLibCheck": true,

    //Tell the TypeScript compiler what libraries we expect to exist
    //In this case, we expect the user's browser to have ES6 support (haha) and a dom
    //The idea is that we will polyfill anything additional that we use from es6 in the client code
    "lib": ["es6", "dom"],

    //This causes ES6 modules to be used instead of CommonJS
    //This is important since it enables webpack to do tree shaking optimizations (dead code removal)
    //This has nothing to do with the compilation target, which is still ES3 by default
    "module": "es6",

    //When setting "module": "es6", the typescript compiler defaults to the "classic" module resolution strategy
    //It is important that we use the "node" module resolution strategy instead
    "moduleResolution": "node"
  },
  "exclude": [
    "node_modules"
  ]
}
```

Note that the 'dom' library will not exist on the server, so don't attempt to use it in server code.

We could have adjusted our build process to use a separate tsconfig file for the server code.  In my experience, this works perfectly fine, but IDEs such as WebStorm will always default to using the same tsconfig file for the entire project.  It is easier to just use es6 everywhere, and add polyfills as needed in the client code.  Just remember to test the application in IE to make sure you didn't forget any polyfills.

## webpack.config.js

Next, we will change a few things in the webpack configuration.  First of all, we will give each configuration in the array a name.  In this case, we will name the first configuration `name: 'web'` and the second configuration `name: 'server'`.  This way, we can reference individual configurations by name instead of by index.

In the server configuration, we will change the `node` option to have `__dirname` and `__filename` both be true instead of false.  This will make it so that __dirname and __filename will be preserved relative to the original source code.  That way, when you reference a file relative to your source code, you don't need to know exactly where it will end up being built.  Additionally, this fixes the behavior of `path.resolve( ... )` in some scenarios.

In the web configuration, we will add a `publicPath` field to the `output` option, which will be `'/build'`.  We will also change the `devtool` option to be `"cheap-module-eval-source-map"`.  This is much faster to build during development, particularly when re-building using the webpack dev middleware we will be configuring later.  However, this will not build anything for production.  If you find yourself in need of a source map in production code, then you can change this to something like `devtool: process.env.NODE_ENV === 'production' ? 'source-map' : 'cheap-module-eval-source-map'`.

_webpack.config.js_
```js
const path = require('path');
const nodeExternals = require('webpack-node-externals');

var config = [
    //web configuration
    {
        name: 'web',
        entry: [
            './src/web/boot-client.tsx'
        ],
        output: {
            path: path.resolve(__dirname, './public/build'),
            publicPath: '/build',
            filename: 'bundle.js',
        },
        resolve: {
            //automatically infer '.ts' and '.tsx' when importing files
            extensions: ['.js', '.jsx', '.ts', '.tsx']
        },
        module: {
            rules: [
                {
                    test:/\.css$/,
                    use:['style-loader','css-loader'],
                },
                {
                    test:/\.tsx?$/,
                    include: path.resolve(__dirname, "./src/web/"),
                    loader: "awesome-typescript-loader"
                },
                {
                    test: /\.(png|jpg|jpeg|gif|svg|ttf|otf|woff|woff2|eot)$/,
                    loader: 'url-loader?limit=25000'
                }
            ]
        },

        //see https://webpack.js.org/configuration/devtool/ for options
        devtool: "cheap-module-eval-source-map"
    },

    //server configuration
    {
        name: 'server',
        entry: ['./src/server/main.ts'],
        target: 'node',
        externals: [nodeExternals()],
        output: {
            path: path.resolve(__dirname, './build'),
            filename: '[name].js',
        },
        resolve: {
            //automatically infer '.ts' and '.tsx' when importing files
            extensions: ['.js', '.jsx', '.ts', '.tsx']
        },
        module: {
            rules: [
                {
                    test:/\.tsx?$/,
                    include: path.resolve(__dirname, "./src/server/"),
                    loader: "awesome-typescript-loader"
                }
            ]
        },

        //by default, webpack will set these to '/', so override that behavior on the server
        node: {
            //setting these to 'true' causes them to keep the same values relative to the source code
            //setting these to 'false' would cause them to have values relative to the output code
            __dirname: true,
            __filename: true
        }
    }
];
module.exports = config;
```

## Webpack Dev Middleware
### Overview

Webpack dev middleware is used to automatically watch for changes to our source code.  We will update our node server to include a new middleware that will do this.  If you are not using a node server, you can still