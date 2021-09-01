# community-protocols

As the diversity of custom elements grows wider and deeper, so do the benefits of those elements being able to communicate with each other. Making this possible via reusably well-known protocols can enable components not just from X vendor to communicate with themselves, but also to interoperate with components from Y vendor and Z vendor. This project looks to define community protocols for cross-component coordination that reduce the overhead required to make that possible.

Think of things like a context API, async work (Suspense-like), SSR-interop, incremental hydration, etc., many of these capabilities get locked into framework APIs that increase the barriers to interoperability. With a community defined protocol, these features and more become attainable and portable across the web components ecosystem.

## Get involved

Check out the [Issues](https://github.com/webcomponents/community-protocols/issues) and [PRs](https://github.com/webcomponents/community-protocols/pulls) currently open on this repo to join in on conversations already inflight regarding a number of exciting areas. If you have ideas on a protocol you like raised to the community open a new [issues](https://github.com/webcomponents/community-protocols/issues/new) so that authors and consumers of libraries and components from across the community that likely have (or wanted to have) implemented features in that area can join in with their use cases as well. Once the rough edges have been hammered out, submit a PR with specs of the protocol and append it to the "Protocols" list below so it can be formalized and put into use across the community.

## Proposals

| Proposal       | Author           | Status |
|----------------|------------------|--------|
| [Context]      | Benjamin Delarre | Draft  |
| [Pending Task] | Justin Fagnani   | Draft  |

[Context]: https://github.com/webcomponents/community-protocols/blob/main/proposals/context.md
[Pending Task]: https://github.com/webcomponents/community-protocols/blob/main/proposals/pending-task.md

## Status

Community Protocols will go through a number of phases during which they can gather further community insight and approval before becoming "Accepted". These phases are as follows:

- *Proposal*
  "Proposal" status applies to just about anything that community members are interested in putting some thought into. While an issue submitted to this repo can help generate initial ideas on the protocol or space of interest and clarify the information needed to kick off a more fruitful conversation, a PR will serve to make known that you are interested in support to drive the protocol in question forward. Having an initial explainer included in this PR, while adding the protocol to the "Proposals" table shown above, will prepare the community to both communicate about and contribute to the development of the protocol asynchronously.

- *Draft*
  A protocol generally agreed to be an interesting area of investigation is given "Draft" status. At this point, the conversation will pivot away from proving the need for a protocol and towards proving a specific pattern by which the protocol can be achieved across the web components community. Developmental work in this area can be addressed in PRs or via group meetings of the [w3c's Web Components Community Group](https://github.com/w3c/webcomponents-cg) as needed.

- *Candidate*
  Once a protocol has received proper incubation and a pattern for applying it has solidified it will be given "Candidate" status. This status outlines that it will soon be accepted. Community members should use this opportunity to outline any final concerns or adjustments they'd like to see in the protocol before adoption. By this point, the work should mostly be done and the focus will be polish, presentation, and implementation.

- *Accepted*
  An "Accepted" protocol is one that is ready to put into action across the community. With clear APIs, types, usage patterns included they are ready to serve the greater purpose of supporting interoperability between web components built from various contexts.

  To move to "Accepted" a proposal needs __2__ implementations.

### Status Graduation

Community protocols are "generally agreed upon patterns" and not "browser specs", so they will always be a choice you make more so than rules a component developer or library author has to follow. In this way, graduation of a protocol from one status to the next will primarily happen in the absence of hard "nay"s. Your active participation in issues, PRs, or [w3c's Web Components Community Group](https://github.com/w3c/webcomponents-cg) meetings regarding specific protocols will be the best way to advocate for a protocol making its way through this process.
