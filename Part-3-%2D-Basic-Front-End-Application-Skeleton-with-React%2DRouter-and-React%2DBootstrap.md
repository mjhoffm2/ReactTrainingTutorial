[[_TOC_]]

## Overview

This part of the tutorial is continued from [Part 2 - Basic Front End Application Skeleton with React and Redux](/Part-2-%2D-Basic-Front-End-Application-Skeleton-with-React-and-Redux).

In this part of the tutorial, we will continue to hook up various frameworks to our front end application, and learning how to use them.  We will still not be implementing any real features, but we will be laying the groundwork for how these frameworks will be used in our application.

Primary technologies in this part of the tutorial:
 - React
 - TypeScript
 - Redux
 - React-Redux
 - React-Router
 - Connected-React-Router
 - React-Bootstrap

All of the dependencies that we need for this part of the tutorial were included in the `package.json` file from the previous part.

# React-Router

### Overview

React makes it easy to create complex applications with many different views and components without having the user perform any actual page navigation.  This is usually referred to as a Single Page Application (SPA).  React-Router is a library to help manage navigation within a single page application, and ties it together with the actual browser url.  This allows us to keep the display of our application in sync with the url in the browser, which is a big win in terms of user experience.

For more information about React-Router, you should read the official documentation: https://reacttraining.com/react-router/web

### Set up our Routes

We are going to set up a new root component for our application.  As a result, I am renaming the current `AppRoot` component to be the `Home` component.  The new root component will be located at `src/web/routes.tsx`.

_routes.tsx_
```ts
import * as React from 'react';
import {Route, Switch, Redirect} from 'react-router';
import {ChannelList} from "./components/Channels";
import {Home} from "./components/Home";

export const Routes = () =>
    <Switch>
        <Route exact path={'/'} component={Home} />
        <Route path={'/channels'} component={ChannelList} />
        <Redirect to={'/'}/>
    </Switch>;

```
![image.png](/.attachments/image-d66576cd-9125-45a0-a402-bcd3818bbd50.png)
Disregard the `ViewChannel.tsx` file for now.

The `Routes` component above defines some rules for how urls will be matched to react components.  In this case, we have set up the `Home` component to be rendered when the url path is exactly `/`.  Next, we have set up the `ChannelList` component to be rendered when the url path starts with `/channels`.  Finally, any routes that did not match will be redirected to `/`.

Relevant Documentation
 - Route Component - https://reacttraining.com/react-router/web/api/Route
 - Switch Component - https://reacttraining.com/react-router/web/api/Switch
 - Redirect Component - https://reacttraining.com/react-router/web/api/Redirect

In addition to renaming `AppRoot` to `Home`, we will also be updating the component to contain a link to `/channels` instead of rendering the `ChannelList` component directly.

_Home.tsx_
```ts
import * as React from "react";
import {Link} from "react-router-dom";

export class AppRoot extends React.Component<{}> {
    constructor(p: {}) {
        super(p);
    }

    render() {
        return (
            <div>
                <h2>Hello World</h2>
                <Link to='/channels'>Go to channel list</Link>
            </div>
        );
    }
}
```

We are using the built in `Link` component provided by `react-router-dom`.  This isn't really doing anything special, it is just performing page navigation without actually reloading the page.  This navigation is picked up by `React-Router`, which will change the component rendered by the `Routes` component.

### Set up the Router

Similarly to React-Redux, React-Router is an interface between React and the browser's History.  Just like we used a `<Provider />` at the root of our application to set up React-Redux, we will use a `<Router />` to set up React-Router.  We will update our `boot-client.tsx` file as follows:

_boot-client.tsx_
```ts
import * as React from 'react';
import * as ReactDOM from 'react-dom'
import createBrowserHistory from "history/createBrowserHistory";
import {rootReducer} from "./reducers/reducer";
import {createStore} from "redux";
import {Provider} from 'react-redux';
import {Router} from "react-router";
import {Routes} from "./routes";

//polyfills for IE
import './util/polyfills';
import 'es6-promise/auto';

const message: string = "this is the client";
console.log(message);

const store = createStore(rootReducer);

const history = createBrowserHistory();

ReactDOM.render(
    <Provider store={store}>
        <Router history={history}>
            <Routes/>
        </Router>
    </Provider>,
    document.getElementById('root')
);

```

