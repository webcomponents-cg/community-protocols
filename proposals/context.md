# Context Protocol

An open protocol for data passing between components.

Author: Benjamin Delarre

Document status: Candidate

Last update: 2021-8-26

# Background

There are a number of scenarios where a web component might require data that is provided from outside itself. While components can specify properties and attributes to receive that data imperatively or declaratively, it is often the case that the data may be owned somewhere further up the DOM tree.

One approach to passing this data down to components is commonly referred to as *prop drilling*, whereby components pass properties all the way down through the hierarchy, passing from component to component until it is consumed at its destination. This is generally considered undesirable as it often requires intermediate components in the tree to have knowledge of the data necessary in its descendents.

Frameworks and libraries often provide mechanisms for this, these can range from the simple Context implementation available in React, to more complex Dependency Injection frameworks. Web components need a similar protocol in order to solve this problem.

# Goals

- Allow elements in the DOM to retrieve data based on their contextual position in the DOM
- Alleviate the problem of *prop drilling*
- Simple API that is easily implemented in any framework / library
- Synchronous protocol, while supporting asynchronous patterns
- Support single or multiple delivery of context values

# Non-Goals

**Context API !== Dependency Injection Framework**

The Context API does not intend to cover all cases and forms of Dependency Injection. It does not specify constructor, factory or property injection patterns. Its only goal is to formalize the pattern of sharing data across the hierarchy in the DOM, specifically avoiding *prop drilling* type scenarios. Dependency Injection patterns could be implemented using this protocol, but this is not the goal and should remain explicitly outside the scope of Context API for simplicity.

**Context API is not a state management alternative**

State management libraries often need to solve some similar problems that the Context API helps to solve. An element deep in the DOM tree may need access to some state, and may need to respond to that state being changed.

While state management could be built using the Context API, it is not a primary goal of the Context API to solve this problem. It is, however, appropriate for state management libraries to use the Context API to resolve state stores and other associated dependencies from deep within the DOM hierarchy; e.g. a component could request a Redux state store via Context.

# Overview

At a high level, the Context API is an event-based protocol that components can use to retrieve data from any location in the DOM:

- A component requiring some data fires a `context-request` event.
- The event carries a `context` value that denotes the data requested, and a `callback` which will receive the data.
- Providers can attach event listeners for `context-request` events to handle them and provide the requested data.
- Once a provider satisfies a request it calls `stopImmediatePropagation()` on the event.

# Details

## The `context-request` event

Components which wish to receive some data from their ancestors should initiate the request by firing a composed, bubbling, `context-request` Event, with a `callback` property.

TypeScript interface:

```typescript
interface ContextRequestEvent<T extends Context<unknown, unknown>> extends Event {
  /**
   * The name of the context that is requested
   */
  readonly context: T;
  /**
   * A boolean indicating if the context should be provided more than once.
   */
  readonly subscribe?: boolean;
  /**
   * A callback which a provider of this named callback should invoke.
   */
  readonly callback: ContextCallback<T>;
}
```

