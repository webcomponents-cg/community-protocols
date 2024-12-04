# On-Demand Definitions Protocol

A protocol for defining custom elements on-demand without relying on top-level
side-effects.

**Author**: [dgp1130](https://github.com/dgp1130/)

**Status**: Draft

**Created**: 2024-11-26

**Last updated**: 2024-11-26

## Background

Custom elements are defined via the
[`customElements.define`](https://developer.mozilla.org/en-US/docs/Web/API/CustomElementRegistry/define)
API. Defining an element is inherently a side-effectful operation, as it
upgrades all elements with the associated tag name which already exist in the
document.

Traditionally, the `customElements.define` call is placed adjacent to the custom
element class declaration like so:

```javascript
export class MyElement extends HTMLElement {
  // ...
}

customElements.define('my-element', MyElement);
```

For the purposes of this protocol, this pattern is considered a "top-level"
custom element definition because the element is defined in the top-level scope.
Any consumer which imports this file will always call `customElements.define`
just by nature of importing it. Code inside functions, classes, or other
constructs which is _not_ executed during the `import` is not considered
"top-level".

```javascript
console.log('This is the top-level');

export function foo() {
  console.log('This is also top-level because it is called from the top-level below.');
}
foo(); // Calling `foo` from the top-level.

export function bar() {
  console.log('This is *not* top-level because a consumer needs to explicitly call it after import.');
}
```

Defining custom elements in the top-level scope is useful to ensure they are
always defined, but this creates a top-level side-effect which has a few
negative consequences:

### Problem 1: Top-level side-effects are not tree shakable

Consider a file which exports two elements:

```javascript
export class MyFirstElement extends HTMLElement {
  // ...
}

customElements.define('my-first-element', MyFirstElement);

export class MySecondElement extends HTMLElement {
  // ...
}

customElements.define('my-second-element', MySecondElement);
```

A consumer of this file can trivially choose to import only `MyFirstElement` xor
`MySecondElement`. It could also run
`document.createElement('my-first-element')` xor
`document.createElement('my-second-element')`. However, any bundler needs to
retain _both_ `customElements.define` calls because they exist in the top-level
scope. This is necessary for correctness (as `MySecondElement` could depend on
`MyFirstElement`) and because a bundler cannot confidently prove that either
element will not used at runtime.

Any module which imports _any_ symbol from this file is forced to retain _all_
of them in the final JS bundle. This is just one example, but the issue extends
to all usages of all custom elements. In order to safely use a custom element, a
module must import that element, but since doing so typically calls
`customElements.define` in the top-level scope, that element can _never_ be tree
shaken.

### Problem 2: Easy to forget to import a custom element

It is trivially easy to use a custom element while forgetting to import its
definition, meaning there is no guarantee the element will be defined at
runtime.

```javascript
document.querySelector('my-element').doSomething();
// ^ Could error: No guarantee `my-element` has been defined.
```

Contrast this with:

```javascript
import './my-element.js';

document.querySelector('my-element').doSomething();
// ^ Know `my-element` definition has been loaded.
```

An import is necessary to express that this file depends on the definition of
`my-element` as a top-level side-effect from `my-element.js`.

Technically, even importing `my-element.js` is not sufficient to know for
certain that `<my-element>` has been defined. Even if `MyElement` is in that
file, there is no real guarantee `customElements.define` was called in the
top-level scope.

### Problem 3: Side-effectful imports are sub-optimal

Relying on side-effectful imports brings negative consequences. Tooling can not
statically detect whether such an import is required or not. If a future
developer removes `.doSomething()` in the above example, no automated tooling
will instruct them to also remove `import './my-element.js';`. This leads to
extra, unneeded dependencies and reduces confidence for developers who may be
hesitant to remove the import because it is hard to know whether anything in the
module actually does depend on any side effects from `my-element.js`.

For human developers, it is in no way obvious that calling `.doSomething()`
requires an import of `my-element.js` or that the import exists to provide
`.doSomething()`. Extensive comments are necessary to communicate the
relationship between these two statements for developers unfamiliar with these
constraints.

### Problem 4: File ordering

When forgetting an import on a custom element, using that element is actually
not guaranteed to fail. It only has the _potential_ to fail, which is in many
ways worse. When two JavaScript files do not have any `import`-relationship
between them, which one is actually executed first is ultimately
_unpredictable_.

The ordering between such files is largely dependent on which other files in the
program import them. Different bundlers may have different behavior and sort
these files completely differently. Normally that's fine: if the files don't
import each other and have no dependency relationship between them, it doesn't
actually matter which one comes first.

However, side-effects from the first file are observable in the second. Since
defining a custom element is inherently a side-effectful operation, there is a
potential file ordering hazard. When one file defines a custom element, and the
other uses that custom element, the system will work only when that ordering
aligns in this unpredictable way.

```javascript
// my-element.js

class MyElement extends HTMLElement {
  doSomething() {
    console.log('Doing something!');
  }
}

customElements.define('my-element', MyElement);
```

```javascript
// my-user.js

document.querySelector('my-element').doSomething();
// ^ MIGHT fail.
```

The lack of an import between `my-user.js` and `my-element.js` means that this
will work with iff `my-element.js` happens to be sorted _before_ `my-user.js`,
which depends entirely on the bundler implementation in use as well as other
files and import edges in the program. Unrelated refactorings can change the
ordering of these two files and lead to unexpected errors.

## Goals

*   Provide a common primitive for custom element libraries and frameworks to
    confidently rely on a custom element they don't own being correctly defined.
*   Support tree-shakable custom elements.
*   More closely couple a custom element's usage with a dependency on its
    implementation.
*   Improve the ability for standard JavaScript tools (bundlers, type checkers,
    linters, etc.) to reason about custom element usage.
*   Reduce reliance on side-effects from files not explicitly depended upon.
*   Continue to support custom element definition as a top-level side-effect.

## Non-goals

*   _Do not_ address the problem of identifying and defining
    ["entry-point components"](#defining-entry-point-elements).

## Overview

This protocol specifies a `static` `define` property on custom elements which
calls `customElements.define` if the element has not already been defined. This
allows anything with a reference to a component's class to define that component
"on demand" before using it.

### Example

An element can implement this protocol by specifying a static `define` function:

```javascript
export class MyElement extends HTMLElement {
  static define() {
    if (customElements.get('my-element')) return;

    customElements.define('my-element', MyElement);
  }
}
```

A consumer can use this protocol by checking if the `define` property exists and
calling it before using the element.

```javascript
import {MyElement} from './my-element.js';

MyElement?.define();
document.querySelector('my-element').doSomething();
// ^ Always works!
```

## Design detail

This structure removes usage of `customElements.define` in the top-level scope
and allows web component libraries / frameworks to design APIs which depend on
this primitive.

### Framework utilization

As two small examples, [HydroActive](https://github.com/dgp1130/HydroActive/)
already implements this draft. All `HydroActive` components come with a built in
`define` implementation and using a component
[requires a reference to its component class](#hydroactive) before making that
component accessible to developers. HydroActive uses this class to automatically
call `define` and ensure they are properly defined being being interacted with
by the developer.

DISCLAIMER: The author of this proposal is also the maintainer of HydroActive.

```javascript
import {component} from 'hydroactive';
import {SomeComp} from './some-comp.js';

export const MyElement = component('my-element', (host) => {
  // `.access(SomeComp)` implicitly calls `SomeComp.define()`.
  host.query('some-comp').access(SomeComp).element.doSomething();
});
```

This approach provides a guarantee that `SomeComp` is defined before
`doSomething` is called. It also allows `SomeComp` to be tree-shaken from the
bundle if it is not used by `MyElement` or if `MyElement` is itself unused and
eligible for tree shaking.

One more potential example would be `lit-html`, which currently relies on
[`lit-analyzer`](#lit-analyzer) to ensure all dependencies of a custom element
have been defined. Lit could hypothetically be updated to use explicit
references to custom element classes:

```javascript
import {MyElement} from './my-element.js';

function renderMyElement() {
  return html`
    <${MyElement}>Hello, World!</${MyElement}>
  `;
}
```

The `html` tagged template literal could implicitly call `MyElement.define()` to
ensure it is defined prior to rendering. This ensures `MyElement` is indeed
defined and allows it to be tree shaken if `renderMyElement` is itself tree
shaken.

### Benefits

This protocol fixes or improves all of the above specified problems:

1.  Removing `customElements.define` from the top-level scope allows bundlers to
    effectively tree shake any unused components. In order to define an element
    with `MyElement.define()`, users must have a reference to `MyElement` which
    is known to the bundler.
2.  This API provides a common primitive for web component libraries and
    frameworks to [build on top of](#framework-utilization). Designing APIs with
    this protocol in mind allows compilers and linters to effectively require
    that an import is used where necessary.
3.  Similar to 2., web component APIs can
    [be designed to leverage this protocol](#framework-utilization) and more
    closely associate a web component class with its usage.
4.  Leveraging an import on a custom element class enforces strict ordering and
    makes it harder for users of that class to depend on an element prior to its
    class' definition.

### Compatibility with top-level definitions

Any usages of a custom element which do require it to be defined at top-level
execution can still be supported by calling `MyElement.define()` at that
location. It is perfectly valid to write:

```javascript
export class MyElement extends HTMLElement {
  static define() {
    if (customElements.get('my-element')) return;

    customElements.define('my-element', MyElement);
  }

  doSomething() {
    console.log('Doing something!');
  }
}

MyElement.define();

document.querySelector('my-element').doSomething();
```

Even though `MyElement.define` is called in the top-level scope, this is still a
correct implementation and usage of the static `define`. Any other consumers of
`MyElement` can call `MyElement.define` to enforce that the element is indeed
defined before they use it, so this is still a useful pattern. The fact that the
element happens to always be defined before any consumers attempt to use it is
an implementation detail those consumers are intentionally abstracting away from
themselves by calling `MyElement.define`.

## Open questions

Some more nuanced questions with this API.

### No-op when already defined

`define` silently no-ops when its element is already defined, meaning multiple
consumers can safely call `define` without fear of negative effects.

ALTERNATIVE PROPOSAL: Let `define` throw if the element is already defined and
have callers check `customElements.get` before invoking it.

Requiring callers to check `customElements.get` means that every usage of
`define` either needs separate knowledge of the element's tag name or needs to
manually call `customElements.getName`, which is extra effort which accomplishes
very little. No user of a custom element can safely assume it is the _only_ user
of that element and unconditionally call `define`.

It is perhaps a little confusing that a function called `define` might not
actually _define_ the component if it happens to already be defined. `define`
could potentially be renamed to be more clear about this possibility, but the
end result of calling the function is that the element is defined. So calling
`define` at a time when the element is already defined does not necessarily
violate any potential assumptions about its behavior which come from the name.

### Built-in tag name

`define` only supports defining the element with one hard-coded tag name. This
behavior is equivalent to calling `customElements.define` in the top-level
scope. Consumers of the class are not able to influence the tag name used.

ALTERNATIVE PROPOSAL: Let the user provide an optional tag name to give
consumers more flexibility.

```javascript
export class MyElement extends HTMLElement {
  static define(tagName) {
    customElements.define(tagName ?? 'my-element', MyElement);
  }
}
```

Supporting user-defined tag names means a custom element no longer has static
knowledge of its tag name. While some custom element authors may be comfortable
that constraint, it is likely not something this protocol would want to force on
its implementers.

`customElements.define` throws when given the same class for multiple tag names.
This is an issue when multiple consumers call `MyElement.define` with different
tag names, expecting to safely define their chosen tag name, only to encounter
an error.

The main goal of this protocol is to allow multiple, unrelated consumers to
safely use a custom element without interfering with each other or relying on
unexpected side-effects. Allowing each consumer to pick its own tag name and
break any other consumer using a different tag name actively fights against the
goals of this protocol.

### Defining entry-point elements

A problem deliberately _not_ solved by this protocol is identification of
"entry-point elements". In this context, an "entry-point element" is one which
must be defined in the top-level scope of some script in order to create a
desired behavior on the page.

If all custom elements move their definitions into a static `define` and no
script calls any of them from the top-level scope, then _no_ element is ever
defined or upgraded. For example, consider an application with the following
HTML:

```html
<my-app></my-app>

<script src="./my-app.js" type="module"></script>
```

Consider also that `MyApp` is defined in line with this protocol as:

```javascript
// my-app.js

import {SomeComp} from './some-comp.js';

export class MyApp extends HTMLElement {
  static define() {
    if (customElements.get('my-app')) return;

    customElements.define('my-app', MyApp);
  }

  // ...
}
```

`my-app.js` _must_ call `customElements.define('my-app', MyApp)` /
`MyApp.define()` in its top-level scope in order for the application to start.

However in this scenario, `MyApp.define` is never called and the entire element,
as well as its dependencies, are never defined or have any effect on the
document.

This illustrates the need for _some_ custom element to be intentionally defined
in the top-level scope in order to upgrade its element during script execution
as well as define any custom element dependencies it may require. In this case,
something needs to identify `MyApp` as an "entry-point element" which must be
defined in the top-level scope and call `MyApp.define()`. This can be hard-coded
by the developer in the top-level scope of `my-app.js` or handled by some other
tool.

Identifying entry-point elements, retaining them in the bundle, and triggering
their definition is a complex problem in its own right and out of scope for this
particular proposal.

## Previous considerations

There is some prior art to be aware of in this space.

### `lit-analyzer`

Lit components expect any of their dependencies to already be defined on the
page prior to being used. This results in a pattern where a Lit element needs to
have a side-effectful import on its dependencies to ensure any custom elements
it needs are already defined.

```javascript
import './my-element.js';

function renderMyElement() {
  return html`
    <my-element>Hello, World!</my-element>
  `;
}
```

A problem with this developer experience is that it is very easy to forget to
import `my-element.js` or accidentally rely on the side-effect of another file
importing it.

[`lit-analyzer`](https://www.npmjs.com/package/lit-analyzer) solves this with
its `no-unknown-tag-name` and `no-missing-import` checks which analyze source
code to ensure that any usage of an element in the `html` literal is covered by
a direct import. This validates that every Lit component has a direct dependency
on any elements they require.

This requires the developer to integrate as distinct analyzer into their
toolchain for Lit, which otherwise doesn't require any such tooling.

### HydroActive

A key goal of [HydroActive](https://github.com/dgp1130/HydroActive/) is to
ensure that custom elements are not accessible to the developer until they have
been defined and hydrated. To that end, HydroActive intentionally hides custom
elements from developers and throws when attempting to access them directly.

```javascript
import {component} from 'hydroactive';

export const MyElement = component('my-element', (host) => {
  // Can access native elements like `<div>` directly.
  host.query('div').access().element.textContent = 'Hello, World!';

  // ERROR: Custom element `SomeComp` requires an element class.
  host.query('some-comp').access();
});
```

This throws even when `some-comp` is already defined because it could be defined
coincidentally due the [file ordering problem](#problem-4-file-ordering).

Instead, HydroActive requires the custom element class to be directly provided
to
[`.access()`](https://github.com/dgp1130/HydroActive/blob/605adcc0bfac70a820c7fdf8fce2a4bfc5c55765/src/dehydrated.ts#L85).

```javascript
// Works.
host.query('some-comp').access(SomeComp).element.textContent = 'Hello, World!';
```

This ensures that the class declaration of `SomeComp` comes before `MyElement`.
HydroActive intentionally makes this the only easy way of getting access to an
custom element to ensure that this dependency exists.

However even this approach is forced to assume that a custom element class
declaration is co-located with a top-level `customElement.define` call. This
assumption also prevents tree-shaking of any dependencies. HydroActive
implemented the on-demand definitions proposal to mitigate these issues.

HydroActive's design with respect to file ordering is described more thoroughly
in [this video](https://youtu.be/euFQRqrTSMk?si=i5HKHayt3QvuNytf&t=736), though
it is primarily focused on this problem within the context of hydration.
However, correctly hydrating an element also requires defining it so this is
effectively the same core problem.
