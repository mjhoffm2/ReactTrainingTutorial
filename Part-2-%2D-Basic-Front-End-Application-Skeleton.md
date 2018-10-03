## Overview

In this part of the tutorial, we will be hooking up various frameworks to our front end application, and learning how to use them.  While we will not be implementing any real features, we will still be laying the groundwork for how these frameworks will be used in our application.  This will serve as a foothold for more interesting  features as they are implemented.  For now, most of our components will be example components, and our build process is going to be sub-optimal.  Those things will come later.

Primary technologies in this part of the tutorial:
 - React
 - TypeScript
 - Redux
 - React-Router
 - React-Bootstrap

## Install New Dependencies

As you would expect, we need to download all of the above technologies as dependencies from NPM.  To save time, I will provide an updated package.json file with the modules required for now.

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
    "host": "node ./build/main.js"
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
    "react-redux": "^5.0.7",
    "react-router": "^4.3.1",
    "react-router-dom": "^4.3.1",
    "redux": "^4.0.0",
    "style-loader": "^0.23.0",
    "typescript": "^3.0.3",
    "url-loader": "^1.1.1",
    "webpack-node-externals": "^1.7.2",
    "whatwg-fetch": "^3.0.0"
  },
  "devDependencies": {
    "webpack": "^4.17.2",
    "webpack-cli": "^3.1.0"
  }
}
```

A couple of things to note about the specific dependencies chosen:
1. Bootstrap 3.3.7 is chosen since that is the peer dependency targeted by react-bootstrap.  We will only be using the stylesheets and static resources from it.
2. Connected-react-router is chosen since it supports react-router 4.x.x.  The alternative is react-router-redux, which is deprecated.

Simply update or replace your existing package.json file, and run `npm install` to update your project's dependencies.

## Redux

### Introduction
This is not a comprehensive tutorial for using Redux, but I will explain some of the basics.  For more information, you should read the official documentation: https://redux.js.org/

There are three things that are required when using Redux.
1. The **State** - Your application state should be contained in a single serializable structure.  It should also be immutable.
2. The **Actions** - Any update to the State should be expressed as an action.  These are simple serializable objects which contain arbitrary information related to your application.  These must be dispatched to redux.
3. The **Reducer** - The reducer is a pure function which takes in the current State and an Action, and returns the next State.  Whenever an Action is dispatched to redux, the Reducer is called to calculate the new state.  This function should have no side effects, so the old state should not be modified.  Typically, multiple reducers are composed together into one root Reducer for the entire state.

These three requirements are both a blessing and a curse.  While this imposes a lot of restrictions on the way you can manage data in your application, it also forces a consistent and understandable pattern with good separation of concerns.  I have written more about this in the [React Knowledge Repository](https://sites.google.com/a/echelon.org/test/Home/react-knowledge-repository#TOC-Pros-and-Cons-to-using-Redux)

### The State

When integrating Redux into your application, it is usually easiest to start by designing the State and deciding what information needs to be held in it.  First, we will create a simple TypeScript interface which defines the shape of our state.  For this basic application skeleton, we will use the following:

_stateDefinitions.ts_
```ts
export interface State {
    users: User[] | null;
    channels: Channel[] | null;
}

export interface User {
    userId: number;
    username: string;
    displayName: string | null;
    email: string | null;
    status: string | null;
    description: string | null;
}

export interface Channel {
    channelId: number;
    ownerId: number | null;
    displayName: string;
    isPublic: boolean;
    canAnyoneInvite: boolean;
    isGeneral: boolean;
    isActiveDirectMessage: boolean;
    owner?: User;
}
```

In the above code, we have defined an interface called `State` which describes an object with two properties, `users` and `channels`.  Each of these properties can either contain `null`, or contain an array of a corresponding object, the interfaces for which are defined in the same code.  These interfaces are based on the requirements for the [Slack Training App](/Slack-Training-App-%2D-Tutorial-Overview), but we could have chosen to design our state in any way we saw fit for the purposes of this part of the tutorial.

As a reminder, this code is merely a type definition, no JavaScript will actually get created from it.

### The Actions

Our skeleton application for this part of the tutorial will only have two actions.  One of them will load a list of users, the other one will load a list of channels.  Here is the code for them:

_actionTypes.ts_
```ts
import * as defs from '../definitions/definitions';

export enum ActionTypes {
    LOAD_USERS = "LOAD_USERS",
    LOAD_CHANNELS = "LOAD_CHANNELS"
}

export type Action = loadUsersAction | loadChannelsAction;

export interface loadUsersAction {
    type: ActionTypes.LOAD_USERS;
    users: defs.User[];
}

export interface loadChannelsAction {
    type: ActionTypes.LOAD_CHANNELS;
    channels: defs.Channel[];
}
```

We will dispatch these actions after completing some kind of load.  The actions contain all the information necessary for our reducer (to be made later) to calculate the next state, which will contain the loaded users/channels.

Again, it is important to understand that the code above is still just type definitions, and no JavaScript will actually get created from it (enums can optionally be represented as runtime objects).

### The Reducer

Finally, we need to bring everything together by creating a reducer to handle our actions and calculate the next state.  As a reminder, the Reducer is a pure function which takes the current state and an action as parameters, and returns the next state.  The Reducer in this case is also responsible for setting the default state.  We could define a trivial reducer that does nothing as follows:

```ts
import * as defs from '../definitions/definitions';
import {Action} from "../actions/actionTypes";
import {Reducer} from "redux";

const initialState: defs.State = { users: null, channels: null };

export const rootReducer: Reducer<defs.State> = (state = initialState, action: Action) => state;
```

The reducer is initially called with a state of `undefined`, in which case your reducer should assume an initial state.  Note that it is not allowed for the reducer to return `undefined`.  In this case, the initial state set by the reducer has the `users` and `channels` properties set to `null`.  Besides setting the initial state, the reducer does nothing, and will ignore any actions passed to it and always return the same state.  Let's address that:



## React-Router