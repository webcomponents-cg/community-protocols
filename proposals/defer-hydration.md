# Defer Hydration Protocol

An open protocol for controlling hydration on the client.

Author: Justin Fagnani

Status: Draft

Last update: 2021-06-24

# Background

In server-side rendered (SSR) applications, the process of a component running code to re-associate its template with the server-rendered DOM is called "hydration". The defer-hydration protocol is design to allow controler over hydration to solve two related problems:

1. Interoperable incremental hydration across web components. Componentents should not automatically hydrate upon being defined, but wait for a signal from their parent or other coordinator.
2. Hydration ordering independent of definition order. Because components usually depend on data from their parent, and the parent won't usually set data on a child until it's hydrated, we need hydration to occur in a top-down order.

`defer-hydration` enables us to decouple loading the code for web component definitions from starting the work of hydration, and enables top-down ordering and sophisticated coordination of hydration, including triggering hydration only on interaction or data changes for specific components.

# Overview

The Defer Hydration Protocol specifies an attribute named `defer-hydration` that is placed on elements, usually during server rendering, to tell them not to hydrate when they are upgraded. Removing this attribute is a signal that the element should hydrate. Elements can observe the attribute being removed via `observedAttribute` and `attributeChangedCallback`.

When an element hydrates it can remove the `defer-hydration` attribute from its shadow children to hydrate them, or keep the attribute if itself can determine a more optimal time to hydrate all or certain children. By making the parent responsible for removing the `defer-hydration` attribute from it's children, we ensure top-down ordering.

## Use case 1: Auto-hydration with top-down odering

In this use case we want to page to hydrate as soon as elements are defined, but we want to force top-down ordering to avoid invalid child states. Here we configure the server-rendering step to add the `defer-hydration` attribute to all elements _except_ the top-most defer-hydration-aware elements in the document.

When the top-most elements are defined, they will run their hydrations steps since they don't have a `defer-hydration` attribute, and will trigger their subtrees to hydrate by removing `defer-hydration` from children.

Example HTML:

```html
<!doctype html>
<html>
  <head>
    <script type="module" src="./app.js"></script>
  </head>
  <body>
    <x-nav>
      <template shadowroot="open">
        <x-header defer-hydration>
          <template shadowroot="open">
            <h1>Example</h1>
          </template>
        </header>
      </template>
    <x-article>
      <template shadowroot="open">
        <x-figure defer-hydration>...</x-figure>
      </template>
    </x-article>
  </body>
<html>
```

## Use case 2: On-demand hydration

Hydration can be deferred until some data or user-driven signal, such as interacting with an element. In this case server-rendering is configured to add `defer-hydration` to all elements so that nothing will automatically hydrate.

An app-level coordinator may implement an event delegation/buffering/replay system to detect user-events within an element and remove `defer-hydration` on demand before replaying events.

*TODO: do we need to have an event that signals that hydration is complete before replaying events?*

## Hydrating children

```ts
class MyElement extends HTMLElement {
  static observedAttributes = ['defer-hydration'];

  attributeChangedCallback(name, oldValue, newValue) {
    if (name === 'defer-hydration' && newValue === null) {
      this._hydrate();
    }
  }

  _hydrate() {
    // do template hydrate work
    // ...

    // hydrate children
    const deferredChildren =
        this.shadowRoot.querySelectorAll('[defer-hydration]');
    for (const child of deferredChildren) {
      child.removeAttribute('defer-hydration');
    }
  }
}
```
