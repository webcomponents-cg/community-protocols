# Performance Markers Protocol

**Author**: Matheus Cardoso [@cardoso](https://github.com/cardoso)

**Status**: Proposal

**Created**: 2025-02-25

**Last updated**: 2025-02-25

# Summary

A protocol for web component authors to use [performance markers](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API/User_timing#adding_performance_markers) that is consistent across different libraries and frameworks, and that can be used to track performance in developer tools.

# Example

TODO: Define the example of the API using type definitions and/or code example.

```typescript
// TODO: Define the devtools object
performance.mark(markName);

performance.measure(measureName, {
    start: markName,
    detail: {
        devtools: {
            dataType: 'track-entry',
            track: 'Web Components',
            ...devtools,
        },
    },
});
```

# Goals

- Naming conventions for performance markers.
- Mapping of performance markers to component lifecycle events.
- Provide a simple way to hook into performance markers in web components.

# Non-goals

- Define how markers are used by tooling.

# Design detail

TODO: Define the design of the API.

# Open questions

Should SSR be considered in the protocol?

# Previous considerations

List any previous proposals or priority art that inspired this proposal.

## Prior art

- [User Timing API](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API/User_timing)
- [Performance API](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API)
- Chrome DevTools Performance Panel
    - [API Design Document](https://docs.google.com/document/d/1Fp4LLvq2VAv9ksgcxDGLf472Rbi9vz_Tqs74DdcT0hw)
    - [Final Documentation](https://developer.chrome.com/docs/devtools/performance/extension)
- Lightning Web Components ([LWC](https://github.com/salesforce/lwc)) Implementation PRs
    - [feat(engine): enhance performance timings](https://github.com/salesforce/lwc/pull/4535)
    - [feat(engine): add tooltips for performance timings](https://github.com/salesforce/lwc/pull/4541)
    - [feat: add mutation logging to DevTools profile](https://github.com/salesforce/lwc/pull/4544)
    - [feat(ssr): add perf timings](https://github.com/salesforce/lwc/pull/5143)
