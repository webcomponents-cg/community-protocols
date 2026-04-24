## Exposing ElementInternals to Accessibility Audit Tools

**Authors**: Dave Rupert, Benny Powers

**Status**: Draft

**Created**: April 22, 2025

**Last updated**: April 24, 2026

## Summary

There's currently no way for  automated accessibility testing tools to gain necessary access to element internals. This leads to false positives (e.g. cannot find `aria-role`), and can confuse downstream users of web components. Problems that arise from this are best summed up on the Deque Labs aXe-core GitHub repo:

https://github.com/dequelabs/axe-core/issues/4259

Both aXe (on thread) and Microsoft Accessibility Insights (privately) have indicated that a community standard here would go a long way to supporting the probing of `ElementInternals` for valid roles. Deque (the makers of aXe) have confirmed in video calls with representatives from Adobe and Red Hat that they will implement support for the global WeakMap approach described in this proposal.

Without a protocol like this, teams using `ElementInternals` face false positives in accessibility audits that can break CI pipelines, lead to disabled lint rules, or discourage adoption of the platform's built-in accessibility features altogether.

For those reasons, it would be good for the Web Components Community Group to have an agreed upon standard for exposing `ElementInternals` so other tooling can build from our convention.

## Goals

- Determine an agreed upon side-channel convention (e.g. a global WeakMap) for exposing `ElementInternals` to third-party diagnostic and accessibility audit tooling.
- Enable high-fidelity accessibility auditing (e.g., ARIA roles, custom states) for components using `ElementInternals` without requiring developers to intentionally bloat their public APIs.

## Non Goals

- **Providing a public API for application logic:** While this proposal does technically expose internals globally, creating potential for abuse, this is an accepted risk strictly in exchange for critical accessibility audit gains. Any use of this registry for application state management, styling, or cross-component communication is explicitly unsupported.
- **Permanence:** This protocol is a temporary stopgap. When browser vendors provide native APIs to expose the accessibility tree to external tools (see [Related Work](#related-work)), this registry can and will be removed. We do not guarantee backwards compatibility for unauthorized or abusive use cases when that time comes.

## Protocol

In order to retain the _internal_ aspect of `ElementInternals`, it's ideal to not expose it as a public property.

```ts
#internals = this.attachInternals();
```

A global WeakMap is the most appropriate side-channel for this because:

- It does not alter the element's class shape or public API surface
- It avoids property naming conflicts (no convention for `.internals` vs `._internals` vs `.elementInternals`)
- WeakMap keys are weak references, so detached elements are still eligible for garbage collection
- It can be implemented in a single shared location (e.g. a base class, mixin, or controller) without requiring changes to every component class

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

## Limitations

This protocol operates within the page's main JavaScript execution context. Tools running in browser extension [isolated worlds](https://developer.chrome.com/docs/extensions/develop/concepts/content-scripts#isolated_world) cannot access `globalThis._elementInternals` or any other JavaScript-side state set by page scripts. This is a known limitation acknowledged by Deque, and is one of the reasons native browser APIs for exposing the accessibility tree remain the long-term goal. This protocol targets in-page tool injection scenarios (e.g. axe-core loaded via `<script>`, Playwright's `page.evaluate`, CI-based auditing).

## Security Considerations

Because this protocol exposes `ElementInternals` (which contains form state) to `globalThis`, it theoretically allows third-party scripts to access encapsulated data. However, in a browser environment, any script with sufficient privileges to read `globalThis._elementInternals` already possesses the ability to monkey-patch `HTMLElement.prototype.attachInternals` to achieve the exact same access. Therefore, this proposal does not introduce a fundamentally new attack vector.

## Related Work

### Standards Efforts

- [`.elementInternals` accessor property (whatwg/html#11040)](https://github.com/whatwg/html/issues/11040) - WHATWG proposal for a native DOM API to access `ElementInternals`
- [Expose implicit ARIA semantics (w3c/aria#2663)](https://github.com/w3c/aria/issues/2663) - W3C ARIA proposal to expose browser-computed and `ElementInternals`-set ARIA semantics
- [aXe-core ElementInternals tracking issue (dequelabs/axe-core#4259)](https://github.com/dequelabs/axe-core/issues/4259) - Discussion that motivated this protocol

### Prior Art

- [Benny Powers' `attachInternals` patch suggestion](https://github.com/dequelabs/axe-core/issues/4259#issuecomment-2238912894) - Early prototype of monkey-patching `attachInternals` to populate a WeakMap
- [Westbrook Johnson's axe-core-element-internals](https://github.com/Westbrook/axe-core-element-internals) - Library demonstrating how aXe-core could leverage `ElementInternals` for accessibility tree validation

