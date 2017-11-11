# Hyperapp Effects

[![Build Status](https://travis-ci.org/okwolf/hyperapp-effects.svg?branch=master)](https://travis-ci.org/okwolf/hyperapp-effects)
[![Codecov](https://img.shields.io/codecov/c/github/okwolf/hyperapp-effects/master.svg)](https://codecov.io/gh/okwolf/hyperapp-effects)
[![npm](https://img.shields.io/npm/v/hyperapp-effects.svg)](https://www.npmjs.org/package/hyperapp-effects)

A [Hyperapp](https://github.com/hyperapp/hyperapp) Higher-Order App giving your `app` superpowers to write your [effects as data](https://youtu.be/6EdXaWfoslc), inspired by [Elm Commands](https://guide.elm-lang.org/architecture/effects).

Effects are always represented as arrays. For a single effect this array represents a tuple containing the effect type string and an object containing the properties of this effect. For multiple effects each array element is either a tuple or an array of tuples, which may be nested.

```js
EffectTuple = [ string, object ]
Effect = EffectTuple | EffectTuple[] | Effect[]
```

## Installation

### Node.js

Install with npm / Yarn.

<pre>
npm i <a href="https://www.npmjs.com/package/hyperapp-effects">hyperapp-effects</a>
</pre>

Then with a module bundler like [rollup](https://github.com/rollup/rollup) or [webpack](https://github.com/webpack/webpack), use as you would anything else.

```js
import { withEffects } from "hyperapp-effects"
```

Or using require.

```js
const { withEffects } = require("hyperapp-effects")
```

### Browser

Download the minified library from the [CDN](https://unpkg.com/hyperapp-effects).

```html
<script src="https://unpkg.com/hyperapp-effects"></script>
```

You can find the library in `window.effects`.

## API

### `withEffects`

```js
withEffects = function(App): App
```

This Higher-Order App function enables `actions` to return arrays which will be treated as effects.

Example:

```js
import { withEffects } from "hyperapp-effects"

withEffects(app)({
  actions: {
    foo: () => [
      // ... effects go here
    ],
  }
}).foo())
```

### `action`

```js
action = function(name: string, data?: any): EffectTuple
```

Describes an effect that will fire another action, optionally with `data`.

Example:

```js
import { withEffects, action } from "hyperapp-effects"

withEffects(app)({
  actions: {
    foo: () => [
      action("bar", { message: "hello" }),
      action("baz", { message: "hola" }),
      // ... other effects
    ],
    bar: () => data => {
      // data will have { message: "hello" }
    },
    baz: () => data => {
      // data will have { message: "hola" }
    }
  }
}).foo()
```

Note that you may also use a single action effect without an array wrapper and that nested `actions` may be called by separating the slices with dots:

```js
import { withEffects, action } from "hyperapp-effects"

withEffects(app)({
  actions: {
    foo: () => action("bar.baz", { message: "hello" }),
    bar: {
      baz: () => data => {
        // data will have { message: "hello" }
      }
    }
  }
}).foo()
```

This same convention follows for all the other effects as well.

### `frame`

```js
frame = function(name: string, data?: object): EffectTuple
```

Describes an effect that will call an action from inside [`requestAnimationFrame`](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame), which is also where the render triggered by the action will run, optionally with a `data` object. A relative timestamp will be provided as the `time` property on the action `data`.

Example:

```js
import { withEffects, frame } from "hyperapp-effects"

withEffects(app)({
  actions: {
    foo: () => frame("spawn", { character: "goomba" }),
    spawn: () => data => {
      // This action is running inside requestAnimationFrame
      // data will have { time: xxxx, character: "goomba" }
    }
  }
}).foo()
```

If you wish to have an action that continuously updates the `state` and rerenders inside of `requestAnimationFrame` (such as for a game), remember to include another `frame` effect in your return:

```js
import { withEffects, action, frame } from "hyperapp-effects"

withEffects(app)({
  state: {
    time: 0,
    delta: 0
  },
  actions: {
    init: () => frame("update"),
    update: () => ({ time }) => [
      action("incTime", time),

      // ...
      // Other actions to update the state based on delta time
      // ...

      // End with a recursive frame effect to perform the next update
      frame("update")
    ],
    incTime: ({ time: lastTime, delta: lastDelta }) => time => ({
      time: time || lastTime,
      delta: time && lastTime ? time - lastTime : lastDelta
    })
  }
}).init()
```

### `delay`

```js
delay = function(duration: number, name: string, data?: any): EffectTuple
```

Describes an effect that will call an action after a delay using [`setTimeout`](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout), optionally with `data`.

Example:

```js
import { withEffects, delay } from "hyperapp-effects"

withEffects(app)({
  actions: {
    startTimer: () => delay(
      60000,
      "alarm",
      { name: "minute timer" }
    ),
    alarm: () => data => {
      // This action will run after a minute delay
      // data will have { name: "minute timer" }
    }
  }
}).startTimer()
```

### `time`

```js
time = function(name: string, data?: object): EffectTuple
```

Describes an effect that will provide the current timestamp to an action using [`performance.now`](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now), optionally with a `data` object. The timestamp will be provided as the `time` property on the action `data`.

Example:

```js
import { withEffects, time } from "hyperapp-effects"

withEffects(app)({
  actions: {
    foo: () => time("bar", { some: "data" }),
    bar: () => data => {
      // data will have { time: xxxx, some: "data" }
    }
  }
}).foo()
```

### `log`

```js
log = function(...args: any[]): EffectTuple
```

Describes an effect that will call [`console.log`](https://developer.mozilla.org/en-US/docs/Web/API/Console/log) with arguments. Useful for development and debugging. Not recommended for production.

Example:

```js
import { withEffects, log } from "hyperapp-effects"

withEffects(app)({
  actions: {
    foo: () => log(
        "string arg",
        { object: "arg" },
        ["list", "of", "args"],
        someOtherArg
    )
  }
}).foo()
```

### `http`

```js
http = function(url: string, action: string, options?: object): EffectTuple
```

Describes an effect that will send an HTTP request using [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/Window/fetch) and then call an action with the response. If you are using a browser from the Proterozoic Eon like Internet Explorer you will want a [`fetch` polyfill](https://github.com/github/fetch). An optional `options` parameter supports the same [options as `fetch`](https://developer.mozilla.org/en-US/docs/Web/API/Window/fetch#Parameters) plus an additional `response` property specifying which [method to use on the response body](https://developer.mozilla.org/en-US/docs/Web/API/Body#Methods), defaulting to "json".

A simple HTTP GET request with a JSON response:

```js
import { withEffects, http } from "hyperapp-effects"

withEffects(app)({
  actions: {
    foo: () => http("/data", "dataFetched"),
    dataFetched: () => data => {
      // data will have the JSON-decoded
      // response from /data
    }
  }
}).foo()
```

An HTTP GET request with a text response:

```js
import { withEffects, http } from "hyperapp-effects"

withEffects(app)({
  actions: {
    foo: () => http(
      "/data/name",
      "textFetched",
      { response: "text" }
    ),
    textFetched: () => data => {
      // data will have the response
      // text from /data
    }
  }
}).foo()
```

An HTTP POST request using JSON body and response:

```js
import { withEffects, http } from "hyperapp-effects"

withEffects(app)({
  actions: {
    login: () => form => http(
      "/login",
      "loginComplete",
      {
        method: "POST",
        body: form
      }
    ),
    loginComplete: () => loginResponse => {
      // loginResponse will have the JSON-decoded
      // response from POSTing to /login
    }
  }
}).login()
```

## License

Hyperapp Effects is MIT licensed. See [LICENSE](LICENSE.md).