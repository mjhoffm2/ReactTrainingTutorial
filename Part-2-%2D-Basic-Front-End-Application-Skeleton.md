[[_TOC_]]


## Overview

This part of the tutorial is continued from [Part 1 - Achieving 'Hello World'](/Part-1-%2D-Achieving-'Hello-World').

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

There are three things that must be set up when using Redux.
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

_reducer.ts_
```ts
import * as defs from '../definitions/definitions';
import {Action} from "../actions/actionTypes";
import {Reducer} from "redux";

const initialState: defs.State = { users: null, channels: null };

export const rootReducer: Reducer<defs.State> = (state = initialState, action: Action) => state;
```

The reducer is initially called with a state of `undefined`, in which case your reducer should assume an initial state.  Note that it is not allowed for the reducer to return `undefined`.  In this case, the initial state set by the reducer has the `users` and `channels` properties set to `null`.  Besides setting the initial state, this reducer does nothing, and will ignore any actions passed to it and always return the same state.  Let's change that:

_reducer.ts_
```ts
import * as defs from '../definitions/definitions';
import {Action, ActionTypes} from "../actions/actionTypes";
import {Reducer} from "redux";

const initialState: defs.State = { users: null, channels: null };

export const rootReducer: Reducer<defs.State> = (state = initialState, action) => {
    switch(action.type) {
        case ActionTypes.LOAD_USERS: {
            return {
                ...state,
                users: action.users
            };
        }
        case ActionTypes.LOAD_CHANNELS: {
            return {
                ...state,
                channels: action.channels
            };
        }
    }
    return state;
};
```

In this new root reducer, we check the type of the action, and will return a new state if it is one of the two actions that we defined.  It is important to return the same state as before if the actions is not matched, since redux will check to see if the reference returned by your reducer is the same as the reference that was provided to detect if any changes were made.  If you are unfamiliar with the `...` syntax, that is called a spread operator.  Essentially, that will create a shallow copy of the object.  This is very often seen in React and Redux due to the fact that state needs to be kept immutable.  Being able to create a shallow copy of an object with a couple of differences in a concise way is very useful.  In this case, the spread operator isn't doing much yet, but as we add more fields to our State, we won't have to go back and update these parts of our reducer.

As our State grows more and more complex, it is generally a good idea to separate out your reducer into several functions, and compose them together into one larger function.  A common pattern for this is to have one reducer for each top level field in the state.

_reducer.ts_
```ts
import * as defs from '../definitions/definitions';
import {Action, ActionTypes} from "../actions/actionTypes";
import {combineReducers, Reducer} from "redux";

const initialUserState: defs.State['users'] = null;

export const userReducer: Reducer<defs.State['users']> = (state = initialUserState, action) => {
    switch(action.type) {
        case ActionTypes.LOAD_USERS: {
            return action.users;
        }
    }
    return state;
};

const initialChannelState: defs.State['channels'] = null;

export const channelReducer: Reducer<defs.State['channels']> = (state = initialChannelState, action) => {
    switch(action.type) {
        case ActionTypes.LOAD_CHANNELS: {
            return action.channels;
        }
    }
    return state;
};

export const rootReducer = combineReducers<defs.State, Action>({
    users: userReducer,
    channels: channelReducer
});
```

In this example, we have one reducer which is only concerned with the `users` part of the state, and another reducer which is only concerned with the `channels` part of the state.  These two separate reducers are then composed together into one `rootReducer` using the `combineReducers` utility.  This may look like overkill in our simple application so far, but as your application grows and the state becomes more deeply nested and complex, this is a good way to organize your code.

### The Store

Of course, at this point all we have done is defined some type interfaces and a function called `rootReducer`.  We still need to set up redux to use this function as our reducer in our application.  This is done by calling the `createStore` method.

_boot-client.tsx_
```ts
import { createStore } from "redux";
import { rootReducer } from "./reducers/reducer";
...

const store = createStore(rootReducer);
...
```

This creates our store using our reducer, which is now ready for use.  If we want, we can manually dispatch actions to the store using `store.dispatch( ... )`, and get the current state using `store.getState()`.  We can also subscribe to changes using `store.subscribe( ... )` and update the reducer using `store.replaceReducer( ... )`.  However, we will not be calling any of these directly.  Instead we will use the React-Redux library to manage all of these for us, among other things.

## React-Redux

## React-Router