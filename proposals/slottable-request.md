# Slottable Request Protocol
aka "Render Props for Web Components"

**Author**: Kevin Schaaf (@kevinpschaaf, Google)

**Status**: Draft

**Last Updated**: 2023-08-28


# Background

There are often use cases that require a component to render UI on-demand and/or based on data, but where the specific rendering of the data should be controllable by the user. Use cases of this include:

*   A data table or tree component, which manages the rendering of cells or tree nodes, but where the given look and feel of the items is customizable by the user.
*   A virtual list component, which optimizes rendering a list of data by only rendering a subset of user-templated items based on which array instances are visible on the screen.
*   A tooltip component whose user-provided slotted content will be rendered rarely and on-demand based on user interactions handled by the tooltip.
*   Components that fetch data themselves, but provide the ability for the user to customize the rendering of the data.

This proposal describes an interoperable protocol for components to request that their owning context render slotted content into themselves, using data provided via the protocol.

# Goals

The high-level goal is to introduce a protocol to allow a component to request that the owner provide content to be rendered on-demand, and optionally based on data that is provided by the component.

Specific goals within this include:

*   The ability for users of a component to render requested content using a templating system of their choice.
*   The ability for users of a component to render requested content using data supplied by the component.
*   The ability for requested content to be rendered into the requesting component's shadow root at a position of its choice.
*   The ability for requested content to be styled in the same "user" scope the component in question was created in, and using styling mechanisms of the user's choice.
*   The ability to use a component that requests content via the protocol in plain JS without requiring a framework.
*   The ability to implement the protocol inside framework-specific wrappers or helpers that expose a user API that looks similar or identical to framework-specific patterns (i.e. "render props" for React).
*   The ability to provide slotted content lazily for better performance, since the component may defer rendering slotted content based on the state of the component.

# Design / Proposal


## Protocol Definition

Concretely, the protocol involves dispatching a well-known DOM event to request slotted content (aka "[slottables](https://dom.spec.whatwg.org/#light-tree-slotables)", per the DOM spec) be rendered with the provided `data` argument and assigned to the given `slotName`:

```ts
export class SlottableRequestEvent<Name extends string, T = unknown> extends Event {
  readonly data: T | typeof remove;
  readonly name: Name;
  readonly slotName: string;
  constructor(name: Name, data: unknown, key?: string) {
    super('slottable-request', {bubbles: false, composed: false});
    this.name = name;
    this.data = data;
    this.slotName = key !== undefined ? `${name}.${key}` : name;
  }
}

export const remove = Symbol('slottable-request-remove');
```

When a component wants to render customizable content, it (1) fires a `slottable-request` event for a given slot name, and (2) renders a `<slot>` element into its shadow root corresponding to the requested slot name.

Each time the `slottable-request` event is received, the user should use the provided `data` to render (or update) slotted content into the component that fired the event, where each top-level child rendered for a given request is assigned to the provided `slotName`. The `name` field on the event is used to determine what type of content to render; each unique `name` will generally map to a specific user-managed template. 

Because components may request multiple slotted instances for the same conceptual `name` (for example, in the case of a `list-item` slot request), the `slotName` may differ from the `name`. Thus, `name` should be used to select which template to render, and `slotName` should be assigned to the given rendered _instance_ and used as a unique key to identify which instance to update for subsequent requests.

When requesting a specific slot instance, the event accepts an optional 3rd `key` argument; this is used to generate a unique `slotName` by suffixing the `name` with `key`.

The `remove` symbol is provided as a sentinel value for `data` to indicate that the given content assigned to `slotName` should be removed, and should be fired when the associated `<slot>` is no longer rendered to avoid leaking light-DOM children.


## Examples

*   ["Travel Calendar"](https://lit.dev/playground/#gist=205ee0ccc0ea4d0420608808942d2655) customization example (raw with no helpers)
*   [Lit proof-of-concept](https://lit.dev/playground/#gist=2974fec927ef67b30d82a6ff7d05740a). Includes demos of:
    *   Raw implementation of handling the `slottable-request` event using Lit
    *   Implementing the protocol using a Lit directive (see [this description](https://gist.github.com/kevinpschaaf/0fe117368411f340aa3019dceeaa465e) of possible API for Lit-specific helpers)
    *   Implementing the protocol in a React web component wrapper


## Comparison with Render Props

Simply modeling the need expressed above as a "render prop", aka a property on the component that accepts a function that receives data and produces/updates DOM based on that data, introduces a number of complications making it ill-suited for interoperable web components:

1. **Different rendering libraries may be used between the caller and the author.** When both the usage site and the component itself are implemented using the same rendering library, render props are as simple as providing a framework-specific template reference (often a function) to the component that the component can natively render. However, Web Components introduce the capability to implement the "inside" of a component with a different rendering library than the outside library rendering the component (if any). Requiring a render prop to be provided using the same rendering library the component was authored with leaks implementation details and has negative DX implications, forcing the user to context-switch into a different templating language they may not otherwise be familiar with.
2. **Rendering a user-provided template to Light DOM is risky.** The light DOM children of a component are logically owned by the calling scope (the "user" of the component, not the component itself). Rendering children outside of the shadow root may conflict with user-scope rendering libraries by violating assumptions it makes for reconciling DOM (e.g., incremental DOM diffs against live DOM and may remove children it did not previously render).
3. **Rendering a user-provided template to Shadow DOM is problematic for styling.** If the component used a render prop to render DOM into its shadow root, that DOM would be subject to the component's shadow styling, rather than the styling of the user's scope. Providing a mechanism to render user styles into the shadow root along with the protocol would likely feel foreign to the caller.

Defining a "framework-agnostic render prop" protocol was explored in  but rejected due to these complications. Instead, this proposal defines an event-based protocol to request the outside scope render and provide slotted children, side-stepping virtually all of the issues raised there.

## API alternatives:

*   The "remove" case is handled as a sentinel using the same event. We could have a separate `remove-slottable-request` event, but that seemed a bit annoying to listen for two events.
*   Explicitly treating `key` as a first-class thing is a choice; it could be up to the user to generate and request unique slot names, but then it would become another protocol/parsing exercise for the receiver to know that e.g. `item.4` and `item.42` should use the `item` template but `header.3` should use the header template. That seems annoying, so I landed on the requestor providing the `name` and `key` and the user getting a pre-baked `slotName` to use for it, but they still need the un-concatenated `name` to select the template.
*   Rather than sending individual `slottable-request` events for each name/key instance, we could define a more complex payload for the event to describe all of the instance data & slots required. That does not lend itself as nicely to helpers to allow rendering different named slots into different parts of the template, since all of those requests would need to be batched by one orchestrator. It's certainly possible to build such a thing, with the batching keyed off of e.g. `part.options.host`, but didn't want to go directly there first. Also, each slottable-request conceptually mapping to a render-prop call seems good for keeping the concepts aligned. We clearly need to perf-test this, however, and that might push us to a single-event model.
