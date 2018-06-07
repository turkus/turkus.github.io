---
layout: post
title: FOUC issue in React [SOLVED!]
---

FOUC
---

FOUC - so called _Flash of Unstyled Content_ can be as very problematic as so many tries of solving this issue. 

To the point
---

Let's consider following configuration of routing ([react-router](https://github.com/ReactTraining/react-router)):

~~~ javascript
...
<PageLayout>
  <Switch>
    <Route exact path='/' component={Home} />
    <Route exact path='/example' component={Example} />
  <Switch>
</PageLayout>
...
~~~

where `PageLayout` is a simple [hoc](https://reactjs.org/docs/higher-order-components.html), containing div wrapper with `page-layout` class and returning it's children. 

Now, let's focus on the component rendering based on route. Usually you would use as `component` prop a React `Compoment`. But in our case we need to get it dynamically, to apply feature which helps us to avoid FOUC. So our code will look like this:

~~~ javascript
import asyncRoute from './asyncRoute'

const Home = asyncRoute(() => import('./Home'))
const Example = asyncRoute(() => import('./Example'))

...

<PageLayout>
  <Switch>
    <Route exact path='/' component={Home} />
    <Route exact path='/example' component={Example} />
  <Switch>
</PageLayout>

...
~~~

to clarify let's also show how `asyncRoute.js` module looks like:

~~~ javascript
import React, { Component } from 'react'
import PropTypes from 'prop-types'

import Loader from 'components/Loader'

class AsyncImport extends Component {
  static propTypes = {
    load: PropTypes.func.isRequired,
    children: PropTypes.node.isRequired
  }

  state = {
    component: null
  }

  toggleFoucClass () {
    const root = document.getElementById('react-app')
    if (root.hasClass('fouc')) {
      root.removeClass('fouc')
    } else {
      root.addClass('fouc')
    }
  }

  componentWillMount () {
    this.toggleFoucClass()
  }

  componentDidMount () {
    this.props.load()
      .then((component) => {
        setTimeout(() => this.toggleFoucClass(), 0)
        this.setState(() => ({
          component: component.default
        }))
      })
  }

  render () {
    return this.props.children(this.state.component)
  }
}

const asyncRoute = (importFunc) =>
  (props) => (
    <AsyncImport load={importFunc}>
      {(Component) => {
        return Component === null
          ? <Loader loading />
          : <Component {...props} />
      }}
    </AsyncImport>
  )

export default asyncRoute
~~~

>`hasClass`, `addClass`, `removeClass` are polyfills which operates on DOM class attribute.

>`Loader` is a custom component which shows spinner.

Why `setTimeout`? 

Just because we need to remove `fouc` class in the second tick. Otherwise it would happen in the same as rendering the Component. So it won't work.

As you can see in the `AsyncImport` component we modify react root container by adding `fouc` class. So HTML for clarity:

~~~ html 
<html lang="en">
<head></head>
<body>
  <div id="react-app"></div>
</body>
</html>
~~~

and another piece of puzzle:

~~~ sass 
#react-app.fouc
    .page-layout *
        visibility: hidden
~~~

sass to apply when importing of specific component (ie.: `Home`, `Example`) takes place.

Why not `display: none`?

Because we want to have all components which rely on parent width, height or any other css rule to be properly rendered.

How it works?
---

The main assumption was to hide all elements until compoment gets ready to show us rendered content. First it fires `asyncRoute` function which shows us `Loader` until `Component` mounts and renders. In the meantime in `AsyncImport` we switch visibility of content by using a class `fouc` on react root DOM element. When everything loads, it's time to show everything up, so we remove that class.


Hope that helps!


Thanks to
---

This [article](https://tylermcginnis.com/react-router-code-splitting/), which idea of dynamic import has been taken (I think) from [react-loadable](https://github.com/jamiebuilds/react-loadable).
