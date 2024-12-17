# On-Demand Definitions Protocol

A protocol for defining custom elements on-demand without relying on top-level
side-effects.

**Author**: [dgp1130](https://github.com/dgp1130/)

**Status**: Draft

**Created**: 2024-11-26

**Last updated**: 2024-12-16

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

### Problem 1: Easy to forget to import a custom element

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
`my-element` as a top-level side-effect from `my-element.js`. This import could
be hundreds or even thousands of lines away from its usage with no clear
association between the two.

### Problem 2: Top-level side-effects are not tree shakable

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
shaken, even if it is ultimately never used. Consider the following dev-mode
only usage of a particular element:

```javascript
import './my-element.js';

// Set to a compile-time constant known by the bundler.
if (import.meta.env.DEV) {
  document.querySelector('my-element').doSomething();
}
```

Even if a bundler correctly configures the production build such that
`import.meta.env.DEV === false`, it can only remove the `doSomething` call from
the bundle. `import './my-element,js';` still needs to remain due to the global
side-effect it creates and which the bundler cannot prove is unused.

### Problem 3: Side-effectful imports are sub-optimal

Relying on side-effectful imports brings negative consequences. Tooling can not
statically detect whether such an import is required or not. If a future
developer removes `.doSomething()` in the above example, no automated tooling
will instruct them to also remove `import './my-element.js';`. This leads to
extra, unneeded dependencies and reduces confidence for developers who may be
hesitant to remove the import because it is hard to know whether anything in the
module actually does depend on any side-effects from `my-element.js`.

For human developers, it is in no way obvious that calling `doSomething`
requires an import of `./my-element.js` or that the import exists to provide
`doSomething`. Extensive comments are necessary to communicate the relationship
between these two statements for developers unfamiliar with these constraints.

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

However, side-effects from the first file executed are observable in the second.
Since defining a custom element is inherently a side-effectful operation, there
is a potential file ordering hazard. When one file defines a custom element, and
the other uses that custom element, the system will work only when that ordering
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

// Note the absence of an `import` here.

document.querySelector('my-element').doSomething();
// ^ MIGHT fail.
```

The lack of an import between `my-user.js` and `my-element.js` means that this
will work if-and-only-if `my-element.js` happens to be sorted _before_
`my-user.js`, which depends entirely on the bundler implementation in use as
well as other files and import edges in the program. Unrelated refactorings can
change the ordering of these two files and lead to unexpected errors.

## Goals

*   Provide a common primitive for custom element libraries and frameworks to
    confidently rely on a custom element they don't own being correctly defined.
*   Support tree-shakable custom elements.
*   More closely couple a custom element's usage with a dependency on its
    implementation.
*   Improve the ability for standard JavaScript tools (bundlers, type checkers,
    linters, etc.) to reason about custom element usage.
*   Reduce reliance on side-effects from files not explicitly depended upon.
*   Continue to support custom element definition as a top-level side-effect
    when necessary.

## Non-goals

*   _Do not_ address the problem of identifying and defining
    ["entry-point components"](#defining-entry-point-elements).
*   _Do not_ remove all top-level side-effects from custom element definitions.
    Some may still be necessary.

## Overview

This protocol specifies a static `define` property on custom elements which
calls `customElements.define` if the element has not already been defined. This
allows anything with a reference to a component's class to define that component
"on demand" before using it.

### Example

An element can implement this protocol by specifying a static `define` function:

```javascript
export class MyElement extends HTMLElement {
  static define() {
    // Check if the tag name was already defined by another class.
    const existing = customElements.get('my-element');
    if (existing) {
      if (existing === MyElement) {
        return; // Already defined as the correct class, no-op.
      } else {
        throw new Error(`Tag name \`my-element\` already defined as \`${
            existing.name}\`.`);
      }
    }

    customElements.define('my-element', MyElement);
  }
}
```

A consumer can use this protocol by checking if the `define` property exists and
calling it before using the element.

```javascript
import {MyElement} from './my-element.js';

MyElement.define();
document.querySelector('my-element').doSomething();
// ^ Always works!
```

## Design detail

This structure removes usage of `customElements.define` in the top-level scope
and allows web component libraries / frameworks to design APIs which depend on
this primitive.

### Framework utilization

As two small examples, [HydroActive](https://github.com/dgp1130/HydroActive/)
already implements this draft. All `HydroActive` components come with a built-in
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
eligible for tree-shaking.

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
defined and allows it to be tree-shaken if `renderMyElement` is itself
tree-shaken.

### Benefits

This protocol fixes or improves all of the above specified problems:

1.  Removing `customElements.define` from the top-level scope allows bundlers to
    effectively tree-shake any unused components. In order to define an element
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
  static define() { /* ... */ }

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

### Naming

The name `define` was chosen for being short, direct, and explicit about what it
does: defining an element in the registry.

This name might be too short such that developers may want to use the name for
their own purposes. For example, a custom element representing words in a
dictionary may want a `define` function which shows the definition of its word.

Another challenge is that `define` does not always define its element. If the
associated element is already defined, then the `define` function is technically
a no-op, which can be confusing.

```javascript
import { MyElement } from './my-element.js';

// Defines the element.
MyElement.define();

// Silently does nothing.
MyElement.define();
```

The necessary post-condition for this function is that `MyElement` is defined in
the custom elements registry. Even if the second call technically had no effect,
the outcome is still that `MyElement` is definitely defined.

Alternatives names may be considered for bike-shedding.

### Scoped Custom Element Registries support

The
[Scoped Custom Element Registries proposal](https://github.com/WICG/webcomponents/blob/gh-pages/proposals/Scoped-Custom-Element-Registries.md)
is not yet finalized and therefore not included in the implementation of this
proposal. However, if and when that proposal is completed, it can be easily
included with a small addition which accepts a custom element registry and a tag
name as input parameters:

```javascript
export class MyElement extends HTMLElement {
  static define(registry = customElements, tagName = 'my-element') {
    // Tag name can only be modified when not in the global registry.
    if (registry === customElements && tagName !== 'my-element') {
      throw new Error('Cannot use a non-default tag name in the global custom element registry.');
    }

    // Check if the tag name was already defined by another class.
    const existing = registry.get(tagName);
    if (existing) {
      if (existing === Clazz) {
        return; // Already defined as the correct class, no-op.
      } else {
        throw new Error(`Tag name \`${tagName}\` already defined as \`${
            existing.name}\`.`);
      }
    }

    // Define the class.
    registry.define(tagName, Clazz, options);
  }
}

// Usage:
const registry = new CustomElementRegistry();
MyElement.define(registry);

// With custom tag name:
MyElement.define(registry, 'my-other-element');
```

This implementation registers the custom element on any given registry,
defaulting to the global registry.

Attempting to define a custom tag name on the global registry still throws in
order to maintain the invariant that all consumers of the global registry use
the agreed-upon name. See [custom tag name](#custom-tag-name).

### Scoped Custom Element Registries as an alternative

Scoped Custom Element Registries have many nice properties which potentially
allow them to serve as an alternative solution to many of the goals of this
proposal.

Scoped registries do not require global side-effects like a top-level
`customElements.define` call and allow every consumer to create their own
registry and define dependencies at an appropriate time. This enables components
to be tree-shaken when not used.

```javascript
class MyElement extends HTMLElement {
  // ...
}

function useMyElement() {
  const myRegistry = new CustomElementRegistry();
  myRegistry.define('my-element', MyElement);
  // ...
}
```

However, scoped custom element registries have some drawbacks which make them a
non-ideal solution to this proposal.

First, scoped registries are coupled to shadow DOM, which not all custom
elements use. This On-Demand Definitions proposal supports all custom elements,
even light DOM components. Note that a
[more recent scoped registry proposal](https://github.com/whatwg/html/issues/10854)
may lift this particular restriction.

Second, scoped registries require creating an entirely distinct registry with
potentially decoupled tag names. This places a constraint on consumers which
need to manually define a mapping of `some-tag-name` -> `MyElement`. This
constraint is reasonable within the context of a scoped registry, but is
completely unnecessary for the goals of this proposal. Not every consumer of an
element wants its own custom registry or to decouple and own its own mapping of
tag names.

Third, as shown in
[scoped registries support](#scoped-custom-element-registries-support), this
proposal can support scoped registries and even provides some benefit for them.
Having a `define` function owned by the component author provides an abstraction
over the tag name in the global registry and
[the `options` field](#allowing-options).

Fourth, scoped registries also require removing the top-level
`customElements.define` call anyways to realize their benefits, which On-Demand
Definitions naturally achieves as well.

Fifth, some component consumers may use a scoped custom elements registry, but
others may not and should still receive the benefits of this proposal. Using a
scoped registry does not address any of these problems for components in the
global registry, while this On-Demand Definitions proposal does. It is perfectly
valid for two different consumers to call `MyElement.define()` in the global
registry while a third consumer uses a scoped registry. All three receive the
benefits of this proposal.

For these reasons, scoped registries are not a better solution for the goals of
this proposal.

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

### Custom tag name

`define` only supports defining the element with one hard-coded tag name. This
behavior is equivalent to calling `customElements.define` in the top-level
scope. Consumers of the class are not able to influence the tag name used.

ALTERNATIVE PROPOSAL: Let the user provide an optional tag name to give
consumers more flexibility.

```javascript
export class MyElement extends HTMLElement {
  static define(tagName) {
    // ...

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

### Allowing options

The static `define` function does not support any additional options, such as
[the options of `customElements.define`](https://developer.mozilla.org/en-US/docs/Web/API/CustomElementRegistry/define#options).

ALTERNATIVE PROPOSAL: Allow `options` to be passed through as parameters to
`define`:

```javascript
class MyElement extends HTMLElement {
  static define(options) {
    // ...

    customElements.define('my-element', MyElement, options);
  }
}
```

Currently, the only option supported by `customElements.define` is
[`extends`](https://developer.mozilla.org/en-US/docs/Web/API/CustomElementRegistry/define#extends)
which allows a custom element to extend an existing built-in tag name and
leverage the
[`is` attribute](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements#using_a_custom_element).

Whether or not an element extends a built-in class is a decision made by the
author of that element, not its consumers. Allowing consumers to decide the
`extends` option would increase the likelihood of the field being set
incorrectly and encountering unsupported behavior.

It also raises the question of what happens when two `define` calls disagree on
the option.

```javascript
import { MyElement } from './my-element.js';

export function createParagraph() {
  MyElement.define({ extends: 'p' });
  return document.createElement('p', {
    is: 'my-element',
  });
}

export function createSection() {
  MyElement.define({ extends: 'section' });
  return document.createElement('section', {
    is: 'my-element',
  });
}
```

Whichever function is called first will dictate the actual `extends` value used.
Meanwhile the other function will find `MyElement` defined with the wrong
`extends` value and its returned element will not be an instance of `MyElement`
as expected. If ordering is non-deterministic or unpredictable, this behavior
can easily lead to bugs.

Instead, custom element definition options may only be specified within the
`define` function itself, where the component author can control its value.

```javascript
class MyElement extends HTMLParagraphElement {
  static define() {
    // ...

    customElements.define('my-element', MyElement, { extends: 'p' });
  }
}
```

Future additions to this options object maybe be more appropriate to make
configurable for individual consumers and considered on a case-by-case basis.
However, `customElements.define` is naturally creating a side-effect which
stores the provided component class and its configuration. When multiple
consumers are defining a component on-demand, they need to agree on that
configuration. This implies that no consumer can have direct control over the
configuration or else it would risk breaking other consumers when they are
forced to use a component with a configuration they did not expect. It is highly
unlikely future options introduced to `customElements.define` will support being
independently configurable for multiple consumers, therefore that functionality
is intentionally _not_ exposed.

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

Consider also that `MyApp` is defined in line with this protocol, omitting
top-level side-effects:

```javascript
// my-app.js

import {SomeComp} from './some-comp.js';

export class MyApp extends HTMLElement {
  static define() {
    // ...
  }

  // ...
}

// No top-level `customElements.define` or `MyApp.define`.
```

`my-app.js` _needs_ to call `customElements.define('my-app', MyApp)` /
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

### Development-only checks

The [example implementation](#example) includes a check which throws if the tag
name was already defined by a different class. This is useful for development
purposes in case of a tag name conflict, However, it is very unlikely to be a
problem in production applications after the developer has resolved any relevant
issues.

Therefore it is acceptable to omit this particular check and no-op in production
for the case of conflicting class definitions. This enables a small bundle size
improvement without affecting valid usage. A _minimal_ implementation of the
protocol looks like:

```javascript
export class MyElement extends HTMLElement {
  static define() {
    // If already defined, no-op.
    // Might be the wrong class, but we don't care in a well-formed application.
    if (customElements.get('my-element')) return;

    customElements.define('my-element', MyElement);
  }
}
```

### Why not inline the `define` implementation?

ALTERNATIVE PROPOSAL: The static `define` implementation is relatively small and
condensed, just execute that whenever there is a need to use a custom element.

```javascript
import {MyElement} from './my-element.js';

if (!customElements.get('my-element')) {
  customElements.define('my-element', MyElement);
}

document.querySelector('my-element').doSomething();
```

This is functionally equivalent to calling `MyElement.define` but does not
require `MyElement` to opt-in to this community protocol. Inlining `define` does
come with a few costs however.

First, `MyElement.define` provides an abstraction which encapsulates the tag
name and options passed to `customElements.define`. Without this abstraction, it
becomes more likely multiple consumers will define the same element multiple
times with different choices and run into the same problems as
[allowing options in the static `define` function](#allowing-options) does.

Second, inlining requires knowledge of the tag name. To call
`customElements.define`, the caller must know the tag name to define. In
general, every consumer of an element likely does need to know the tag name it
is consuming, however libraries or frameworks may want to handle calling
`customElements.define` automatically in a context where they don't necessarily
know the expected tag name and would require a product developer to manually
provide this information every time.

Third, when given a custom element which has not been defined, is it reasonable
to directly define that element? Most existing custom elements expect a single,
centralized `customElements.define` call and do not anticipate that they may be
defined at any time.

Consider a component written in the traditional style with an adjacent top-level
side-effect and no knowledge of the On-Demand Definitions protocol or an
equivalent "Just call `customElements.define` before you use it" convention:

```javascript
export class MyElement extends HTMLElement {
  // ...
}

doSomething(MyElement);

customElements.define('my-element', MyElement); // Throws an error.

function doSomething(elClass) {
  // Conditionally define the class if necessary.
  if (!customElements.get('my-element')) {
    customElements.define('my-element', elClass);
  }

  document.querySelector('my-element').doSomethingElse();
}
```

Because `doSomething` uses the convention of conditionally defining the element,
it is able to define `MyElement` *before* the adjacent top-level side-effect,
causing it to throw an error. Any code with a reference to `MyElement` prior to
its definition could potentially cause this problem. An unexpected early
definition can also be observed through other `customElements` APIs like `get`,
`getName`, or `whenDefined` which could affect component logic in unanticipated
ways.

Therefore On-Demand Definitions is better implemented as an opt-in decision by
any given component. By implementing the static `define` function, a component
essentially states: "I do not expect to be defined by a specific, centralized
`customElements.define` call." This guarantee is what allows decentralized
`define` calls to work consistently.

Fourth, to support tree-shaking, top-level side-effects need to be removed
regardless of whether a separate `define` abstraction is used. This begs the
question: What should a web component author do with their existing
`customElements.define` call? There are a few options:

1.  Move it to a separate ES module and tell consumers to import that when
    top-level side-effects are needed.
1.  Delete it entirely and expect every consumer to follow the convention of
    conditionally defining `MyElement` before using it.
1.  Move it into their own (not specified by a community protocol) version of a
    static `defineMyComponent` function with none of the interoperability
    benefits.

These each have their own trade offs and every component author is likely to
make an independent decision leading to divergence within the web component
space. This makes consuming web components even harder because consumers have to
ask "How do I ensure this component is defined?" every time they adopt a new
custom element.

On-Demand Definitions provides a recommended answer to this complicated question
which maximizes compatibility with the rest of the ecosystem.

Finally, there is a small bundle size and stability argument to make here.
Conditionally defining a custom element is not quite trivial and has a few
unexpected edge cases (ex. the component is already defined but with a different
class). The fewer times this function is implemented, the less likely bugs will
be introduced and the less JavaScript users need to download. Also components
are likely to be consumed more frequently than they will be implemented.
Therefore it follows that it will be slightly more optimal for component
definitions to implement `define` rather than asking every consumer of the
component to implement conditional define logic itself.

Note that custom element libraries and frameworks do skew this reasoning
slightly, however the general rule of component usage (even if implemented by a
small number of frameworks) outnumbering component definitions (even if also
implemented by a small number of frameworks) should hold in most environments.

### Why not use `customElements.whenDefined`?

It is possible to await a custom element definition via
`customElements.whenDefined` which would allow a module to be more tolerant of a
dependency component being defined after it.

ALTERNATIVE PROPOSAL: Use `customElements.whenDefined` to wait for a component
to be defined somewhere else.

```javascript
const el = document.querySelector('my-element');

// Wait until the element is defined.
customElements.whenDefined('my-element').then(() => {
  el.doSomething(); // Definitely defined now.
});
```

While `customElements.whenDefined` is a useful primitive, it is insufficient to
meet the goals of this proposal because it:

1.  relies on _some other module_ in the program to import the definition
    of `my-element`, which is not guaranteed to happen synchronously or ever at
    all.
1.  "colors" all usage of custom elements to be async which can
    block many otherwise-reasonable API contracts.
    *   For example, both the Lit and HydroActive use cases are synchronous and
        would be incompatible with asynchronously waiting for dependencies to be
        defined.
1.  requires independent knowledge of the tag name for an element which might
    not be known in generic contexts such as libraries or frameworks.
    *   `MyElement.define();` only requires a reference to the custom element
        class, not its tag name.
1.  does not improve tree-shakability of components.

Beyond those functional points, `customElements.whenDefined` notably does *not*
define a custom element or provide a definition, weakening the relationship
between a custom element and this specific usage. This is very different from
the intent behind On-Demand Definitions which opts to strengthen this
relationship to ensure that a specific, known custom element class is defined
before it is used.

`customElements.whenDefined` is great for use cases attempting to use a custom
element's definition which may or may not be provided by something else on the
page at any time. That does not describe the problem statement of this proposal
which wants a specific custom element to always be defined at a specific moment
in time. This indicates that `customElements.whenDefined` is the wrong primitive
to solve this particular problem.

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

While useful, `lit-analyzer` unfortunately requires the developer to integrate a
distinct service into their toolchain with special knowledge of Lit templates,
which otherwise does not require any such tooling.

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
implemented the On-Demand Definitions proposal to mitigate these issues.

HydroActive's design with respect to file ordering is described more thoroughly
in [this video](https://youtu.be/euFQRqrTSMk?si=i5HKHayt3QvuNytf&t=736), though
it is primarily focused on this problem within the context of hydration.
However, correctly hydrating an element also requires defining it so this is
effectively the same core problem.
