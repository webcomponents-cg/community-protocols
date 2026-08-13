# Context Protocol

An open protocol for passing contextual data between components.

Author: Benjamin Delarre

Document status: Draft

Last update: 2026-08-13

# Background

Web components often need data owned by an ancestor, such as a theme, locale, logger, or state store.
Passing that data through every intermediate component couples those components to data they do not otherwise use.
Frameworks commonly solve this with context or dependency-injection APIs, but components built with different libraries need a shared browser API.

# Goals

- Resolve context from the nearest matching provider on an element's composed event path.
- Avoid passing contextual data through unrelated intermediate components.
- Provide synchronous discovery when a provider is already available.
- Let one-time requests wait for a later provider.
- Support subscriptions, provider replacement, and explicit cancellation.
- Give frameworks and plain custom elements the same API.

# Non-goals

The Context Protocol is not a dependency-injection framework.
It does not define constructor, factory, or property injection.
A dependency-injection system may use this protocol to locate dependencies, but those higher-level patterns are outside its scope.

The Context Protocol is also not a state-management system.
A component may use it to obtain a state store, but the protocol does not define how state is stored, updated, or observed.

# API

The protocol consists of two event interfaces and two methods on `Element`:

- `ContextRequestEvent` discovers the nearest provider synchronously.
- `ContextProviderEvent` announces that a provider's availability has changed.
- `requestContext()` obtains one value, waiting for a later provider if needed.
- `subscribeContext()` receives values from the nearest provider until its signal aborts.

The element that dispatches a request is the consumer.
A provider is an ancestor element whose request listener responds with a value for the requested context key.

```webidl
callback ContextCallback = undefined (
  any value,
  optional VoidFunction unsubscribe
);

callback ContextConsumerCallback = undefined (any value);

[Exposed=Window]
interface ContextRequestEvent : Event {
  constructor(DOMString type, ContextRequestEventInit eventInitDict);

  readonly attribute any context;
  readonly attribute Element? contextTarget;
  readonly attribute ContextCallback _callback;
  readonly attribute boolean subscribe;
  readonly attribute AbortSignal? signal;

  undefined respond(any value, optional VoidFunction unsubscribe);
};

dictionary ContextRequestEventInit : EventInit {
  required any context;
  required ContextConsumerCallback _callback;
  boolean subscribe = false;
  AbortSignal? signal = null;
};

[Exposed=Window]
interface ContextProviderEvent : Event {
  constructor(DOMString type, ContextProviderEventInit eventInitDict);

  readonly attribute any context;
  readonly attribute Element? contextTarget;
};

dictionary ContextProviderEventInit : EventInit {
  required any context;
};

partial interface Element {
  Promise<any> requestContext(
    any context,
    optional ContextRequestOptions options = {}
  );

  undefined subscribeContext(
    any context,
    ContextConsumerCallback callback,
    ContextSubscriptionOptions options
  );
};

dictionary ContextRequestOptions {
  AbortSignal signal;
};

dictionary ContextSubscriptionOptions {
  required AbortSignal signal;
};
```

The event constructors accept the usual `EventInit` members and preserve the values supplied by the author.
Only the qualifying context events defined below participate in the protocol.
If `ContextRequestEventInit.subscribe` is true and its `signal` is null, the `ContextRequestEvent` constructor throws a `TypeError`.

The `callback` constructor option is the consumer callback.
The `callback` attribute returns a user-agent-created delivery callback for providers.
A provider retains the delivery callback to send subscription updates.
The delivery callback validates the current provider before forwarding a value to the consumer callback.
It invokes the consumer callback with the request's `contextTarget` as the callback `this` value.

The delivery callback has a stable identity for the lifetime of the event.
Fresh events that the user agent creates for an existing operation use that operation's delivery callback.

# Context keys

A context key identifies the requested value.
Keys are compared using strict equality (`===`).
A key can be any JavaScript value other than `NaN`.
Because `NaN` does not strictly equal itself, it could never match a provider.

A unique symbol or object creates a context that can only be shared by code with access to that value.
A string or `Symbol.for()` key can intentionally identify the same context across independently loaded modules.

Constructing either context event with `NaN` as its context throws a `TypeError`.
Calling `requestContext()` with `NaN` returns a promise rejected with a `TypeError`.
Calling `subscribeContext()` with `NaN` throws a `TypeError`.

# Qualifying context events

A qualifying context request is a `ContextRequestEvent` that:

- has a `type` of exactly `context-request`;
- has `bubbles` and `composed` set to true;
- has `cancelable` set to false; and
- is dispatched from an `Element`.

A qualifying provider announcement is a `ContextProviderEvent` that has a `type` of exactly `context-provider`, has `bubbles` and `composed` set to true, has `cancelable` set to false, and is dispatched from an `Element`.