The important changes are:
1. We create a history object similarly to how we created the store for Redux.
2. We add a `<Router />` component with the history object.
3. We are using `<Routes />` as our new root component instead of the old `<AppRoot />` (renamed to `Home`)

### Try it out

At this point, we can go ahead and run our code to check it out.

```
npm run build
npm run host
```

We should see the following when we navigate to http://localhost:3000

![image.png](/.attachments/image-6e279c1e-5fa1-4b23-ab22-debf94974fc2.png)

If we Click on the 'Go to channel list' link, we should see the `ChannelList` component shown:

![image.png](/.attachments/image-7293126d-bbe1-416d-924c-6f221409e495.png)

So far so good.

However, if you try to do a hard refresh on the page (F5), you may see something similar to this:

![image.png](/.attachments/image-a9418a82-5bcb-4576-adc0-a58b7e07164a.png)

Refreshing the perfectly valid page has resulted in a 404 error.

### Adjust the Server

The issue above is that while our front end knows how to handle the `/channels` route, our node server doesn't know it.  All we did is configure it to serve the `main.html` file when a user requests the `/` url.  When a user tries to request the `/channels` url, the server decides it doesn't exist and gives us a 404 error.  This doesn't happen when a user first requests the index page at `/`, and then navigates to `/channels` using the client-sided router.

