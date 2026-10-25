---
name: fulcro
description: >-
  Use when writing, editing, reviewing, or discussing Fulcro full-stack
  applications. Triggers: .cljs / .cljc files containing
  com.fulcrologic.fulcro.*, com.fulcrologic.fulcro-css.*,
  com.fulcrologic.rad.*, com.wsscode.pathom3.* (or legacy
  com.wsscode.pathom.*); the macros defsc, defmutation, defrouter, defresolver;
  the helpers df/load!, df/load-field!, dr/change-route!, comp/get-query,
  comp/get-ident, comp/get-initial-state, comp/computed, comp/factory, fs/*,
  fo/*, uism/*; ident vectors `[:table/id value]` and `[:component/id
  ::ComponentName]`; shadow-cljs.edn with a :main build that requires Fulcro;
  deps.edn containing com.fulcrologic/fulcro, com.fulcrologic/fulcro-rad,
  com.fulcrologic/guardrails, or com.wsscode/pathom3; transit endpoints `/api`;
  Fulcro Inspect setup (com.fulcrologic/fulcro-inspect,
  com.fulcrologic.devtools.chrome-preload, fulcro.inspect.tool/add-fulcro-inspect!,
  com.fulcrologic.fulcro.inspect.dom-picker-preload, or the legacy
  com.fulcrologic.fulcro.inspect.preload); or any mention of Fulcro,
  fulcrologic, Fulcro Inspect, or Fulcro RAD. Covers defsc component
  definition, idents and the normalized client database, EQL queries,
  mutations and their action / remote / ok-action / error-action sections,
  df/load! and load targeting, dynamic routing, forms, UI state machines, the
  Pathom 3 server resolver shape, and the Fulcro Inspect workflow.
user-invocable: false
---

# Fulcro

Layered on top of three baselines:

- `clojure` (in [clojure-skills](https://github.com/brackendev/clojure-skills)) for host-neutral style, threading, collections, atoms, dispatch, formatting, namespaces, REPL conventions.
- `clojurescript` (in [clojurescript-skills](https://github.com/brackendev/clojurescript-skills)) for JavaScript interop, externs inference, macro stage separation, `catch :default`, JS-flavored numbers and truthiness, and the `cljs.main` / `shadow-cljs` workflow surface.
- `clojure-jvm` (in [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills)) for Java interop on the server, JVM-typed exceptions, `with-open`, refs / agents / STM, `alter-var-root`, and the Clojure CLI / `tools.build` / `clj-kondo` / `cljfmt` / `test-runner` / nREPL workflow.

This skill applies when the code uses Fulcro on the client (`com.fulcrologic.fulcro.*`) or Pathom on the server (`com.wsscode.pathom3.*`). It owns the framework-specific deltas: component definition, query / ident / initial-state composition, the normalized client database, mutations, loading, dynamic routing, forms, UI state machines, the Pathom 3 resolver shape, the Fulcro Inspect workflow, and the typical project shape (`shadow-cljs.edn` + `deps.edn` + `dev/user.clj`).

For the deepest framework reference, [The Fulcro Developer's Guide](https://book.fulcrologic.com/) is the canonical source. Cross-check API specifics against the [Fulcro source](https://github.com/fulcrologic/fulcro) and its commit history between release tags (`fulcro-3.9.0`, `fulcro-3.10.0`, and so on). The upstream `CHANGELOG.md` stops at 3.7.2. The framework evolves quickly enough that older blog posts often describe deprecated forms.

This skill reflects Fulcro 3.10, which requires React 18 or later and is tested against React 19.

## Key Rules

1. **Define components with `defsc`.** Co-locate `:query`, `:ident`, `:initial-state`, and the render body. Do not split these across multiple forms; they describe one component. Fulcro 3 has no `defui` macro, so rewrite Fulcro 2.x `defui` components with `defsc`.
2. **Idents are two-tuples `[:table/id value]`.** Use the same `:table/id` keyword for every entity of that type, across query, ident, and initial-state. Singletons use `[:component/id ::ComponentName]`. Ident hygiene is the most common source of silent denormalization.
3. **Compose child queries and child initial-state through the framework helpers.** `(comp/get-query ChildComponent)` and `(comp/get-initial-state ChildComponent params)` rather than inlining either by hand. This is what lets refactors of a child propagate without breaking parent composition.
4. **Mutate state only through `defmutation`.** Never `swap!` the state atom from UI code. Place optimistic state in `action`, server work in `remote`, response handling in `ok-action` / `error-action` / `result-action`.
5. **Pass parent-derived data and callbacks through `(comp/computed props {...})`, and keep the callbacks stable.** Fulcro keeps computed props when a targeted update refreshes a child that has a query and an ident from the database. A callback passed in ordinary props is lost on that update. A closure built inline in render, such as `(ui-person (comp/computed person {:on-click (fn [] (delete! this))}))`, is a new function on every parent render, so the child's props never compare equal and the child re-renders every time. Lift the callback into a top-level function that receives the component.
6. **Load data with `df/load!` and `df/load-field!`.** Use `:target`, `:post-mutation`, and `:marker` rather than wrapping `load!` in custom helpers or hand-merging server responses into the state atom.
7. **Route with dynamic routers.** `defrouter` + `dr/change-route!` from `com.fulcrologic.fulcro.routing.dynamic-routing`. Legacy union routers (the `defrouter` form from the Fulcro 2.x era, available in Fulcro 3 as `defsc-router` in `com.fulcrologic.fulcro.routing.legacy-ui-routers`) are legacy. Upstream still supports them but advises against them in new applications, so do not reach for them in new code.
8. **Manage form state through `fs/add-form-config*` and `fs/*` helpers.** Track dirty / pristine, validation, and submission with the form-state algorithms rather than ad-hoc component-local state.
9. **Pathom 3 resolvers live in `com.wsscode.pathom3.*` and use `(pco/defresolver ...)`.** Mirror client mutation symbols on the server so EQL flows symmetrically through the API.
10. **Keep transit-serializable values on the wire.** Mutation parameters, load parameters, and EQL responses hold only plain Clojure data (maps, vectors, sets, lists, keywords, strings, numbers, booleans, nil, UUIDs, dates). Atoms, channels, refs, functions, and most deftypes throw or silently drop fields when encoded to transit. Pathom returns Clojure data, and Fulcro merges it into the normalized DB.
11. **Reach for UI State Machines (`com.fulcrologic.fulcro.ui-state-machines`) when a flow needs explicit states.** Login flows, multi-step forms, network retry loops, and complex routing transitions belong in a UISM rather than scattered across mutations.

## References

The Key Rules are the short form. Each reference file below holds the detailed guidance and the examples for one topic. Before writing, reviewing, or refactoring Fulcro code, read every reference whose topic the task touches, and read the next one when the work moves to a new topic.

| When the task involves | Read |
|------------------------|------|
| Writing or changing a `defsc` component, its query, ident, or initial state, normalization, or EQL queries | `references/components.md` |
| Mutations and their `action`, `remote`, `ok-action`, and `error-action` sections, or loading data with `df/load!` and load targeting | `references/mutations-and-loading.md` |
| Dynamic routing with `defrouter` and `dr/change-route!` | `references/routing.md` |
| Forms with `fs/*` form state or Fulcro RAD form options | `references/forms.md` |
| UI state machines (`uism/*`) | `references/state-machines.md` |
| Pathom 3 resolvers and the server parser | `references/pathom-server.md` |
| Fulcro Inspect setup or debugging with Inspect | `references/inspect.md` |
| Project layout, `deps.edn` and `shadow-cljs.edn`, builds, the server, or the development workflow | `references/project-workflows.md` |

## Project Workflow

The Fulcro project shape (`deps.edn` aliases, `shadow-cljs.edn` builds, `dev/user.clj`, Pathom 3 server wiring, the `fulcro-template` scaffold), the test stack (`fulcro-spec`, `kaocha`, `shadow-cljs` test build), the `guardrails` toggle, and an ecosystem map (`fulcro-rad`, `fulcro-spec`, `fulcro-i18n`, `fulcro-incubator`, `statecharts`, `guardrails`) live in `references/project-workflows.md`. Load it on demand.

## Gotchas

### Missing `:initial-state` keeps a component out of the DB

A component without `:initial-state` (or that omits itself from a parent's `:initial-state` composition) never appears in the normalized DB at boot. The component renders as if its props are `nil`. Add `(comp/get-initial-state ChildComponent params)` to the parent's `:initial-state`, or load the data with `df/load!` before navigating.

### Over-querying causes wasted re-renders

A component that queries data it does not render re-renders whenever that data changes. Either prune the query to what render reads, or split the component so the parent passes a smaller props slice.

### Fulcro 3 hooks components are real React function components

Setting `:use-hooks? true` produces a function component. Standard React hook rules apply (no hooks in conditionals, no hooks in loops). The trade-off is fewer Fulcro lifecycle hooks; reach for hooks components when you genuinely want React hooks behavior, not as a default.

### Fulcro 3.9 removed the pre-React-18 renderers

Fulcro 3.9 requires React 18 or later and removed rendering code that could not work with React's asynchronous rendering. The removed pieces are the `com.fulcrologic.fulcro.rendering.multiple-roots-renderer`, `ident-optimized-render`, and `keyframe-render2` namespaces, and the `hooks/use-fulcro-mount` function. An app that sets `:optimized-render!` to one of those renderers, or mounts components with `use-fulcro-mount`, fails to compile after the upgrade. Remove the `:optimized-render!` override so the default keyframe renderer applies, and replace floating roots with `hooks/use-component` or `hooks/use-root`. The `react.version18` helpers (`with-react18`, `react18-options`) are now deprecated no-ops and can be deleted.

### Release builds keep Fulcro's log calls and their file paths

Fulcro logs through Timbre, and each Timbre log call records the file it comes from. A `shadow-cljs release` build therefore keeps every Fulcro log call together with the absolute path of the Fulcro jar on the build machine (`jar:file:/<home>/.m2/repository/...`), which publishes the builder's home directory and enlarges the bundle. Timbre removes the calls below a minimum level at compile time when the compiling JVM sees `TAOENSSO_TIMBRE_MIN_LEVEL_EDN`. The compiler cache and a running shadow-cljs server both defeat the setting, so follow the steps under "Removing Fulcro's log calls from the release build" in `references/project-workflows.md`.

### `with-redefs` is unreliable across the wire

Stubbing a server-side resolver from a client-side test does not work; they run in different processes. Reach for `fulcro-spec`'s component stubs on the client, and for ordinary `with-redefs` (JVM) or `with-redefs` + reload (CLJS) only within a single process.
