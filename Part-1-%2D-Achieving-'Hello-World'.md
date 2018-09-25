[[_TOC_]]

# Overview

In this project, I will be using a node server running express.  However, this guide is mostly independent of the actual server architecture used.  I chose node because it requires very little configuration, and because we will already need to use it to run webpack.

# Prerequisites

Before starting this tutorial, you should make sure to install npm with Node 8+.  You can get both from here: https://www.npmjs.com/get-npm

You should also use an IDE with TypeScript support.  Visual Studio, Eclipse, and IntelliJ all have extensions for TypeScript.  For this project, if you are not already using one of these IDEs, I recommend using either VSCode or WebStorm.  Both require little to no additional configuration, and have excellent TypeScript support.

# Initialize the project

## The package.json

Initialize the project with a package.json file in the root directory.  This can be done by using the command `npm init -y` to create a default package.json file and then installing individual dependencies.

A package.json is used by npm to describe the project and manage dependencies.  We will also be using the package.json file to run our build/deploy scripts.  For more information about it, you can refer to the official npm documentation: https://docs.npmjs.com/getting-started/using-a-package.json

To save time, I will provide a starting package.json that contains the dependencies we need for now.

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
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@types/express": "^4.16.0",
    "@types/react": "^16.4.13",
    "@types/react-bootstrap": "^0.32.13",
    "@types/react-dom": "^16.0.7",
    "@types/react-router": "^4.0.30",
    "awesome-typescript-loader": "^5.2.1",
    "css-loader": "^1.0.0",
    "express": "^4.16.3",
    "react": "^16.5.0",
    "react-bootstrap": "^0.32.4",
    "react-dom": "^16.5.0",
    "react-router": "^4.3.1",
    "redux": "^4.0.0",
    "style-loader": "^0.23.0",
    "typescript": "^3.0.3",
    "url-loader": "^1.1.1",
    "webpack-node-externals": "^1.7.2"
  },
  "devDependencies": {
    "webpack": "^4.17.2",
    "webpack-cli": "^3.1.0"
  }
}
```

With a console open to the same folder that the package.json is located in, run `npm install` to download all the required dependencies.

## The tsconfig.json

Next, we need to set up a tsconfig.json file.  This will be used by the TypeScript compiler to convert the TypeScript we write into JavaScript.  Most IDEs will also use this for syntax highlighting and improved code completion.  Again, I will provide a minimal starting version to save time:

_tsconfig.json_
```json
{
  "compilerOptions": {
    //use strict type checking
    "strict": true,

    "jsx": "react",

    //skip type checking .d.ts files, particularly from third party libraries
    "skipLibCheck": true
  },
  "exclude": [
    "node_modules"
  ]
}
```

We will be invoking the TypeScript compiler as a part of our webpack build process, so we don't need to use it to transpile down to browser-compatible code yet.

## The webpack.config.js

The last initialization related file we will need is the webpack.config.js file.  This is used to configure our webpack build process.  This involves things like setting up the entry point, output files, and the loaders used to process each type of file.  Before we can create it, we will need to decide something about our project structure.  Here is the folder structure that I will be using for this tutorial:

![image.png](/.attachments/image-df1364c9-dfaa-4fae-9cf4-58315d0cf16c.png)

With that structure in mind, here is the starting webpack config:

_webpack.config.js_

```js
const path = require('path');
const nodeExternals = require('webpack-node-externals');

var config = [
    //web configuration
    {
        entry: ['./src/web/boot-client.tsx'],
        output: {
            path: path.resolve(__dirname, './public'),
            filename: 'build/bundle.js',
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
                    test: /\.(png|jpg|jpeg|gif|svg|ttf|otf)$/,
                    loader: 'url-loader?limit=25000'
                },
            ]
        },
        devtool: "source-map"
    }
];
module.exports = config;
```

The important things that the webpack configuration is doing is:
1. Define the entry point.
2. Define the loaders that will be used for different files.
3. Configure how the final JavaScript bundle will be put together.
3. Define the output location and name.

More information about configuring webpack can be found from the official documentation: https://webpack.js.org/configuration/

## The main.html

We will also need an html file, which we will name `main.html` and place in the `public` folder:

_main.html_
```html
<!DOCTYPE html>
<html>
<head>
    <title>Demo</title>
