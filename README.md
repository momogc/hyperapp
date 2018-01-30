 <!-- # [Hyperapp](https://hyperapp.js.org) -->

# <img height=24 src=https://cdn.rawgit.com/JorgeBucaran/f53d2c00bafcf36e84ffd862f0dc2950/raw/882f20c970ff7d61aa04d44b92fc3530fa758bc0/Hyperapp.svg> Hyperapp

[![Travis CI](https://img.shields.io/travis/hyperapp/hyperapp/master.svg)](https://travis-ci.org/hyperapp/hyperapp) [![npm](https://img.shields.io/npm/v/hyperapp.svg)](https://www.npmjs.org/package/hyperapp) [![Slack](https://hyperappjs.herokuapp.com/badge.svg)](https://hyperappjs.herokuapp.com "Join us")

Hyperapp is a JavaScript library for building web applications.

* **Minimal** — Hyperapp was born out of the attempt to do more with less. We have aggressively minimized the concepts you need to understand while remaining on par with what other frameworks can do.
* **Functional** — Hyperapp's design is inspired by [The Elm Architecture](https://guide.elm-lang.org/architecture). Create scalable browser-based applications using a functional paradigm. The twist is you don't have to learn a new language.
* **Batteries-included** — Out of the box, Hyperapp combines state management with a Virtual DOM engine that supports keyed updates & lifecycle events — all with no dependencies.

## Getting Started

Our first example is a counter that can be incremented or decremented. Go ahead and try it online [here](https://codepen.io/hyperapp/pen/zNxZLP).

```js
import { h, app } from "hyperapp"

const state = { count: 0 }

const actions = {
  down: value => state => ({ count: state.count - value }),
  up: value => state => ({ count: state.count + value })
}

const view = (state, actions) => (
  <main>
    <h1>{state.count}</h1>
    <button onclick={() => actions.down(1)}>-</button>
    <button onclick={() => actions.up(1)}>+</button>
  </main>
)

app(state, actions, view, document.body)
```

This example assumes you are using a JavaScript compiler like [Babel](https://babeljs.io) or [TypeScript](https://www.typescriptlang.org) and a module bundler like [Parcel](https://parceljs.org), [Rollup](https://rollupjs.org), [Webpack](https://webpack.js.org), etc. Usually all you need to do is install the JSX [transform plugin](https://babeljs.io/docs/plugins/transform-react-jsx) and add the pragma option to your <samp>.babelrc</samp> file.

```json
{
  "plugins": [["transform-react-jsx", { "pragma": "h" }]]
}
```

If you are not familiar with JSX, it is a language syntax extension that lets you write HTML tags interspersed with JavaScript. Because browsers don't understand JSX, we use a compiler to transform it into <samp>hyperapp.h</samp> function calls (hyperscript).

```jsx
const view = (state, actions) =>
  h("div", {}, [
    h("h1", {}, state.count),
    h("button", { onclick: () => actions.down(1) }, "-"),
    h("button", { onclick: () => actions.up(1) }, "+")
  ])
```

Note that JSX is not required for building applications with Hyperapp. You can use hyperscript syntax without a compilation step as shown above. More alternatives to JSX include [hyperapp/html](https://github.com/hyperapp/html), [hyperx](https://github.com/substack/hyperx) and [t7](https://github.com/trueadm/t7).

If you don't want to set up a build environment, you can download Hyperapp from a CDN like [unpkg](https://unpkg.com/hyperapp) and it will be globally available through the <samp>window.hyperapp</samp> object.

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <script src="https://unpkg.com/hyperapp"></script>
</head>
</html>
```

## Overview

Hyperapp applications consist of three interconnected parts: the [State](#state), [View](#view), and [Actions](#actions).

Once initialized, the application executes in a continuous loop, taking in actions from users or from external events, updating the state, and representing changes in the view through a [Virtual DOM](#virtual-dom). Think of an action as a signal that notifies Hyperapp to update the state and schedule the next view redraw. After processing an action, the new state is presented back to the user.

### State

The state is a plain JavaScript object that describes your entire program. It consists of all the dynamic data that is moved around in the application during its execution. The state cannot be mutated after it is created. To update our program, we generate a new state object using actions.

```js
const state = {
  count: 0
}
```

Like any JavaScript object, the state can be a deeply nested tree of objects. We refer to nested objects in the state as partial state. A single state tree does not conflict with modularity — see [Slices] to learn how to update nested objects and split your state and actions.

```js
const state = {
  top: {
    count: 0
  },
  bottom: {
    count: 0
  }
}
```

### Actions

The only way to change the state is via actions. An action is like a blueprint that describes how to derive the next state from the current one. Under the hood, Hyperapp wires every function in your actions object to schedule a view redraw whenever the state changes.

The state is immutable. If you mutate the state within an action and return it, the view will not be redrawn as you expect. To update the state, actions must return a partial state object. The new state will be the result of a shallow merge of this object and the current state.

```js
const actions = {
  down: value => state => ({ count: state.count - value }),
  up: value => state => ({ count: state.count + value })
}
```

Immutability enables time-travel debugging, helps prevent introducing hard-to-track-down bugs by making state changes more predictable, and allows cheap memoization of components using shallow equality <samp>===</samp> checks.

The <samp>app</samp> function returns a copy of your actions where every function has been wired to state changes. Exposing this object to the outside world can be useful to operate your application from another program, subscribe to global events, etc.

To see this in action, modify the [previous example](#getting-started) to save the wired actions to a variable.

```jsx
const main = app(state, actions, view, document.body)
```

Then open the developer console and try calling some actions. You should see the counter update accordingly.

```js
main.up(9000)
```

#### Asynchronous Actions

Actions used for side effects don't need to have a return value. Actions which return a Promise, <samp>null</samp> or <samp>undefined</samp> do not trigger redraws or update the state.

```js
const actions = {
  upLater: value => (state, actions) => {
    setTimeout(actions.up, 1000, value)
  },
  up: value => state => ({ count: state.count + value })
}
```

Actions that return a Promise can be written as <samp>[async](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)</samp> functions. Note that <samp>async</samp> functions return a Promise, and not a partial state object. You need to call another action in order to update the state.

```js
const actions = {
  upLater: () => async (state, actions) => {
    await new Promise(done => setTimeout(done, 1000))
    actions.up(10)
  },
  up: value => state => ({ count: state.count + value })
}
```

### View

Every time your application state changes as a result of calling an action, the view function is called so that you can specify how you want the DOM to look based on the new state. The view returns your specification in the form of a [Virtual DOM](#virtual-dom) and Hyperapp takes care of updating the actual DOM to match it.

This operation doesn't replace the entire DOM tree, but only update the parts of the DOM that actually changed. Incremental updates are calculated by diffing the old and the new Virtual DOM, then patching the actual DOM to reflect the new version.

```js
const view = (state, actions) =>
  h("div", {}, [
    h("h1", {}, state.count),
    h("button", { onclick: () => actions.down(1) }, "-"),
    h("button", { onclick: () => actions.up(1) }, "+")
  ])
```

## Virtual DOM

A virtual DOM is a description of what a DOM should look like using a tree of nested JavaScript objects known as virtual nodes. Unlike real DOM elements, virtual nodes are cheap to create and move around.

```js
{
  name: "div",
  props: {
    id: "app"
  },
  children: [{
    name: "h1",
    props: {},
    children: ["Hi."]
  }]
}
```

At any point in time, you describe how you want your UI to look like. It is important to understand that the result of render is not an actual DOM node. Those are just lightweight JavaScript objects. We call them the virtual DOM.

React is going to use this representation to try to find the minimum number of steps to go from the previous render to the next.

A Virtual DOM allows us to write code as if the entire page is rendered on each change, while we only update the parts of the DOM that actually changed. We try to do this in as few operations as possible, by comparing the new virtual DOM against the previous one. This leads to high efficiency, since typically only a small percentage of nodes need to change, and changing real DOM nodes is costly compared to recalculating a virtual DOM.

Hyperapp creates this in-memory data structure cache, computes the resulting differences, and then updates the browser's actual DOM efficiently.

To help you create virtual nodes in a more compact way, Hyperapp provides the <samp>h</samp> function.

```js
import { h } from "hyperapp"

const node = h(
  "div",
  {
    id: "app"
  },
  [h("h1", null, "Hi.")]
)
```

A virtual node props may include any valid [HTML](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes) or [SVG](https://developer.mozilla.org/en-US/docs/Web/SVG/Attribute) attributes, [DOM events](https://developer.mozilla.org/en-US/docs/Web/Events), [lifecycle events](lifecycle-events.md), and [keys](#keys).

A function that returns a virtual node is also known as a [component](#components).

## Examples

You can find more [examples here](https://codepen.io/hyperapp).

## Links

* [Slack](https://hyperappjs.herokuapp.com)
* [Twitter](https://twitter.com/hyperappjs)
* [/r/Hyperapp](https://www.reddit.com/r/hyperapp)

## License

Hyperapp is MIT licensed. See [LICENSE](LICENSE.md).
