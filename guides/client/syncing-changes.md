# Syncing changes and optimistic UIs

When using LiveView, whenever you change the state in your LiveView process, changes are automatically sent and applied in the client.

However, in many occasions, the client may have its own state: inputs, buttons, focused UI elements, and more. In order to avoid server updates from destroying state on the client, LiveView provides several features and out-of-the-box conveniences.

Let's start by discussing which problems may arise from client-server integration, which may apply to any web application, and explore how LiveView solves it automatically.

## The problem in a nutshell

Imagine your web application has a form. The form has a single email input and a button. We have to validate that the email is unique in our database and render a tiny “✗” or “✓“ accordingly close to the input. Because we are using server-side rendering, we are debouncing/throttling form changes to the server. And, to avoid double-submissions, we want to disable the button as soon as it is clicked.

Here is what could happen. The user has typed “hello@example.” and debounce kicks in, causing the client to send an event to the server. Here is how the client looks like at this moment:

```
[ hello@example.    ]

    ------------
       SUBMIT
    ------------
```

While the server is processing this information, the user finishes typing the email and presses submit. The client sends the submit event to the server, then proceeds to disable the button, and change its value to “SUBMITTING”:

```
[ hello@example.com ]

    ------------
     SUBMITTING
    ------------
```

Immediately after pressing submit, the client receives an update from the server, but this is an update from the debounce event! If the client were to simply render this server update, the client would effectively roll back the form to the previous state shown below, which would be a disaster:

```
[ hello@example.    ] ✓

    ------------
       SUBMIT
    ------------
```

This is a simple example of how client and server state can evolve and differ for periods of times, due to the latency (distance) between them, in any web application, not only LiveView.

LiveView solves this in two ways:

* The JavaScript client is always the source of truth for current input values

* LiveView tracks how many events are currently in flight in a given input/button/form. The changes to the form are applied behind the scenes as they arrive, but LiveView only shows them once all in-flight events have been resolved