A context event that does not meet these requirements is dispatched as an ordinary DOM event.
It does not create, fulfill, or re-evaluate a context operation, and calling `respond()` on a non-qualifying request throws an `InvalidStateError` `DOMException`.

This rule preserves the normal `Event` constructor shape while making protocol participation explicit.
In particular, a `CustomEvent` named `context-request`, or a `ContextRequestEvent` constructed with `composed: false`, cannot accidentally enter the protocol.

`contextTarget` is initially null.
At the start of each dispatch, the user agent sets it to the `Element` on which `dispatchEvent()` was called, or to null for another kind of `EventTarget`.
It is not retargeted across shadow boundaries or changed during dispatch.
After dispatch, it remains set to the target of the most recent dispatch.
For a request event, `contextTarget` identifies the consumer.
For a provider announcement, it identifies the element whose provider availability changed.
An event dispatched from a non-`Element` is not a qualifying context event.
The first qualifying dispatch of a subscription request binds that event to its `contextTarget`.
If `dispatchEvent()` is later called with that event on a different target, it throws an `InvalidStateError` `DOMException` before dispatch begins and leaves `contextTarget` unchanged.

Each `ContextRequestEvent` has a responded flag.
The user agent sets it to false at the start of each dispatch.
`respond()` sets it to true for the rest of that dispatch.
The flag is not exposed to authors.

# Responding to a request

The first matching provider encountered during the bubbling portion of DOM dispatch responds by calling `event.respond(value, unsubscribe)`.
A provider must use a non-capturing request listener.
A capturing listener cannot respond.
At a shadow host, DOM can expose `eventPhase` as `AT_TARGET` while invoking a non-capturing listener during the bubbling portion of dispatch.
That listener can respond; the `eventPhase` value alone does not determine response eligibility.

A conforming provider must not install more than one active request listener on an element for the same context key.
Otherwise, listener registration order would determine which listener responds.

The `respond(value, unsubscribe)` method steps are:

1. If this is not a qualifying context request whose current listener is being invoked during the bubbling portion of DOM dispatch, throw an `InvalidStateError` `DOMException`.
2. If `currentTarget` is not an `Element`, is `contextTarget`, or another provider has already responded, throw an `InvalidStateError` `DOMException`.
3. If `subscribe` is false and `unsubscribe` was supplied, throw a `TypeError`.
4. If `subscribe` is true and `unsubscribe` was omitted, throw a `TypeError`.
5. If `subscribe` is true and `signal` is aborted, throw an `InvalidStateError` `DOMException`.
6. If the request is associated with a completed or terminated operation, throw an `InvalidStateError` `DOMException`.
7. If the request is associated with an operation whose current provider is `currentTarget`, and `unsubscribe` is not that operation's current unsubscribe function, throw an `InvalidStateError` `DOMException`.
8. Set the responded flag to true and call `stopImmediatePropagation()`.
9. If the request is associated with a subscription operation, either confirm its current provider registration and set it to active or record `currentTarget` and `unsubscribe` as its prospective provider registration.
   A one-time operation remains dispatching during this step.
10. Except for the same-provider confirmation defined under provider re-evaluation, invoke the delivery callback synchronously with `value` and, for a subscription, `unsubscribe`.
   The delivery callback validates the provider registration and invokes the consumer callback with `value`.
11. If initial delivery to a prospective provider succeeds and the operation has not terminated, make the prospective registration current, set the operation to active, and clear the prospective registration.
    If the operation had a previous provider, invoke its unsubscribe function and report any exception.

If `respond()` throws before step 8, the user agent has not accepted an unsubscribe function.
If the provider registered the delivery callback before calling `respond()`, it must remove that registration before propagating or reporting the exception.

Once step 8 has run, the request remains responded to even if the callback throws.
The user agent reports the exception, and it does not propagate to the provider or escape `dispatchEvent()`.
This prevents a second provider from answering the same request.

For a new subscription, the provider registers the delivery callback and its stable, idempotent unsubscribe function before calling `respond()`.
If initial delivery throws, the user agent invokes that unsubscribe function, removes the new registration, terminates the subscription, and reports the exception.
The exception does not escape `dispatchEvent()`.

The user agent adds the subscription's abort algorithm before invoking the initial callback.
If the signal aborts during initial delivery, termination wins: the new provider is unsubscribed and the operation does not become active.
If this occurs during a transfer, both the old and new provider registrations are removed.

Calling the delivery callback does not respond to a request.
For initial delivery, `respond()` authorizes and invokes it as part of the atomic response steps.
For a later subscription update, it invokes the consumer callback only when the supplied unsubscribe function is the operation's current provider unsubscribe function.
In every other state, including before `respond()`, after completion or termination, or with a missing or stale unsubscribe function, it throws an `InvalidStateError` `DOMException` and does not invoke the consumer callback.

