# Cross-root ARIA Reflection API explainer

- [source](https://github.com/WICG/webcomponents/issues/917)
- [**WIP** spec draft](https://westbrook.github.io/cross-root-aria-reflection/)

It is critically important that content on the web be accessible. When native elements are not the solution, ARIA is the standard which allows us to describe accessible relationships among elements.  Unfortunately, it's mechanism for this is historically based on `IDREF`s, which cannot express relationships between different DOM trees, thus creating a problem for applying these relationships across a `ShadowRoot`.

Today, because of that, authors are left with only incomplete and undesirable choices:

* Observe and move ARIA-related attributes across elements (for role, etc.).
* Use non-standard attributes for ARIA features, in order to apply them to elements in a shadow root.
* RequirE usage of custom elements to wrap/slot elements so that ARIA attributes can be placed directly on them. This gets very complicated as the number of slotted inputs and levels of shadow root nesting increase.
* Duplicating nodes across shadow root boundaries.
* Abandoning Shadow DOM.
* Abdandoning accessibility.

It is important that this be addressed and authors be able to establish, enable and manage important relationships.

This proposal introduces a reflection API which would allow ARIA attributes and properties set on elements in a shadow root can be reflected by their host element into the parent DOM tree..

This mechanism will allow users to apply standard best practices for ARIA and resolve a large margin of accessibility use cases for applications of native Web components and native Shadow DOM. This API is most suited for one-to-one delegation, but should also work for one-to-many scenarios. There is no mechanism for directly relating two elements in different shadowroots together, but this will still be possible manually with the element reflection API.

The proposed extension adds a new `reflects*` (e.g.: `reflectsAriaLabel`, `reflectsAriaDescribedBy`) options to the `.attachShadow` method similarly to the `delegatesFocus`, while introducing a new content attribute `auto*` (e.g.: `reflectarialabel`, `reflectariadescribedby`) to be used in the shadowroot inner elements. This has an advantage that it works with Declarative Shadow DOM as well (though, it requires another set of HTML attributes in the declarative shadow root template), and it is consistent with `delegatesFocus`. The declarative form works better with common developer paradigm where they may not necessarily have access to a DOM node right where they are creating / declaring it.

```html
<input aria-controlls="foo" aria-activedescendent="foo">Description!</span>
<template id="template1">
  <ul reflectariacontrols>
    <li>Item 1</li>
    <li reflectariaactivedescendent>Item 2</li>
    <li>Item 3</li>
  </ul>
</template>
<x-foo id="foo"></x-foo>
```

```html
const template = document.getElementById('template1');

class XFoo extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: "open", reflectsAriaControls: true, reflectsAriaActivedescendent: true });
    this.shadowRoot.appendChild(template.content.cloneNode(true));
  }
}
customElements.define("x-foo", XFoo);
```

In the example above, `x-foo` would be able to play the role of both the `aria-controls` element and the `aria-activedescendent` element for the `input`, setting that basis for a combo box style interface.

For instance when reflecting `aria-activeelement` it is desirable that readers know that it applies to the `input` so that focus does not need to be thrown to a different element when updating the active element. Current workarounds include copying the DOM and referencing something in the same DOM tree in order to complete the reader relationship, while placing that content invisibly in the same place as the content that a pointer device user might leverage to interact with the same data.

Live example: TBD

## Usage with Declarative Shadow DOM

This extension allows usage of attributes with [Declarative Shadow DOM](https://github.com/mfreed7/declarative-shadow-dom/blob/master/README.md) and won't block SSR.

Following the same rules [to add declarative options from `attachShadow`](https://github.com/mfreed7/declarative-shadow-dom/blob/master/README.md#additional-arguments-for-attachshadow) such as `delegateFocus`, we should expect ARIA delegation to be used as:

```html
<input aria-controlls="foo" aria-activedescendent="foo">Description!</span>
<x-foo id="foo">
  <template shadowroot="open" shadowrootreflectscontrols shadowrootreflectsariaactivedescendent>
    <ul reflectariacontrols>
      <li>Item 1</li>
      <li reflectariaactivedescendent>Item 2</li>
      <li>Item 3</li>
    </ul>
  </template>
</x-foo>
```

In the example above, the `<template shadowroot>` tag has the content attributes `shadowrootreflectscontrols` and `reflectsariaactivedescendent`, matching the options in the imperative `attachShadow`:

```javascript
this.attachShadow({ mode: "open", delegatesAriaLabel: true, reflectsAriaControls: true, reflectsAriaActivedescendent: true });
```

This mirrors the usage of delegatesFocus as:

```html
<template shadowroot="open" shadowrootdelegatesfocus>
```

being the equivalent of:


```javascript
this.attachShadow({ mode: "open", delegatesFocus: true });
```

For now, consistency is being preserved, but otherwise the ARIA delegation attributes can be simplified as:

```html
<span id="foo">Description!</span>
<x-foo aria-label="Hello!" aria-describedby="foo">
  <template shadowroot="open" reflectscontrols reflectsariaactivedescendent>
    <input id="input" autoarialabel autoariadescribedby />
    <span autoarialabel>Another target</span>
  </template>
</x-foo>
```

Or even further by accepting a token list on `reflects` or a similarly shaped attribute:


```html
<span id="foo">Description!</span>
<x-foo aria-label="Hello!" aria-describedby="foo">
  <template shadowroot="open" reflects="controls aria-activedescendent">
    <input id="input" autoarialabel autoariadescribedby />
    <span autoarialabel>Another target</span>
  </template>
</x-foo>
```

* * *

# Appendix

## No silver bullet

Not all cross-shadow use cases are covered. Cases like radio groups, tab groups, or combobox might require complexity that is not available yet at this current cross-root delegation API. Though custom versions of this might be possible where they weren't before, like building a custom radio group with the `<input>` inside of a shadow root:

```html
<div role="radiogroup">
  <x-radio>
    <template shadowroot="open" reflects="role aria-checked">
      <input type="radio" reflectrole reflectariachecked />
    </template>
  </x-radio>
</div>
```

The reflection API might also not resolve attributions from multiple shadow roots in parallel or attributes that would point to DOM trees containing the current host component.

## Thoughts: attribute names are too long

The attributes names such as `shadowrootreflectss*` are very long and some consideration for shorter names by removing the `shadowroot` prefix can be discussed as long the discussion is sync'ed with the stakeholders of the respective Declarative Shadow DOM proposal. This can be further shortened by taking the `reflects` collection as a DOM token list on a single attribue.

## Non-"aria" attributes for consideration

### datalist

Leveraging the `datalist` attribute on an encapsulated `<input>` element requires that support for this feature is built into the same DOM tree. Support for reflection of this attribute would free this feature to be leveraged more widely:

```html
<label for="ice-cream-choice">Choose a flavor:</label>
<d-input list="ice-cream-flavors" id="ice-cream-choice">
  <template shadowroot="open" delegates="list label">
    <input auto-list auto-label name="ice-cream-choice" />
  </template>
</d-input>

<d-datalist id="ice-cream-flavors">
  <template shadowroot="open" reflects="list">
    <datalist reflect-list>
      <option value="Chocolate">
      <option value="Coconut">
      <option value="Mint">
      <option value="Strawberry">
      <option value="Vanilla">
    </datalist>
  </template>
</d-datalist>
```

### OpenUICG Popup API

The OpenUICG is developing an API to support content that popsup over a page. In thier example of 
[using the new attributes in shadow DOM](https://open-ui.org/components/popup.research.explainer#shadow-dom)
the element owning the `popup` attribute is encapsulated in a shadow root to protect it from the 
surrounding application context. This means that the relationship required for a `togglepopup`, 
`showpopup`, or `hidepopup` bearing element can no longer be made from that level:

```html 
<my-tooltip>
    <template shadowroot=closed>
      <div popup=hint>This is a tooltip: <slot></slot></div>
    </template>
    Tooltip text here!
  </my-tooltip>
```

This could be addressed by reflecting the `popup` element to the host like so:

```html 
<button togglepopup=foo>Toggle the pop-up</button>
<my-tooltip id="foo">
    <template shadowroot=closed reflects="popup">
        <div popup=hint reflect-popup>This is a tooltip: <slot></slot></div>
    </template>
    Tooltip text here!
</my-tooltip>
```

Here, the declarative relationship between the `[togglepopup]` element and the `[popup]` elements 
can be made without surfacing the `.showPopUp()` method directly on the `<my-tooltip>` container.

## Public summary from WCCG

https://w3c.github.io/webcomponents-cg/#cross-root-aria

**GitHub Issue(s):**

* [WICG/aom#169](https://github.com/WICG/aom/issues/169)
* [WICG/aom#107](https://github.com/WICG/aom/issues/107)
* [WICG/webcomponents#917](https://github.com/WICG/webcomponents/issues/917)
* [WICG/webcomponents#916](https://github.com/WICG/webcomponents/issues/916)

