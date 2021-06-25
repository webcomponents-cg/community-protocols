# Context Protocol

An open protocol for data passing between components.

Author: Benjamin Delarre

Document status: Draft

Last update: 2021-6-25

# Background

There are a number of scenarios where a web component might require data that is provided from outside itself. While components can specify properties and attributes to receive that data imperatively or declaratively, it is often the case that the data may be owned somewhere further up the DOM tree.

One approach to passing this data down to components is commonly referred to as 'prop drilling', whereby components pass properties all the way down through the hierarchy, passing from component to component until it is consumed at its destination. This is generally considered undesirable as it often requires intermediate components in the tree to have knowledge of the data necessary in its descendents.

Frameworks and libraries often provide mechanisms for this, these can range from the simple Context implementation available in React, to more complex Dependency Injection frameworks. Web components need a similar protocol in order to solve this problem.

# Goals

- Allow elements in the DOM to retrieve data based on their contextual position in the DOM
- Alleviate the problem of 'prop drilling'
- Simple API that is easily implemented in any framework / library
- Synchronous protocol, while supporting asynchronous patterns
- Support single or multiple delivery of context values

# Non-Goals

**Context API !== Dependency Injection Framework**

The Context API does not intend to cover all cases and forms of Dependency Injection. It does not specify constructor, factory or property injection patterns. Its only goal is to formalize the pattern of sharing data across the hierarchy in the DOM, specifically avoiding 'prop drilling' type scenarios. Dependency Injection patterns could be implemented using this protocol, but this is not the goal and should remain explicitly outside the scope of Context API for simplicity.

**Context API is not a state management alternative**

State management libraries often need to perform similar behaviors to the problems that Context API helps to solve. An element deep in the DOM tree made need access to some state, and may need to respond to that state being changed. While state management could be built using the Context API, it is not a primary goal of Context API to solve this problem. It is however most appropriate for state management libraries to use Context API to resolve state stores and other associated dependencies from deep within the DOM hierarchy, e.g. a component could request a Redux state store via Context.

# Overview

At a high level, the Context API is an event based protocol that components can use to retrieve data from any location in the DOM:

- A component requiring some data fires a `context-request` event.
- The event carries a `context` value that denotes the data requested, and a `callback` which will receive the data.
- Providers can attach event listeners for `context-request` events to handle them and provide the requested data.
- Once a provider satisfies a request it calls `stopPropagation()` on the event.

# Details

## The `context-request` event

Components which wish to receive some data from their ancestors should initiate the request by firing a composed, bubbling, `context-request` Event, with a `callback` property.

Typescript interface:

```typescript
interface ContextEvent<T extends Context<unknown>> extends Event {
  /**
   * The name of the context that is requested
   */
  readonly context: T;
  /**
   * A boolean indicating if the context should be provided more than once.
   */
  readonly multiple?: boolean;
  /**
   * A callback which a provider of this named callback should invoke.
   */
  readonly callback: ContextCallback<T>;
}
```

