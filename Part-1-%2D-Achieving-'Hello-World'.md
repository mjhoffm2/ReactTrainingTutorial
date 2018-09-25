# Overview

In this project, I will be using a node server running express.  However, this guide is mostly independent of the actual server architecture used.  I chose node because it requires very little configuration, and because we will already need to use it to run webpack.

# Prerequisites

Before starting this tutorial, you should make sure to install npm with Node 8+.  You can get both from here: https://www.npmjs.com/get-npm

You should also use an IDE with TypeScript support.  Visual Studio, Eclipse, and IntelliJ all have extensions for TypeScript.  For this project, if you are not already using one of these IDEs, I recommend using either VSCode or WebStorm.  Both require little to no additional configuration, and have excellent TypeScript support.

# Initialize the project

Initialize the project with a package.json file in the root directory.  This can be done by using the command `npm init -y` to create a default package.json file and then installing individual dependencies.  To save time, I will provide a starting package.json that contains the dependencies we need for now.

_package.json_
```json
{
  "name": "react-demo",
  "version": "1.0.0",
  "description": "",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
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

Next, we need to set up a tsconfig.json and webpack.config.js file.  Again, I will provide starting versions to save time.

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

For the webpack.config.js file, we will need to decide something about our project structure.  Here is the folder structure that I will be using for this tutorial

![image.png](/.attachments/image-df1364c9-dfaa-4fae-9cf4-58315d0cf16c.png)
