# Cross-root ARIA

- [**WIP** Spec draft](https://leobalter.github.io/cross-root-aria-delegation/)

## Public summary from WCCG

https://w3c.github.io/webcomponents-cg/#cross-root-aria

**GitHub Issue(s):**

* [WICG/aom#169](https://github.com/WICG/aom/issues/169)
* [WICG/aom#107](https://github.com/WICG/aom/issues/107)
* [WICG/webcomponents#917](https://github.com/WICG/webcomponents/issues/917)
* [WICG/webcomponents#916](https://github.com/WICG/webcomponents/issues/916)

## Description

1. Shadow root encapsulation currently prevents references between elements in different roots. Cross-root ARIA references would re-enable this platform feature within shadow roots.
2. It's not possible to "reflect" ARIA attributes from am element in a shadow root up to the host of that shadow root. For instance, the DOM for a listbox fully encapsulated in a shadow root cannot be reflected into the parent DOM tree as would be needed for it to be referenced by the `aria-controls` attribute on an `<input>` element. Similarly, the list item descendent could not reflect its content for reference by an `aria-activedescendent` attribute on the same `<input>` element. These DOM elements are rightfully encapsulated within their shadow root, but a system of aria reflection could map their content (sans element reference) to their host, and the host could pass that content when referenced by `aria-controls` or `aria-activedescendent`.

## Motivation

Content on the web being accessible is critically important. Making web component content accessible currently requires many complicated and only partially-successful workarounds, such as:

* Observing and moving ARIA-related attributes across elements (for role, etc.)
* Using non-standard attributes for ARIA features, in order to apply them to elements in a shadow root.
* Requiring that custom elements users wrap/slot elements so that ARIA attributes can be placed directly on them. This gets very complicated as the number of slotted inputs and levels of shadow root nesting increase.
* Duplicating nodes across shadow root boundaries
* Abandoning Shadow DOM

## Explainers

- [Cross-root ARIA Reflection API](cross-root-aria-reflection.md)
