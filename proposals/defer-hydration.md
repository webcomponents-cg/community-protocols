# Defer Hydration Protocol

An open protocol for controlling hydration on the client.

Author: Justin Fagnani

Status: Draft

Last update: 2021-06-24

# Background

In server-side rendered (SSR) applications, the process of a component running code to re-associate its template with the server-rendered DOM is called "hydration". We want to enable interoperable incremental hydration across web components, so that componentents do not automatically hydrate upon being defined, but wait for a signal from their parent or other coordinator.

The enables us to decouple loading the code for web component definitions from starting the work of hydration, and enables sophisticated coordination of hydration, including triggering hydration only on interaction or data changes for specific components.

# Overview

The Defer Hydration Protocol specifies an attribute named `defer-hydration` that is placed on elements, usually during server rendering, to tell them not to hydrate when they are upgraded. Removing this attribute is a signal that the element should hydrate. Elements can observe the attribute being removed via `observedAttribute` and `attributeChangedCallback`.

When an element hydrates it ca remove the `defer-hydration` attribute from its shadow children to hydrate them, or keep the attribute if itself can determin a more optimal time to hydrate all or certain children.
