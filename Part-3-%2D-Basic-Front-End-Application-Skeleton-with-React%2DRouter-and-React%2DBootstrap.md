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
                <Link to='/channels' />
            </div>
        );
    }
}
```

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

### Adjust the Server

## Connected-React-Router

## React-Bootstrap

### Examples

### Including Bootstrap CSS