During a transfer, the old provider remains active until initial delivery from the new provider succeeds.
The user agent then invokes the previous provider's unsubscribe function.
If initial delivery from the new provider throws, the new registration is removed and the old provider remains active.

A provider's unsubscribe function must remove its registration before performing other cleanup, and later calls must have no effect.
If provider cleanup throws, the user agent reports the exception and still completes the cancellation or transfer.

The `context`, `subscribe`, and `signal` attributes return their corresponding `ContextRequestEventInit` values.
The `callback` attribute returns the delivery callback associated with the consumer callback.

The `context` attribute of `ContextProviderEvent` returns its corresponding `ContextProviderEventInit` value.

# One-time requests

## `requestContext()`

`element.requestContext(context, options)` returns a new promise for a value from the nearest matching provider on the path that a composed event dispatched from `element` would follow.

The `requestContext(context, options)` method steps are:

1. If `context` is `NaN`, return a promise rejected with a `TypeError`.
2. If `options.signal` is present and aborted, return a promise rejected with its abort reason.
3. Create a new promise and a one-time context operation owned by `element`'s node document.
   Its delivery callback completes the operation and resolves the promise with the supplied value.
4. If a signal is present, add an abort algorithm that terminates the operation with the signal's abort reason.
5. Append the operation to the document's context operations and dispatch it.
6. Return the promise.

If a provider responds, the operation's state becomes completed and the returned promise is resolved with the supplied value.
Promise resolution follows the normal JavaScript rules, so a promise or thenable supplied as the context value is adopted.

If no provider responds, the operation becomes pending.
A later matching provider announcement causes the user agent to dispatch a fresh request from the element.
The dispatched event itself is never retained or redispatched.

If the signal aborts before the operation completes, the abort algorithm terminates the operation and rejects the promise with the signal's abort reason.
Once a provider responds, aborting the signal has no effect on the supplied value or on asynchronous work represented by it.