The fix for this is extremely simple, but it is important to understand the logic behind it and the implications of making the fix.  We will simply change the url which serves the index page from `"/"` to `"*"`.

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

        //configure endpoints that are handled by api calls
        this.configureApi();

        //static files
        this.express.use(express.static(path.join(__dirname, '/../public')));

        //serve static home page for all remaining requests
        this.express.get("*", (req, res, next) => {
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

Now, every request to the server that doesn't match a previously handled route will end up fetching the `main.html` file.  In this situation, the only route that won't serve the `main.html` file is a request for a static file such as a request for the `bundle.js` file.  However, this has introduced new problems.  If a user tries to download a file which has been removed, normally we would want them to get a 404 error.  However, instead they will be served main.html.  This is a problem we will not be addressing here.

On a related note, we now have a problem with our main.html file.  When we request `<script src="./build/bundle.js"></script>`, this is dependent on the url.  If a user tries to request the url `/channels/part2`, then their browser will attempt to download the javascript bundle from `/channels/build/bundle.js`, which doesn't exist.  Instead of getting a 404 error, they will get served the main.html file again, which the browser will try to parse as JavaScript and fail.

To fix this problem, we need to add a `<base href="/" />` tag to our html to define the base url for the page.

_main.html_
```html
<!DOCTYPE html>
<html>
<head>
    <title>Demo</title>
    <base href="/" />
</head>
<body>
    <div id="root"></div>
    <script src="./build/bundle.js"></script>
</body>
</html>
```

With these fixes applied, we should find that our application is working as intended when a user tries to directly request a sub-page.

### Creating a Nested Route

Right now, all of the routes in our application are handled by the `Routes` component.  However, we can have routes distributed throughout our component tree.  For example, our `/channels` route does not have the `exact` prop.  This means that it will render the `ChannelList` component as long as the url _starts_ with `/channels`.  We can have longer urls with more path parameters which can then in turn be handled by the `ChannelList` component.

Let's create a new component which represents an individual channel.

_ViewChannel.tsx_
```ts
import * as React from 'react';
import * as defs from '../definitions/definitions';
import {RouteComponentProps} from "react-router";
import {Dispatch} from "redux";
import {Action} from "../actions/actionTypes";
import {connect} from "react-redux";
import {Link} from "react-router-dom";

interface urlParams {
    channelId: string;
}

interface params extends RouteComponentProps<urlParams> {}

interface connectedState {
    channel: defs.Channel | null;
}

interface connectedDispatch {

}

const mapStateToProps = (state: defs.State, ownProps: params): connectedState => {
    //select the specific channel from redux matching the channelId route parameter
    if(state.channels) {
        const channelId = parseInt(ownProps.match.params.channelId);

        //Note: Array.prototype.find( ... ) requires a polyfill for internet explorer
        const channel = state.channels.find(channel => channel.channelId === channelId);

        if(channel) {
            return {
                channel
            };
        }
    }

    return {
        channel: null
    };
};

const mapDispatchToProps = (dispatch: Dispatch<Action>): connectedDispatch => ({

});

type fullParams = params & connectedState & connectedDispatch;

interface localState {}

class ViewChannelComponent extends React.Component<fullParams, localState> {
    render() {
        return (
            <div>
                <div>
                    Channel Id: {this.props.match.params.channelId}
                </div>
                <div>
                    {
                        this.props.channel ?
                            <div>Channel Name: {this.props.channel.displayName}</div> :
                            <div>Loading...</div>
                    }
                    <Link to='/channels'>
                        Close
                    </Link>
                </div>
            </div>
        );
    }
}

export const ViewChannel: React.ComponentClass<params> =
    connect(mapStateToProps, mapDispatchToProps)(ViewChannelComponent);

```

In the above code, we have defined a component which expects to receive a url parameter called `channelId`.  If we define a route that looks like
```ts
<Route path={`/channels/:channelId/view`} component={ViewChannel}/>
```
Then when the ViewChannel component, is rendered, the section of the url which matches `:channelId` will be assigned to `props.match.params.channelId`.  Note that even if we navigate to a url like `/channels/3/view`, the url parameter will be the string `"3"`, not the number `3`.

In the `mapStateToProps` method, the `ViewChannel` component uses the `channelId` url parameter to select a specific channel from the Redux store.  It would be more efficient if we stored the channels in the Redux store as a map instead of an array, but this isn't an issue for our small application.

We also have included a link which changes the url to `/channels`.  This has the effect of removing the part of the url which caused this component to be rendered, resulting in this component being removed.

Let's update the `ChannelList` component to make use of this new component:

_Channels.tsx_
```ts
import * as React from 'react';
import * as defs from '../definitions/definitions';
import {Route, RouteComponentProps, Switch} from "react-router";
import {Dispatch} from "redux";
import {Action, ActionTypes} from "../actions/actionTypes";
import {connect} from "react-redux";
import {ViewChannel} from "./ViewChannel";
import {Link} from "react-router-dom";

interface urlParams {
    channelId: string;
}

interface params extends RouteComponentProps<urlParams> {}

interface connectedState {
    channels: defs.Channel[] | null;
}

interface connectedDispatch {
    reloadChannels: () => Promise<void>;
}

const mapStateToProps = (state: defs.State): connectedState => ({
    channels: state.channels
});

const mapDispatchToProps = (dispatch: Dispatch<Action>): connectedDispatch => ({
    reloadChannels: async () => {
        //TODO: load data from server

        dispatch({
            type: ActionTypes.LOAD_CHANNELS,
            channels: [{
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
            }]
        });
    }
});

type fullParams = params & connectedState & connectedDispatch;

interface localState {}

class ChannelListComponent extends React.Component<fullParams, localState> {

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
                                    <Link to={`${this.props.match.url}/${channel.channelId}/view`}>
                                        Open {channel.displayName}
                                    </Link>
                                </div>
                            ) :
                            "Loading..."
                    }
                </div>
                <Switch>
                    <Route path={`${this.props.match.url}/:channelId/view`} component={ViewChannel}/>
                    <Route
                        render={() =>
                            <div>Please select a Channel</div>
                        }
                    />
                </Switch>
                <Link to='/'>
                    Return to home
                </Link>
            </div>
        );
    }
}

export const ChannelList: React.ComponentClass<params> =
    connect(mapStateToProps, mapDispatchToProps)(ChannelListComponent);
```

The major additions to this component are:
1. We have included a new set of routes below the list of channels.  When the url matches the `${this.props.match.url}/:channelId/view` path, then the ViewChannel component will be rendered on the page.  Otherwise, it will default to rendering a div that says 'Please select a Channel'.
2. Each channel in the list is now displayed as a link to open that channel.
3. We have added a link to change the url to `/`.  This has the effect of unmounting the `ChannelList` component, and mounting the `Home` component instead.

We can rebuild and re-run the server to take a look at what this looks like so far:

![image.png](/.attachments/image-baa8d2c4-6086-469c-b2b1-a709e9fffe72.png)

![image.png](/.attachments/image-76cf3600-28b7-405e-8be2-de2d38cd04d9.png)

With any luck, you should be able to see something similar to the above images.

# Connected-React-Router

In a nutshell, what we have done with React-Router is integrate the state from the url into React.  Depending on the application, it might make sense to also integrate that state with Redux.  These bindings can be accomplished using Connected-React-Router.  In my experience, this is entirely optional and is more of an organizational decision than a functional one.  If you like the idea of keeping navigational state in Redux, and updating the page location by dispatching Redux actions, then this library is for you.

### Update the State

We will need to update our State definition to include the state which will be used by Connected-React-Router

_stateDefinitions.ts_
```ts
import { RouterState } from 'connected-react-router';

export interface State {
    users: User[] | null;
    channels: Channel[] | null;
    router: RouterState;
}

...
```

We will also need to update the way our store is created to know how to handle navigational actions being dispatched.

```ts
//configure store based on https://github.com/supasate/connected-react-router
const store = createStore(
    connectRouter(history)(rootReducer),
    compose(
        applyMiddleware(
            routerMiddleware(history)
        )
    )
);
```

We will also be using `<ConnectedRouter />` instead of `<Router />`.  Our final `boot-client.tsx` file should look like this:

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

const message: string = "this is the client";
console.log(message);

const history = createBrowserHistory();

//configure store based on https://github.com/supasate/connected-react-router
const store = createStore(
    connectRouter(history)(rootReducer),
    compose(
        applyMiddleware(
            routerMiddleware(history)
        )
    )
);

//const store = createStore(rootReducer);

ReactDOM.render(
    <Provider store={store}>
        <ConnectedRouter history={history}>
            <Routes/>
        </ConnectedRouter>
    </Provider>,
    document.getElementById('root')
);
```

Finally, we add the RouterAction to our discriminated union of Actions in `actionTypes.ts`:

```ts
import {RouterAction} from "connected-react-router";

export type Action = RouterAction | loadUsersAction | loadChannelsAction;
```

In a perfect world, we would be done.  Unfortunately, at the time of writing this, there are some issues with the TypeScript definitions for Connected-React-Router that will show up as errors in our application.  We need to make the following changes to fix them.

First, we will export a special `ReducerAction` type which will exclude the `RouterAction` type.  Our final `actionTypes.ts` file should look like this:

_actionTypes.ts_
```ts
import * as defs from '../definitions/definitions';
import {RouterAction} from "connected-react-router";

export enum ActionTypes {
    LOAD_USERS = "LOAD_USERS",
    LOAD_CHANNELS = "LOAD_CHANNELS"
}

export type ReducerAction = loadUsersAction | loadChannelsAction;
export type Action = RouterAction | ReducerAction;

export interface loadUsersAction {
    type: ActionTypes.LOAD_USERS;
    users: defs.User[];
}

export interface loadChannelsAction {
    type: ActionTypes.LOAD_CHANNELS;
    channels: defs.Channel[];
}
```

In our `reducer.ts` file, we will replace all uses of `Action` with `ReducerAction`.

Additionally, you should see a type error with the `combineReducers` call mentioning that the 'router' property is missing.  This property will be managed outside of this reducer, so we don't need our reducer to know about it.  Essentially, we want to delete the 'router' property from our type definition, but only for the reducer file.  This can be accomplished similarly to how we did the same for the `ReducerAction` type, where we would have a `ReducerState` interface that excludes the 'router' property, and then a `State` type which extends `ReducerState` to include the 'router' property.  This will work just fine, but to me it makes more sense to start with the complete `State` interface, and subtract the 'router' property from it.

To accomplish this, we will use the following helper types:

```ts
//helper types
export type KeysExcept<T, K extends keyof T> = {
    [Key in keyof T]: K extends Key ? never : Key
}[keyof T]

export type RemoveKeys<T, K extends keyof T> = Pick<T, KeysExcept<T, K>>;
```

I recommend placing these in your definitions somewhere.

The `RemoveKeys` type lets us create types which are a copy of an existing type, with specific keys deleted.  In our case, we will define:

```ts
type stateExceptRouter = RemoveKeys<defs.State, 'router'>
```

Using this, we can now use `stateExceptRouter` in our `reducer.ts` file to pretend like we don't have a 'router' property in our state.

Finally, we need to change the type of our `rootReducer` by casting it to `Reducer<stateExceptRouter>`.

Putting it all together, this is what our `reducer.ts` file should look like:

_reducer.ts_
```ts
import * as defs from '../definitions/definitions';
import {ActionTypes, ReducerAction} from "../actions/actionTypes";
import {combineReducers, Reducer} from "redux";

export const initialUserState: defs.State['users'] = null;

export const userReducer: Reducer<defs.State['users'], ReducerAction> = (state = initialUserState, action) => {
    switch(action.type) {
        case ActionTypes.LOAD_USERS: {
            return action.users;
        }
    }
    return state;
};

export const initialChannelState: defs.State['channels'] = null;

export const channelReducer: Reducer<defs.State['channels'], ReducerAction> = (state = initialChannelState, action) => {
    switch(action.type) {
        case ActionTypes.LOAD_CHANNELS: {
            return action.channels;
        }
    }
    return state;
};

type stateExceptRouter = defs.RemoveKeys<defs.State, 'router'>

//our root reducer ignores the 'router' state
export const rootReducer = combineReducers<stateExceptRouter, ReducerAction>({
    users: userReducer,
    channels: channelReducer
}) as Reducer<stateExceptRouter>;
```

With that, we should have solved all of our TypeScript errors.

### Using Connected-React-Router

Now that we have hooked up React-Router to Redux, we can dispatch actions to change the page location and manipulate the browser history.  We will hook this up using React-Redux.

```ts
import * as React from 'react';
import {Action} from "../actions/actionTypes";
import {push} from "connected-react-router";

interface connectedDispatch {
    //go to a local url
    push: (url: string) => void;
}

const mapDispatchToProps = (dispatch: Dispatch<Action>): connectedDispatch => ({
    push: url => dispatch(push(url))
});
```

Now, we can call things like `this.props.dispatch('/some/url/3/view');` from anywhere in our component to trigger a page reload.  Keep in mind, we could already do this using `this.props.history.push('some/url/3/view');` for components which were rendered from a `<Route />` component and accepted `RouteComponentProps`.  However, now this goes through Redux.

# React-Bootstrap

### Overview

React-Bootstrap will be used to generate most of the actual DOM elements that the user sees.  It has a lot of useful components for organizing the layout of our application, as well as individual inputs, buttons, and more.  The best way to get an understanding of what you can do with React-Bootstrap is to look through the examples in their documentation.

https://react-bootstrap.github.io/getting-started/introduction

To practice React-Bootstrap, you can go ahead and test out various styles and components in the application.  I will apply some arbitrary styling to our application using React-Bootstrap to give an idea on what you can do with it.

### Examples

#### Home
_Home.tsx_
```ts
import * as React from "react";
import {Link} from "react-router-dom";
import {Row, Col, Grid, Panel} from "react-bootstrap";

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
                            <Panel.Body>Hello World</Panel.Body>
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

This will render a Panel on the left half of the screen, and some text on the right half of the screen.  See the bottom of the page for an image of what this should look like.

In this component, I have put a `<Grid />` component near the root of the dom.  In React-Bootstrap, you can have `<Row />` components and `<Col />` components nested within each other to create a nested grid effect, but you should not place a `<Grid />` inside of another `<Grid />`.

#### Channels

_Channels.tsx_
```ts
import * as React from 'react';
import * as defs from '../definitions/definitions';
import {Route, RouteComponentProps, Switch} from "react-router";
import {Dispatch} from "redux";
import {Action, ActionTypes} from "../actions/actionTypes";
import {connect} from "react-redux";
import {ViewChannel} from "./ViewChannel";
import {Link} from "react-router-dom";
import {Row, Col, Grid, Panel, ListGroup, ListGroupItem, Alert} from "react-bootstrap";

interface urlParams {
    channelId: string;
}

interface params extends RouteComponentProps<urlParams> {}

interface connectedState {
    channels: defs.Channel[] | null;
}

interface connectedDispatch {
    reloadChannels: () => Promise<void>;
}

const mapStateToProps = (state: defs.State): connectedState => ({
    channels: state.channels
});

const mapDispatchToProps = (dispatch: Dispatch<Action>): connectedDispatch => ({
    reloadChannels: async () => {
        //TODO: load data from server

        dispatch({
            type: ActionTypes.LOAD_CHANNELS,
            channels: [{
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
            }]
        });
    }
});

type fullParams = params & connectedState & connectedDispatch;

interface localState {}

class ChannelListComponent extends React.Component<fullParams, localState> {

    componentDidMount() {
        this.props.reloadChannels();
    }

    render() {
        return (
            <Grid>
                <Row>
                    <Col xs={12}>
                        <Panel>
                            <Panel.Heading>Available Channels</Panel.Heading>
                            {
                                this.props.channels ?
                                    <ListGroup>
                                        {
                                            this.props.channels.map(channel =>
                                                <ListGroupItem key={channel.channelId}>
                                                    <Row>
                                                    <Col xs={6}>
                                                        {channel.displayName}
                                                    </Col>
                                                    <Col xs={6}>
                                                        <Link to={`${this.props.match.url}/${channel.channelId}/view`}>
                                                            Open
                                                        </Link>
                                                    </Col>
                                                    </Row>
                                                </ListGroupItem>
                                            )
                                        }
                                        </ListGroup> :
                                    <Panel.Body>Loading...</Panel.Body>
                            }
                        </Panel>
                    </Col>
                </Row>
                <Switch>
                    <Route path={`${this.props.match.url}/:channelId/view`} component={ViewChannel}/>
                    <Route
                        render={() =>
                            <Alert bsStyle="warning">Please select a Channel</Alert>
                        }
                    />
                </Switch>
                <Row>
                    <Col xs={12}>
                        <Link to='/'>
                            Return to home
                        </Link>
                    </Col>
                </Row>
            </Grid>
        );
    }
}

export const ChannelList: React.ComponentClass<params> =
    connect(mapStateToProps, mapDispatchToProps)(ChannelListComponent);
```

#### ViewChannel

_ViewChannel.tsx_
```ts
import * as React from 'react';
import * as defs from '../definitions/definitions';
import {RouteComponentProps} from "react-router";
import {Dispatch} from "redux";
import {Action} from "../actions/actionTypes";
import {connect} from "react-redux";
import {Link} from "react-router-dom";
import {Row, Col, Button, Panel, Glyphicon} from "react-bootstrap";
import {push} from "connected-react-router";

interface urlParams {
    channelId: string;
}

interface params extends RouteComponentProps<urlParams> {}

interface connectedState {
    channel: defs.Channel | null;
}

interface connectedDispatch {
    //go to a local url
    push: (url: string) => void;
}

const mapStateToProps = (state: defs.State, ownProps: params): connectedState => {
    //select the specific channel from redux matching the channelId route parameter
    if(state.channels) {
        const channelId = parseInt(ownProps.match.params.channelId);
        const channel = state.channels.find(channel => channel.channelId === channelId);

        if(channel) {
            return {
                channel
            };
        }
    }

    return {
        channel: null
    };
};

const mapDispatchToProps = (dispatch: Dispatch<Action>): connectedDispatch => ({
    push: url => dispatch(push(url))
});

type fullParams = params & connectedState & connectedDispatch;

interface localState {}

class ViewChannelComponent extends React.Component<fullParams, localState> {

    render() {
        return (
            <Row>
                <Col xs={12}>
                    <Panel>
                        <Panel.Heading>
                            Channel Id: {this.props.match.params.channelId}
                        </Panel.Heading>
                        <Panel.Body>
                            {
                                this.props.channel ?
                                    <div>Channel Name: {this.props.channel.displayName}</div> :
                                    <div>Loading...</div>
                            }
                            <Link to='/channels'>
                                Close using a link
                            </Link>
                        </Panel.Body>
                        <Panel.Footer>
                            <Button
                                onClick={e => {
                                    this.props.push('/channels');
                                }}
                            >
                                <Glyphicon glyph="remove"/> Close
                            </Button>
                        </Panel.Footer>
                    </Panel>
                </Col>
            </Row>
        );
    }
}

export const ViewChannel: React.ComponentClass<params> =
    connect(mapStateToProps, mapDispatchToProps)(ViewChannelComponent);

```

### Including Bootstrap CSS

React-Bootstrap does not include its own styling.  Instead, it requires Bootstrap version 3.3.7 as a peer dependency.  Only the css stylesheet is needed, the JavaScript used by Bootstrap is completely re-implemented in React-Bootstrap.

To add the css we need, we will need to make changes to our `boot-client.tsx` file, and our `webpack.config.js` file.

First, simply import the css file directly from bootstrap.

_boot-client.tsx_
```ts
import 'bootstrap/dist/css/bootstrap.css'
```

We already have rules in webpack which tells it how to handle the css.  In this case, the bootstrap stylesheet will be put in the JavaScript bundle as a string.  This is not great for the size of our bundle, but that is something we can deal with later.  However, the bootstrap css references some static files that we do not have rules configured in webpack to handle.  We will add a rule targeting images and fonts using the `url-loader`.  This loader will include these resources directly in our JavaScript bundle as a base 64 string.  We can also provide a `limit` parameter, which limits how large a resource can be before it won't be included in the bundle.  In this case, any images or fonts which are larger than 25 kb will be kept as separate files and output into to the `public/build` folder.

```js
        ...
        module: {
            rules: [
                ...
                {
                    test: /\.(png|jpg|jpeg|gif|svg|ttf|otf|woff|woff2|eot)$/,
                    loader: 'url-loader?limit=25000'
                }
            ]
        }
        ...
```

Add the rule to use the `url-loader` with limit 25000 to the web part of the webpack.config.js (not server).

The final `webpack.config.json` should look like this:

_webpack.config.js_
```js
const path = require('path');
const nodeExternals = require('webpack-node-externals');

var config = [
    //web configuration
    {
        entry: ['./src/web/boot-client.tsx'],
        output: {
            path: path.resolve(__dirname, './public/build'),
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
                }
            ]
        },
        devtool: "source-map"
    },

    //server configuration
    {
        entry: ['./src/server/main.ts'],
        target: 'node',

        //make sure that webpack doesn't waste time compiling and bundling code that is already available in node
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


# Putting it all together
## File Structure

![image.png](/.attachments/image-40622091-fc93-4c35-837d-e4ed773ec115.png)

## Build and Run

```
npm run build
npm run host
```

## What it should look like

### Home
![image.png](/.attachments/image-ac8ecac5-93fe-4e7e-996e-1e91a7bdbe3c.png)

### Channels
![image.png](/.attachments/image-9bade8f5-e4e2-4295-a17b-87fefde868af.png)

### ViewChannel
![image.png](/.attachments/image-1ad3cc4d-6d2d-49aa-8bba-e003a519b110.png)

## Download Source
https://dev.azure.com/echeloncons/_git/Slack%20Training%20App?version=GBPart-3