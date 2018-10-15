# Part 4 - Improving the Development Experience

## Overview

This part of the tutorial is continued from [Part 3 - Basic Front End Application Skeleton with React-Router and React-Bootstrap](/Part-3-%252D-Basic-Front-End-Application-Skeleton-with-React%252DRouter-and-React%252DBootstrap.md), but it should be applicable to any similarly built application.  Much of the code in this part of the application will be specific to the node server we have configured.

In this part of the tutorial, we will be hooking up and implementing a couple of features to make developing the application easier and faster.

Primary Technologies
 - Webpack
 - Webpack dev middleware
 - Webpack hot module replacement
 - React Hot Loader

## Update existing configuration

There are a couple of options that I chose for the earlier parts of this tutorial that turned out to be non-optimal, incorrect, or simply made the application more complex than necessary.  We will go through these options to correct them.

### tsconfig.json

First, we will switch the tsconfig `lib` option to include `es6`.  Since we are using the same tsconfig.json file for both the server, and the client, it is inconvenient to limit ourselves to es5 code on the server.  Additionally, some third party type definitions will reference things like 'Map' and 'Set', which are only available in es6.  For more information about the consequences of changing this option, please review the section about polyfills on [Part 2 - Basic Front End Application Skeleton with React and Redux](/Part-2-%252D-Basic-Front-End-Application-Skeleton-with-React-and-Redux.md).

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

### webpack.config.js

Next, we will change a few things in the webpack configuration.  First of all, we will give each configuration in the array a name.  In this case, we will name the first configuration `name: 'web'` and the second configuration `name: 'server'`.  This way, we can reference individual configurations by name instead of by index.

In the server configuration, we will change the `node` option to have `__dirname` and `__filename` both be true instead of false.  This will make it so that __dirname and __filename will be preserved relative to the original source code.  That way, when you reference a file relative to your source code, you don't need to know exactly where it will end up being built.  Additionally, this fixes the behavior of `path.resolve( ... )` in some scenarios.

In the web configuration, we will add a `publicPath` field to the `output` option, which will be `'/build/'`.  We will also change the `devtool` option to be `"cheap-module-eval-source-map"`.  This is much faster to build during development, particularly when re-building using the webpack dev middleware we will be configuring later.  However, this will not build anything for production.  If you find yourself in need of a source map in production code, then you can change this to something like `devtool: process.env.NODE_ENV === 'production' ? 'source-map' : 'cheap-module-eval-source-map'`.

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
            publicPath: '/build/',
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

## Install new dependencies

Here is an updated package.json containing all of the dependencies that will be required for this part of the tutorial:

_package.json_
```json
{
  "name": "react-demo",
  "version": "1.0.0",
  "description": "",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "node_modules/.bin/webpack --mode=development",
    "build:prod": "node_modules/.bin/webpack --mode=production",
    "host": "node ./build/main.js --mode=development",
    "host:prod": "node ./build/main.js --mode=production",
    "dev": "node_modules/.bin/webpack --mode=development --config-name=server & node ./build/main.js --mode=development"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@types/es6-promise": "^3.3.0",
    "@types/express": "^4.16.0",
    "@types/react": "^16.4.13",
    "@types/react-bootstrap": "^0.32.13",
    "@types/react-dom": "^16.0.7",
    "@types/react-redux": "^6.0.9",
    "@types/react-router": "^4.0.30",
    "@types/react-router-dom": "^4.3.1",
    "@types/webpack": "^4.4.14",
    "@types/webpack-dev-middleware": "^2.0.2",
    "@types/webpack-env": "^1.13.6",
    "@types/webpack-hot-middleware": "^2.16.4",
    "awesome-typescript-loader": "^5.2.1",
    "bootstrap": "^3.3.7",
    "connected-react-router": "^4.5.0",
    "css-loader": "^1.0.0",
    "es6-promise": "^4.2.5",
    "express": "^4.16.3",
    "file-loader": "^2.0.0",
    "react": "^16.5.0",
    "react-bootstrap": "^0.32.4",
    "react-dom": "^16.5.0",
    "react-hot-loader": "^4.3.11",
    "react-redux": "^5.0.7",
    "react-router": "^4.3.1",
    "react-router-dom": "^4.3.1",
    "redux": "^4.0.0",
    "style-loader": "^0.23.0",
    "typescript": "^3.0.3",
    "url-loader": "^1.1.1",
    "webpack-dev-middleware": "^3.4.0",
    "webpack-hot-middleware": "^2.24.3",
    "webpack-node-externals": "^1.7.2",
    "whatwg-fetch": "^3.0.0"
  },
  "devDependencies": {
    "webpack": "^4.17.2",
    "webpack-cli": "^3.1.0"
  }
}
```