A full TypeScript definition for this event and its associated types can be found in the [Definitions](#definitions) section at the end of this document.

## Context objects

`ContextRequestEvent`s carry a `context` value that is used to identify specific contexts. This value may sometimes be referred to as the "context key", and can be of any type.

### Context equality

Matching contexts between a provider and consumer is done with strict equality (`===`). This means that a context can be made guaranteed unique by using a key value that's unique under strict equality, like a unique Symbol (not using `Symbol.for()`) or an object. A context can be intentionally made to match other contexts by using a key that's not unique under strict equality like a string or `Symbol.for()`.

### Context types

In TypeScript it's possible to cast values to an interface that carries additional type information. This is useful for contexts to convey the type of the value that the context provides.

To be interoperable, implementations should cast context keys to a type with the `__context__` key:

```ts
export type Context<KeyType, ValueType> = KeyType & {__context__: ValueType};
```

Then values can be cast to this to create a typed context key:

```ts
export const myContext = 'my-context' as Context<string, number>;
```

The value type of a Context can then be extracted with a utility type:

```ts
export type ContextType<T extends UnknownContext> =
  T extends Context<infer _, infer V> ? V : never;
```

Usage:
```ts
// MyContextType = number
type MyContextType = ContextType<typeof myContext>;
```

It is recommended that TypeScript-based implementations provide both `Context` and `ContextType` types.

### `createContext` functions

It is recommended that TypeScript implementations provide a `createContext()` function which is used to create a `Context`. This function can just cast to a `Context`:

```ts
export const createContext = <ValueType>(key: unknown) =>
    key as Context<typeof key, ValueType>;
```

## Context Providers

A context provider will satisfy a `context-request` event, passing the `callback` the requested data whenever the data changes. A provider will attach an event listener to the DOM tree to catch the event, and if it will be able to satisfy the request it _MUST_ call `stopImmediatePropagation` on the event.

If the provider has data available to satisfy the request then it should immediately invoke the `callback` passing the data. If the event has a truthy `subscribe` property, then the provider can assume that the `callback` can be invoked multiple times, and may retain a reference to the callback to invoke as the data changes. If this is the case the provider should pass the second `unsubscribe` parameter to the callback when invoking it in order to allow the requester to disconnect itself from the providers notifications.

The provider _MUST NOT_ retain a reference to the `callback` nor pass an `unsubscribe` callback if the `context-request` event's `subscribe` property is not truthy. Doing so may cause a memory leak as the consumer may not ever call the `unsubscribe` callback.

To safeguard against memory leaks caused by non-compliant consumers that don't call the `unsubscribe` callback, it is recommended that the provider uses WeakRefs to reference subscription callbacks.

The provider _SHOULD_ call `stopImmediatePropagation` before invoking the callback, or call the callback in a try/catch block, to ensure that an error thrown by the callback does not prevent immediate propagation from being stopped:

```js
this.addEventListener('context-request', event => {
  event.stopImmediatePropagation();
  // If the callback throws, propagation is already stopped
  event.callback('some data');
});
```

A provider does not necessarily have to be a Custom Element, but this may be a convenient mechanism.

## Usage

An element which wishes to receive some context and participate in the Context API should emit an event with the `context-request` type. It is suggested that an implementation of the `ContextRequestEvent` would be used something like this:

```js
// get a context from somewhere (this could be in any module)
const coolThingContext = createContext('cool-thing');

this.dispatchEvent(
  new ContextRequestEvent(
    coolThingContext, // the context we want to retrieve
    (coolThing) => {
      this.myCoolThing = coolThing; // do something with value
    }
  )
);
```

If a provider listening for this event can provide the requested context it will invoke the callback passed in the payload of the event. The element can then do whatever it wishes with this value.

It may also be the case that a provider can retain a reference to this callback, and can then invoke the callback multiple times. In this case providers should pass an unsubscribe function as a second argument to the callback to allow consumers to inform the provider that it should no longer update the element, and should dispose of the callback.

An element may also provide a `subscribe` boolean on the event detail to indicate that it is interested in receiving updates to the value.

Consumers should be aware that given that there is a loose coupling between implementations with this protocol that they may need to implement the `callback` handling defensively. An example of a defensive consumer that only wants a value once is provided below:

```js
let providedAlready = false;
this.dispatchEvent(
  // Note, this event is not a subscribing event:
  new ContextRequestEvent(coolThingContext, (coolThing, unsubscribe) => {
    // Guard against multiple callback calls in case of bad actor providers
    if (!providedAlready) {
      this.myCoolThing = coolThing; // do something with value
    }
    // `unsubscribe()` should be given if `subscribe` was true on the request 
    // event. But if a bad provider passed an unsubscribe callback anyway,
    // you could unsubscribe immediately since we only want it once.
    unsubscribe?.();
  })
);
```

It is recommended that custom elements which participate in the Context API should fire their `context-request` events in their `connectedCallback()` method. Likewise in their `disconnectedCallback()` method they should invoke any unsubscribe callbacks they have received.

A more complete example is as follows:

```js
class SimpleElement extends HTMLElement {
  connectedCallback() {
    this.dispatchEvent(
      new ContextRequestEvent(
        loggerContext,
        (value, unsubscribe) => {
          // Call the old unsubscribe callback if the unsubscribe call has
          // changed. This probably means we have a new provider.
          if (unsubscribe !== this.loggerUnsubscribe) {
            this.loggerUnsubscribe.?();
          }
          this.logger = value;
          this.loggerUnsubscribe = unsubscribe;
        },
        true // we want this event multiple times (if the logger changes)
      )
    );
  }
  disconnectedCallback() {
    this.loggerUnsubscribe?.();
    this.loggerUnsubscribe = undefined;
    this.logger = undefined;
  }
}
```

# Open Questions / Previous considerations

## Could we use Promises instead of callbacks?

Many have commented that Promises could be used instead of callback functions. One major drawback of promises is that they cannot be resolved synchronously which would complicate usage of the Context API for simple use-cases.

Another issue with promises is that they do not allow multiple-resolution. Therefore we would not be able to handle cases where a requested value changes over time. This is a capability that we see in the React Context API and believe is valuable for a variety of use cases.

While we could restrict this API to only support a single resolution of a requested value, and then use observable mechanisms on that value to achieve data update behaviors, it is believed that this will complicate simple use-cases unnecessarily.

## Should requesters get to 'accept' providers?

The current API as proposed does not allow a requestor to 'approve' that a provider is going to give it the right object. We have some capability to enforce this in TypeScript, but we could provide a slightly different API that would allow the requesting component to check the value it will receive:

```js
this.dispatchEvent(
  new ContextRequestEvent(loggerContext, (candidate) => {
    if (typeof candidate.log === 'function' && typeof candidate.info === 'function') {
      // we can accept this candidate so return the callback to the provider
      return (logger, unsubscribe) => {
        this.logger = logger;
      };
    }
  });
)
```

In this alternative, we expect to always synchronously receive a 'candidate' data value that a provider may give us. We can then inspect this candidate value in our component to determine if it has the correct shape, and then return our callback function to accept updates to it.

In this proposal we would likely enforce that the callback always be invoked synchronously.

Alternative APIs could also be explored in this approach, we could for instance have providers append themselves to a list of potential providers along with candidate value objects, and then allow our components to pick which provider they wish to use:

```js
const contextRequest = new ContextRequestEvent(loggerContext);
this.dispatchEvent(context);
if (!contextRequest.providers) {
  // no providers for logger
  return;
}
const provider = contextRequest.providers.find((loggerProvider) => {
  // test if the provider is the type we want or the value it provides is right
});
const unsubscribe = provider.provide(this, (logger) => {
  this.logger = logger;
});
// later...
unsubscribe(); // don't need updates anymore
```

These alternatives do provide more capability, but it's an open question as to whether or not this complexity is warranted or desired. It also opens up a larger question about what would the candidate value be? Would it have to be an object of the requested type, could it be some other protocol to determine uniformity between the requested data and the actual data? This begins to seem more complex than we really need here for unnecessary type safety overhead. It is suggested if consumers want type safety then they should use TypeScript to achieve this.

# Definitions

Below are some TypeScript definitions for the common parts of the proposed protocol:

```typescript
/**
 * A context key.
 *
 * A context key can be any type of object, including strings and symbols. The
 *  Context type brands the key type with the `__context__` property that
 * carries the type of the value the context references.
 */
export type Context<KeyType, ValueType> = KeyType & {__context__: ValueType};

/**
 * An unknown context type
 */
export type UnknownContext = Context<unknown, unknown>;

/**
 * A helper type which can extract a Context value type from a Context type
 */
export type ContextType<T extends UnknownContext> =
  T extends Context<infer _, infer V> ? V : never;

/**
 * A function which creates a Context value object
 */
export const createContext = <ValueType>(key: unknown) =>
  key as Context<typeof key, ValueType>;

/**
 * A callback which is provided by a context requester and is called with the value satisfying the request.
 * This callback can be called multiple times by context providers as the requested value is changed.
 */
export type ContextCallback<ValueType> = (
  value: ValueType,
  unsubscribe?: () => void
) => void;

/**
 * An event fired by a context requester to signal it desires a named context.
 *
 * A provider should inspect the `context` property of the event to determine if it has a value that can
 * satisfy the request, calling the `callback` with the requested value if so.
 *
 * If the requested context event contains a truthy `subscribe` value, then a provider can call the callback
 * multiple times if the value is changed, if this is the case the provider should pass an `unsubscribe`
 * function to the callback which requesters can invoke to indicate they no longer wish to receive these updates.
 */
export class ContextRequestEvent<T extends UnknownContext> extends Event {
  public constructor(
    public readonly context: T,
    public readonly callback: ContextCallback<ContextType<T>>,
    public readonly subscribe?: boolean
  ) {
    super('context-request', {bubbles: true, composed: true});
  }
}

declare global {
  interface HTMLElementEventMap {
    /**
     * A 'context-request' event can be emitted by any element which desires
     * a context value to be injected by an external provider.
     */
    'context-request': ContextRequestEvent<Context<unknown, unknown>>;
  }
}
```
