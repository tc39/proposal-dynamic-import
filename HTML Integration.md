# HTML Integration for `import()`

The following outlines the spec changes to the [HTML Standard](http://html.spec.whatwg.org/multipage/) that would be necessary to support the `import()` proposal in a web browser host environment.

## Script Infrastructure

The [Scripting Definitions](https://html.spec.whatwg.org/multipage/webappapis.html#definitions-2) section, which defines HTML's concepts of **script**, **classic script**, and **module script**, would be modified as follows:

- The _base URL_, _cryptographic nonce_, and _parser state_ properties, currently on module scripts, would move to the base script type

The procedure to [**create a classic script**](https://html.spec.whatwg.org/multipage/webappapis.html#creating-a-classic-script) would be updated to take these three properties, which it sets on the created classic script. All of its callers would be updated:

- The [script fetching algorithms](https://html.spec.whatwg.org/multipage/webappapis.html#fetching-scripts) would be updated to pass the appropriate values that they were passed (similar to module scripts), and their callers in turn would be updated to derive them appropriately as they do for module scripts
- The [inline classic script setup algorithm](https://html.spec.whatwg.org/multipage/scripting.html#script-processing-model:creating-a-classic-script) would pass along the values derived from the `script` element (similar to modules)
- The [`javascript:` URL processing algorithm](https://html.spec.whatwg.org/multipage/browsers.html#javascript-protocol) would pass along the _address_ variable, no cryptographic nonce, and `"not parser-inserted"`
- The [timer initialization steps](https://html.spec.whatwg.org/#timer-initialisation-steps) used by `setTimeout` and `setInterval` would pass along the caller realm's API base URL, no cryptographic nonce, and `"not parser-inserted"`

## Integration with the JavaScript module system

The [integration with the JavaScript module system](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-module-system) would be updated in a few ways.

The introductory sentence of [**resolve a module specifier**](https://html.spec.whatwg.org/multipage/webappapis.html#resolve-a-module-specifier) would changed to say and link to _script_, instead of _module script_, as with the base URL changes above, it can now accept both.

The algorithm for [HostResolveImportedModule](https://html.spec.whatwg.org/multipage/webappapis.html#hostresolveimportedmodule(referencingmodule,-specifier)) would be updated with the new parameter name, _referencingScriptOrModule_, and the variable name _referencing module script_ would be changed to _referencing script_.

We would define a new section for HostFetchImportedModule. The implementation would be as follows:

### HostFetchImportedModule(_referencingScriptOrModule_, _specifier_)

1. Let _referencing script_ be _referencingScriptOrModule_.[[HostDefined]].
1. Let _settings object_ be _referencing script_'s [settings object](https://html.spec.whatwg.org/multipage/webappapis.html#settings-object).
1. Return undefined but continue running the following steps in parallel:
1. Let _url_ be the result of resolving a module specifier given _referencing script_ and _specifier_. If the result is failure, asynchronously complete this algorithm with a `TypeError` exception and abort these steps.
1. Let _credentials mode_ be `"omit"`.
1. If _referencing script_ is a [module script](https://html.spec.whatwg.org/#module-script), set _credentials mode_ to _referencing script_'s [credentials mode](https://html.spec.whatwg.org/#concept-module-script-credentials-mode).
1. [Fetch a module script tree](https://html.spec.whatwg.org/multipage/webappapis.html#fetch-a-module-script-tree) given _url_, _settings object_, `"script"`, _referencing script_'s [cryptographic nonce](https://html.spec.whatwg.org/#concept-module-script-nonce), _referencing script_'s [parser state](https://html.spec.whatwg.org/#concept-module-script-parser), and _credentials mode_. If fetching a module script tree asynchronously completes with null, asynchronously complete this algorithm with an abrupt completion whose value is a new `TypeError`. Otherwise, asynchronously complete this algorithm with a normal completion whose value is undefined.
