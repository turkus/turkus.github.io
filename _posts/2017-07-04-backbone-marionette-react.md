---
layout: post
title: Backbone (Marionette) to React 
last_modified_at: 2017-07-04T11:23:20+01:00
---

How it took place?
---

One day somebody gave me - a backend guy - an old school project written in Backbone with Marionette. At the beginning I was trying to get into, learn how it works etc, but after some while - in the time when React community was growing really fast - I said: That's it! Let's move to React.

First was to convince my teammates to do it. Everybody wasn't sure if it's a good idea: "We have tons of code lines...", and I agreed. 

From the business point of view we couldn't rewrite whole Backbone app to React. We had to do it smoothly, piece by piece. Another thing was to decrease amount of data (`dist.js` - containing whole application logic) being sent to the client browser. Now I can say that we did it! In production.

I was trying to find a good article about that case. Reading many meaningless articles and of course I found that:

[http://jgaskins.org/blog/2015/02/06/gentle-migration-from-marionette-to-react](http://jgaskins.org/blog/2015/02/06/gentle-migration-from-marionette-to-react)

And that's really cool, anyway thanks guys! But... we decided to go a little bit further.

Imagine that you're not using gulp, Backbone with Marionette by building big `dist.js`, but you do it using webpack and React.

How could that possible?

Well you have to mix it. Create a connection between React and our lovely Backbone app. Let's say: create a hybrid.

Recipe
---

I need you to focus. I won't sell here the boilerplate, but I will try to explain HOWTO in really easy and straight way.

First we need to deal with webpack and install base React sources + React Router. We can even use simple boilerplate (we used React, React Router and Redux). Then put everything in the same static directory next to Backbone app. When we have it and by running `npm start` we see simple React app on [http://localhost:8080](http://localhost:8080) we can move to step two.

In that step we look into Backbone app, find (or define) place like this:

~~~ javascript
var App = new Marionette.Application({
  onStart: function(options) {
    // some code here
  }
});
~~~

and put there a global variable, let's call it `backboneAppReady` and set it to `true`:

~~~ javascript
var App = new Marionette.Application({
  onStart: function(options) {
    // some code here
    window.backboneAppReady = true
  }
});
~~~

That will tell React that Backbone application has been started and we can talk with. How? Let's go to the next step.

We have something like Backbone.Wreqr ([https://github.com/marionettejs/backbone.wreqr](https://github.com/marionettejs/backbone.wreqr)). Now it's depreciated, but which old project isn't? :) After installing it we can use radio channel which is the clue.  

Of course if you're up to date with Marionette, you can use in Marionette v3.x.x built-in radio option with channels ([https://marionettejs.com/docs/v3.0.0/backbone.radio.html#channel](https://marionettejs.com/docs/v3.0.0/backbone.radio.html#channel))

Now, I'm sure you have implemented msgbus as a module in some kind of this way (if not I suggest to do that):

~~~ javascript
// static/backboneapp/utils/msgbus.js

define(["backbone", "backbone.wreqr"], function(Backbone) {
  var API;
  API = {
    vent: new Backbone.Wreqr.EventAggregator(),
    command: new Backbone.Wreqr.Commands(),
    reqres: new Backbone.Wreqr.RequestResponse()
  };
  return API;
});
~~~

the only one thing we should change is to use the power of channels! So change to this:

~~~ javascript
// static/backboneapp/utils/msgbus.js

define(["backbone", "backbone.wreqr"], function(Backbone) {
  var API, globalChannel;
  globalChannel = Backbone.Wreqr.radio.channel('global');
  API = {
    vent: globalChannel.vent,
    command: globalChannel.commands,
    reqres: globalChannel.reqres
  };
  return API;
});
~~~

Now you created a global channel to talk through.

If we have it, then we can move to React app. First, let's create url dispatcher for our Backbone views:

~~~ javascript
// static/reactapp/routes/Backbone/dispatcher.js

const showHome = (msgbus, matches) => {
  msgbus.reqres.request('show:home')
}

const showMembers = (msgbus, matches) => {
  msgbus.reqres.request('show:company:members')
}

const showMember = (msgbus, matches) => {
  let id = matches[1]
  msgbus.commands.execute('show:company:member', {id: id})
}

const notFound = (msgbus, matches) => {
  msgbus.reqres.request('show:notfound')
}

const backboneAppRoutes = [
  [/^\/members/, showMembers],
  [/^\/members\/(([0-9]+)*)/, showMember],
  [/\//, showHome],
  [/.*/, notFound]
]

const dispatchUrl = (path) => {
  const msgbus = Backbone.Wreqr.radio.channel('global')

  for (let route of backboneAppRoutes) {
    let [regex, show] = route
    let matches = path.match(regex)
    if (matches) {
      show(msgbus, matches)
      return
    }
  }
}

export default dispatchUrl
~~~

then an util to deal with:

~~~ javascript
// static/reactapp/routes/Backbone/utils.js

import dispatchUrl from './dispatcher'

const showBackbonePage = (pathname) => {
  let intervalId = setInterval(() => {
    if (window.backboneAppReady) {
      dispatchUrl(pathname)
      clearInterval(intervalId)
    }
  }, 250)
}

export default showBackbonePage
~~~

and finally React `NotFound.js` view implementation:

~~~ javascript
// static/reactapp/routes/NotFound/components/NotFound/NotFound.js

import { Component } from 'react'
import PropTypes from 'prop-types'
import { isEqual } from 'underscore'

import showBackbonePage from 'routes/Backbone/utils'

class NotFound extends Component {
  static propTypes = {
    location: PropTypes.object.isRequired
  }

  constructor (props, context) {
    super(props, context)

    this.currentUrl = props.location.pathname + props.location.search
  }

  componentDidMount () {
    showBackbonePage(this.props.location.pathname)
  }

  componentDidUpdate () {
    showBackbonePage(this.props.location.pathname)
  }

  shouldComponentUpdate (nextProps) {
    const url = nextProps.location.pathname + nextProps.location.search
    const shouldUpdate = !isEqual(this.currentUrl, url)
    if (shouldUpdate) this.currentUrl = url
    return shouldUpdate
  }

  render () {
    return false
  }
}

export default NotFound
~~~

Ooops! I forget about `CoreLayout.js` (called also `PageLayout.js` - example reference: [https://github.com/davezuko/react-redux-starter-kit/blob/master/src/layouts/PageLayout/PageLayout.js](https://github.com/davezuko/react-redux-starter-kit/blob/master/src/layouts/PageLayout/PageLayout.js)):

~~~ javascript
import React, { Component } from 'react'
import PropTypes from 'prop-types'

class CoreLayout extends Component {
  static propTypes = {
    children: PropTypes.element.isRequired,
    location: PropTypes.oneOfType([PropTypes.oneOf([null]), PropTypes.object]),
    dispatch: PropTypes.func.isRequired
  }

  static contextTypes = {
    router: PropTypes.object.isRequired
  }

  constructor (...args) {
    super(...args)
    this.setLocation = this.setLocation.bind(this)
  }

  componentWillMount () {
    this.initLocationHandlers()
  }

  setLocation (params) {
    const pathname = params.pathname || '/'
    const queryParams = params.queryParams ? '?' + params.queryParams : ''
    this.context.router.push(pathname + queryParams)
  }

  initLocationHandlers () {
    let intervalId = setInterval(() => {
      if (window.backboneAppReady) {
        const msgbus = Backbone.Wreqr.radio.channel('global')
        msgbus.reqres.setHandler('set:location', this.setLocation)
        clearInterval(intervalId)
      }
    }, 250)
  }

  render () {
    return (
      <div className='main-wrapper'>
        <div className='core-layout__viewport'>
          <Header />
          <div className='core-layout__children'>
            {this.props.children}
          </div>
        </div>
        <div className='core-layout__backbone' />
      </div>

    )
  }
}

export default CoreLayout
~~~

What is `setLocation` and `set:location` handler for?
---

Until everything works fine (Backbone and React apps), we need to implement one location handler on React app side and apply it to Backbone app, otherwise browser history won't work.
In every place you navigate around Backbone app, you have to use `msgbus` in this way:

~~~ javascript
msgbus.reqres.request("set:location", {pathname: '/members'})
~~~

Now your Backbone app navigates through React Router routes.

Remember to remove Router from Backbone app, so this part:

~~~ javascript
class Router extends Marionette.AppRouter
    ...
~~~

and its usage.

What about `index.html`?
---

In `index.html` you have to put Backbone's root div into React's which is recommended to mix views (`<div className='core-layout__backbone' />` in `CoreLayout.js` in code example above) and of course add javascript dists of both apps.

What about template tags which backend renders?<br>
We used [https://www.npmjs.com/package/ejs](https://www.npmjs.com/package/ejs) to escape them.

How it works?
---

Main assumption was to pass the whole traffic through the React Router. So we set `NotFound.js` route as in the example above and when applications starts, it searches route in React and if doesn't find then goes to Backbone app.


Hope that helps!

Big thanks to
---
I'd like to thank my mate Dawid who went with me through all this stuff that we've got this done!
