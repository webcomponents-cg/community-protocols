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

# Goals

- Determine an agreed upon declaration naming convention for exposing `ElementInternals` to third-party tooling.
- Identify all the places (ARIA roles, custom states, forms, SSR, etc) where consistent interface would be ideal.

In order to retain the _internal_ aspect of `ElementInternals`, it's ideal to not expose it as a public property.

```ts
#internals = this.attachInternals();
```

But based on the established goal of consistently exposing internals specifically to accessibility tooling, a global weak map is most appropriate

```ts
declare global {
  var _elementInternals: WeakMap<Element, ElementInternals>;
}

globalThis._elementInternals ??= new WeakMap<Element, ElementInternals>();

class HasInternals extends HTMLElement {
  #internals = this.attachInternals();
  constructor() {
    super();
    globalThis._elementInternals.set(this, this.#internals);
  }
}

customElements.define('has-internals', HasInternals);
```

Here's an example of a Lit Reactive Controller which could handle this for all elements in a project:

```ts
import type { ReactiveController } from 'lit';
export class InternalsController implements ReactiveController {
  constructor(host: Element) {
    this.#internals = host.attachInternals();
    globalThis._elementInternals.set(host, this.#internals);
  } 
}
```
```ts
import { InternalsController } from '../internals-controller.ts';
import { LitElement } from 'lit';
import { customElement } from 'lit/decorators.js';

@customElement('has-internals')
class HasInternals extends LitElement {
  // internals not exposed on element public api
  #internals = new InternalsController(this);
}
```

# Open questions

1. Would closed shadowroots need/desire something a private field like `#internals`? 

# Previous considerations

Prior art:

- Benny Powers' suggested patch to aXe https://github.com/dequelabs/axe-core/issues/4259#issuecomment-2238912894
- Westbrook Johnson's [axe-core-element-internals](https://github.com/Westbrook/axe-core-element-internals)


