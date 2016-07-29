# Polymer "alacarte"
Exploratory code working up towards the Polymer 2.0 release.
## Overarching goals
* Custom Elements and Shadow DOM v1 support
* Polymer 2.0 components look just like “vanilla” web components from the outside
  * Remove the need for `Polymer.dom` calls
  * Remove the requirement for `set`/`notifyPath` path notifications in data-binding
* Rough edges sanded off of current data binding system
  * Batch changes and run effects for set of changes in well-defined order (compute, notify, propagate, observe)
  * Remove multi-property `undefined` rule
  * TBD: provide alternatives to object-identify array tracking
* Improved code factoring
  * Refactored into decoupled libraries that can stand on their own and be composed using raw ES6 classes
  * Any optional parts (e.g. shady shim, style shim, template elements, etc.) opt-in and not loaded/required by default
* Provide a minimally-breaking API surface area from Polymer 1.0, to the extent allowed given the above goals

## How to use & caveats
Alacarte includes a Polymer 1.0 "Backward Compatibility" (BC) layer loadable via `alacarte/polymer.html` that attempts to provide as close to the same API and semantics for using Polymer as possible.  Notes on usage:
* In order to test existing code that references `polymer/polymer.html`, you'll need to check out `alacarte` as `polymer`, or else redirect `polymer/polymer.html` to `alacarte/polymer.html`.
* At this moment, the Shady DOM shim is not included in `polymer.html`, meaning elements that create shadow roots will only run in Chrome.  To opt-in to testing Shady DOM, load `alacarte/src/shady/shady.html` in your app/demo.  When loaded, all browsers will use Shady DOM, even where native Shadow DOM exists.
* By default, `Polymer()` will attempt to register V1 `customElements.define` if present (via polyfill or native), but will fallback to V0 `document.registerElement`
* To use the Custom Elements V1 polyfill, check out / bower link the `v1-polymer-edits` branch of `webcomponentsjs` and load `webcomponentsjs/webcomponents-lite.js`.  You can control which polyfill/native support is used via these query string flags:
  - `?wc-ce=v1` uses V1 polyfill & uses native V1 when present (start Canary with `--enable-blink-features=CustomElementsV1` flag to test native V1 support)
  - `?wc-ce=v1&wc-register` uses V1 polyfill _even if_ native present
  - `?wc-ce=v0` uses V0 polyfill & uses native V0 when present (forces `window.customElements = null`)
  - `?wc-ce=v0&wc-register` uses V0 polyfill _even if_ native present
* You should continue to use `created` and `attached` Polymer callbacks when using the V1 CE polyfill, despite the name changes in the V1 spec.
* Style selector shimming is implemented when needed. This is necessary when ShadyDom is in use and provides style encapsulation.
* Custom properties: 
   * Native css custom properties are used by default (different from 1.0) on all browser that support them: Chrome, FF, Safari 10/Tech Preview. Native @apply is used where supported: Canary + experimental web platform features. When not available, @apply is emulated via custom properties (e.g. --foo { color: red; } becomes --foo_-_color: red;)
   * Custom properties are shimmed on other browsers (Safari 9, IE/Edge) [A flag will be added to force this on other browsers]. This is currently implemented for elements, not yet implemented for custom-style.
* custom-style: does not currently work in HTMLImports when native v1 custom elements are used (edited)

## Not yet implemented
* Some utility functions are not yet implemented
    * A number of utility functions that were previously on the Polymer 1.0 element prototype are not ported over yet.  These will warn with "not yet implemented" warnings.
* Array notification API's not yet implemented.  Note due to removal of object/array dirty check, you should be able to just make changes using normal array methods, then re-set the array to an element and it will "re-go"
* `<array-selector>` not yet implemented

## Breaking Changes
This is a list of proposed/intentional breaking changes as implemented in the current incantation of this repo.  If you find changes that broke existing code or rules, please raise and we'll need to decide whether they are expected/intentional or not.

Note that some of the items listed below have been temporarily shimmed to make testing existing code easier, but will be removed in the future.

### Styling
* Drop invalid syntax
  * :root {}
    * Should be :host > * {}
  * var(--a, --b)
    * Should be var(--a, var(--b))
  * @apply(--foo)
    * Should be @apply --foo;
* Native CSS Custom Properties by default
* TBD: Drop `element.customStyle`

### Element definitions
* TBD: `dom-if`, `dom-repeat`, `dom-bind`, `array-selector`, etc. will not included in `polymer.html` by default (going forward; they currently are); users should import those elements when needed
* TBD: For now, all template type extensions have now been changed to standard custom elements that take a `<template>` in their light dom, which allows them to be used in the current native V1 implementation in Canary (which does not yet support `is`) and a future version of Safari (that likely won't support `is`).  e.g. 

  ```
<template is="dom-repeat" items="{{items}}">...</template>
  ```

  should change to

  ```
<dom-repeat items="{{items}}">
   <template>...</template>
</dom-repeat>
  ```

  For the time being, `Polymer()` will automatically wrap template extensions used in Polymer element templates during template processing for backward-compatibility, although we may decide to remove this auto-wrapping in the future.  Templates used in the main document must be manually wrapped.

### Polymer element prototype
* Methods starting with `_` are not guaranteed to exist (most have been removed)

### Element lifecycle
* Attached: no longer deferred until first render time. Instead when measurement is needed use... API TBD.
* `lazyRegister` option removed and is now “on” by default
* Experimental: `listeners` and `hostAttributes` are deferred until "afterNextRender", since the majority uses of these should not be initial paint-blocking.  Please help identify use cases where paint-blocking host attributes/listeners are useful/needed.
* Requst to early-users: We would really like to remove the `ready` callback, since its use is generally anti-pattern-ish, and it's hard to document when a "one-shot callback that runs after all local dom & observers have flushed" should actually be used, as opposed to running said code in an observer.  In exploring alacarte, please try to avoid `ready` and help identify use cases where it is useful/needed.

### Data
* Template not stamped & data system not initialized (observers, bindings, etc.) until attached (or until microtask, if we go the async by default route)
  * Fallout from V1 changes, since attributes not readable in constructor
* Re-setting an object or array no longer ditry checks, meaning you can make deep changes to an object/array and just re-set it, without needing to use `set`/`notifyPath`.
* Inline computed annotations run unconditionally at initialization, regardless if any arguments are defined (and will see undefined for undefined arguments) <-- the inconsistency between this and how normal observers/computed runs is probably not justifiable; will probably revisit
* Inline computed annotations are always “dynamic” (e.g. setting/changing the function will cause the binding to re-compute)
* Other method effects (multi-prop observers, computed) called once at initialization if any arguments are defined (and will see undefined for undefined arguments)
‘notify’ events not fired when value changes as result of binding from host 
* In order for a property to be deserialized from its attribute, it must be declared in the `properties` metadata object

Other
* Shadow DOM v1
  * `<content select="...">` → `<slot name="...">`
  * Selectively distributed content needs to add `slot` attribute
* Custom Elements v1
  * Applies to any “raw” custom elements (e.g. test-fixture)
    * createdCallback → constructor
      * Basically can only set instance properties
      * Must not inspect attributes, children, parent
    * attachedCallback → connectedCallback
* Alacarte code uses limited ES2015 syntax (mostly just `class`, so it can be run without transpilation in current Chrome, Safari, FF, and Edge; note Safari 9 does not currently support `=>`, among others).  Transpilation is required to run in IE11.