A full typescript definition for this event and its associated types can be found in the [Definitions](#definitions) section at the end of this document.

## The `createContext` function

Implementations should provide a `createContext` function which is used to create a well known value to identify a specific Context. These well known values could be published in lightweight NPM packages to facilitate community compatibility.

It is suggested the `createContext` implementation take a string identifier to facilitate debugging, and an optional initial value which consumers could use if no Context is provided. The function should return a `Context` object with the following type definition:

```ts
export type Context<T> = {
  name: string;
  initialValue?: T;
};
```

It is proposed that the `community-protocols` repository will publish a package that contains a default `createContext` implementation. An example implementation is as follows:

```ts
export type UnknownContext = Context<unknown>;

export type ContextType<T extends UnknownContext> = T extends Context<infer Y>
  ? Y
  : never;

export function createContext<T>(name: string, initialValue?: T): Context<T> {
  return {
    name,
    initialValue,
  };
}
```

## Context Providers

A context provider will satisfy a `context-request` event, passing the `callback` the requested data whenever the data changes. A provider will attach an event listener to the DOM tree to catch the event, and if it will be able to satisfy the request _MUST_ call `stopPropagation` on the event.

If the provider has data available to satisfy the request then it should immediately invoke the `callback` passing the data. If the event has a truthy `multiple` property, then the provider can assume that the `callback` can be invokved multiple times, and may retain a reference to the callback to invoke as the data changes. If this is the case the provider should pass the second `dispose` parameter to the callback when invoking it in order to allow the requester to disconnect itself from the providers notifications.

A provider does not necessarily have to be a Custom Element, but this may be a convenient mechanism.

## Usage

An element which wishes to receive some context and participate in the Context API should emit an event with the `context-request` type. It is suggested that an implementation of the `ContextEvent` would be used something like this:

```js
// get a context from somewhere (this could be in any module)
const coolThingContext = createContext('cool-thing');

this.dispatchEvent(
    new ContextEvent(
        coolThingContext, // the context we want to retrieve
        callback: (coolThing) => {
            this.myCoolThing = coolThing; // do something with value
        }
    )
);
```

If a provider listening for this event can provide the requested context it will invoke the callback passed in the payload of the event. The element can then do whatever it wishes with this value.

It may also be the case that a provider can retain a reference to this callback, and can then invoke the callback multiple times. In this case providers should pass a dispose function as a second argument to the callback to allow consumers to inform the provider that it should no longer update the element, and should dispose of the callback.

An element may also provide a `multiple` boolean on the event detail to indicate that it is interested in receiving updates to the value.

Consumers should be aware that given that there is a loose coupling between implementations with this protocol that they may need to implement the `callback` handling defensively. An example is provided below:

```js
this.dispatchEvent(
  new ContextEvent(coolThingContext, (coolThing, dispose) => {
    // if we were given a disposer, this provider is likely to send us updates
    if (dispose) {
      // so dispose immediately if we only want it once
      dispose();
    }
    // guard against multiple assignment in case of bad actor providers
    if (!this.myCoolThing) {
      this.myCoolThing = coolThing; // do something with value
    }
  })
);
```

It is recommended that custom elements which participate in the Context API should fire their `context-request` events in their `connectedCallback` handler. Likewise in their `disconnectedCallback` they should invoke any dispose functions they have received.

A more complete example is as follows:

```js
class SimpleElement extends HTMLElement {
  connectedCallback() {
    this.dispatchEvent(
      new ContextEvent(
        loggerContext,
        (value, dispose) => {
          // protect against changing providers
          if (dispose && dispose !== this.loggerDisposer) {
            this.dispose();
          }
          this.logger = value;
          this.loggerDisposer = dispose;
        },
        true // we want this event multiple times (if the logger changes)
      )
    );
  }
  disconnectedCallback() {
    if (this.loggerDisposer) {
      this.loggerDisposer();
    }
    this.loggerDisposer = undefined;
    this.logger = undefined;
  }
}
```

# Open Questions / Previous considerations

## Could we use Promises instead of callbacks?

Many have commented that Promises could be used instead of callback functions. One major drawback of promises is that they cannot be resolved synchronously which would complicate usage of the Context API for simple use-cases.

Another issue with promises is that they do not allow multiple-resolution. Therefore we would not be able to handle cases where a requested value changes over time. This is capability that we see in the React Context API and believe is valuable for a variety of use cases.

While we could restrict this API to only support a single resolution of a requested value, and then use observable mechanisms on that value to achieve data update behaviors, it is believe this will complicate simple use-cases unnecessarily.

## Should requesters get to 'accept' providers?

The current API as proposed does not allow a requestor to 'approve' that a provider is going to give it the right object. We have some capability to enforce this in Typescript, but we could provide a slightly different API that would allow the requesting component to check the value it will receive:

```js
this.dispatchEvent(
  new ContextEvent(loggerContext, (candidate) => {
    if (typeof candidate.log === 'function' && typeof candidate.info === 'function) {
      // we can accept this candidate so return the callback to the provider
      return (logger, dispose) => {
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
const contextRequest = new ContextEvent(loggerContext);
this.dispatchEvent(context);
if (!contextRequest.providers) {
  // no providers for logger
  return;
}
const provider = contextRequest.providers.find((loggerProvider) => {
  // test if the provider is the type we want or the value it provides is right
});
const dispose = provider.provide(this, (logger) => {
  this.logger = logger;
});
// later...
dispose(); // don't need updates anymore
```

These alternatives do provide more capability, but its an open question as to whether or not this complexity is warranted or desired. It also opens up a larger question about what would the candidate value be? Would it have to be an object of the requested type, could it be some other protocol to determine uniformity between the requested data and the actual data? This begins to seem more complex than we really need here for unnecessary type safety overhead. It is suggested if consumers want type safety then they should use Typescript to achieve this.

# Definitions

Below are some typescript definitions for the common parts of the proposed protocol:

```typescript
/**
 * A Context object defines an optional initial value for a Context, as well as a name identifier for debugging purposes.
 */
export type Context<T> = {
  name: string;
  initialValue?: T;
};

/**
 * An unknown context typeU
 */
export type UnknownContext = Context<unknown>;

/**
 * A helper type which can extract a Context value type from a Context type
 */
export type ContextType<T extends UnknownContext> = T extends Context<infer Y>
  ? Y
  : never;

/**
 * A function which creates a Context value object
 */
export function createContext<T>(
  name: string,
  initialValue?: T
): Readonly<Context<T>> {
  return {
    name,
    initialValue,
  };
}

/**
 * A callback which is provided by a context requester and is called with the value satisfying the request.
 * This callback can be called multiple times by context providers as the requested value is changed.
 */
export type ContextCallback<ValueType> = (
  value: ValueType,
  dispose?: () => void
) => void;

/**
 * An event fired by a context requester to signal it desires a named context.
 *
 * A provider should inspect the `context` property of the event to determine if it has a value that can
 * satisfy the request, calling the `callback` with the requested value if so.
 *
 * If the requested context event contains a truthy `multiple` value, then a provider can call the callback
 * multiple times if the value is changed, if this is the case the provider should pass a `dispose`
 * method to the callback which requesters can invoke to indicate they no longer wish to receive these updates.
 */
export class ContextEvent<T extends UnknownContext> extends Event {
  public constructor(
    public readonly context: T,
    public readonly callback: ContextCallback<ContextType<T>>,
    public readonly multiple?: boolean
  ) {
    super("context-request", { bubbles: true, composed: true });
  }
}

declare global {
  interface HTMLElementEventMap {
    /**
     * A 'context-request' event can be emitted by any element which desires
     * a context value to be injected by an external provider.
     */
    "context-request": ContextEvent<UnknownContext>;
  }
}
```