In other words, for the most common cases, **LiveView will automatically sync client and server state for you**. This is a huge benefit of LiveView, as many other stacks would require developers to tackle these problems themselves. For complete detail in how LiveView handles forms, see [the JavaScript client specifics in the Form Bindings page](form-bindings.md#javascript-client-specifics).

## Optimistic UIs via loading classes

Whenever an HTML element pushes an event to the server, LiveView will attach a `-loading` class to it. For example the following markup:

```heex
<button phx-click="clicked" phx-window-keydown="key">...</button>
```

On click, would receive the `phx-click-loading` class, and on keydown would receive the `phx-keydown-loading` class. The CSS loading classes are maintained until an acknowledgement is received on the client for the pushed event. If the element is triggered several times, the loading state is removed only when all events are resolved.

This means the most trivial optimistic UI enhancements can be done in LiveView by simply adding a CSS rule. For example, imagine you want to fade the text of an element when it is clicked, while it waits for a response:

```css
.phx-click-loading.opaque-on-click {
  opacity: 50%;
}
```

Now, by adding the class `opaque-on-click` to any element, the elements give an immediate feedback on click.

The following events receive CSS loading classes:

  - `phx-click` - `phx-click-loading`
  - `phx-change` - `phx-change-loading`
  - `phx-submit` - `phx-submit-loading`
  - `phx-focus` - `phx-focus-loading`
  - `phx-blur` - `phx-blur-loading`
  - `phx-window-keydown` - `phx-keydown-loading`
  - `phx-window-keyup` - `phx-keyup-loading`

Events that happen inside a form have their state applied to both the element and the form. When an input changes, `phx-change-loading` applies to both input and form. On submit, both button and form get the `phx-submit-loading` classes. Buttons, in particular, also support a `phx-disabled-with` attribute, which allows you to customize the text of the button on click:

```heex
<button phx-disable-with="Submitting...">Submit</button>
```

### Tailwind integration

If you are using Tailwind, you may want to use [the `addVariant` plugin](https://tailwindcss.com/docs/plugins#adding-variants) to make it even easier to customize your elements loading state.

```javascript
plugins: [
  plugin(({ addVariant }) => {
    addVariant("phx-click-loading", [".phx-click-loading&", ".phx-click-loading &",]);
    addVariant("phx-submit-loading", [".phx-submit-loading&", ".phx-submit-loading &",]);
    addVariant("phx-change-loading", [".phx-change-loading&", ".phx-change-loading &",]);
  }),
],
```

Now to fade one element on click, you simply need to add:

```heex
<button phx-click="clicked" class="phx-click-loading:opacity-50">...</button>
```

## Optimistic UIs via JS commands

While loading classes are extremely handy, they do not cover all possible scenarios. For  more complex cases, you can use [JS commands](`Phoenix.LiveView.JS`). JS commands support adding/removing classes, toggling attributes, hiding elements, transitions, and more.

For example, imagine that you want to immediately remove an element from the page on click, you can do this:

```heex
<button phx-click={JS.push("delete") |> JS.hide()}>Delete</button>
```

If the element you want to delete is not the clicked button, but its parent (or other element), you can pass a selector to hide:

```heex
<button phx-click={JS.push("delete") |> JS.hide("#post-row-13")}>Delete</button>
```

Or imagine that, instead of removing the element from the page, you want to fade it. We already saw how we can use loading classes for this in the previous section. However, loading classes target the element that was clicked. Luckily, you can also configure the selector that gets the loading state:

```heex
<button phx-click={JS.push("delete", loading: "#post-row-13")}>Delete</button>
```

Transitions are also only a few characters away:

```heex
<div id="item">My Item</div>
<button phx-click={JS.transition("shake", to: "#item")}>Shake!</button>
```

Whenever you perform any of these commands alongside a `push`, LiveView knows which push event caused the DOM element to hide or transition, and LiveView will prevent any server update from conflicting or erasing the client state until the push is acknowledged by the server. This means LiveView automatically prevents your elements from flickering or flashing, as events round-trip between client and server.

JS commands also include a `dispatch` function, which dispatches an event to the DOM element to trigger client-specific functionality. For example, to trigger copying to a clipboard, you may implement this event listener:

```javascript
window.addEventListener("app:clipcopy", (event) => {
  if ("clipboard" in navigator) {
    if (event.target.tagName === "INPUT") {
      navigator.clipboard.writeText(event.target.value);
    } else {
      navigator.clipboard.writeText(event.target.textContent);
    }
  } else {
    alert(
      "Sorry, your browser does not support clipboard copy.\nThis generally requires a secure origin — either HTTPS or localhost.",
    );
  }
});
```

And then trigger it as follows:

```heex
<button phx-click={JS.dispatch("app:clipcopy", to: "#printed-output")}>Copy</button>
```

See `Phoenix.LiveView.JS` for more examples and documentation.

## Optimistic UIs via JS hooks

On the most complex cases, you can assume control of a DOM element, and control exactly how and when server updates apply to the element on the page. See [the Client hooks via `phx-hook` section in the JavaScript interoperability page](js-interop.md#client-hooks-via-phx-hook) to learn more.

## Live navigation

LiveView also provides mechanisms to customize and interact with navigation events.

### Navigation classes

The following classes are applied to the LiveView's parent container:

  - `"phx-connected"` - applied when the view has connected to the server
  - `"phx-loading"` - applied when the view is not connected to the server
  - `"phx-error"` - applied when an error occurs on the server. Note, this
    class will be applied in conjunction with `"phx-loading"` if connection
    to the server is lost.

### Navigation events

For live page navigation via `<.link navigate={...}>` and `<.link patch={...}>`, their server-side equivalents `push_navigate` and `push_patch`, as well as form submits via `phx-submit`, the JavaScript events `"phx:page-loading-start"` and `"phx:page-loading-stop"` are dispatched on window. This is useful for showing main page loading status, for example:

```javascript
// app.js
import topbar from "topbar"
window.addEventListener("phx:page-loading-start", info => topbar.delayedShow(500))
window.addEventListener("phx:page-loading-stop", info => topbar.hide())
```

Within the callback, `info.detail` will be an object that contains a `kind`
key, with a value that depends on the triggering event:

  - `"redirect"` - the event was triggered by a redirect
  - `"patch"` - the event was triggered by a patch
  - `"initial"` - the event was triggered by initial page load
  - `"element"` - the event was triggered by a `phx-` bound element, such as `phx-click`
  - `"error"` - the event was triggered by an error, such as a view crash or socket disconnection

Additionally, any `phx-` event may dispatch page loading events by annotating the DOM element with the `phx-page-loading` attribute.

For all kinds of page loading events, all but `"element"` will receive an additional `to` key in the info metadata pointing to the href associated with the page load. In the case of an `"element"` page loading event, the info will contain a `"target"` key containing the DOM element which triggered the page loading state.

A lower level `phx:navigate` event is also triggered any time the browser's URL bar is programmatically changed by Phoenix or the user navigation forward or back. The `info.detail` will contain the following information:

  - `"href"` - the location the URL bar was navigated to.
  - `"patch"` - the boolean flag indicating this was a patch navigation.
  - `"pop"` - the boolean flag indication this was a navigation via `popstate`
    from a user navigation forward or back in history.
