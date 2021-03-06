# Part 2 - Basic Front End Application Skeleton with React and Redux

## Overview

This part of the tutorial is continued from [Part 1 - Achieving 'Hello World'](/Part-1-%252D-Achieving-'Hello-World'.md).

In this part of the tutorial, we will be hooking up various frameworks to our front end application, and learning how to use them.  While we will not be implementing any real features, we will still be laying the groundwork for how these frameworks will be used in our application.  This will serve as a foothold for more interesting  features as they are implemented.  For now, most of our components will be example components, and our build process is going to be sub-optimal.  Those things will come later.

Primary technologies in this part of the tutorial:
 - React
 - TypeScript
 - Redux
 - React-Redux

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
    "@types/react": "^16.7.17",
    "@types/react-bootstrap": "^0.32.13",
    "@types/react-dom": "^16.0.7",
    "@types/react-redux": "^6.0.11",
    "@types/react-router": "^4.0.30",
    "@types/react-router-dom": "^4.3.1",
    "awesome-typescript-loader": "^5.2.1",
    "bootstrap": "^3.3.7",
    "connected-react-router": "^5.0.1",
    "css-loader": "^1.0.0",
    "es6-promise": "^4.2.5",
    "express": "^4.16.3",
    "file-loader": "^2.0.0",
    "react": "^16.6.3",
    "react-bootstrap": "^0.32.4",
    "react-dom": "^16.5.0",
    "react-redux": "^5.1.1",
    "react-router": "^4.3.1",
    "react-router-dom": "^4.3.1",
    "redux": "^4.0.0",
    "style-loader": "^0.23.0",
    "typescript": "^3.2.2",
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


## Polyfills

### Overview

Before we get too far, it is important that we have an understanding of polyfills and when we need them.  Even though we can transpile code from ES6 to ES3/5, this only transforms the syntax.  The actual methods and features available vary from browser to browser, and anything not available must be polyfilled.  If we wish to use ES6 promises, we need to use a polyfill to add support for Internet Explorer.

For example, if we wrote the following code:

```ts
const myFunction = () => Promise.resolve(null);
```

When targeting es3/5, this might be transpiled into the following:

```js
var myFunction = function() { return Promise.resolve(null); };
```

While transpilation can turn `const` into `var` and arrow functions into regular functions, it will leave things like `Promise.Resolve( ... )` unchanged.

### Example

In our `tsconfig.json`, we declare what libraries we expect to be available in the user's browser.  This will cause TypeScript to throw an error whenever we use a feature that is not part of those libraries.  Take the following `tsconfig.json` file for example:

_tsconfig.json_
```ts
{
  "compilerOptions": {
    //Use strict type checking
    //This makes it so 'null' and 'undefined' are considered explicit values
    //It shows type errors where types could not be inferred, among other things
    "strict": true,

    //Transforms react JSX syntax into React.createElement() calls
    "jsx": "react",

    //Tell the TypeScript compiler what libraries we expect to exist
    //In this case, we expect the user's browser to have ES5 support and a dom
    //We provide a polyfill for es2015 promises, and other polyfills provide their own definitions
    //Another common approach is to simply declare that the user's browser has es6 support, and then polyfill anything necessary
    "lib": ["es5", "dom", "es2015.promise"],

    //This causes ES6 modules to be used instead of CommonJS
    //This is important since it enables webpack to do tree shaking optimizations (dead code removal)
    //This is different from the compilation target, which is still ES3 by default
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
In the above code, we have declared that we expect the user's browser to support `es5`, `dom` and `es2015.promise`.  Since not all browsers actually support promises, we will need to provide a polyfill for it if we want our application to work on those browsers.

To do this, we will add the following in the entry point of our application:

```ts
import 'es6-promise/auto';
```
This library will automatically polyfill promises into the user's browser, if necessary.

### Type-Safe Polyfills

If we want to use other features from ES6 such as `Array.prototypes.find( ... )`, then we will need to polyfill that as well.  However, there isn't a `lib` option specific to this `find` method, so we will also need to provide our own type definition for it.

_polyfills.js_
```js
//polyfill for array.find
//https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/find
// https://tc39.github.io/ecma262/#sec-array.prototype.find
if (!Array.prototype.find) {
    Object.defineProperty(Array.prototype, 'find', {
        value: function(predicate) {
            // 1. Let O be ? ToObject(this value).
            if (this == null) {
                throw new TypeError('"this" is null or not defined');
            }

            var o = Object(this);

            // 2. Let len be ? ToLength(? Get(O, "length")).
            var len = o.length >>> 0;

            // 3. If IsCallable(predicate) is false, throw a TypeError exception.
            if (typeof predicate !== 'function') {
                throw new TypeError('predicate must be a function');
            }

            // 4. If thisArg was supplied, let T be thisArg; else let T be undefined.
            var thisArg = arguments[1];

            // 5. Let k be 0.
            var k = 0;

            // 6. Repeat, while k < len
            while (k < len) {
                // a. Let Pk be ! ToString(k).
                // b. Let kValue be ? Get(O, Pk).
                // c. Let testResult be ToBoolean(? Call(predicate, T, « kValue, k, O »)).
                // d. If testResult is true, return kValue.
                var kValue = o[k];
                if (predicate.call(thisArg, kValue, k, o)) {
                    return kValue;
                }
                // e. Increase k by 1.
                k++;
            }

            // 7. Return undefined.
            return undefined;
        },
        configurable: true,
        writable: true
    });
}
```

_polyfills.d.ts_
```ts
interface Array<T> {
    /**
     * Returns the value of the first element in the array where predicate is true, and undefined
     * otherwise.
     * @param predicate find calls predicate once for each element of the array, in ascending
     * order, until it finds one where predicate returns true. If such an element is found, find
     * immediately returns that element value. Otherwise, find returns undefined.
     * @param thisArg If provided, it will be used as the this value for each invocation of
     * predicate. If it is not provided, undefined is used instead.
     */
    find<S extends T>(predicate: (this: void, value: T, index: number, obj: T[]) => value is S, thisArg?: any): S | undefined;

