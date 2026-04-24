## Proposal name

**Authors**: Dave Rupert, Benny Powers

**Status**: Proposal

**Created**: April 22, 2025

**Last updated**: April 24, 2026

## Summary

There's currently no way for  automated accessibility testing tools to gain necessary access to element internals. This leads to false positives (e.g. cannot find `aria-role`), and can confuse downstream users of web components. Problems that arise from this are best summed up on the DequeuLabs aXe-core Github Repo:

https://github.com/dequelabs/axe-core/issues/4259

Both aXe (on thread) and Microsoft Accessibility Insights (privately) have indicated that a community standard here would go a long way to supporting the probing of `ElementInternals` for valid roles.

Another potential win here would be an SSR story where tooling to make static HTML could elevate internals like `role` or a default `:state()` to the template.

For those reasons, it would be good for the Web Components Community Group to have an agreed upon standard for exposing `ElementInternals` so other tooling can build from our convention.

## Goals

- Determine an agreed upon side-channel convention (e.g. a global WeakMap) for exposing `ElementInternals` to third-party diagnostic and accessibility audit tooling.
- Enable high-fidelity accessibility auditing (e.g., ARIA roles, custom states) for components using `ElementInternals` without requiring developers to intentionally bloat their public APIs.

## Non Goals

- **Providing a public API for application logic:** While this proposal does technically expose internals globally, creating potential for abuse, this is an accepted risk strictly in exchange for critical accessibility audit gains. Any use of this registry for application state management, styling, or cross-component communication is explicitly unsupported.
- **Permanence:** This protocol is a temporary stopgap. When browser vendors provide native APIs to expose the accessibility tree to external tools, this registry can and will be removed. We do not guarantee backwards compatibility for unauthorized or abusive use cases when that time comes.

## Protocol

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

Alternatively, page owners may choose to add a blocking script before any elements are defined:

```ts
globalThis._elementInternals ??= new WeakMap<Element, ElementInternals>();
const originalAttachInternals = HTMLElement.attachInternals;
HTMLElement.attachInternals = function(...args) {
  const internals = originalAttachInternals.apply(this, args);
  globalThis._elementInternals.set(this, internals);
  return internals;
};
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

## Security Considerations

Because this protocol exposes `ElementInternals` (which contains form state) to `globalThis`, it theoretically allows third-party scripts to access encapsulated data. However, in a browser environment, any script with sufficient privileges to read `globalThis._elementInternals` already possesses the ability to monkey-patch `HTMLElement.prototype.attachInternals` to achieve the exact same access. Therefore, this proposal does not introduce a fundamentally new attack vector.

# Previous considerations

Prior art:

- Benny Powers' suggested patch to aXe https://github.com/dequelabs/axe-core/issues/4259#issuecomment-2238912894
- Westbrook Johnson's [axe-core-element-internals](https://github.com/Westbrook/axe-core-element-internals)