The major additions are:
1. webpack-dev-middleware
2. webpack-hot-middleware
3. TypeScript types for webpack
4. New npm scripts, which I will go into more detail about later

## Webpack Dev Middleware
### Overview

Webpack dev middleware is used to automatically watch for changes to our source code.  We will update our node server to include this as a new middleware.  If you are not using a node server, you can still perform something similar by running webpack in `watch` mode.  See https://webpack.js.org/configuration/watch/#watchoptions for more information.  However, this section will be specific to using a node server running express.

### Update Server

In our server code, we are going to add a new middleware to our express app to configure and enable the dev middleware.

```ts
import * as webpack from 'webpack';
import * as devMiddleware from 'webpack-dev-middleware';
import {Configuration} from "webpack";

const configs: Configuration[] = require('../../webpack.config');
const config = configs.find(config => config.name === "web");

const compiler = webpack({
    ...config,
    mode: "development",
});

//enable dev middleware to rebuild and serve files
this.express.use(devMiddleware(compiler, {
     publicPath: config!.output!.publicPath!
}));
```
I recommend adding this to our `configureMiddleWare` method in the `App.ts` file.

This code will set up a middleware which will monitor for requests to resources in the "/build" folder, and will automatically serve updated files if needed.  Additionally, it will run webpack to build the "web" bundle, and will rebuild the bundle whenever any source file changes that the bundle used.  Please note that the bundle built by the dev middleware is not output to disc, it is stored in memory.  As a result, the files you see in the public/build folder may be out of date.  You can manually run webpack with `npm run build` or `npm run build:prod` to rebuild these files and output them to disc.

## Hot Module Replacement

### Overview

Running webpack-dev-middleware or webpack in `watch` mode allows your bundle code to be kept as up-to-date as possible, but it is still necessary to refresh the page to get the updated resources in your browser.  By using hot module replacement, the resources used by your browser will be automatically updated without completely refreshing the entire application.  This is much faster than a page refresh, and it maintains your application state.