    find(predicate: (value: T, index: number, obj: T[]) => boolean, thisArg?: any): T | undefined;
}

```

Then we import the polyfills in the entry point of our application:

```ts
import './util/polyfills';
```

From this point on, we have access to the `.find( ... )` method on our arrays wherever we use them.

### Non-Type-Safe Polyfills

Alternatively, another common approach is to simply do the following:

```ts
{
  "compilerOptions": {
    //Use strict type checking
    //This makes it so 'null' and 'undefined' are considered explicit values
    //It shows type errors where types could not be inferred, among other things
    "strict": true,

    //Transforms react JSX syntax into React.createElement() calls
    "jsx": "react",

    //Tell the TypeScript compiler what libraries we expect to exist
    //In this case, we expect the user's browser to have ES6 support (haha) and a dom
    //The idea is that we will polyfill anything additional that we use from es6 without needing our own type definitions for them
    "lib": ["es6", "dom"],

    //This causes ES6 modules to be used instead of CommonJS
    //This is important since it enables webpack to do tree shaking optimizations (dead code removal)
    //This is different from the compilation target, which is still ES3 by default
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

In this example, we have simply declared that we expect the user's browser to have full es6 support.  This is of course not going to be the case, but the idea is that we will polyfill whatever es6 methods and features we need without needing to provide a separate type definition for them.  TypeScript will simply pull those definitions from the built-in es6 type library.  The downside is that TypeScript will not show an error if you forget to polyfill something, and you need to make sure that the polyfills you use match the built-in definitions.

# Redux

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

In the above code, we have defined an interface called `State` which describes an object with two properties, `users` and `channels`.  Each of these properties can either contain `null`, or contain an array of a corresponding object, the interfaces for which are defined in the same code.  These interfaces are based on the requirements for the [Slack Training App](https://github.com/mjhoffm2/ReactTrainingTutorial/blob/master/Slack-Training-App-%252D-Tutorial-Overview.md), but we could have chosen to design our state in any way we saw fit for the purposes of this part of the tutorial.

As a reminder, this code is merely a type definition, no JavaScript will actually get created from it.

I recommend organizing the various types and interfaces in your application into different definition files.  In this case, interfaces related to our Redux store will be stored in the `stateDefinitions.ts` file.  You can then have a single central `definitions.ts` file which will import and re-export all of the definitions from your various other definition files.  This way, you can decide to re-orgnaize and refactor your types and interfaces, and the rest of your codebase doesn't need to be updated.

_definitions.ts_
```ts
export * from './stateDefinitions';
```

For this tutorial, I will place these files in a `definitions` folder inside the `src/web` directory:

![image.png](/.attachments/image-08680fef-6893-4988-ade7-3a551a40e675.png)

You can disregard the `actions` folder and `preferences.ts` file for now.

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

For this tutorial, I will place everything related to Redux actions in an `actions` folder located in the `src/web` directory:

![image.png](/.attachments/image-4d9e60f8-0397-4a03-adc3-d392123de82e.png)


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

export const userReducer: Reducer<defs.State['users'], Action> = (state = initialUserState, action) => {
    switch(action.type) {
        case ActionTypes.LOAD_USERS: {
            return action.users;
        }
    }
    return state;
};

const initialChannelState: defs.State['channels'] = null;

export const channelReducer: Reducer<defs.State['channels'], Action> = (state = initialChannelState, action) => {
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

For this tutorial, I will place everything related to Redux reducers in a `reducers` folder located in the `src/web` directory:

![image.png](/.attachments/image-731d54c7-5a80-4899-9431-618b100e0c3e.png)

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

# React-Redux

### Overview

In order to connect our React components to Redux, we will use a library called `React-Redux`.  This will give us an easy way to have components subscribe and re-render in response to changes in the Redux store, as well as dispatch actions to update the store.

The entire API for React-Redux boils down to two simple pieces.
1. The `<Provider />` component - This component can be placed at or near the root of your React application, which will make the Redux store available for the rest of the application, without needing to pass the store explicitly through to each component.
2. The `connect()` method - This method is used to create 'higher order' components, and encapsulates the process of getting data into and out of the Redux store.  The idea is that you can write a React component which uses a bunch of data from the Redux store via props, and then use the `connect()` method to wrap the component and export it such that the parent doesn't need to provide that data via props.

You can read more information from the official documentation: https://react-redux.js.org/introduction/quick-start

### Create a Component

Let's make a new React component called `ChannelListComponent`.  This component will render all the channels in the Redux store, and will also be in charge of loading them.

For this tutorial, I will place this component in a file called `Channels.tsx` located in the `components` folder:

![image.png](/.attachments/image-29db764c-792d-4230-9b57-bdc777f1c676.png)

To get starting creating our component, we will define a couple of interfaces.  First we need to define an interface for the props that we will receive from the parent component.  In this case, we don't need any parameters from the parent component.
```ts
interface params { }
```
Next, we define an interface for the data we will get from the Redux store:
```ts
interface connectedState {
    channels: defs.Channel[] | null;
}
```
Finally, we define an interface for the callback method we need to dispatch an action to load the channels into the Redux store.  This describes a callback which will asyncronously load a list of channels, and then dispatch an action to update the Redux store to contain those channels.  That implementation will come later.
```ts
interface connectedDispatch {
    reloadChannels: () => Promise<void>;
}
```
Now that we have all these interfaces set up, we can combine all of them together to create the complete `props` for our `ChannelListComponent`.
```ts
type fullParams = params & connectedState & connectedDispatch;
```

Now let's actually implemented our component:

_Channels.tsx_
```ts
import * as React from 'react';
import * as defs from '../definitions/definitions';

interface params {}

interface connectedState {
    channels: defs.Channel[] | null;
}

interface connectedDispatch {
    reloadChannels: () => Promise<void>;
}

type fullParams = params & connectedState & connectedDispatch;

class ChannelListComponent extends React.Component<fullParams> {

    componentDidMount() {
        this.props.reloadChannels();
    }

    render() {
        return (
            <div>
                <div>
                    <h3>
                        Available Channels
                    </h3>
                </div>
                <div>
                    {
                        this.props.channels ?
                            this.props.channels.map(channel =>
                                <div key={channel.channelId}>
                                    {channel.displayName}
                                </div>
                            ) :
                            "Loading..."
                    }
                </div>
            </div>
        );
    }
}

```

### Connect the Component to Redux

The component that we have created takes 2 props, `channels` and `reloadChannels`.  We want both of these props to be provided by the Redux store, and the parent component shouldn't need to provide any props at all.  To accomplish this, we will use the `connect()` method from `React-Redux`, which will wrap our component in a higher order component as discussed earlier.

The `connect()` method takes two parameters.
1. mapStateToProps - A function to map the redux Store to the properties we defined in `connectedState`
2. mapDispatchToProps - A function to generate callbacks using the `dispatch` method from the store.

At this point, we can no longer put a black box around the `reloadChannels()` method, since we will need to provide an implementation in `mapDispatchToProps`. 
 We could set up our node server to have an API endpoint which returns this data, but for now we will just hard code some data.  The implementation will look something like:

_Channels.tsx_
```ts
import * as React from 'react';
import * as defs from '../definitions/definitions';
import { Action, ActionTypes } from "../actions/actionTypes";
import { connect } from "react-redux";
import { Dispatch } from "redux";

...

const mapStateToProps = (state: defs.State): connectedState => ({
    channels: state.channels
});

const tempChannels: defs.Channel[] = [{
    channelId: 1,
    displayName: "General",
    canAnyoneInvite: true,
    isActiveDirectMessage: false,
    isGeneral: true,
    isPublic: true,
    ownerId: null
}, {
    channelId: 2,
    displayName: "Random",
    canAnyoneInvite: true,
    isActiveDirectMessage: false,
    isGeneral: false,
    isPublic: true,
    ownerId: 1
}, {
    channelId: 3,
    displayName: "Secret",
    canAnyoneInvite: false,
    isActiveDirectMessage: false,
    isGeneral: false,
    isPublic: false,
    ownerId: 1
}];

const mapDispatchToProps = (dispatch: Dispatch<Action>): connectedDispatch => ({
    reloadChannels: async () => {
        //TODO: load data from server

        dispatch({
            type: ActionTypes.LOAD_CHANNELS,
            channels: tempChannels
        });
    }
});

export const ChannelList: React.ComponentClass<params> =
    connect(mapStateToProps, mapDispatchToProps)(ChannelListComponent);
```

#### Add support for Async/Await
At this point, you may notice an error on the async method that looks like this:
`TS2705: An async function or method in ES5/ES3 requires the 'Promise' constructor. Make sure you have a declaration for the 'Promise' constructor or include 'ES2015' in your '--lib' option.`

This error is because we have declared in our `tsconfig.json` file that our target platform only supports the `es5` and `dom` libraries.  Since promises aren't part of es5, we will need to provide a polyfill for them.  Please refer to the Polyfills section at the top of this part of the tutorial for more information about this.  For now, we can silence this error by updating the `lib` option in our `tsconfig.json` file to declare support for Promises:

_tsconfig.json_
```js
"compilerOptions": {
...

  //Tell the TypeScript compiler what libraries we expect to exist
  //In this case, we expect the user's browser to have ES5 support and a dom
  //We provide a polyfill for es2015 promises, and other polyfills provide their own definitions
  "lib": ["es5", "dom", "es2015.promise"],

...
}
```

Then we add the following polyfill to the entry point for our code:

_boot-client.tsx_
```ts
//polyfills for IE
import 'es6-promise/auto';
```
This makes promises available to the user's browser, if it wasn't already.

### Final Connected Component

Putting everything together, our component now looks like this:

_Channels.tsx_
```ts
import * as React from 'react';
import * as defs from '../definitions/definitions';
import { Dispatch } from "redux";
import { Action, ActionTypes } from "../actions/actionTypes";
import { connect } from "react-redux";

interface params {}

interface connectedState {
    channels: defs.Channel[] | null;
}

interface connectedDispatch {
    reloadChannels: () => Promise<void>;
}

const mapStateToProps = (state: defs.State): connectedState => ({
    channels: state.channels
});

const tempChannels: defs.Channel[] = [{
    channelId: 1,
    displayName: "General",
    canAnyoneInvite: true,
    isActiveDirectMessage: false,
    isGeneral: true,
    isPublic: true,
    ownerId: null
}, {
    channelId: 2,
    displayName: "Random",
    canAnyoneInvite: true,
    isActiveDirectMessage: false,
    isGeneral: false,
    isPublic: true,
    ownerId: 1
}, {
    channelId: 3,
    displayName: "Secret",
    canAnyoneInvite: false,
    isActiveDirectMessage: false,
    isGeneral: false,
    isPublic: false,
    ownerId: 1
}];

const mapDispatchToProps = (dispatch: Dispatch<Action>): connectedDispatch => ({
    reloadChannels: async () => {
        //TODO: load data from server

        dispatch({
            type: ActionTypes.LOAD_CHANNELS,
            channels: tempChannels
        });
    }
});

type fullParams = params & connectedState & connectedDispatch;

class ChannelListComponent extends React.Component<fullParams> {

    componentDidMount() {
        this.props.reloadChannels();
    }

    render() {
        return (
            <div>
                <div>
                    <h3>
                        Available Channels
                    </h3>
                </div>
                <div>
                    {
                        this.props.channels ?
                            this.props.channels.map(channel =>
                                <div key={channel.channelId}>
                                    {channel.displayName}
                                </div>
                            ) :
                            "Loading..."
                    }
                </div>
            </div>
        );
    }
}

export const ChannelList: React.ComponentClass<params> =
    connect(mapStateToProps, mapDispatchToProps)(ChannelListComponent);

```
The `connect()` method returns a function that we can pass our `ChannelListComponent` to, creating a new 'higher-order' component called `ChannelList`.  This component can then be exported and used elsewhere in the application.  This higher-order component will automatically get the props it needs from the Redux store, so they don't need to be provided by a parent component.  In the case of this component, there are no props at all to be provided by the parent component, as defined in the `params` interface.  We can update our `AppRoot` component to use our new `ChannelList` higher-order component.

_AppRoot.tsx_
```ts
import * as React from "react";
import {ChannelList} from "./Channels";

export class AppRoot extends React.Component<{}> {
    constructor(p: {}) {
        super(p);
    }

    render() {
        return (
            <div>
                <h2>Hello World</h2>
                <ChannelList />
            </div>
        );
    }
}
```



### The Provider

In order for the Redux store to actually be available to our higher-order components that we create using `connect()`, we need to add it to our React component tree.  This is done using the `<Provider />` component provided by `React-Redux`.  We will update our `boot-client.tsx` file, which is the entry point to our React application, to use this component.

_boot-client.tsx_
```ts
import * as React from 'react';
import * as ReactDOM from 'react-dom'
import {rootReducer} from "./reducers/reducer";
import {createStore} from "redux";
import {Provider} from 'react-redux';
import {AppRoot} from "./components/AppRoot";

//polyfills for IE
import 'es6-promise/auto';

const message: string = "this is the client";
console.log(message);

const store = createStore(rootReducer);

ReactDOM.render(
    <Provider store={store}>
            <AppRoot/>
    </Provider>,
    document.getElementById('root')
);
```

# Putting Everything Together

### File Structure

![image.png](/.attachments/image-71eb816a-bca6-403c-a4f6-73b7c55eccaf.png)

### Build and Run the application

```
npm run build
npm run host
```

```
> npm run build
> react-demo@1.0.0 build C:\Users\Matt Hoffman\git\react-demo\node
> webpack --mode=development

i ｢atl｣: Using typescript@3.0.3 from typescript
i ｢atl｣: Using tsconfig.json from C:/Users/Matt Hoffman/git/react-demo/node/tsconfig.json
i ｢atl｣: Using typescript@3.0.3 from typescript
i ｢atl｣: Using tsconfig.json from C:/Users/Matt Hoffman/git/react-demo/node/tsconfig.json
i ｢atl｣: Checking started in a separate process...
i ｢atl｣: Time: 1174ms
i ｢atl｣: Checking started in a separate process...
i ｢atl｣: Time: 753ms
Hash: d69fcfeb0abe91b4cd3e05fb8f65fbe47b3d2eab
Version: webpack 4.17.2
Child
    Hash: d69fcfeb0abe91b4cd3e
    Time: 7520ms
    Built at: 2018-10-04 12:02:18
                  Asset      Size  Chunks             Chunk Names
        build/bundle.js   902 KiB    main  [emitted]  main
    build/bundle.js.map  1020 KiB    main  [emitted]  main
    Entrypoint main = build/bundle.js build/bundle.js.map
    [./node_modules/webpack/buildin/global.js] (webpack)/buildin/global.js 509 bytes {main} [built]
    [./node_modules/webpack/buildin/harmony-module.js] (webpack)/buildin/harmony-module.js 573 bytes {main} [built]
    [./src/web/actions/actionTypes.ts] 192 bytes {main} [built]
    [./src/web/boot-client.tsx] 572 bytes {main} [built]
    [./src/web/reducers/reducer.ts] 718 bytes {main} [built]
    [./src/web/util/polyfills.js] 1.69 KiB {main} [built]
    [0] multi ./src/web/boot-client.tsx 28 bytes {main} [built]
        + 48 hidden modules
Child
    Hash: 05fb8f65fbe47b3d2eab
    Time: 5993ms
    Built at: 2018-10-04 12:02:17
      Asset      Size  Chunks             Chunk Names
    main.js  7.81 KiB    main  [emitted]  main
    Entrypoint main = main.js
    [./src/server/App.ts] 819 bytes {main} [built]
    [./src/server/main.ts] 499 bytes {main} [built]
    [express] external "express" 42 bytes {main} [built]
    [http] external "http" 42 bytes {main} [built]
    [path] external "path" 42 bytes {main} [built]
    [0] multi ./src/server/main.ts 28 bytes {main} [built]
```
```
> npm run host

> react-demo@1.0.0 host C:\Users\Matt Hoffman\git\react-demo\node
> node ./build/main.js

this is the server
server env: development
Listening on port 3000
```


### What our final application should look like
![image.png](/.attachments/image-d71d39fd-2b5f-4935-9c21-a8097e2cc436.png)



### Download Source
The complete code for this part of the tutorial can be found here: https://github.com/mjhoffm2/react-demo/tree/Part-2

Up next: [Part 3 - Basic Front End Application Skeleton with React-Router and React-Bootstrap](/Part-3-%252D-Basic-Front-End-Application-Skeleton-with-React%252DRouter-and-React%252DBootstrap.md)