</head>
<body>
<div id="root"></div>
<script src="./build/bundle.js"></script>
</body>
</html>
```

This simply creates a single html element for react to mount to, and includes the bundle created by webpack as a script.

# Create the React App

Our front end will be implemented using React, but for this part of the tutorial we will only go as far as printing out "Hello World" to the dom.

Our one component is going to be very simple:

_AppRoot.tsx_
```ts
import * as React from "react";

export class AppRoot extends React.Component<{}> {
    constructor(p: {}) {
        super(p);
    }

    render() {
        return "Hello World";
    }
}
```

and the entry point to our client javascript:

_boot-client.tsx_
```ts
import * as React from 'react';
import * as ReactDOM from 'react-dom'
import {AppRoot} from "./components/AppRoot";

const message: string = "this is the client";
console.log(message);

ReactDOM.render(<AppRoot/>, document.getElementById('root'));
```

These files will be located in the project as follows:

![image.png](/.attachments/image-27a3ede8-8e66-486a-b6ec-8b400c88a4f2.png)

Based on the client (web) part of our webpack.config.json file, boot-client.tsx is our entry point, and our code will be output as a single bundle.js file in the `public/build` folder.

We can now build our front end code by running `npm run build` in the console.  After that, you should be able to open the main.html file in a web browser and see the app in action.

# Setting up the server

Just opening up the application as a static file in a web browser isn't going to cut it going forward, so let's set up a server to actually serve the bundle.  Since we are already using TypeScript for the front end, we will be using it for the server as well.  We will need to update our configuration to support this.

## Update the webpack.config.js

We will update the existing webpack.config.js we already have to include a new server configuration.

_webpack.config.js_
```js
const path = require('path');
const nodeExternals = require('webpack-node-externals');

var config = [
    //web configuration
    {
        entry: ['./src/web/boot-client.tsx'],
        output: {
            path: path.resolve(__dirname, './public'),
            filename: 'build/bundle.js',
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
                    test: /\.(png|jpg|jpeg|gif|svg|ttf|otf)$/,
                    loader: 'url-loader?limit=25000'
                },
            ]
        },
        devtool: "source-map"
    },

    //server configuration
    {
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
            __dirname: false,
            __filename: false
        }
    }
];
module.exports = config;
```

We have separate configurations for the server and the client, since we need to compile both from TypeScript to JavaScript.  We could set up the project to avoid using webpack to pre-process the server code, but I opted to go down this route.

We could also use separate tsconfig.json files for both the server and the client.  I have tried this before and it works well, but I have also found that IDE's tend to not pick up the correct one.  Therefore, we will just be using the one generic tsconfig file.

## Server Code

We will configure our server to run from port 3000 on our localhost, and will server the static main.html file when a user requests the "/" url.  The following code will accomplish this:

_App.ts_
```ts
import * as express from "express";
import * as path from "path";

export class App {
    // ref to Express instance
    public express: express.Application;
    //Run configuration methods on the Express instance.
    constructor() {
        this.express = express();
        this.configureMiddleWare();
        this.configureApi();

        //static files
        this.express.use(express.static(path.join(__dirname, '/../public')));

        //temporary static home page
        this.express.get("/", (req, res, next) => {
            let filePath: string = path.resolve(__dirname, '../public/main.html');
            res.sendFile(filePath);
        });
    }

    // Configure Express middleware.
    private configureMiddleWare(): void {

    }

    // Configure API endpoints.
    private configureApi(): void {

    }
}
```

_main.ts_
```ts
import * as http from 'http';
import {App} from "./App";

