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

## Examples

### Unlocking the Combobox

Take this simplified ["Editable Combobox With List Autocomplete Example"](https://www.w3.org/WAI/ARIA/apg/example-index/combobox/combobox-autocomplete-list.html) from the ARIA Authoring Practices Guide.

```html
<label for="cb1-input">State</label>
<div class="combobox combobox-list">
    <div class="group">
        <input
            id="cb1-input"
            class="cb_edit"
            type="text"
            role="combobox"
            aria-autocomplete="list"
            aria-expanded="true"
            aria-controls="cb1-listbox"
            aria-activedescendant="lb1-ak"
        />
        <button
            id="cb1-button"
            tabindex="-1"
            aria-label="States"
            aria-expanded="true"
            aria-controls="cb1-listbox"
        ></button>
    </div>
    <ul id="cb1-listbox" role="listbox" aria-label="States">
        <li id="lb1-al" role="option">Alabama</li>
        <li id="lb1-ak" role="option">Alaska</li>
    </ul>
</div>
```

Currently, to fully achieve the relationships outlined therein, if you wanted to convert
this DOM to custom elements, you'd really only have the option to decorate the example:

```html
<x-label>
    <label for="cb1-input">State</label>
</x-label>
<x-combobox>
    <x-input-group>
        <x-input>
            <input
                id="cb1-input"
                class="cb_edit"
                type="text"
                role="combobox"
                aria-autocomplete="list"
                aria-expanded="true"
                aria-controls="cb1-listbox"
                aria-activedescendant="lb1-ak"
            />
        </x-input>
        <x-button>
            <button
                id="cb1-button"
                tabindex="-1"
                aria-label="States"
                aria-expanded="true"
                aria-controls="cb1-listbox"
            ></button>
        </x-button>
    </x-input-group>
    <x-listbox>
        <ul id="cb1-listbox" role="listbox" aria-label="States">
            <li id="lb1-al" role="option">Alabama</li>
            <li id="lb1-ak" role="option">Alaska</li>
        </ul>
    </x-listbox>
</x-combobox>
```

While this offers some additional custom element-based control over the styles attributed to this
UI, the approach continues to hoist the responsibility of building this DOM to the parent component or
application.

When moving that DOM management responsibility to the individual custom elements
themselves, the responsibility of the consuming developer begins to diminish, but the ID based
relationships begin to change or become impossible. To support this, we've assumed the presence of
the ARIA Attribute Delegation API in following code examples:

```html
<x-label id="cb1-label">State</x-label>
<x-combobox>
    <x-input-group>
        <x-input
            aria-labeledby="cb1-label"
            role="combobox"
            aria-autocomplete="list"
            aria-expanded="true"
            aria-controls="cb1-listbox"
            aria-activedescendant="lb1-ak"
        >
            #shadow-root delegates="aria-labeledby role aria-autocomplete aria-expanded aria-controls aria-activedescendant"
                <input
                    type="text"
                    auto-role
                    auto-aria-labeledby
                    auto-aria-autocomplete
                    auto-aria-expanded
                    auto-aria-controls
                    auto-aria-activedescendant
                />
        </x-input>
        <x-button
            aria-labeledby="cb1-label"
            tabindex="-1"
            aria-expanded="true"
            aria-controls="cb1-listbox"
        ></x-button>
    </x-input-group>
    <x-listbox
        aria-labeledby="cb1-label"
        options='[["Alabama", "lb1-al"], ["Alaska", "lb1-ak"]]'
    ></x-listbox>
</x-combobox>
```

Here we've moved from the `for` attribute on a `<label>` element to giving its host an ID so that
other elements can reference it via `aria-labelledby`. This persists the accessible relationship, 
but it removes the previously managed interactions, like clicking the `<label>` focusing on the
input. We also see the `aria-activedescendant` relationship broken as the ID `lb1-ak` moves into 
the shadow root of the `<x-listbox>` element. This is the first place the ARIA Attribute Reflection 
benefits the refactor of the pattern from raw DOM to custom elements.

```html
<x-label id="cb1-label">State</x-label>
<x-combobox>
    <x-input-group>
        <x-input
            aria-labeledby="cb1-label"
            role="combobox"
            aria-autocomplete="list"
            aria-expanded="true"
            aria-controls="cb1-listbox"
            aria-activedescendant="lb1-ak"
        >
            #shadow-root delegates="aria-labeledby role aria-autocomplete aria-expanded aria-controls aria-activedescendant"
                <input
                    type="text"
                    auto-role
                    auto-aria-labeledby
                    auto-aria-autocomplete
                    auto-aria-expanded
                    auto-aria-controls
                    auto-aria-activedescendant
                />
        </x-input>
        <x-button
            aria-labeledby="cb1-label"
            tabindex="-1"
            aria-expanded="true"
            aria-controls="cb1-listbox"
        >
            #shadow-root delegates="aria-expanded aria-controls aria-label"
                <button auto-aria-expanded auto-aria-controls auto-aria-label></button>
        </x-button>
    </x-input-group>
    <x-listbox
        aria-labeledby="cb1-label"
        id="cb1-listbox"
        options='["Alabama", "Alaska"]'
    >
        #shadow-root delegates="label" reflects="aria-activedescendant role"
            <ul role="listbox" reflect-role autolabel>
                <li role="option">Alabama</li>
                <li role="option" reflect-aria-activedescendant>Alaska</li>
            </ul>
    </x-listbox>
</x-combobox>
```

As we begin to see the benefits and capabilities that the Delegation and Reflection APIs open for 
out custom element architectures, this example can be further simplified.

```html
<x-label for="cb1">
    #shadow-root delegates="for"
        <label autofor><slot></slot></label>
    State
</x-label>
<x-combobox
    id="cb1"
    options='["Alabama", "Alaska"]'
>
    #shadow-root delegates="focus label"
        <x-input
            aria-labeledby="cb1-label"
            role="combobox"
            aria-autocomplete="list"
            aria-expanded="true"
            aria-controls="listbox"
            aria-activedescendant="listbox"
        >
            #shadow-root delegates="aria-labeledby role aria-autocomplete aria-expanded aria-controls aria-activedescendant"
                <input
                    type="text"
                    auto-role
                    auto-aria-labeledby
                    auto-aria-autocomplete
                    auto-aria-expanded
                    auto-aria-controls
                    auto-aria-activedescendant
                />
        </x-input>
        <x-button
            autolabel
            tabindex="-1"
            aria-expanded="true"
            aria-controls="listbox"
        >
            #shadow-root reflects="role"
                <button reflect-role></button>
        </x-button>
        <x-listbox
            autolabel
            id="listbox"
            options=options
        >
            #shadow-root delegates="label" reflects="aria-activedescendant role"
                <ul role="listbox" reflect-role autolabel>
                    <li role="option">Alabama</li>
                    <li role="option" reflect-aria-activedescendant>Alaska</li>
                </ul>
        </x-listbox>
</x-combobox>
```

In the above example, the move to leveraging a shadow root on `<x-combobox>` even opened the one 
to many relationship of the `autolabel` delegation allowing for the return to the `for` attribute, 
which is delegated to an actual `<label>` element to surface the interaction relationships we had 
previously lost with the move to `aria-labelledby`. All the while, we've reduced the DOM that a 
consumer of this pattern is required to write to:

```html 
<x-label for="cb1">State</x-label>
<x-combobox
    id="cb1"
    options='["Alabama", "Alaska"]'
></x-combobox>
```

This feels like a really nice refactor. All of these changes are powered by highly useful API in 
the form of ARIA Attribute Delegation and ARIA Attribute Reflection and were previously not possible 
when placing shadow boundaries between important content in an interface. However, it does assume 
a greenfield implementation. A more realistic look at what these APIs can surface will be 
derived from decorating or composing existing patterns into these complex interfaces.

Take this interpretation of a popular custom elements library's "input" and "list" components:

```html 
<y-textfield
    label="State"
>
    #shadow-root
        <label>
            ${label}
            <input />
        </label>
</y-textfield>
<y-button></y-button>
<y-list>
    <y-list-item>Alabama</y-list-item>
    <y-list-item>Alaska</y-list-item>
</y-list>
```

Let's see how we might be able to update this example with the ARIA Delegation and Reflection 
APIs in order to complete the Combobox contract for screen readers.

```html 
<y-textfield
    label="State"
    id="cb2-textfield"
    role="combobox"
    aria-autocomplete="list"
    aria-expanded="true"
    aria-controls="cb2-listbox"
    aria-activedescendant="lb2-ak"
>
    #shadow-root delegates="role aria-autocomplete aria-expanded aria-controls aria-activedescendant" reflects="label"
        <label reflects-label>
            ${label}
            <input
                auto-role
                auto-aria-labeledby
                auto-aria-autocomplete
                auto-aria-expanded
                auto-aria-controls
                auto-aria-activedescendant
            />
        </label>
</y-textfield>
<y-button
    aria-labelledby="cb2-textfield"
    aria-expanded="true"
    aria-controls="cb2-listbox"
    icon="expand_more"
>
    #shadow-root delegates="label aria-expanded aria-controls"
        <button auto-label auto-aria-expanded auto-aria-controls>
            <y-icon>expand_more</y-icon>
        </button>
</y-button>
<y-list
    aria-labelledby="cb2-textfield"
    id="cb2-listbox"
>
    <y-list-item id="lb2-al">Alabama</y-list-item>
    <y-list-item id="lb2-ak">Alaska</y-list-item>
</y-list>
```

This places a pretty high burden on consumers:

```html 
<y-textfield
    label="State"
    id="cb2-textfield"
    role="combobox"
    aria-autocomplete="list"
    aria-expanded="true"
    aria-controls="cb2-listbox"
    aria-activedescendant="lb2-ak"
></y-textfield>
<y-button
    aria-labelledby="cb2-textfield"
    aria-expanded="true"
    aria-controls="cb2-listbox"
    icon="expand_more"
></y-button>
<y-list
    aria-labelledby="cb2-textfield"
    id="cb2-listbox"
>
    <y-list-item id="lb2-al">Alabama</y-list-item>
    <y-list-item id="lb2-ak">Alaska</y-list-item>
</y-list>
```

However, it could be easily composed into a single shadow root:

```html 
<y-combobox
    label="state"
    aria-activedescendant="lb2-ak"
>
    #shadow-root delegates="aria-activedescendant"
        <y-textfield
            label=label
            id="cb2-textfield"
            role="combobox"
            aria-autocomplete="list"
            aria-expanded="true"
            aria-controls="cb2-listbox"
            auto-aria-activedescendant
        ></y-textfield>
        <y-button
            aria-labelledby="cb2-textfield"
            aria-expanded="true"
            aria-controls="cb2-listbox"
            icon="expand_more"
        ></y-button>
        <y-list
            aria-labelledby="cb2-textfield"
            id="cb2-listbox"
        >
            <slot name="items"></slot>
        </y-list>
    <y-list-item id="lb2-al" slot="items">Alabama</y-list-item>
    <y-list-item id="lb2-ak" slot="items">Alaska</y-list-item>
</y-combobox>
```

This comes out to the following for a consuming developer:

```html 
<y-combobox
    label="state"
>
    <y-list-item id="lb2-al" slot="items">Alabama</y-list-item>
    <y-list-item id="lb2-ak" slot="items">Alaska</y-list-item>
</y-combobox>
```

Even less if you choose to encapsulate DOM management for the list items by accepting an array of 
items the way our `<x-combobox>` did above. However, in both cases the ARIA Attribute Delegation 
and Reflection APIs are making it possible for more complex interfaces to be accessible when built 
with custom element and shadow DOM without needing to architect your whole implementation around 
the intricacies of keeping ID references in a single DOM tree.

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


## Public summary from WCCG

https://w3c.github.io/webcomponents-cg/#cross-root-aria

**GitHub Issue(s):**

* [WICG/aom#169](https://github.com/WICG/aom/issues/169)
* [WICG/aom#107](https://github.com/WICG/aom/issues/107)
* [WICG/webcomponents#917](https://github.com/WICG/webcomponents/issues/917)
* [WICG/webcomponents#916](https://github.com/WICG/webcomponents/issues/916)

