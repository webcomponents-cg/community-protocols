# pending-task-protocol

An open protocol for interoperable asynchronous Web Components.

Author: Justin Fagnani

Status: Draft

Last update: 2021-06-20

# Background

There are a number of scenarios where a web component might depend on or perform some asynchronous task. Components may lazy-load parts of their implementation or content, perform I/O in response to user input, manage long-running computations, etc.

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
* Completed
* Failed

This protocol represents three of the states (Started, Completed, and Failed) with a promise-carrying DOM event.

## The `pending-task` event

Components with pending tasks indicate so by firing a composed, bubbling, `pending-task` Event, with a `promise` property:

TypeScript interface:
```ts
interface PendingTaskEvent extends Event {
  complete: Promise<void>;
}
```

Example:

```ts
class PendingTaskEvent extends Event {
  constructor(complete: Promise<void>) {
    super('pending-task', {bubbles: true, composed: true});
    this.complete = complete;
  }
}

// Inside a component definition:
class DoWorkElement extends HTMLElement {
  async doWork() { /* ... */ }

  startWork() {
    const workComplete = this.doWork();
    this.dispatchEvent(new PendingTaskEvent(workComplete));
  }
}

// Inside a container component:
class IndicateWorkElement extends HTMLElement {
  #pendingTaskCount = 0;

  constructor() {
    super();
    this.addEventListener('pending-task', async (e) => {
      e.stopPropagation();
      if (++this.#pendingTaskCount === 1) {
        this.showSpinner();
      }
      await e.complete;
      if (--this.#pendingTaskCount === 0) {
        this.hideSpinner();
      }
    });
  }
}
```

The completion Promise must be resolved when the task is complete and rejected if the task fails. The value the Promise resolves to is unspecified.

Using an event with a Promise allows us to represent three of the four asynchronous states:

* Not-started: not represented
* Started: `pending-task` event fired
* Completed: Promise resolved
* Failed: Promise rejected

## Cancelling tasks

This proposal does _not_ cover cancelling tasks. Similar to Promises, this proposal assumes that task cancellation is best done by the task initiators with an [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal). Objects being notified of a task shouldn't neccessarily be able to cancel it.

If a task is canceled by other means, the `completed` Promise should be rejected.

## Intercepting PendingTask events

If a part of a UI shows a loading affordance for a subtree, it is recommended that it stop propagation of the PendingTask event so that only one loading affordance is shown.

```ts
this.addEventListener('pending-task', async (e) => {
  e.stopPropagation();
  // show loading indicator
  await e.complete;
  // hide loading indicator
});
```

## Default actions

Some UI controls are able to show their own pending task indicators. One example is a form submit button with an embedded spinner. Such a controller may not want to show the loading indicator if a component above it in the tree is also showing one. This can be accomplished with event default actions.

Listeners can call `e.preventDefault()` on the event:

```ts
this.addEventListener('pending-task', async (e) => {
  e.preventDefault();
  // show loading indicator
  await e.complete;
  // hide loading indicator
});
```

And the control can check if the event is defaulted:

```ts
const workComplete = this.doWork();
const event = new PendingTaskEvent(workComplete);
this.dispatchEvent(event);
if (!event.defaultPrevented) {
  this.showLoadingIndicator();
  await workComplete;
  this.hideLoadingIndicator();
}
```

# Open Questions

## Types of Async Tasks

There are different types of async work. Whether this proposal should attempt to classify them and add a `type` field to `PendingTaskEvent` is an open question.

The argument against is that we simply do not yet know what categories there should be and how to definitively guide authors towards choosing the right type. It's also just additional complexity for component authors.

An argument for is that there are some types of async work that we may not want to display UI affordances (like spinners) for. Async rendering used in order to yield to the browser's task queue for input hanlding and layout/paint for instance, should probably not trigger a progress indicator. Yet, code that measures style or layout may need to wait for rendering to complete, and so could potentially utilize pending-task for that.

How can we allow UI affordances like spinners, but not to frequently create them during async rendering?

We could add a field to the event that indicates the type of work and standardize a small number of types, such as `loading` and `rendering`. An event to indicate async rendering starting and stopping has existing analogies with the `animationstart` and `animationend` events.

We may also want to specifically recommend against firing `pending-task` events for rendering work because of the pervasiveness of such async rendering with modern web component base classes like LitElement and Stencil.

## Work Estimation

Some use cases, like a progress bar that shows how much work is remaining, could benefit from estimating how much work is pending.

The `pending-task` event could carry a numeric work estimate property so that containers can estimate the total amount of pending work and incremental progress.

On the other hand, this may be better suited for ProgressEvent.
