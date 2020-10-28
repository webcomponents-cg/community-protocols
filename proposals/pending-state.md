# pending-task-protocol

An open protocol for interoperable asynchronous Web Components.

Author: Justin Fagnani

Document status: Draft

Last update: 2020-10-28

# Background

There are a number of scenarios where a web component might depend on or perform some asynchronous task. Components may lazy-load parts of their implementation or content, perform I/O in response to user input, manage long-running computation, etc.

It's often desirable to communicate the state of async tasks up the component tree to parent components so that they can display user affordances indicating whether the UI or content is pending some async operation.

Frameworks have the ability to invent custom APIs and patterns to handle this kind of cross-component communication. To enable the same with web components in a way that's interoperable between components from diffrent sources, and possibly implemented with different libraries, we need a protocol that components can implement without reliance on a common implementation.

# Goals

* Allow components to communicate that they have pending asynchonous tasks to ancestors in the DOM tree.
* Allow components to know if there are pending asynchronous tasks in descendants in the DOM tree.
* Allow components to intercept, modify, and block pending task notifications.
* Give guidance on the type of asynchronous work that should be notified.
* Allow for some labelling or differentiation between types of asynchronous work.

# Details

## Asynchronous Task States

There are four main states that an asynchronous task can be in:

* Not-started
* Started
* Sucessfully completed
* Failed

## The `pending-task` event

Components with pending tasks indicate so by firing a composed, bubbling, `pending-task` Event, with a `promise` property:

TypeScript interface:
```ts
interface PendingTaskEvent extends Event {
  promise: Promise<void>;
}
```

Example:

```js
class PendingTaskEvent extends Event {
  constructor(promise) {
    super('pending-task', {bubbles: true, composed: true});
    this.promise = promise;
  }
}

// Inside a component definition:
class DoWorkElement extends HTMLElement {
  async doWork() { /* ... */ }

  startWork() {
    this.dispatchEvent(new PendingTaskEvent(this.doWork()));
  }
}

// Inside a container component:
class IndicateWorkElement extends HTMLElement {
  constructor() {
    super();
    this.addEventListener('pending-task', async (e) => {
      this.showSpinner();
      await e.promise;
      this.hideSpinner();
    });
  }
}
```

The Promise must be resolved when the task is complete and rejected if the task fails. The value the Promise resolves to is unspecified.

Using an event with a Promise allows us to represent three of the four asynchronous states:

* Not-started: not represented
* Started: `pending-task` event fired
* Sucessfully completed: Promise resolved
* Failed: Promise rejected

# Open Questions

## Should the event carry a Promise or should there be two events?

Animations have multiple events: `animationstart` and `animationend`. Promises make it very easy to correlate the start of an operation with the end of that operation.

## Types of Async Tasks

There are some types of async work that we may not want to display UI affordances (like progress indicators) for. Async rendering used in order to yield to the browser's task queue for input hanlding and layout/paint for instance, should probably not trigger a progress indicator. Yet, code that measures style or layout may need to wait for rendering to complete.

We could add a field to the event that indicates the type of work and standardize a small number of types, such as `loading` and `rendering`. An event to indicate async rendering starting and stopping has existing analogies with the `animationstart` and `animationend` events.

We may also want to specifically reccomend against firing `pending-task` events for rendering work because of the pervasiveness of such async rendering with modern web component base classes like LitElement and Stencil.

## Work Estimation

Some use cases, like a progress bar that shows how much work is remaining, could benefit from estimating how much work is pending. The `pending-task` event could carry a numeric work estimate property so that containers can estimate the total amount of pending work and incremental progress.