console.log("this is the server");

console.log(`server env: ${process.env.NODE_ENV}`);

const app = new App().express;
const port =  3000;
app.set('port', 3000);
//create a server and pass our Express app to it.
const server = http.createServer(app);
server.listen(port);
server.on('listening', onListening);

//function to note that Express is listening
function onListening(): void {
    console.log(`Listening on port ${port}`);
}
```
![image.png](/.attachments/image-69ec849e-dd38-4aac-9b7e-ee1bbf79105a.png)

Since node doesn't natively understand TypeScript, we have included this as a part of our webpack build process.  Webpack will convert the TypeScript code into JavaScript that node can understand and run.  Now, when you run `npm run build` in the console, you should find that in addition to the client being compiled into a `bundle.js` file, the server code will be built into the build folder as `main.js`.

# Running the demo

We can now run the 2 following commands to compile our code and run the server

`npm run build`
```
> npm run build

> react-demo@1.0.0 build C:\Users\Matt\Documents\gitkraken\react-demo\node
> webpack --mode=development

i ｢atl｣: Using typescript@3.0.3 from typescript
i ｢atl｣: Using tsconfig.json from C:/Users/Matt/Documents/gitkraken/react-demo/node/tsconfig.json
i ｢atl｣: Using typescript@3.0.3 from typescript
i ｢atl｣: Using tsconfig.json from C:/Users/Matt/Documents/gitkraken/react-demo/node/tsconfig.json
i ｢atl｣: Checking started in a separate process...
i ｢atl｣: Time: 74ms
i ｢atl｣: Checking started in a separate process...
i ｢atl｣: Time: 36ms
Hash: 743e889d2595ff9c98dbfff76cc22d007244b3c0
Version: webpack 4.20.1
Child
    Hash: 743e889d2595ff9c98db
    Time: 5933ms
    Built at: 09/25/2018 4:39:12 PM
                  Asset     Size  Chunks             Chunk Names
        build/bundle.js  744 KiB    main  [emitted]  main
    build/bundle.js.map  861 KiB    main  [emitted]  main
    Entrypoint main = build/bundle.js build/bundle.js.map
    [./src/web/boot-client.tsx] 319 bytes {main} [built]
    [0] multi ./src/web/boot-client.tsx 28 bytes {main} [built]
        + 12 hidden modules
Child
    Hash: fff76cc22d007244b3c0
    Time: 5259ms
    Built at: 09/25/2018 4:39:12 PM
      Asset      Size  Chunks             Chunk Names
    main.js  6.89 KiB    main  [emitted]  main
    Entrypoint main = main.js
    [./src/server/App.ts] 896 bytes {main} [built]
    [./src/server/main.ts] 541 bytes {main} [built]
    [express] external "express" 42 bytes {main} [built]
    [http] external "http" 42 bytes {main} [built]
    [path] external "path" 42 bytes {main} [built]
    [0] multi ./src/server/main.ts 28 bytes {main} [built]
```
`node ./build/main.js`
```
> npm run host

> react-demo@1.0.0 host C:\Users\Matt\Documents\gitkraken\react-demo\node
> node ./build/main.js

this is the server
server env: development
Listening on port 3000
```

Once the server is running, you can navigate to `localhost:3000` in a web browser to check that it is working.

![image.png](/.attachments/image-22c56936-f5e9-4908-a47e-ff8bf39cae7f.png)

I also recommend adding an npm script to the package.json file to host the server:

```json
  ...
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "node_modules/.bin/webpack --mode=development",
    "build:prod": "node_modules/.bin/webpack --mode=production",
    "host": "node ./build/main.js"
  },
  ...
```

This lets us use `npm run host` instead of `node ./build/main.js` in the console.

The complete code for this part of the tutorial can be found here: https://dev.azure.com/echeloncons/_git/Slack%20Training%20App?version=GCe46523c72431d8837e8ec6de566bbb126867bda8