The hot module replacement that I will be going over is specific to running a node server, and requires webpack-dev-middleware to be configured.  If you are not using a node server running express, hot module replacement is still possible.  You will essentially need to run a node express server as a separate process in the background, and have your hot module reloading on the UI connect to it.  I will not be going into detail on how to accomplish this in this tutorial, but if you are using .NET Core, you can use [JavaScript Services](https://github.com/aspnet/JavaScriptServices/tree/master/src/Microsoft.AspNetCore.SpaServices#webpack-hot-module-replacement) for this.

Also keep in mind that support for hot module reloading is implemented at the loader level.  That means that many loaders do not implement hot module reloading, and will simply do a full reload every time.  Fortunately, most of the basic loaders do have support, such as the `style-loader` for inlining css.

### Update Server

In our server code, we are going to add another middleware after our dev middleware which will handle hot module reloading.  Building off of the dev middleware we already configured, this will be very simple:

```ts
import * as hotMiddleware from 'webpack-hot-middleware';

this.express.use(hotMiddleware(compiler));
```

Here is the final code for our express server with both webpack-dev-middleware and webpack-hot-middleware configured:

_App.ts_
```ts
import * as express from "express";
import * as path from "path";
import * as webpack from 'webpack';
import * as devMiddleware from 'webpack-dev-middleware';
import * as hotMiddleware from 'webpack-hot-middleware';
import {Configuration} from "webpack";

const configs: Configuration[] = require('../../webpack.config');
const config = configs.find(config => config.name === "web");

export class App {
    // ref to Express instance
    public express: express.Application;
    //Run configuration methods on the Express instance.
    constructor() {
        this.express = express();

        this.configureMiddleWare();

        //configure endpoints that are handled by api calls
        this.configureApi();

        //static files
        this.express.use(express.static(path.join(__dirname, '../../public')));

        //serve static home page for all remaining requests
        this.express.get("*", (req, res, next) => {
            let filePath: string = path.resolve(__dirname, '../../public/main.html');
            res.sendFile(filePath);
        });
    }

    // Configure Express middleware.
    private configureMiddleWare(): void {

        if(process.env.NODE_ENV === 'development') {

            const compiler = webpack({
                ...config,
                mode: "development",
            });

            //enable hot module replacement part 1
            this.express.use(devMiddleware(compiler, {
                publicPath: config!.output!.publicPath!
            }));

            //enable hot module replacement part 1
            this.express.use(hotMiddleware(compiler));
        }
    }

    // Configure API endpoints.
    private configureApi(): void {

    }
}
```

Both the dev middleware and the hot middleware are added using the `configureMiddleWare()` method, which is called before the other routes/middlewares are configured.  Additionally, these middlewares are only enabled when the server is running in 'development' mode.

### Update client

We will also need to make adjustments to our front end so that it will know to connect to the hot module reloader and reload changes.

#### Update boot-client.tsx

In our `boot-client.tsx` file, we will need to define exactly what can be hot reloaded, and how it will be done.  In the case of this application, I will make it so that the `<Routes />` component can be hot reloaded, as well as its children.  When reloaded, it will simply re-render the react application.

Here is the updated boot-client.tsx file:

_boot-client.tsx_
```ts
import * as React from 'react';
import * as ReactDOM from 'react-dom'
import createBrowserHistory from "history/createBrowserHistory";
import {rootReducer} from "./reducers/reducer";
import {applyMiddleware, compose, createStore} from "redux";
import {Provider} from 'react-redux';
import {ConnectedRouter, connectRouter, routerMiddleware} from "connected-react-router";
import {Routes} from "./routes";

//polyfills for IE
import './util/polyfills';
import 'es6-promise/auto';

//styles
import 'bootstrap/dist/css/bootstrap.css'

const message: string = "this is the client";
console.log(message);

const history = createBrowserHistory();

//configure store based on https://github.com/supasate/connected-react-router
const store = createStore(
    connectRouter(history)(rootReducer),
    compose(
        applyMiddleware(
            routerMiddleware(history)
        ),
        (window as any).devToolsExtension ? (window as any).devToolsExtension() : (f: any) => f
    )
);

function render(rootContainer: JSX.Element) {
    ReactDOM.render(
        <Provider store={store}>
            <ConnectedRouter history={history}>
                {rootContainer}
            </ConnectedRouter>
        </Provider>,
        document.getElementById('root')
    );
}

render(<Routes />);

//configure hot module replacement during development
if(module.hot) {
    module.hot.accept('./routes', () => {
        const NewRoutes = require('./routes').Routes;
        render(<NewRoutes />);
    });
}
```

The major changes include the small refactor so that there is a render method which takes a rootContainer (our `<Routes />` component) which will call `ReactDom.render( ... )`, and the addition of the last 6 lines, which configure hot module reloading.  In this case, the code will allow changes to the `'./routes'` resource to be hot reloaded, and will reload them by calling our new render method with an updated Routes component.

Please note the dynamic require statement.  If you are following the tutorial exactly, this require statement is not actually necessary due to the way ES6 Harmony Modules work.  However, in all other cases it is necessary since that resource will have been changed.  If you try to simply render the Routes class that has already been imported, it will be out of date.  If you are unsure, you can simply include it and it will work either way.

#### Update 'Web' Webpack config

Finally, we will need to update our webpack configuration so that our front end will be able to support hot reloading.  This is done with two changes:
1. Add `'webpack-hot-middleware/client'` as a new entry point to the 'web' configuration.
2. Add `new webpack.HotModuleReplacementPlugin()` as a plugin.

The updated webpack.config.js will look like:
_webpack.config.js_
```js
const path = require('path');
const nodeExternals = require('webpack-node-externals');
const webpack = require('webpack');

var config = [
    //web configuration
    {
        name: 'web',
        entry: [
            './src/web/boot-client.tsx',

            //we configure the hot reloader as a separate entry point to the application
            'webpack-hot-middleware/client'
        ],
        output: {
            path: path.resolve(__dirname, './public/build'),
            publicPath: '/build/',
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
        devtool: "cheap-module-eval-source-map",

        plugins: [
            new webpack.HotModuleReplacementPlugin()
        ]
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

### Try it out

At this point, you can try out hot module reloading.  Start your server with the usual `npm run build` and `npm run host` and navigate to http://localhost:3000 to start the application.  Then make some sort of change to one of the components of the application, such as changing the text in the `Home` component.   When you save your changes to disc, you should see it updated within a few seconds.

![image.png](/.attachments/image-43298dcf-0600-4bdd-ae4d-b5d0c12eb274.png)

If you make a change that cannot be compiled, you will see an overlay with errors:

![image.png](/.attachments/image-c2459aa4-d82b-4ceb-b18b-d28fe357105f.png)


## Hot Reloading and React State

### Overview
Due to the way we decided to handle hot reloading, react components are remounted when they are hot-reloaded.  This means that the component's state will be reset when a hot reload happens.  However, the Redux store will persist.  If you wish to also preserve the state of react components, then you can use [react-hot-loader](https://github.com/gaearon/react-hot-loader) to do this.  The downside is that the react-hot-loader removes some of the current flexibility that we have in our build process.

Additionally, while react-hot-reloader works in 99% of cases, it is possible to create valid react components that cannot be properly reloaded by react-hot-reloader.  Supporting react-hot-loader imposes some limitations on the way components can be defined.  See https://github.com/gaearon/react-hot-loader/issues/1024 for more details.

### Setup

Setting this up is very simple. All we should need to do is go to `boot-client.tsx` and wrap the part we want to hot reload in an `<AppContainer />` element from react-hot-loader, and update some configuration.  However, after messing with this for a while I could not get this to preserve any component state.  Anyway, as of `react-hot-loader` v4, it is recommended to use the new `hot(module)( ... )` api instead of the `<AppContainer />` component, which requires much less configuration.

First, we go to the root component that we want to hot-reload, in this case the `Routes` component in `routes.tsx`, and we wrap the export in a `hot(module)( ... );` call.

_routes.tsx_
```ts
import * as React from 'react';
import {Route, Switch, Redirect} from 'react-router';
import {hot} from 'react-hot-loader'
import {ChannelList} from "./components/Channels";
import {Home} from "./components/Home";

export const Routes = hot(module)(() =>
    <Switch>
        <Route exact path={'/'} component={Home} />
        <Route path={'/channels'} component={ChannelList} />
        <Redirect to={'/'}/>
    </Switch>
);
```

Next, we update boot-client.tsx, reverting it to what we had before we added hot module replacement.

_boot-client.tsx_
```ts
import * as React from 'react';
import * as ReactDOM from 'react-dom'
import createBrowserHistory from "history/createBrowserHistory";
import {rootReducer} from "./reducers/reducer";
import {applyMiddleware, compose, createStore} from "redux";
import {Provider} from 'react-redux';
import {ConnectedRouter, connectRouter, routerMiddleware} from "connected-react-router";
import {Routes} from "./routes";

//polyfills for IE
import './util/polyfills';
import 'es6-promise/auto';

//styles
import 'bootstrap/dist/css/bootstrap.css'
import {AppContainer} from "react-hot-loader";

const message: string = "this is the client";
console.log(message);

const history = createBrowserHistory();

//configure store based on https://github.com/supasate/connected-react-router
const store = createStore(
    connectRouter(history)(rootReducer),
    compose(
        applyMiddleware(
            routerMiddleware(history)
        ),
        (window as any).devToolsExtension ? (window as any).devToolsExtension() : (f: any) => f
    )
);

ReactDOM.render(
    <Provider store={store}>
        <ConnectedRouter history={history}>
            <Routes />
        </ConnectedRouter>
    </Provider>,
    document.getElementById('root')
);
```
As you may be able to tell, the `hot(module)( ... );` approach is more streamlined and handles a lot of the hard stuff for us.  However, it also takes a more opinionated approach at the build process and how we are doing hot module reloading.  We are now dependent on `babel-loader`, which `awesome-typescript-loader` uses internally.  If we want to hot reload something outside of our react components, we will need to configure that separately.

We also lose flexibility with how react components are hot reloaded.  In particular, it is helpful to be able to configure the hot reloader to skip an update with compilation errors.  This is done by default with the way we had configured hot module reloading previously.  The default behavior for react-hot-loader's `hot(module)( ... )` api is to accept errored modules anyway, which will then cause the module to become 'unaccepted' and no longer hot reload until the page is refreshed.  I could not figure out a way to address this.  In theory you should be able to exert more control using the `<AppContainer />` component, but I was not able to get that to correctly preserve the state of the React components.

Whichever way we do it, we still need to have the hot module replacement plugin in webpack, as well as the `webpack-hot-middleware/client` entry point.

### Try it out

To test out hot module replacement while maintaining the state of our react components, we are going to add a new `<Counter />` component to the application, which will be displayed by the `Home` component.

_Counter.tsx_
```ts
import * as React from "react";
import {Row, Col, Button} from "react-bootstrap";

interface params {}

interface localState {
    count: number;
}

export class Counter extends React.Component<params, localState> {
    constructor(p: params) {
        super(p);

        this.state = {
            count: 0
        }
    }

    render() {
        return (
            <Row>
                <Col xs={6}>
                    <div>Count: {this.state.count}</div>
                </Col>
                <Col xs={6}>
                    <Button
                        onClick={e => this.setState({count: this.state.count+1})}
                    >
                        Increment
                    </Button>
                </Col>
            </Row>
        );
    }
}
```
![image.png](/.attachments/image-77be6521-0819-4353-be22-df1952c3591b.png)

We will update `Home.tsx` to import and use the new `Counter.tsx` component:

_Home.tsx_
```ts
import * as React from "react";
import {Link} from "react-router-dom";
import {Row, Col, Grid, Panel} from "react-bootstrap";
import {Counter} from "./Counter";

export class Home extends React.Component<{}> {
    constructor(p: {}) {
        super(p);
    }

    render() {
        return (
            <Grid>
                <Row>
                    <Col xs={6}>
                        <Panel>
                            <Panel.Heading>App Root</Panel.Heading>
                            <Panel.Body>
                                <Counter/>
                            </Panel.Body>
                            <Panel.Footer>
                                <Link to='/channels'>Go to channel list</Link>
                            </Panel.Footer>
                        </Panel>
                    </Col>
                    <Col xs={6}>
                        Content on right half of screen
                    </Col>
                </Row>
            </Grid>
        );
    }
}
```

Now, you can build and run the application and try it out for yourself.  Simply increment the counter and then make some kind of change to `Counter.tsx` or `Home.tsx`.  You will see the change get applied while maintaining the current count.

![image.png](/.attachments/image-6b45bf49-90f2-4418-bc10-c5b98052f01d.png)

## Add Better NPM Script

So far, we have been building and running our code using two separate scripts, `npm run build` and `npm run host`.  You may also have noticed that since we enabled the webpack dev middleware that automatically rebuilt our code, it was necessary to wait for the web code to be built a second time after a server restart.  To address these issues, we will be using a new NPM script to both build and run our code for development.

_package.json_
```json
{
  "scripts": {
    "dev": "node_modules/.bin/webpack --mode=development --config-name=server & node ./build/main.js --mode=development"
  }
}
```

This script will first build only the server code, then it will run the server in development mode.  The server will then automatically build the web code as a part of the webpack dev middleware.  Please note that as a result, the web bundle that is built will never be output to disc in development mode.  You can manually run `npm run build` to output the web bundle.

The dev middleware also has an option to output the code it builds to disc.  However, this will also cause the hot-update files to be written to disc as well, so I would not recommend it.  Just keep in mind that your bundle file on disc may be out of date.

## Hot Reloading CSS

### Overview

If you have already configured part of your application to hot reload using either webpack-hot-middleware or react-hot-loader, then you already support hot reloading css.  All you need to do is make sure you are using `style-loader` to handle css (or less/scss), and any updates to your styles will be automatically reloaded.  You also have to make sure you are importing them through javascript via `import` statements, and not just adding the stylesheet directly to your html bundle.  You don't even need to import the css in a react module being hot reloaded, so you can import directly in `boot-client.tsx` if you wish.

### CSS Extracting Plugins

While it is recommended to use a plugin like [Extract Text Plugin](https://github.com/webpack-contrib/extract-text-webpack-plugin) or [mini-css-extract-plugin](https://github.com/webpack-contrib/mini-css-extract-plugin) to separate your css code from your JavaScript bundle in production, this will prevent your css from being hot reloaded.  In some cases, this might cause other issues when hot reloading non-css resources.  You should set up your webpack config so that these plugins are only used during production.

### Try it out

To test out reloading css, we are going to create a new file called `Counter.css` and place it in a new folder called `styles`.

_Counter.css_
```css
.my-counter {
    color: blue;
    padding-top: 6px;
}
```
![image.png](/.attachments/image-e277aecd-af3e-412c-a573-2de2ff5839a3.png)

In `Counter.tsx`, we are going to simply add a line to import it:

`import '../styles/Counter.css';`

Then we are going to add a `className` property to the div containing the 'Counter: ...' text.

`<div className={'my-counter'}>Counter: {this.state.count}</div>`

Now we can go ahead and run the demo, and you will see that the 'Counter: ...' text is now blue and better centered vertically.

![image.png](/.attachments/image-d4ac532c-f0ee-49ae-86e5-5535f28da1cb.png)

We can edit the properties of the `my-counter` style and have them immediately reload.  For example, we can change the text color from `blue` to `red`:

![image.png](/.attachments/image-dcfce9d4-2720-4052-8ab8-c758794636ce.png)

You will see the text turn red, without reloading the component itself.

## Download Source

https://github.com/mjhoffm2/react-demo/tree/Part-4

Up Next: [Part 5 - Bundling for Production](/Part-5-%252D-Bundling-for-Production.md)