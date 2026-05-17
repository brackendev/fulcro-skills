---
name: fulcro-lenses
description: >-
  Translate code-lenses design philosophies into Fulcro-specific patterns.
  Auto-triggers when working on Fulcro code (com.fulcrologic.fulcro.*,
  com.fulcrologic.rad.*, com.wsscode.pathom3.*, defsc, defmutation, defrouter,
  df/load!, ident vectors) alongside the code-lenses plugin. Layers on top of
  clojure-lenses (in clojure-skills) and clojurescript-lenses (in
  clojurescript-skills); records only the Fulcro-specific deltas that come from
  the normalized client database, ident hygiene, mutations as data, EQL queries,
  the action / remote / ok-action / error-action split, df/load! over hand-rolled
  merges, dynamic routing, form-state algorithms, UI state machines, the Pathom
  3 server, the Fulcro Inspect workflow, and the framework's strong stance on
  pure render functions. Covers the four default code-lenses philosophies (grug,
  Honest Code, Tidy First, Parse Don't Validate) and the two opt-in philosophies
  (APOSD, Legacy Code) that activate when their lens is added with `+aposd` or
  `+legacy-code` or invoked directly.
user-invocable: false
---

# Code Lenses for Fulcro

Layered on top of [clojure-lenses](https://github.com/brackendev/clojure-skills) (in `clojure-skills`) for host-neutral translations and [clojurescript-lenses](https://github.com/brackendev/clojurescript-skills) (in `clojurescript-skills`) for the JavaScript-host deltas. This skill records only the Fulcro-specific additions: the normalized client database, ident composition, mutations as data, EQL queries, the framework's `action` / `remote` / `ok-action` / `error-action` split, `df/load!` over hand-rolled merges, dynamic routing, form-state algorithms, UI state machines, the Pathom 3 server, and Fulcro Inspect.

The default code-lenses review set is `grug`, `honest-code`, `tidy-first`, and `parse-dont-validate`. `aposd` and `legacy-code` are opt-in (added with `+aposd` or `+legacy-code`, or invoked directly). The sections below apply when the corresponding lens is active; APOSD and Legacy Code translations are present so the lens can use them when explicitly invoked, but they are not part of the default trigger set.

When the [code-lenses](https://github.com/brackendev/code-lenses) plugin is active in a Fulcro project, apply the baseline `clojure-lenses` and `clojurescript-lenses` translations first; reach for this skill for the Fulcro-specific additions below.

## Grug Brain

- One ident shape per entity type. Pick `:person/id` once and use it everywhere — query, ident, initial-state, mutations. Mixing `:person/id` with `:user/id` for the same conceptual entity is the kind of complexity that bites back six months later.
- `defsc` is a flat building block. Co-locate `:query`, `:ident`, `:initial-state`, and the render body. Splitting these across files or wrapping `defsc` in a custom macro is the complexity demon's invitation.
- Prefer `df/load!` over hand-rolled merges into the state atom. `df/load!` is the simple thing that handles normalization, post-mutations, and loading markers in one call.
- One mutation per logical operation. A single `(defmutation login ...)` with `action` + `remote` + `ok-action` is simpler than a chain of `swap!` calls in a button handler.
- Reach for UI State Machines only when the flow has explicit phases. A two-state "loading / loaded" condition does not need a UISM; a login flow with `:initial`, `:authenticating`, `:authenticated`, `:failed`, `:locked` does.
- Pathom 3 resolvers are tiny functions. One resolver per attribute or per logical join. Resist the urge to build a "router resolver" that dispatches over keys; the index already does that.
- Fulcro Inspect is a free debugger. Reach for it before reaching for `println` debugging. The DB tab makes ident hygiene problems obvious in a glance.

## A Philosophy of Software Design

- The Fulcro client app is a deep module. It hides the normalization, transaction log, remote pipeline, and re-render machinery behind a small surface: `defsc`, `defmutation`, `df/load!`, `comp/transact!`. Application code should not reach into the internals.
- A feature is a deep module spanning client and server. Group the client component, its mutations, its load helpers, the Pathom resolver, and the schema fragment under one namespace prefix (`app.feature.profile.ui`, `app.feature.profile.mutations`, `app.feature.profile.resolvers`). The feature's public API is the component factory and the public mutation symbols; everything else is private.
- The ident is information hiding. The component's identity shape (`[:person/id v]`) belongs inside the namespace. Callers receive a component factory; they do not construct idents by hand.
- The Pathom resolver indexes hide path-finding from callers. A consumer asks for `[:person/id :person/full-name {:person/posts [:post/title]}]`; the index figures out which resolvers to invoke. Resist the urge to build wrappers that bypass the index.
- Errors at the right layer. Network errors belong in `error-action` (or a UISM transition). Schema errors belong in Pathom resolvers (return `nil` or throw `ex-info` with structured data). Render-layer errors are bugs, not failure modes.
- Define schemas at boundaries. Malli specs on the Pathom output edge and Fulcro form-state validation on the input edge form the parse layer. Inside the app, components receive parsed data and do not re-validate.

## Tidy First

- Extract a helper component before adding a fifth query join. A `defsc` with `:query [a b c {:d ...} {:e ...} {:f ...} {:g ...}]` is doing too much. Split into a child component and compose the queries.
- Extract a targeting helper before its third use. `(targeting/replace-at [:component/id ::Page :selected])` repeated three times becomes `(def selection-target [:component/id ::Page :selected])`.
- Read order in a Fulcro file: namespace, requires, schema, mutations, helpers, component (with `:query`, `:ident`, `:initial-state`, then render), factory. Files that interleave components and mutations make the dependency graph harder to read.
- Guard clauses in mutation `action` bodies: bail early on missing state, idents, or env keys before mutating. `(when-let [ident (get env :ref)] ...)` reads better than a deeply nested `if-let`.
- Threading mutations through `swap!`: prefer one `swap!` per `action` body that calls a pure `update-fn` rather than several small `swap!` calls. The state atom is updated atomically per `swap!` invocation; multiple `swap!`s leave intermediate states observable.
- Tidying a route: if the same `(dr/change-route! this ...)` appears in three places, lift it into a named function that takes the parameters and add the route segment.
- Reading order across namespaces: `app.ui.*` reads `app.model.*` which reads `app.server.*` only through Pathom remotes. A component that imports from `app.server.*` directly is a layering violation; tidy by routing through a resolver and `df/load!`.

## Parse, Don't Validate

- The Pathom resolver is the parse layer for server responses. A resolver returns `{:person/id 42 :person/name "Alice"}` — already parsed Clojure data, ready to merge into the normalized DB. Validation against a schema (Malli on the resolver output) belongs here, not in the UI.
- The `:pre-merge` hook on a component is a parse hook for the client. Use it when the server response shape needs transformation before normalization: filling in defaults, computing derived fields, splitting a flat payload into nested entities.
- Form-state algorithms parse user input into a validated entity. `(fs/validity profile)` and `(fs/form-field-valid? attr value)` are the parse predicates; a `:save` mutation that fires only when `(fs/valid? profile)` is the smart constructor.
- Closed maps via Malli's `:closed true` reject unexpected keys at the boundary. Apply at the resolver edge (rejecting malformed client mutations) and at the form edge (rejecting unexpected form payloads).
- Tagged maps for sum types in UISM: a session is `{:session/status :awaiting-credentials}` or `{:session/status :authenticated :session/user user}`. The status tag is the discriminator.
- Idents are parsed identities. The ident `[:person/id 42]` is the parsed form of "this entity in the people table"; the components consume idents, not raw entity values.
- Mutations declared with `m/declare-mutation` plus a Malli spec on the parameter map produce a parsed mutation contract. Calls with malformed parameters fail at the transaction boundary, not in the `action` body.

## Honest Code

- Mutations are data. `(login {:email "alice@example.com"})` returns an EQL data structure; `comp/transact!` interprets it. The data form is honest; hidden side effects in random `swap!` calls are dishonest.
- Components render pure functions of props. The render body should not perform side effects (no `js/fetch`, no `localStorage`, no atom swaps). Side effects belong in mutation `action` / `ok-action` bodies, in `componentDidMount`, or in `:use-hooks?` effects.
- The normalized client database is the single source of truth. State that lives outside the DB (component-local state, top-level `defonce` atoms) is dishonest because the framework cannot reason about it, Fulcro Inspect cannot show it, and `with-redefs` cannot stub it.
- Pathom resolvers declare what they need (`::pco/input`) and what they produce (`::pco/output`). The declaration is the honest contract; the body honors it. Resolvers that quietly read or write data outside their declared boundary are dishonest.
- `df/load!` is honest data flow. The call declares "this data is wanted at this path"; the framework fetches, normalizes, merges, and notifies. Hand-rolled merges into the state atom (`swap! state assoc-in [:person/id 42] new-person`) bypass normalization and break the honest data flow.
- `defmutation` sections are honest about timing. `action` runs synchronously, `remote` describes wire intent, `ok-action` runs after a successful response, `error-action` runs after a failure. Code that does network work inside `action` is dishonest about when it runs.
- Let It Crash, Fulcro edition: surface errors through `error-action` or a UISM `:event/failure` transition; do not swallow them silently. Network errors should fire `ok-action` with a documented `:status :failed` payload only when the server intentionally returns failure as data.
- The Fulcro client app is a single value at any moment. The state atom dereferences to a plain Clojure map; Fulcro Inspect's DB tab is that value. Test by querying that map, not by mocking framework internals.

## Legacy Code

- Seams in Fulcro: the state atom (mutate it via `defmutation` to replace fixtures), the remote map (`(app/fulcro-app {:remotes {...}})` accepts substitutes), Pathom resolvers (replace `(pci/register ...)` in tests), and the UISM event channel (stub a state machine for testing UI flows).
- Extract Interface translates to extracting a Pathom resolver. A component that hits the server directly through a `js/fetch` call becomes a component that requires data via a `df/load!`; the resolver implements the contract.
- Parameterize Constructor: pass the Fulcro app to functions that need it. Code that reaches for a top-level `(defonce app ...)` cannot be tested without that singleton; code that takes `app` as a parameter substitutes a test app.
- Sprout method translates to a new mutation or resolver. New business logic goes in a new `defmutation` called from a new component; legacy mutations stay frozen.
- Wrap method translates to wrapping a legacy component in a `defsc` shell that adapts the props shape. Useful when migrating from `defui` (Fulcro 2.x) to `defsc` (Fulcro 3.x) one component at a time.
- Scratch refactoring maps to Fulcro Inspect plus the REPL. Trigger a mutation from the Inspect transactions tab, inspect the DB tab afterward, refine. Source-mapped stack traces from CLJS land back in Fulcro source.
- Effect sketching: a change to an ident shape ripples through every query, every initial-state, every mutation that constructs that ident, and every Pathom resolver that returns the matching key. Use Fulcro Inspect's DB tab to confirm the migration is complete (no entities left in the old table) before deleting the legacy code.
- Hard dependencies in Fulcro projects: top-level `defonce` Fulcro app, ad-hoc `swap!` on the state atom from button handlers, Pathom resolvers that reach into top-level dynamic vars, UISM machines that hard-code their actor idents.
- Characterization tests via `fulcro-spec`: build a known DB shape, dispatch a mutation, assert the DB shape after. The "before" and "after" snapshots are the characterization. For Pathom, run a fixed EQL query against a fixture environment and assert the result.
- Union routers and `defui` components are legacy. Both still compile. When touching one, plan a migration to `defsc` and dynamic routers; do not extend the legacy form.

## Gotchas

- A `defmutation` rebuilt as a function (`(defn login [params] ...)`) loses the framework integration: the transaction log, the remote pipeline, Fulcro Inspect visibility, and the `ok-action` / `error-action` callbacks. "Tidy first" should never collapse a mutation into a plain function.
- "Extract helper component" must keep `:query`, `:ident`, `:initial-state`, and the factory together. Pulling the render body into a helper that takes a Fulcro component as an argument and then composing queries by hand is more complexity than the duplication it removed.
- `comp/computed` is the seam for passing parent-derived data and callbacks. Passing them as inline literals on every render is functionally equivalent on day one but causes re-render churn that is hard to debug later.
- Pathom 2 (`com.wsscode.pathom.connect`) and Pathom 3 (`com.wsscode.pathom3.connect.operation`) look similar but the API differs. Apply lens translations to the version actually in use; do not blend examples from the two.
- UI State Machines look like a tidying option, but they have setup cost. For a one-shot toggle, a `swap!` in a mutation is the simpler thing. Reach for UISM when there are at least three states and at least two events with non-trivial transitions.
- `with-redefs` on Fulcro internals is fragile. The framework reads its hooks at app construction time; redefining a hook after `(app/fulcro-app ...)` has run does not retroactively change the app. Pass alternate implementations into the constructor instead.
- Fulcro RAD generates components and forms from attribute models. "Extract helper" inside a RAD-generated component is usually wrong — the generator will overwrite the change on the next regeneration. Customize through RAD's hook points or eject the component from the generator.
