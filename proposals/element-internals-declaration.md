# Proposal name

**Author**: Dave Rupert

**Status**: Proposal

**Created**: April 22, 2025

**Last updated**: April 22, 2025

# Summary

There's no standard API to expose `ElementInternals` to `@public`. Third-party tooling like Automated Accessibility Testing tools are unable to reliably "guess" and find those exposed internals (e.g. cannot find `aria-role`). Problems that arise from this are best summed up on the DequeuLabs aXe-core Github Repo:

https://github.com/dequelabs/axe-core/issues/4259

Both aXe (on thread) and Microsoft Accessibility Insights (privately) have indicated that a community standard here would go a long way to supporting the probing of `ElementInternals` for valid roles.

Another potential win here would be an SSR story where tooling to make static HTML could elevate internals like `role` or a default `:state()` to the template.

For those reasons, it would be good for the Web Components Community Group to have an agreed upon standard for exposing `ElementInternals` so other tooling can build from our convention.

# Example

Here is a softball proposal for how we could consistently expose internals publicly.

```ts
class FooElement extends HTMLElement {
  // @public
  const internals: ElementInternals = this.attachInternals()
}
```

# Goals

- Determine an agreed upon declaration naming convention for exposing `ElementInternals` to third-party tooling.
- Identify all the places (ARIA roles, custom states, forms, SSR, etc) where consistent interface would be ideal.

# Non-goals

- Fight about names forever.

# Design detail

Infinite options exist on what we could name the interface each with its plusses and minuses.

```js
const elementInternals = this.attachInternals() // verbose but specific
const internals = this.attachInternals() // shorter but more vague
const guts = this.attachInternals() // this is nonsense.
```

Another common convention is to attach internals as a private/internal variable.

```js
const #internals = this.attachInternals() // very private
const _internals = this.attachInternals() // this underscores the need for consistency
```

This makes logical sense to make `attachInternals` internal, but based on the established goal of this being a public API that is consistently exposed to third-parties it seems a public declaration would be more suitable.


# Open questions

1. Would closed shadowroots need/desire something a private field like `#internals`? 

# Previous considerations

Prior art:

- Benny Powers' suggested patch to aXe https://github.com/dequelabs/axe-core/issues/4259#issuecomment-2238912894
- Westbrook Johnson's [axe-core-element-internals](https://github.com/Westbrook/axe-core-element-internals)