See [Requesting one value](#requesting-one-value) for an author-facing example.

## Direct one-time events

Authors can dispatch a qualifying `ContextRequestEvent` directly when they need synchronous delivery.

If no provider responds before `dispatchEvent()` returns, a directly dispatched one-time request is a synchronous miss.
The user agent does not retain it.
Its `signal` does not cancel dispatch or prevent a response, because there is no retained operation to terminate.

See [Requesting synchronously with a direct event](#requesting-synchronously-with-a-direct-event) for an author-facing example.

# Subscriptions

## `subscribeContext()`

`element.subscribeContext(context, callback, {signal})` creates a subscription and immediately dispatches a fresh qualifying request from `element` with `subscribe` set to true.

The `subscribeContext(context, callback, options)` method steps are:

1. If `context` is `NaN`, throw a `TypeError`.
2. If `signal` is aborted, return.
3. Create a subscription operation owned by `element`'s node document.
   Its delivery callback accepts a value and provider unsubscribe function.
4. Add an abort algorithm to `signal` that terminates the operation.
5. Append the operation to the document's context operations and dispatch it.

When a provider responds, the user agent invokes `callback(value)` synchronously.
It invokes the callback again when that provider supplies a new value or when the subscription transfers to a nearer provider.
Aborting the signal removes a pending subscription or invokes the active provider's unsubscribe function.

The user agent supplies the delivery callback to providers.
It manages provider handoff and invokes the consumer callback.
Authors using `subscribeContext()` therefore do not receive, compare, retain, or invoke provider unsubscribe functions.

The delivery callback throws an `InvalidStateError` `DOMException` when invoked after the operation terminates or by a provider registration that is no longer current.
If an active provider invokes it with an unsubscribe function other than that registration's stable function, it throws the same exception and does not deliver the value.

If the consumer callback throws during initial delivery, the new subscription is terminated as described for `respond()`.
If it throws during a later update, the user agent reports the exception, the exception does not propagate to the provider, and the subscription remains active.
A provider must continue delivering the update to its other subscribers.

See [Subscribing to a value](#subscribing-to-a-value) and [Providing a subscription](#providing-a-subscription) for author-facing examples.

## Direct subscription events

A directly constructed subscription request must supply a non-null signal.
On its first qualifying dispatch, the user agent binds the event to the dispatching element.
If the signal is already aborted, the event is dispatched without creating an operation, and `respond()` cannot accept it.
Otherwise, before invoking any event listener, the user agent creates a subscription operation in the dispatching state for the event, adds an abort algorithm that terminates the operation, and appends the operation to the element's node document's context operations.

The event remains associated with that operation after dispatch.
Redispatching it from the same element reuses the operation, so it cannot create a duplicate subscription or replace the original signal.
Before invoking a listener during a redispatch of a pending or active operation, the user agent records its state, current provider, and unsubscribe function and sets its state to dispatching.
After an initial dispatch or redispatch, an operation terminated during dispatch remains terminated.
Otherwise, a successful response leaves the operation active.
If no provider responds and the operation was previously active, the user agent restores its active state, current provider, and unsubscribe function.
If no provider responds and the operation was not previously active, the user agent sets it to pending.
The user agent then applies any group transfer deferred during dispatch and resumes any provider re-evaluation pass suspended on the operation.
A provider that responds while the operation remains pending or active can still deliver its current value again.
Calling `dispatchEvent()` with the event on another target throws an `InvalidStateError` `DOMException` before dispatch begins.
A separately constructed event represents a separate subscription, even if it was initialized with the same consumer callback.

Binding a direct subscription to one target keeps its operation owner and callback `this` value stable.
It also prevents one delivery callback from identifying operations for different elements or documents.

If no provider responds, the operation remains pending until a provider responds or its signal aborts.
When active, aborting its signal invokes its current provider's unsubscribe function.

Providers invoke the retained delivery callback for each later value, passing the same unsubscribe function used in `respond()`.
The user agent invokes the consumer callback with the value.
If one consumer callback throws, the user agent reports that exception; the provider continues delivering to its other active subscriptions.

See [Dispatching a direct subscription request](#dispatching-a-direct-subscription-request) for an author-facing example of same-target redispatch and cancellation.

# Context operations and document ownership

Each `Document` has an ordered set of context operations and a first-in-first-out queue of provider re-evaluation passes.
A context operation is a user-agent-owned record with:

- a kind: one-time request or subscription;
- its target element and context key;
- its consumer callback, delivery callback, and optional abort signal;
- a state: dispatching, pending, active, completed, or terminated;
- for a one-time request, its promise and resolving functions; and
- for an active subscription, its current provider element and unsubscribe function; and
- while establishing or transferring a subscription, its prospective provider element and unsubscribe function.

The operation belongs to the target element's node document: the `Document` to which the target belongs.
During a deferred adoption transfer, it remains owned by the old document until the group transfer steps run.
Composed event paths do not cross document boundaries, so context discovery never reaches an embedding or embedded document, regardless of origin.

An author-dispatched one-time event does not create an operation.
A qualifying subscription event creates or reuses an operation as described under direct subscription events.
The two `Element` methods create their operations before dispatch.

The ordered set provides deterministic re-evaluation order.
Operations are ordered by the time they are appended to their current document's set.
Method-created operations are appended before their initial event dispatch, and a directly constructed subscription is appended before the first listener invocation of its initial dispatch.
Re-dispatching an existing subscription does not move it in the order.

## Dispatching a context operation

To dispatch a context operation, the user agent:

1. Returns if the operation is completed or terminated.
2. If it has a signal and that signal is aborted, terminates the operation with the signal's abort reason and returns.
3. Records the operation's state and, if it is active, its current provider and unsubscribe function.
4. Sets its state to dispatching.
5. Creates a fresh `ContextRequestEvent` with type `context-request`, the operation's context and consumer callback, its subscription and signal values, `bubbles` and `composed` set to true, and `cancelable` set to false.
   It associates the event with the operation and its delivery callback.
6. Dispatches the event from the operation's target element.
7. Records whether a provider response completed successfully without terminating the operation or rolling back a prospective provider.
8. If the operation is completed, terminated, or active, leaves its state unchanged.
9. Otherwise, if the recorded state was active, restores its active state, provider, and unsubscribe function.
10. Otherwise, sets the operation to pending.
11. Applies any group transfer that was deferred during dispatch.
12. Resumes any provider re-evaluation pass that was suspended on this operation.
13. Returns whether a provider response completed successfully.

The composed path created by step 6 determines provider selection for that attempt.
No stored path is reused.

## Completing a one-time context operation

To complete a one-time context operation with a value, the user agent:

1. Returns if its state is completed or terminated.
2. Sets its state to completed.
3. Removes its abort algorithm and removes it from its document's context operations.
4. Resolves its promise with the value.

Promise reactions therefore run according to ordinary microtask timing, even when the provider responds synchronously during the method call.

## Terminating a context operation

To terminate a context operation, optionally with a reason, the user agent:

1. Returns if its state is completed or terminated.
2. Sets its state to terminated before running author code.
3. Removes its abort algorithm and removes it from its document's context operations.
4. If it has a current provider unsubscribe function, invokes that function and reports any exception.
5. If it is a one-time operation and a reason was supplied, rejects its promise with that reason.

Setting the state first makes termination idempotent and prevents cleanup from delivering another value during termination.

# Provider announcements and re-evaluation

A provider dispatches a qualifying `ContextProviderEvent` whenever its availability for a context changes.
On activation it installs its bubbling request listener before dispatching a fresh announcement from the providing element:

```js
provider.dispatchEvent(
  new ContextProviderEvent('context-provider', {
    context: themeContext,
    bubbles: true,
    composed: true,
  })
);
```

The announcement states only that provider availability for a context may have changed.
It does not assert that the announcing element provides that context or supports subscriptions.
A provider that cannot fulfill a request leaves it unanswered so another provider can respond.

After dispatching a qualifying provider announcement, the user agent appends a provider re-evaluation pass to the announcing element's node document.
The pass records the announcing element and context key and snapshots the document's affected pending, active, and dispatching operations with that key in operation order.
An operation is affected if:

- the announcing element is the operation's current provider; or
- the announcing element would be on the composed path of a fresh qualifying context request from the operation's target and is not the target itself.

Unless the document is already processing a pass, the user agent processes queued passes before the call to `dispatchEvent()` returns.
When an operation's turn begins, the user agent skips it if it no longer belongs to that document, is no longer affected, or has completed, terminated, or been aborted.
If the operation is dispatching, the user agent suspends the pass at that entry until the dispatch finishes instead of re-entering the operation.
Otherwise, it dispatches the operation and records whether a provider response completed successfully.
If the operation then no longer belongs to that document or has completed or terminated, the user agent skips the remaining rules for its turn.
After the operation's re-evaluation turn finishes, the user agent applies any group transfer deferred during that turn before advancing the pass.
The current-provider case lets a provider that moved, disconnected, or became unavailable release operations it previously fulfilled, even when it is no longer on their request paths.

Re-evaluation occurs even if an announcement listener stops propagation or throws.
The announcement notifies script; it does not control the user agent's processing.

For an active subscription, a successful response from a different provider transfers the operation under the provider handoff rules in Responding to a request.
A response from the same provider during user-agent-initiated re-evaluation confirms the existing registration.
It does not deliver the value again or create another registration, and the provider must supply the same unsubscribe function.
A repeated request dispatched directly by an author does deliver the current value.

If no provider responds successfully during re-evaluation, an active operation normally remains with its current provider.
The exception is when the pass's announcing element is that current provider element.
In that case the announcement means that provider's availability for the context has changed and the lack of a response means it no longer fulfills the operation.
The user agent records its unsubscribe function, clears the current provider registration, and makes the operation pending before invoking the recorded function and reporting any exception.

The user agent determines the path used by the affected-operation check without dispatching an event or invoking author code, and does not reuse a path from an earlier request.
Re-dispatch then determines whether the announcing element or another element is the nearest matching provider.
An announcement from an unrelated branch does not dispatch a request for that operation and has no effect on it.

The user agent does not combine separate announcements.
An announcement dispatched while a pass is running appends another pass, which starts after earlier queued passes finish.
An announcement nested inside a context request can return while its pass is suspended on that request.
The pass resumes after the current request dispatch finishes and before that outer request's `dispatchEvent()` call returns.
This prevents recursive delivery without losing an announcement or a provider that became reachable after the current event path was created.

# Lifecycle and topology

Each fresh request uses its current composed path to select a provider.
An ordinary DOM mutation does not change an active operation or trigger re-evaluation by itself.

- Disconnecting a target does not implicitly cancel its operations.
  A detached tree still has a composed path, and the element can later be reconnected.
- A custom element normally aborts component-lifetime operations in `disconnectedCallback()` and creates new signals when it reconnects.
- After a state-preserving move, the component aborts and reissues affected operations from its new position.
- Moving an ancestor or changing slot assignment can change the nearest provider without directly moving the consumer.
  Code that performs such a topology change must reissue affected operations or dispatch a provider announcement for each affected context.

When a target element is adopted into another document, the user agent transfers all of its unfinished operations as one ordered group.
If any operation in that group is being dispatched or re-evaluated, the user agent defers the group transfer until the current dispatch and response steps finish.
It then collects the operations in their order in the old document and removes them from that document.
For each active subscription, it records the old provider's unsubscribe function, clears the current provider registration, sets the operation to pending, and invokes the recorded function while reporting any exception.
After cleanup, it appends each still-unfinished operation to the new node document in the collected relative order.
It then dispatches each transferred operation once from its new position.
The user agent does not dispatch an operation in both documents at once.

When an element that is the current provider for operations whose targets remain in the old document is adopted, the user agent re-evaluates those operations in the old document after adoption completes, using the same operation order and non-reentrancy rules as a provider re-evaluation pass.
Since the adopted provider is no longer on their request paths, another provider can take over or the user agent applies the same transition to pending used when an announcing current provider no longer responds.
If both a target and its provider are adopted as part of the same subtree, the target-operation transfer rule above takes precedence and the operation is dispatched only in the new document.

When a provider deactivates, it first removes its request listener.
It then dispatches a qualifying provider announcement for each context it was providing.
Re-evaluation lets another matching provider take over; otherwise the rule above makes its affected operations pending.
The user agent invokes their unsubscribe functions during that transition, so no provider registration remains afterward.

Entering the back-forward cache preserves context operations with their document.
Script is suspended, so no context event is dispatched while the document is frozen.
Operations resume unchanged when the document is restored.

When a document is discarded rather than cached, the user agent terminates each unfinished context operation.
It supplies an `AbortError` `DOMException` as the reason when terminating a one-time operation.
Terminating a subscription invokes its provider's unsubscribe function, if present, without invoking its consumer callback.

# Garbage collection

A document's context-operation set keeps each unfinished operation and its target, callbacks, signal, and promise alive.
This prevents garbage collection from silently ending a pending promise or subscription.

Authors use `AbortSignal` to bound that lifetime.
Subscriptions therefore require a signal, and long-lived `requestContext()` calls should normally supply one.
When an operation completes, terminates, or is aborted, the user agent removes it from the document's context-operation set and releases those references.
The whole set can be collected after its document is discarded.

# Capability and security model

Context is intentionally an ancestor capability protocol.
Every listener on a request's composed path can observe its context key, callback, subscription mode, and signal.
The nearest matching ancestor can respond with any JavaScript value.
Closed shadow roots do not hide a request from ancestors and are not a security boundary for context.
Every listener on a provider announcement's composed path can observe its context key and unretargeted `contextTarget`.

An ancestor can also block a request by stopping its propagation without responding.
This follows ordinary DOM event authority: a component cannot bypass an ancestor that controls its event path.

Unique object and symbol keys prevent accidental collisions and responses from code that never observes a request.
They do not protect a consumer from an ancestor that can observe and answer that request.
Components must not request authority from an ancestor they do not trust.

Events and composed paths do not cross document boundaries, so this primitive does not add cross-origin or cross-frame communication.
Values and callbacks can cross JavaScript realm boundaries within one document under the ordinary DOM event and JavaScript object rules.

The API exposes no persistent device, user, or origin information and requires no permission.
Its privacy and security properties are those of same-document DOM events and direct JavaScript object exchange.

# Design decisions and alternatives

## Why a native API

The event-only Context Protocol can be, and has been, implemented by libraries.
Native support is not justified by an inability to provide context in author code.
It instead provides common entry points and one processing model for components that do not share a library implementation.

The user agent can consistently own pending one-time requests, subscription cancellation, provider handoff, adoption, document teardown, and re-evaluation ordering.
Without native ownership, each library must provide that retained state and lifecycle behavior, and independently loaded implementations must agree on more than the event shape to coordinate it.
Direct event dispatch remains available for synchronous discovery and low-level protocol participation.

[Earlier revisions of this proposal](https://github.com/webcomponents-cg/community-protocols/blob/90ddaeb765400d6595a6a18cb63896fdddf5efc3/proposals/context.md) define the event-only protocol, and [`@lit/context`](https://github.com/lit/lit/tree/main/packages/context) is a deployed implementation of that design.
The native API builds on that experience while making different choices where the user agent needs an atomic response and deterministic operation lifetime.

## Explicit `respond()` rather than callback invocation alone

Calling `respond()` gives the user agent an atomic, testable indication that a provider fulfilled a request.
Inferring fulfillment from propagation state or from an arbitrary callback invocation would leave the user agent unable to decide whether a late-provider operation should remain pending.
The callback remains exposed because subscription providers need it for later values.
It is a user-agent-created wrapper rather than the author callback so later updates can be accepted only from the current provider registration and rejected after handoff or termination.
Its stable identity gives providers a subscriber key that does not change across redispatch.

## Per-document operation ownership

Composed event paths do not cross document boundaries, so a per-document operation set matches the scope of provider discovery.
A global or agent-wide coordinator would create cross-document state without adding a provider that a request could reach.
Requiring authors to install a special context-root object would expose bookkeeping that the user agent can perform consistently itself.

## Exact context events rather than event names alone

Event names are shared author-controlled strings.
Requiring the context event interface and exact propagation flags prevents an unrelated `CustomEvent` from creating persistent user-agent state.
It also lets ordinary non-qualifying events continue to use the same names without special dispatch behavior.

## Relationship to earlier protocol implementations

Implementations of the earlier event-only protocol are not retroactively changed by this proposal.
Their author-created events are non-qualifying events, so the user agent dispatches them normally and does not create or fulfill native context operations.
Existing consumers and providers using the same earlier protocol therefore continue to interoperate with one another.

Mixed native and event-only implementations do not interoperate automatically.
An earlier provider fulfills requests by invoking `event.callback(value, unsubscribe)` directly, while a native request requires `event.respond(value, unsubscribe)` to establish the response atomically.
In the other direction, an earlier request event has no `respond()` method for a native-only provider to call.

A library that supports incremental adoption can implement a dual-mode provider.
It can feature-detect a callable `respond` member and use it for native requests, while retaining the earlier callback-invocation behavior for other requests.
Providers must be upgraded or adapted before consumers switch to `requestContext()` or `subscribeContext()`.
The user agent does not infer a native response from direct callback invocation because it could not then atomically validate the provider, stop competing responses, and install or transfer retained state.

## Stable `contextTarget`

The ordinary event `target` is retargeted across shadow boundaries and therefore does not identify one stable requesting or announcing element to every listener.
For a request, `contextTarget` supports operation identity, self-response checks, provider bookkeeping, and the consumer callback's `this` value.
For a provider announcement, it identifies the element whose availability changed.
Earlier implementations have also used an explicit context target for both event types.

Unlike `target`, `contextTarget` intentionally exposes the original element to ancestors across closed shadow roots.
That is additional encapsulation-visible information, even though a context request already gives those ancestors authority to observe, answer, or block it.
The protocol chooses stable participant identity over shadow retargeting because providers may need to associate retained work with the exact consumer or announcing provider.
Components must not dispatch context requests or provider announcements from an internal element whose identity they are unwilling to reveal to ancestors.

## Promise and subscription methods

A one-time operation that may wait is represented by a Promise.
Repeated values use a callback plus an `AbortSignal`, while direct event dispatch remains available for consumers that require synchronous one-time delivery or need to participate in the low-level protocol.
This keeps asynchronous waiting out of the event object itself.

## Optional cancellation for one-time requests

A one-time request may normally be fulfilled during its initial dispatch, so requiring every caller to create an `AbortController` would add ceremony to the common case.
The signal is therefore optional, and an unanswered request intentionally remains pending for a later provider.

That convenience has a lifetime cost: without a signal, the document retains the pending operation and its target, callback, and promise until it is fulfilled or the document is discarded.
Authors that cannot accept that lifetime should supply a signal.
Subscriptions require a signal because they have no successful response that completes the operation automatically.

## Cooperative topology updates

The user agent does not observe every DOM mutation or continuously recompute the nearest provider.
Doing so would make unrelated tree changes trigger protocol work and author code.
Instead, code that changes provider availability or relevant topology announces the change or reissues the affected operation.

Until that happens, an active subscription can remain attached to its previous provider even when a fresh request would select another provider.
This explicit invalidation model keeps observable work scoped to protocol participants and makes re-evaluation timing deterministic.

# Usage

This section is non-normative.

## Requesting one value

```js
const themeContext = Symbol('theme');

this.theme = await this.requestContext(themeContext, {
  signal: this.lifecycleSignal,
});
```

## Requesting synchronously with a direct event

```js
let found = false;

element.dispatchEvent(
  new ContextRequestEvent('context-request', {
    context: themeContext,
    callback: (theme) => {
      found = true;
      element.theme = theme;
    },
    bubbles: true,
    composed: true,
  })
);

if (!found) {
  // No provider responded during dispatch.
}
```

## Providing one value

```js
this.addEventListener('context-request', (event) => {
  if (event.context !== themeContext || event.subscribe) {
    return;
  }

  event.respond(this.theme);
}, { capture: false });

this.dispatchEvent(
  new ContextProviderEvent('context-provider', {
    context: themeContext,
    bubbles: true,
    composed: true,
  })
);
```

## Subscribing to a value

```js
const loggerContext = Symbol('logger');

class LoggerConsumer extends HTMLElement {
  #contextController;

  connectedCallback() {
    this.#contextController = new AbortController();

    this.subscribeContext(
      loggerContext,
      this.loggerCallback,
      { signal: this.#contextController.signal }
    );
  }

  loggerCallback(logger) {
    this.logger = logger;
  }

  disconnectedCallback() {
    this.#contextController.abort();
    this.#contextController = undefined;
    this.logger = undefined;
  }
}
```

## Providing a subscription

```js
this.addEventListener('context-request', (event) => {
  if (event.context !== loggerContext || !event.subscribe) {
    return;
  }

  const callback = event.callback;
  const existing = this.loggerSubscribers.get(callback);

  if (existing) {
    event.respond(this.logger, existing);

    return;
  }

  let active = true;

  const unsubscribe = () => {
    if (!active) {
      return;
    }

    active = false;

    this.loggerSubscribers.delete(callback);
  };

  this.loggerSubscribers.set(callback, unsubscribe);

  try {
    event.respond(this.logger, unsubscribe);
  } catch (error) {
    unsubscribe();

    throw error;
  }
}, { capture: false });

setLogger(logger) {
  this.logger = logger;

  for (const [callback, unsubscribe] of this.loggerSubscribers) {
    try {
      callback(logger, unsubscribe);
    } catch (error) {
      reportError(error);
    }
  }
}
```

## Dispatching a direct subscription request

Direct event dispatch exposes synchronous initial delivery while retaining the subscription for later updates:

```js
const controller = new AbortController();

const request = new ContextRequestEvent('context-request', {
  context: loggerContext,
  callback(logger) {
    console.assert(this === consumer);
    consumer.logger = logger;
  },
  subscribe: true,
  signal: controller.signal,
  bubbles: true,
  composed: true,
});

consumer.dispatchEvent(request);

// Ask the current provider to deliver its value again without creating a
// second subscription.
consumer.dispatchEvent(request);

controller.abort();
```

The request is bound to `consumer` by its first qualifying dispatch.
Dispatching it from another element throws an `InvalidStateError` `DOMException`.

## TypeScript integration

Web IDL defines the native API surface and does not express a relationship between a context key and its value type.
A TypeScript library can add that static relationship with branded keys and typed wrappers:

```ts
declare const contextValueType: unique symbol;

type Context<T> = symbol & {
  readonly [contextValueType]: T;
};

function createContext<T>(description: string): Context<T> {
  return Symbol(description) as Context<T>;
}

function requestTypedContext<T>(
  element: Element,
  context: Context<T>,
  options: {signal?: AbortSignal} = {},
): Promise<T> {
  return element.requestContext(context, options) as Promise<T>;
}

function subscribeTypedContext<T>(
  element: Element,
  context: Context<T>,
  callback: (value: T) => void,
  options: {signal: AbortSignal},
): void {
  element.subscribeContext(context, callback, options);
}

interface Logger {
  log(message: string): void;
}

declare const consumer: Element;
declare const lifecycleSignal: AbortSignal;

const loggerContext = createContext<Logger>('logger');
const logger = await requestTypedContext(consumer, loggerContext, {
  signal: lifecycleSignal,
});

logger.log('Context received');

subscribeTypedContext(
  consumer,
  loggerContext,
  (updatedLogger) => updatedLogger.log('Context updated'),
  {signal: lifecycleSignal},
);
```

These helpers provide compile-time checking only.
They do not validate a provider's value at runtime or change the native methods' Web IDL signatures.

# Processing-model summary

The observable behavior above requires user-agent state for:

- the response status of each `ContextRequestEvent` dispatch;
- each document's ordered context-operation set and provider re-evaluation queue;
- abort algorithms and current provider cleanup for active subscriptions; and
- adoption, document teardown, and garbage-collection integration.

The protocol does not observe every DOM mutation or cache a composed path.
Provider matching always uses ordinary event dispatch.
How a user agent indexes operations is not observable, provided key comparison remains strict equality and the specified ordering is preserved.

# Proposed web-platform test coverage

The corresponding web-platform tests should cover at least:

- WebIDL exposure, constructor defaults, and invalid `NaN` keys;
- qualifying and non-qualifying event combinations, including legacy-shaped author events that remain ordinary events;
- `contextTarget` initialization, stability, same-target redispatch, rejection of different-target redispatch, shadow retargeting, and original-target exposure across closed roots;
- rejection of responses from capturing listeners, the request target, duplicate providers, and late calls, plus acceptance from a non-capturing shadow-host listener whose `eventPhase` is `AT_TARGET`;
- synchronous response and asynchronous Promise reaction ordering;
- one-time immediate, pending, abort-before-dispatch, abort-during-dispatch without state resurrection, direct-event signal behavior, thenable, and late-provider fulfillment;
- method-based and direct subscriptions, required and already-aborted signals, updates, callback `this` values, idempotency, and terminal-event redispatch;
- initial and later callback exceptions, non-propagation of reported consumer exceptions, and cleanup exceptions;
- nested open and closed shadow roots, slots, and nearest-provider selection;
- provider activation, replacement, deactivation, failed transfer, and failed replacement during current-provider deactivation;
- unrelated announcements that dispatch no requests, operation-order processing including reentrant operation creation, and announcements dispatched during initial dispatch or re-evaluation;
- disconnect, reconnect, state-preserving moves, slot changes, adoption cleanup, and transferred-operation ordering;
- same-origin and cross-origin iframe boundaries;
- back-forward cache restoration and document teardown; and
- garbage-collection tests using weak references where the harness supports deterministic collection.

State-machine tests should also cover sequences that combine dispatch, response, abort, announcement, move, adoption, and teardown rather than testing those operations only in isolation.
