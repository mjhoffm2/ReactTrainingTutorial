[[_TOC_]]

## Overview

This part of the tutorial is continued from [Part 4 - Improving the Development Experience](/Part-4-%2D-Improving-the-Development-Experience).

After everything is said and done, we still need to be able to ship some quality code.  If you have been following the tutorial exactly up to this point, you may have noticed that our bundle is about 5.57 MB when built with --mode=development.  However, when built when --mode=production, the bundle is still about 2.25 MB.  This is way too large, so we are going to need to address this.

## Dynamic Webpack Configuration

### Overview
The first thing we are going to want to do is be able to tweak our `webpack.config.js` file based on whether we are compiling for development or production.  You might expect to just be able to check the global `process.env.NODE_ENV` flag just like we can do in the rest of our ui code, but you would be unfortunately mistaken.  When the `webpack.config.js` file is executed, that flag has not yet been set.

There are two ways we could go about this.  One way is to simply have two different webpack configurations, one for development and one for production.  If you use this approach, you can also have a 'common' webpack configuration that each environment specific configuration extends to reduce duplicate code.  The alternate approach is to adjust the webpack configuration to be able to detect the 'mode' parameter, which is the approach I will be taking in this tutorial.

### Adjust webpack.config.js

In order to detect the 'mode' parameter that has been provided, we will need to adjust the configuration that is exported by this file.  Basically, instead of exporting our configuration array directly, we are going to export a function that returns our configuration.  The function looks like this:

**Old webpack.config.js**
```js
var config = [
    //web configuration
    {
        name: 'web',
        ...
    }
];
module.exports = config;
```

**New webpack.config.js**
```js
var config = (env, options) => [
    //web configuration
    {
        name: 'web',
        ...
    }
];
module.exports = config;
```

With the new format, we are able to access the `env` and `options` parameters.  In particular, `options.mode` will be set to `'production'` or `'development'` based on the mode that we provided.  Using this, we can now set up our webpack configuration so that we return a slightly different config based on which environment we are compiling for.

### Update webpack-dev-middleware

One consequence of the way we have changed our configuration, is that the old code we were using on the server to obtain the configuration will have broken.  The fix is pretty simple.  Open `App.ts` and make the following change:

**Old**
```ts
const configs: Configuration[] = require('../../webpack.config');
const config = configs.find(config => config.name === "web");
```

**New**
```ts
const configs: (env: any, options: any) => Configuration[] = require('../../webpack.config');
const config = configs({}, {mode: "development"}).find(config => config.name === "web");
```

This will let us maintain the same behavior as before without adjusting any of our other code.  The downside is that if you end up using more then just `options.mode` in your `webpack.config.js` file, you will need to provide those here as well.  The reason we are able to just hard code `mode: "development"` here is because this config is only used for the webpack-dev-middleware, which is only used in development.

# Reducing Bundle Size

## Source Maps

The biggest contributor to the size of our production bundle is the source maps that are generated in the bundle itself.  In production mode, we should either generate the source maps as a separate file, or omit the source maps altogether.  Using the new `options.mode` parameter that we set up, this is easy to configure.  Simply change the `devtool` option in web configuration section of the `webpack.config.js` file.

_webpack.config.js_
```js
devtool: options.mode === 'production' ? "source-map" : "cheap-module-eval-source-map",
```

Now let's compile our production code and see how we did:

`npm run build:prod`

```
Child web:
    Hash: 8f75d3b406bbaeaec117
    Time: 16097ms
    Built at: 2018-10-10 10:24:12
                                   Asset      Size  Chunks                    Chunk Names
    e18bbf611f2a2e43afc071aa2f4e1512.ttf  44.3 KiB          [emitted]
    89889688147bd7575d6327160d64e760.svg   106 KiB          [emitted]
                               bundle.js   553 KiB       0  [emitted]  [big]  main
                           bundle.js.map  1.26 MiB       0  [emitted]         main
    Entrypoint main [big] = bundle.js bundle.js.map
     [11] (webpack)/buildin/global.js 509 bytes {0} [built]
     [20] ./node_modules/react-redux/es/index.js + 23 modules 43 KiB {0} [built]
          |    24 modules
     [28] ./src/web/actions/actionTypes.ts 192 bytes {0} [built]
     [32] ./node_modules/react-router/es/index.js + 5 modules 16.5 KiB {0} [built]
          |    6 modules
     [58] (webpack)/buildin/harmony-module.js 573 bytes {0} [built]
     [77] ./src/web/routes.tsx 546 bytes {0} [built]
     [82] ./src/web/components/Channels.tsx + 7 modules 24 KiB {0} [built]
          |    8 modules
     [83] ./src/web/components/Home.tsx + 1 modules 3.22 KiB {0} [built]
          |    2 modules
     [84] multi webpack-hot-middleware/client ./src/web/boot-client.tsx 40 bytes {0} [built]
     [85] (webpack)-hot-middleware/client.js 7.59 KiB {0} [built]
     [86] (webpack)/buildin/module.js 519 bytes {0} [built]
     [89] (webpack)-hot-middleware/client-overlay.js 2.16 KiB {0} [built]
     [94] (webpack)-hot-middleware/process-update.js 4.26 KiB {0} [built]
    [165] ./src/web/util/polyfills.js 1.69 KiB {0} [built]
    [175] ./src/web/boot-client.tsx + 1 modules 1.99 KiB {0} [built]
          | ./src/web/boot-client.tsx 1.11 KiB [built]
          | ./src/web/reducers/reducer.ts 843 bytes [built]
        + 326 hidden modules

    WARNING in asset size limit: The following asset(s) exceed the recommended size limit (244 KiB).
    This can impact web performance.
    Assets:
      bundle.js (553 KiB)

    WARNING in entrypoint size limit: The following entrypoint(s) combined asset size exceeds the recommended limit (244 KiB). This can impact web performance.
    Entrypoints:
      main (553 KiB)
          bundle.js


    WARNING in webpack performance recommendations:
    You can limit the size of your bundles by using import() or require.ensure to lazy load some parts of your application.
    For more info visit https://webpack.js.org/guides/code-splitting/
```

With this change alone, we have reduced the size of our production bundle to about 553 KB.  This is a significant improvement over the 2.25 MB bundle we were generating before, but we can do better.

## Disable Hot Module Replacement

In production, we will not be using hot module replacement, so we might as well remove it from our configuration in production mode.  This involves removing the hot module replacement plugin, as well as the entry point we added.

_webpack.config.js_
```js
    //web configuration
    {
        name: 'web',
        entry: options.mode === 'production' ? [
            './src/web/boot-client.tsx'
        ] : [
            //we configure the hot reloader as a separate entry point to the application
            'webpack-hot-middleware/client',

            './src/web/boot-client.tsx'
        ],
        ...
        plugins: options.mode === 'production' ? [] : [
            new webpack.HotModuleReplacementPlugin()
        ]
    }
```

Now, in production mode, we will only have one entry point, and no plugins.

## Separate CSS

Right now, we are including the entire bootstrap stylesheet in the bundle itself.  Let's put 