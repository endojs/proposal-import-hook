# Import Hook

**Stage**: To be presented for advancement to Stage 1 at a future plenary.

**Champions**:

* Mark S. Miller (MM), Agoric, @erights
* Kris Kowal (KKL), Agoric, @kriskowal
* Caridy Patiño (CPO), Salesforce

**Emeritus**:

* Bradley Farias, GoDaddy
* Jean-Francois Paradis, Salesforce

# Synopsis

Building upon [Source phase imports][proposal-source-phase-imports]
and [ESM phase imports][proposal-esm-phase-imports]
which introduce `ModuleSource` and `import source` for integration with Wasm
and sending code to other workers, this proposal extends the [ModuleSource]
constructor to accept a hook for obtaining a module source for each of the
module's static and dynamic imports.  This allows user code to choose whether
an import is satisfied by the host module system, replaced by a mock, or denied
outright.

> This proposal picks up the remaining pieces of the previous proposal for
> [First-class `Module` and `Module Source`](https://github.com/tc39/proposal-compartments/blob/7e60fdbce66ef2d97370007afeb807192c653333/0-module-and-module-source.md)
> from the [HardenedJS](https://hardenedjs.org) [`Compartment` proposal][proposal-compartments].

# Proposal

We extend the `ModuleSource` constructor to accept an optional handler.

```ts
interface ModuleSource {
  constructor(source: string | ModuleSource, handler?: ModuleHandler);
}
```

The `ModuleSource` constructor eagerly captures the `handler` and the
`importHook` and `importMetaHook` properties of the handler in internal slots.
The implementation will thereafter pass the handler as the receiver
object to any invocation of the `importHook` and `importMetaHook`, so the hooks
may consult other properties of the handler.

> Aside, the handler is necessary for capturing a base specifier for resolving
> a relative import specifier, and allows 262 to avoid unnecssary specificity
> about resolution algorithms.

Because the identity of a `ModuleSource` is a key in a realm's module map for
purposes of denoting a corresponding module instance, we introduce the ability
to construct a `ModuleSource` from the precompiled text and host data of
another module source, but producing a distinct identity for purposes of
multiple instantiation.

```ts
type ModuleHandler = {
  importHook?: ImportHook,
  importMetaHook?: ImportMetaHook,
  // ...
  [name: string | symbol | number]: unknown,
};
```

The `importHook` is a function that returns a promise for another
`ModuleSource` for an import specifier.
When the implementation instantiates a `ModuleSource` in a particular execution
context, the corresponding module instance must obtain a module source for
every static `import` statement and its transitive static imports.
After the module has executed, the implementation consults the `importHook` for
each dynamic `import` called from the emitted module instance.
The import hook receives an import attributes object regardless of whether
the import statement or expression expressed any import attributes.
The import attributes must be presented with keys in shortlex order,
and consequently, for the case that dynamic import provides a JavaScript
object for the import attributes bag, the implementation creates a
normalized copy for the second positional argument to the `importHook`.

```ts
type ImportHook = (
  this: ModuleHandler,
  specifier: ImportSpecifier,
  attributes: Record<string, string>,
) => Promise<ModuleSource>;
```

An import specifier is a string, lifted without alteration from the text of a
module source.

```ts
type ImportSpecifier = string;
```

For a module source that expresses an `import.meta` expression anywhere in its
`source` that is also constructed with an `importMetaHook`, the implementation
must invoke the `importMetaHook` immediately before executing the body of the module
so that the hook has an opportunity to populate it with additional properties.

```ts
type ImportMetaHook = (
  this: ModuleHandler,
  importMeta: ImportMeta,
) => void;

type ImportMeta = {
  __proto__: null,
  [name: string | symbol | number]: unknown,
};
```

> The Module Harmony group elected to avoid deferring the import meta hook
> to the first evaluation of an `import.meta` expression, but leaves the option
> open for a host hook, just not a user code hook.

We rely on the existing import mechanism: a dynamic `import` on a
`ModuleSource` in a particular execution context will get or create a module
instance for that source and drive that through execution and return a promise
for the corresponding module namespace.
In that process, the implementation will call `importHook` for each module
imported statically and dynamically in that module, giving it the import
specifier string and import attributes.
For example, importing a module that statically imports a module `x`,

```js
import 'x';
```

If the module map has no entry indicating that this module source has never
before imported `"x"`, will induce the implementation to call the module
source's import hook, albeit the user code import hook, with the module
specifier `"x"` and an empty import attributes object, expecting it to produce
a promise for the `ModuleSource` of `"x"`.

```js
const source = await importHook('x', {});
```

Thereafter, the implementation will never again consult the `importHook`
for the modules specifier `"x"` with no import attributes.

For another example, importing a module that imports a module `"y"` with
import attributes `type: 'json'` and `extraneous: "very"` will induce
the `importHook` for the identified `ModuleSource` once.

```js
import 'y' with { type: 'json', extraneous: 'very' };
```

The syntax of dynamic import does not itself reify a JavaScript object
for the import attributes.
But, for the purposes of calling an `importHook`, the implementation
must produce a _canonicalized_ object with the keys of the import
attributes in shortlex order.

```js
const source = await importHook('y', { extraneous: 'very', type: 'json' });
```

The implementation also consults the `importHook` for:

- dynamic `import` with or without attributes,
- static `import source`, with or without attributes, and
- dynamic `import.source`, with or without attributes.

Dynamic import receives a reified attributes bag.
To provide consistent behavior, the implementation produces
a canonicalized copy of the given attributes bag for the `importHook`.

```js
await import('z', { type: 'json', extraneous: 'very' });
```

So the implementation then calls the `importHook` with the canonicalized import
attributes.

```js
await importHook('z', { extraneous: 'very', type: 'json' });
```

## Examples

## Mocking

With an `importHook`, a test can isolate a specific module and mock one or more
of its dependencies.
In this example, we import a test fixture module and execute it in an
environment where its `fs` dependency is met by `mock-fs` and all other
imports fall through to the host.

```js
import source fixture from 'fixture.js';

await import(new ModuleSource(fixture, {
  importHook(specifier, attributes) {
    if (specifier === 'fs') {
      return import.source('mock-fs');
    } else {
      return import.source(specifier, attributes);
    }
  }
}));
```

## Mocking Membrane

With a slight complication, the fixture can be imported in an environment where
its transitive dependencies experience the same distortion of `fs` to
`mock-fs`.

```js
import source fixture from 'fixture.js';

const handler = {
  importHook(specifier, attributes) {
    if (specifier === 'fs') {
      return import.source('mock-fs');
    } else {
      const source = await import.source(specifier, attributes);
      return new ModuleSource(source, handler);
    }
  },
}

await import(new ModuleSource(fixture, handler));
```

This example is admittedly reductive for illustration purposes.
The `importHook` is also responsible for resolving module-relative and
package-relative URLs.

## Module system extension

In this example, we create a module source that transitively traps all imports
and loads source for them with `fetch` and treats all import specifiers as
URLs, resolving relative URLs to the location of the containing module source,
and provides `import.meta.url`.

```js
const handlerPrototype = {
  async importHook(importSpecifier) {
    const url = new URL(importSpecifier, this.url).href;
    const response = await fetch(url);
    const text = await response.text();
    return new ModuleSource(text, {
      __proto__: handlerPrototype,
      url,
    });
  },
  importMetaHook(meta) {
    meta.url = this.url;
  },
};
const source = new ModuleSource(`
  import 'example.js';
`, { __proto__, url: import.meta.url });
await import(source);
```

With a slight complication, the hook can accommodate the `type` `json` import
attribute, or arbitrary types and arbitrary other import attributes.

```js
const handlerPrototype = {
  async importHook(importSpecifier, { type }) {
    const url = new URL(importSpecifier, this.url).href;
    const response = await fetch(url);
    if (type === 'json') {
      const json = await response.text();
      // Virtual module sources (to be proposed) would obviate the need to
      // embed the JSON in a JS module source.
      return new ModuleSource(`export default ${json}`);
    } else if (type === undefined) {
      const text = await response.text();
      return new ModuleSource(text, {
        __proto__: handlerPrototype,
        url,
      });
    }
  },
  importMetaHook(meta) {
    meta.url = this.url;
  },
};
const source = new ModuleSource(`
  import 'example.js';
`, { __proto__, url: import.meta.url });
await import(source);
```

# Intersection Semantics

## New Global

This import hooks proposal creates a foundation for a [`new Global`
proposal][proposal-new-global], which in turn provides a sufficient foundation
to implement [`Compartment` proposal][proposal-compartments] and
[HardenedJS](https://hardenedjs.org) entirely in user code, without any
compromise in fidelity.
Within HardenedJS, we create a way to evaluate arbitrary guest code and control
its access to globals and modules, denying the guest access to any host modules
and powerful globals by default.

Consider the following potential guest code:

```js
const indirectEval = eval;
indirectEval('import("node:fs")');
```

Then, consider a host that naively attempts to isolate this code with
just import hooks:

```js
const source = new ModuleSource(`
    const indirectEval = eval;
    indirectEval('import("node:fs")');
`, {
  importHook(specifier) {
    // never reached
  }
});
```

We never reach the module source's `importHook` because indirect `eval`
uses the global execution environment associated with the `eval` function
by an internal slot of the function object itself.
So, we need a `ModuleSource` constructor that is associated with a
`Global` with an `importHook`.
And, consequently, we need each `Global` to have its own `eval`, `Function`,
`AsyncFunction`, `ModuleSource`, and `Global` constructors so code evaluated in
that global environment cannot escape.

Concretely, we propose that this scenario would capture all global and module
access for guest code.

```js
const log = [];
const globalThat = new Global({
  importHook(specifier) {
    log.push(`global ${specifier}`);
    return new ModuleSource('');
  },
});
globalThat.log = log;
const source = new globalThat.ModuleSource(`
  import 'static-import';
  import('dynamic-import');
  eval('import("direct-eval-import")');
  globalThis.eval('import("indirect-eval-import")');
  new Function('return import("function-import")');
  const AsyncFunction = (async () => {}).constructor;
  new AsyncFunction('return import("async-function-import")');
  const GeneratorFunction = (function*() {}).constructor;
  new GeneratorFunction('return import("generator-function-import");')().next();
  const AsyncGeneratorFunction = (async function*() {}).constructor;
  new AsyncGeneratorFunction('return import("async-generator-function-import");')().next();
`, {
  importHook(specifier) {
    log.push(`local import ${specifier}`);
  },
});
await import(source);
// log:
// local static-import
// local dynamic-import
// local direct-eval-import
// global indirect-eval-import
// global function-import
// global async-function-import
// global generator-function-import
// global async-generator-function-import
```

## Content Security Policy

This proposal has no intersection semantics with Content Security Policy but
rests on the earlier introduction of module sources and their execution model,
which do.

The execution of a module source that was constructed by the host may
capture sufficient origin information that it is safe to execute without `unsafe-eval`
in a Content Security Policy.
A module source that was parsed from source text, where the text is not a
[Trusted Type][trusted-types], will not have any attached origin information and would be unable
to execute without an `unsafe-eval` Content Security Policy.

## Synchronous import, importNow

We expect a future proposal to introduce `import.now` or `import.sync` as a
mechanism for synchronously growing the module graph and executing a module.
If so, `ModuleSource` handlers would need a corresponding synchronous import
hook that must not return a promise.

# Design Rationales

### Should `importHook` be synchronous or asynchronous?

When a source module imports from a module specifier, you might not have the
source at hand to create the corresponding `Module` to be returned. If
`importHook` is synchronous, then you must have the source ready when the
`importHook` is invoked for each dependency.

Since the `importHook` is only triggered via the kicker (`import(instance)`),
going async there has no implications whatsoever.
In prior iterations of this, the user was responsible for looping thru the
dependencies, and preparing the instance before kicking the next phase.
That's no longer the case here, where the level of control on the different
phases is limited to the invocation of the `importHook`.

### Can cycles be represented?

Yes, `importHook` can return a `Module` that was either `import()`ed already or
was returned by an `importHook` already.

The `import.source` behavior introduced in [ESM source phase
imports](esm-phase-imports) lets an `importHook` defer to its parent module
loader to link an existing module or forward-reference to a module.
But, as yet, only ESM can link cycles.

### Idempotency of dynamic imports in ModuleSource

Any `import()` statement inside a module source will result of a possible
`importHook` invocation on the `Module`, and the decision on whether or not to
call the `importHook` depends on whether or not the `Module` has already
invoked it for the `specifier` in question. So, a `Module`
most keep a map for every `specifier` and its corresponding `Module` to
guarantee the idempotency of those static and dynamic import statements.

User-defined and host-defined import-hooks will likely enforce stronger
consistency between import behavior across module instances, but module
instances enforce local consistency and some consistency in aggregate by
induction of other modules.

### toString

Whether `ModduleSource` instances retain the original source may vary by host
and modules should reuse
[HostHasSourceTextAvailable](https://tc39.es/ecma262/#sec-hosthassourcetextavailable)
in deciding whether to reveal their source, as they might with a `toString`
method.
The [module block][module-blocks] proposal may necessitate the retention of
text on certain hosts so that hosts can transmit use sources in their serial
representation of a module, as could be an extension to `structuredClone`.

### Referrer Specifier

The host-defined or virtual-host-defined import hook for a module is
responsible for taking "import specifiers", the string expressed in both static
and dynamic import statements, and resolving them relative to the logical or
physical specifier for the module.
Although it is possible to construct a host or virtual-host that only supports
absolute or fully-qualified specifiers, nearly every module will have a
referrer of one kind or another.

A module will typically have three distinct but often coincident data:

- This `referrer` concept, the basis for resolving import specifiers.
- The `import.meta.url`, the basis for locating resources adjacent to modules
  hosted on the web using URL math.
  The `import.meta` object should not have a `url` property unless it
  is useful for that purpose, and as such should not exist for modules
  located in bundles or archives, and should not be used in modules that
  will be bundled or archived, but will often be identical to the `referrer`
  when it is present.
- The origin of the source, the basis for a host deciding whether to allow the
  module to be evaluated when a Content-Security-Policy is in place.
  The origin will often be identical to the origin of `import.meta.url`.
  The origin will exist in host data of the module source and must
  not be virtualizable, lest a virtual host be able to escape a same origin
  policy.

So, although these values are often related, they must be independent features.
Virtual host implementations cannot necessarily rely on the `import.meta.url`
to exist and virtual host implementations must be able to virtualize the
referrer.

The authors of this proposal vacilated between versions that made the referrer
explicit or implicit.
All of the authors shared a goal to make the `Module` and `ModuleSource`
features incrementally explainable.
Since it is possible to construct a trivial module system without the concept
of a referrer, a graduated tutorial introducing this feature (like this
explainer!) should be able to defer or formally ignore referral until it
becomes necessary to explain relative module specifier resolution.

To that end, we agreed that the referrer should be optional and appear later
in function signatures than other more essential parameters, if at all.
We relegated the `referrer` to at least be a property of the `Module`
constructor's options bag.
We then realized that the options bag was more analogous to the `Proxy` handler
object, such that the virtual host could choose to extend the handler with any
property of any type to carry the referrer, such that an import hook might
resolve a relative specifier using `this.referrer` (a `string` with the
module's own specifier), `this.slot` (a `number` representing the index into an
array of pre-computed resolutions for a bundle), or whatever other protocol the
virtual host chose to implement.

This was only possible because host-defined resolution behavior and
virtual-host-defined resolution behavior do not need to communicate.
Host-defined modules resolve import specifiers based on information in the
referring module record's host data internal slot.
At no point does a host-defined module need to consult the referrer of a
virtual-host-defined module.

We additionally elected not to repeat what we believe to be a mistake in the
design of `Proxy`.
The `Module` constructor eagerly captures all the optional methods of the
module handler rather than accessing them afresh for each invocation.
So, we sacrifice some dynamism but still thread the handler as the target
object for all method invocations.

A consequence of this design is that any `Module` instance will retain a
`handler`, where an options bag would be immediately released to the GC
nursery.
And, if the handler carries a `referrer`, every `Module` instance needs a
unique handler instance, even though many module instances can share
the same hooks.

### Just one `ModuleSource`

Earlier proposals introduced a separate `Module` or `ModuleInstance` constructor.
This would have made clear that the identity of a `ModuleSource` is meaningless
so that, if you wanted multiple instances of a module namespace from the same
module source, you could do so without reparsing or recompiling the source.

```js
const source = new ModuleSource('export default {}');
const a = new Module(source);
const b = new Module(source);
(await import(a)) !== (await import(b));
```

However, [ESM Phase Imports][proposal-esm-phase-imports] made the identity of a
`ModuleSource` significant, without binding the source to a particular module
map (albeit a Realm or Realm of another Worker).
So, module sources are transferrable and can be imported in multiple realms,
but only singly-instaniated within one realm.

In practice, there is little to no motivation for multiple instances for a
single parsed source in a single realm.
So, we lose very little by consolidating `Module` and `ModuleSource`
in `ModuleSource`.
The `handler` becomes a non-transferrable component of the source that
overrides parent module loader behavior if it is imported into any realm of the
same Agent.

And, in turn, we have one less concept to litigate and one less name to choose
for a fresh global constructor.

### Using dynamic `import` to trigger import behavior

The [ESM Phase Imports][proposal-esm-phase-imports] proposal settles the
issue of how to trigger import behavior.
But, for sake of completeness, an alternate design we discarded would
have put an `import` method on a `Module` instance.
This made sense when a `Module` execution was bound to the realm that
provided the `Module` intrinsic.
However, we elected to make `ModuleSource` identity significant
for the purposes of pass-invariant identity of `import` for a module source
through `postMessage` to another `Worker` in the HTML semantics, or
possibly to another realm of the same agent.
This means that a `ModuleSource` does not intrinsically identfy the module map
it will be used to populate.
The execution context where you use a dynamic `import` indicates the module map
the module source will populate.

### Handler versus options bag

We regret the dynamism of `Proxy` handlers, but also definitely need a
"handler" and not a mere, disposalbe options bag, because we need to use the
handler as a receiver for all subsequent calls to hooks.
We chose a way that has no precedent in the language.
Eagerly capturing all the hooks *and* the handler in `ModuleSource`
internal slots slots makes the hooks invariant over the lifetime
of the `ModuleSource` but also allows us to carry some state, like a referrer
module specifier, as state of the handler accessible like `this.url`.
The handler also allows 262 to evolve without concerning itself with
implementation- or host-specific module resolution algorithms.

[proposal-source-phase-imports]: https://github.com/tc39/proposal-source-phase-imports
[proposal-esm-phase-imports]: https://github.com/tc39/proposal-esm-phase-imports
[proposal-compartments]: https://github.com/tc39/proposal-compartments
[proposal-new-global]: https://github.com/endojs/proposal-new-global
[trusted-types]: https://w3c.github.io/webappsec-trusted-types/dist/spec/
