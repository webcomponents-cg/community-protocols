# Context Protocol

An open protocol for data passing between components.

Author: Benjamin Delarre

Document status: Draft

Last update: 2021-4-27

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

- A component requiring some data fires a `request-context` event.
- The event carries a `name` that denotes the data requested, and a `callback` which will receive the data.
- Providers can attach event listeners for `request-context` events to handle them and provide the requested data.
- Once a provider satisfies a request it calls `stopPropagation()` on the event.

# Details

## The `request-context` event

Components which wish to receive some data from their ancestors should initiate the request by firing a composed, bubbling, `request-context` Event, with a `callback` property.

Typescript interface:

```typescript
interface RequestContextEvent extends Event {
  /**
   * The name of the context that is requested
   */
  readonly name: T;
  /**
   * A boolean indicating if the context should only be provided once.
   */
  readonly once: boolean;
  /**
   * A callback which a provider of this named callback should invoke.
   */
  readonly callback: RequestContextCallback<ContextTypeMap[T]>;
}
```

A full typescript definition for this event and its associated types can be found in the [Definitions](#definitions) section at the end of this document.

## Context Providers

A context provider will satisfy a `request-context` event, passing the `callback` the requested data whenever the data changes. A provider will attach an event listener to the DOM tree to catch the event, and if it will be able to satisfy the request _MUST_ call `stopPropagation` on the event.

If the provider has data available to satisfy the request then it should immediately invoke the `callback` passing the data. If the event does NOT have a truthy `once` property, then the provider can assume that the `callback` can be invokved multiple times, and may retain a reference to the callback to invoke as the data changes. If this is the case the provider should pass the second `dispose` parameter to the callback when invoking it in order to allow the requester to disconnect itself from the providers notifications.

A provider does not necessarily have to be a Custom Element, but this may be a convenient mechanism.

## Usage

An element which wishes to receive some context and participate in the Context API should emit an event with the `request-context` type. It is suggested that an implementation of the `RequestContextEvent` would be used something like this:

```js
this.dispatchEvent(
    new RequestContextEvent(
        'cool-thing', // the name of the context we want to receive
        callback: (coolThing) => {
            this.myCoolThing = coolThing; // do something with value
        }
    )
);
```

If a provider listening for this event can provide the requested context it will invoke the callback passed in the payload of the event. The element can then do whatever it wishes with this value.

It may also be the case that a provider can retain a reference to this callback, and can then invoke the callback multiple times. In this case providers should pass a dispose function as a second argument to the callback to allow consumers to inform the provider that it should no longer update the element, and should dispose of the callback.

As a convenience, and a hint for providers, an element may also provide a once boolean on the event detail to indicate that it is not interested in receiving updates to the value. If this behavior is essential to the correct operation of the consumer, then they should be implemented defensively as there is no guarantee that providers will honor this agreement. An example is provided below:

```js
this.dispatchEvent(
  new RequestContextEvent(
    "cool-thing-we-want-once",
    (coolThing, dipose) => {
      // if we were given a disposer, this provider is likely to send us updates
      if (dispose) {
        // so dispose immediately
        dispose();
      }
      // guard against multiple assignment in case of bad actor providers
      if (!this.myCoolThing) {
        this.myCoolThing = coolThing; // do something with value
      }
    },
    true // we only want the event once
  )
);
```

It is recommended that custom elements which participate in the Context API should fire their context-request events in their connectedCallback handler. Likewise in their disconnectedCallback they should invoke any dispose functions they have received.

A more complete example is as follows:

```js
class SimpleElement extends HTMLElement {
  connectedCallback() {
    this.dispatchEvent(
      new RequestContextEvent("logger", (value, dispose) => {
        // protect against changing providers
        if (dispose && dispose !== this.loggerDisposer) {
          this.dispose();
        }
        this.logger = value;
        this.loggerDisposer = dispose;
      })
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

## What about a `createContext` style API?

In React Context API they provide a function `createContext` which creates a context object that uniquely identifies the context to consumers and providers. This has some advantages in that it means you have a unique object that identifies a context. However the downside is that you need to have an agreed upon module that exports this context value in order to share the context.

For web components this creates a centralization issue. If all components want is a logger context, where would the logger context object be defined? Who owns that module? Will all consumers agree to depend upon it?

Thus far we have determined that a 'looser' approach will serve us better and avoid centralization concerns. But this is still an open question and deserves further debate.

## How to handle name conflicts?

Since we do not have a `createContext` style API, we have the potential for name conflicts. It is suggested that we promote a pattern of namespacing the context type strings. For instance: `common:logger`. The community protocol repo would be a good place to document common namespaces and their intents.

If we have to actually resolve a conflict in an application then a 'MappingContextProvider' is theorized. This provider would receive context events and remap them to namespaced context events then emit them up the tree. In this manner we can map or reroute context events easily.

A possible implementation of this will be added to the `lit-labs` project soon.

## Should requesters get to 'accept' providers?

The current API as proposed does not allow a requestor to 'approve' that a provider is going to give it the right object. We have some capability to enforce this in Typescript, but we could provide a slightly different API that would allow the requesting component to check the value it will receive:

```js
this.dispatchEvent(
  new RequestContextEvent('logger', (candidate) => {
    if (typeof candidate.log === 'function' && typeof candidate.info === 'function) {
      // we can accept this candidate
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
const contextRequest = new RequestContextEvent("logger");
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
 * An interface map to store context type strings to type definition mappings
 */
export interface ContextTypeMap {}

declare global {
  interface HTMLElementEventMap {
    /**
     * A 'context-request' event can be emitted by any element which desires
     * a context value to be injected by an external provider.
     */
    "context-request": ContextEvent<keyof ContextTypeMap>;
  }
}

/**
 * A callback which is provided by a context requester and is called with the value satisfying the request.
 * This callback can be called multiple times by context providers as the requested value is changed.
 */
export type ContextCallback<
  ValueType extends ContextTypeMap[keyof ContextTypeMap]
> = (value: ValueType, dispose?: () => void) => void;

/**
 * An event fired by a context requester to signal it desires a named context.
 *
 * A provider should inspect the `name` property of the event to determine if it has a value that can
 * satisfy the request, calling the `callback` with the requested value if so.
 *
 * A provider can call the callback multiple times if the value is changed, if this is the case the
 * provider should pass a `dispose` method to the callback which requesters can invoke to indicate they
 * no longer wish to receive these updates.
 *
 * If a requester only wishes to ever receive the context once, then they can optionally set the
 * `once` property on the event, providers should respect this property and only execute the
 * callback once.
 */
export class RequestContextEvent<T extends keyof ContextTypeMap> extends Event {
  public readonly name: T;
  public readonly once: boolean;
  public readonly callback: ContextCallback<ContextTypeMap[T]>;

  public constructor(
    name: T,
    callback: ContextCallback<ContextTypeMap[T]>,
    once: boolean = false
  ) {
    super("request-context", { bubbles: true, composed: true });
    this.name = name;
    this.callback = callback;
    this.once = once;
  }
}
```
