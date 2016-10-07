# import()

This repository contains a proposal for adding a "function-like" `import()` module loading syntactic form to JavaScript. It is currently in stage 2 of [the TC39 process](https://tc39.github.io/process-document/). Previously it was discussed with the module-loading community in [whatwg/loader#149](https://github.com/whatwg/loader/issues/149).

You can view the in-progress [spec draft](https://domenic.github.io/proposal-import-function/) and take part in the discussions on the [issue tracker](https://github.com/domenic/proposal-import-function/issues).

## Motivation and use cases

The existing syntactic forms for importing modules are static declarations. They accept a string literal as the module specifier, and introduce bindings into the local scope via a pre-runtime "linking" process. This is a great design for the 90% case, and supports important use cases such as static analysis, bundling tools, and tree shaking.

However, it's also desirable to be able to dynamically load parts of a JavaScript application at runtime. This could be because of factors only known at runtime (such as the user's language), for performance reasons (not loading code until it is likely to be used), or for robustness reasons (surviving failure to load a non-critical module). Such dynamic code-loading has a long history, especially on the web, but also in Node.js (to delay startup costs). The existing `import` syntax does not support such use cases.

Truly dynamic code loading also enables advanced scenarios, such as racing multiple modules against each other and choosing the first to successfully load.

## Proposed solution

This proposal adds an `import(specifier)` syntactic form, which acts in many ways like a function (but see below). It returns a promise for the module namespace object of the requested module, which is created after fetching, instantiating, and evaluating all of the module's dependencies, as well as the module itself.

Here `specifier` will be interpreted the same way as in an `import` declaration (i.e., the same strings will work in both places). However, while `specifier` is a string it is not necessarily a string literal; thus code like `` import(`./language-packs/${navigator.language}.js`) `` will work—something impossible to accomplish with the usual `import` declarations.

`import()` is proposed to work in both scripts and modules. This gives script code an easy asynchronous entry point into the module world, allowing it to start running module code.

Like the existing JavaScript module specification, the exact mechanism for retrieving the module is left up to the host environment (e.g., web browsers or Node.js). This is done by introducing a new host-environment-implemented abstract operation, HostFetchImportedModule, in addition to reusing and slightly tweaking the existing HostResolveImportedModule.

(This two-tier structure of host operations is in place to preserve the semantics where HostResolveImportedModule always returns synchronously, using its argument's [[RequestedModules]] field. In this way, HostFetchImportedModule can be seen as a mechanism for dynamically populating the [[RequestedModules]] field. This is similar to how some host environments already fetch the module tree in ahead of time, to ensure all HostResolveImportedModule calls during module evaluation are able to find the requested module.)

## Example

Here you can see how `import()` enables lazy-loading modules upon navigation in a very simple single-page application:

```html
<!DOCTYPE html>
<nav>
  <a href="books.html" data-entry-module="books">Books</a>
  <a href="movies.html" data-entry-module="movies">Movies</a>
  <a href="video-games.html" data-entry-module="video-games">Video Games</a>
</nav>

<main>Content will load here!</main>

<script>
  const main = document.querySelector("main");
  for (const link of document.querySelectorAll("nav > a")) {
    link.addEventListener("click", e => {
      e.preventDefault();

      import(`./section-modules/${link.dataset.entryModule}.js`)
        .then(module => {
          module.loadPageInto(main);
        })
        .catch(err => {
          main.textContent = err.message;
        });
    });
  }
</script>
```

Note the differences here compared to the usual `import` declaration:

* `import()` can be used from scripts, not just from modules.
* If `import()` is used in a module, it can occur anywhere at any level, and is not hoisted.
* `import()` accepts arbitrary strings (with runtime-determined template strings shown here), not just static string literals.
* The presence of `import()` in the module does not establish a dependency which must be fetched and evaluated before the containing module is evaluated.
* `import()` does not establish a dependency which can be statically analyzed. (However, implementations may still be able to perform speculative fetching in simpler cases like `import("./foo.js")`.)

## Alternative solutions explored

There are a number of other ways of potentially accomplishing the above use cases. Here we explain why we believe `import()` is the best possibility.

### Using host-specific mechanisms

It's possible to dynamically load modules in certain host environments, such as web browsers, by abusing host-specific mechanisms for doing so. Using HTML's `<script type="module">`, the following code would give similar functionality to `import()`:

```js

function importModule(url) {
  return new Promise((resolve, reject) => {
    const script = document.createElement("script");
    const tempGlobal = "__tempModuleLoadingVariable" + Math.random().toString(32).substring(2);
    script.type = "module";
    script.textContent = `import * as m from "${url}"; window.${tempGlobal} = m;`;

    script.onload = () => {
      resolve(window[tempGlobal]);
      delete window[tempGlobal];
      script.remove();
    };

    script.onerror = () => {
      reject(new Error("Failed to load module script with URL " + url));
      delete window[tempGlobal];
      script.remove();
    };

    document.documentElement.appendChild(script);
  });
}
```

However, this has a number of deficiencies, apart from the obvious ugliness of creating a temporary global variable and inserting a `<script>` element into the document tree only to remove it later.

The most obvious is that it takes a URL, not a module specifier; furthermore, that URL is relative to the document's URL, and not to the script executing. This introduces a needless impedance mismatch for developers, as they need to switch contexts when using the different ways of importing modules, and it makes relative URLs a potential bug-farm.

Another clear problem is that this is host-specific. Node.js code cannot use the above function, and would have to invent its own, which probably would have different semantics (based, likely, on filenames instead of URLs). This leads to non-portable code.

Finally, it isn't standardized, meaning people will need to pull in or write their own version each time they want to add dynamic code loading to their app. This could be fixed by adding it as a standard method in HTML (`window.importModule`), but if we're going to standardize something, let's instead standardize `import()`, which is nicer for the above reasons.

### An actual function

Drafts of the [Loader](https://whatwg.github.io/loader/) ideas collection have at various times had actual functions (not just function-like syntactic forms) named `System.import()` or `System.loader.import()` or similar, which accomplish the same use cases.

The biggest problem here, as previously noted by the spec's editors, is how to interpret the specifier argument to these functions. Since these are just functions, which are the same across the entire Realm and do not vary per script or module, the function must interpret its argument the same no matter from where it is called. (Unless something truly weird like stack inspection is implemented.) So likely this runs into similar problems as the document base URL issue for the `importModule` function above, where relative module specifiers become a bug farm and mismatch any nearby `import` declarations.

### A new binding form

At the July 2016 TC39 meeting, in a [discussion](https://github.com/benjamn/reify/blob/master/PROPOSAL.md) of a proposal for [nested `import` declarations](https://github.com/benjamn/reify/blob/master/PROPOSAL.md), the original proposal was rejected, but an alternative of `await import` was proposed as a potential path forward. This would be a new binding form (i.e. a new way of introducing names into the given scope), which would work only inside async functions.

`await import` has not been fully developed, so it is hard to tell to what extent its goals and capabilities overlap with this proposal. However, my impression is that it would be complementary to this proposal; it's a sort of halfway between the static top-level `import` syntax, and the full dynamism enabled by `import()`.

For example, it was explicitly stated at TC39 that the promise created by `await import` is never reified. This creates a simpler programming experience, but the reified promises returned by `import()` allow powerful techniques such as using promise combinators to race different modules or load modules in parallel. This explicit promise creation allows `import()` to be used in non-async-function contexts, whereas (like normal `await` expressions) `await import` would be restricted. It's also unclear whether `await import` would allow arbitrary strings as module specifiers, or would stick with the existing top-level `import` grammar which only allows string literals.

My understanding is that `await import` is for more of a static case, allowing it to be integrated with bundling and tree-shaking tools while still allowing some lazy fetching and evaluation. `import()` can then be used as the lowest-level, most-powerful building block.

## Relation to existing work

So far module work has taken place in three spaces:

- [The JavaScript specification](https://tc39.github.io/ecma262/#sec-modules), which mostly defines the syntax of modules, defering to the host environment via [HostResolveImportedModule](https://tc39.github.io/ecma262/#sec-hostresolveimportedmodule);
- [The HTML Standard](https://html.spec.whatwg.org/multipage/), which defines [`<script type="module">`](https://html.spec.whatwg.org/multipage/scripting.html#the-script-element), how to [fetch modules](https://html.spec.whatwg.org/multipage/webappapis.html#fetching-scripts), and fulfills its duties as a host environment by specifying [HostResolveImportedModule](https://html.spec.whatwg.org/multipage/webappapis.html#hostresolveimportedmodule(referencingmodule,-specifier)) on top of those foundations;
- [The Loader specification](https://whatwg.github.io/loader/), which is a collection of interesting ideas prototyping ways of creating a runtime-configurable loading pipeline and creating modules reflectively (i.e. not from source text)

This proposal would be a small expansion of the existing JavaScript and HTML capabilities, using the same framework of specifying syntactic forms in the JavaScript specification, which delegate to the host environment for their heavy lifting. HTML's infrastructure for fetching and resolving modules would be leveraged to define its side of the story. Similarly, Node.js would supply its own definitions for HostFetchImportedModule and HostResolveImportedModule to make this proposal work there.

The ideas in the Loader specification would largely stay the same, although probably this would either supplant the current `System.loader.import()` proposal or make `System.loader.import()` a lower-level version that is used in specialized circumstances. The Loader specification would continue to work on prototyping more general ideas for pluggable loading pipelines and reflective modules, which over time could be used to generalize HTML and Node's host-specific pipelines.

Concretely, this repository is intended as a TC39 proposal to advance through the stages process, specifying the `import()` syntax and the relevant host environment hooks. It also contains [an outline of proposed changes to the HTML Standard](HTML Integration.md) that would integrate with this proposal.
