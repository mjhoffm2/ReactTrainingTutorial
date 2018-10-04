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

## React-Router

### Overview

React makes it easy to create complex applications with many different views and components without having the user perform any actual page navigation.  This is usually referred to as a Single Page Application (SPA).  React-Router is a library to help manage navigation within a single page application, and ties it together with the actual browser url.  This allows us to keep the display of our application in sync with the url in the browser, which is a big win in terms of user experience.

### Set up our Routes

We are going to set up a new root component for out application.  As a result, I am renaming the current `AppRoot` component to be the `Home` component.  The new root component will be located at `src/web/routes.tsx`.

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

Similarly to React-Redux, React-Router is an interface between React and the browser's History.  Just like we used a `<Provider />` at the root of our application to set up React-Redux, we will use a `<Router />` to set up React-Router.

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

If we Click on the 'Go to channel list', we should see the `ChannelList` component shown.

![image.png](/.attachments/image-7293126d-bbe1-416d-924c-6f221409e495.png)

So far so good.

However, if you try to do a hard refresh on the page (F5), you may see something similar to this.

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
Then when the ViewChannel component, is rendered, the section of the url which matched `:channelId` will be assigned to `props.match.params.channelId`.  Note that even if we navigate to a url like `/channels/3/view`, the url parameter will be the string `"3"`, not the number `3`.

The `ViewChannel` component will use the `channelId` url parameter to select a specific channel from the Redux store.  It would be more efficient if we stored the channels in the Redux store as a map instead of an array, but this isn't an issue for our small application.

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
2. Each channel is now displayed as a link to open that channel.
3. We have added a link to change the url to `/`.  This has the effect of unmounting the `ChannelList` component, and mounting the `Home` component instead.

We can rebuild and re-run the server to take a look at what this looks like so far:

![image.png](/.attachments/image-baa8d2c4-6086-469c-b2b1-a709e9fffe72.png)

![image.png](/.attachments/image-76cf3600-28b7-405e-8be2-de2d38cd04d9.png)

With any luck, you should be able to see something similar to the above images.

## Connected-React-Router

## React-Bootstrap

### Examples

### Including Bootstrap CSS