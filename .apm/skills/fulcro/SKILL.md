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

## Components (`defsc`)

`defsc` is the canonical component macro from `com.fulcrologic.fulcro.components`. The conventional alias is `comp`. The shape every component shares:

```clojure
(ns app.ui.person
  (:require
   [com.fulcrologic.fulcro.components :as comp :refer [defsc]]
   [com.fulcrologic.fulcro.dom :as dom]))

(defsc Person [this {:person/keys [id name email]}]
  {:query         [:person/id :person/name :person/email]
   :ident         (fn [] [:person/id id])
   :initial-state (fn [{:keys [id name email]}]
                    {:person/id    id
                     :person/name  name
                     :person/email email})}
  (dom/div
    (dom/p name)
    (dom/p email)))
```

Equivalent template forms are accepted for `:ident` (`:ident :person/id`), `:initial-state` (a literal map with `:param/keyword` placeholders), and `:query`. Use the lambda form whenever the value depends on the props or a child query.

The `:ident` lambda takes no arguments. It sees `this` and the names bound by the argument list, so `id` above comes from the `{:person/keys [id ...]}` destructuring. The `:query` lambda sees only `this`. Writing `props` inside the lambda only works when the argument list binds `props`, for example `[this {:person/keys [id] :as props}]`. The `:initial-state` lambda sees only its own `params` argument.

A docstring goes between the component name and the argument list: `(defsc Person "A person row." [this props] {...} ...)`. In `(defsc Person [this props] "A person." {:query [...]} ...)`, the string and the options map are body forms. Fulcro 3.10 and later reject that shape at compile time with "Docstring is in the wrong position". Earlier versions compile the component with no query, ident, or initial state, and it fails at runtime in confusing ways, so check for a misplaced docstring whenever a component's props are unexpectedly empty.

| Option | Purpose |
|--------|---------|
| `:query` | EQL the component requires |
| `:ident` | Two-tuple table identity, or `(fn [] ...)` for a custom shape |
| `:initial-state` | Seed data, composable across the parent tree |
| `:pre-merge` | Transform a server response before normalization |
| `:use-hooks?` | `true` renders a function component that can use React hooks. `:pure` also memoizes it with React `memo`. Switching to or from `:pure` requires a browser reload. |
| `:componentDidMount` / `:componentWillUnmount` | Lifecycle hooks on the class form |
| `:initLocalState` | Component-local mutable state (rare; prefer the normalized DB) |

The factory wraps a component for rendering. Build it once per namespace:

```clojure
(def ui-person (comp/factory Person {:keyfn :person/id}))

;; in a parent render
(map ui-person (sort-by :person/name people))
```

`:keyfn` returns React's `key` for each rendered instance. Since Fulcro 3.9, a factory for a component that has an `:ident` defaults the key to the ident's second element as a string, so `{:keyfn :person/id}` above is redundant but harmless. Precedence runs from a `:react-key` entry in the props, to an explicit `:keyfn`, to the ident default. Components without an ident still need `:keyfn` when rendered in a list, or React logs a missing-key warning and re-renders more than necessary.

## Idents and Normalization

The normalized client database is a Clojure map shaped by ident:

```clojure
{:person/id {1 {:person/id 1 :person/name "Alice" :person/email "alice@example.com"}
             2 {:person/id 2 :person/name "Bob"   :person/email "bob@example.com"}}
 :ui/root   {:person/list [[:person/id 1] [:person/id 2]]}}
```

`[:person/id 1]` is an ident. Edges in the graph are idents; nodes live under their table keyword. Fulcro normalizes server responses into this shape automatically as long as each entity has an `:ident` declaration consistent with its query.

Hygiene rules:

- **One table keyword per entity type.** Pick `:person/id` once and use it in every query, ident, initial-state, and mutation for that entity. Mixing `:person/id` and `:user/id` for the same logical thing, or taking the ident value from anywhere other than the props (next rule), produces two parallel tables that drift out of sync. The symptoms are a list that shows stale rows after an update and a Fulcro Inspect DB tab that shows two tables with overlapping keys.
- **The ident's value comes from the props.** `(fn [] [:person/id (:person/id props)])` — never a literal from outside the component, never a value pulled from a parent's bindings.
- **Singletons use `[:component/id ::ComponentName]`.** Reserve this shape for top-level pages, sidebars, and other one-of-a-kind components. The auto-generated keyword is namespaced to the component so it cannot collide.
- **Idents are stable.** Use the entity's persistent identifier (typically a UUID or database id), not a transient value like an index or a position.

Use `(comp/get-ident ComponentName props)` rather than hand-constructing an ident when you need one in a mutation or load. Refactoring a component's ident shape then propagates without searching for literal vectors.

## Queries (EQL)

EQL is the query language: Clojure data shaped as a vector with possibly nested maps.

```clojure
;; simple property
[:person/id :person/name]

;; join (nested component query)
[:person/id :person/name {:person/address (comp/get-query Address)}]

;; parameterized join
[(list {:person/friends (comp/get-query Person)} {:limit 10})]

;; link query: read from a top-level table
[[:current-user '_]]

;; recursive query (e.g., tree of comments)
[:comment/id :comment/body {:comment/replies '...}]
```

Compose parent queries from child queries:

```clojure
(defsc PeoplePage [this {:keys [people]}]
  {:query         [{:people (comp/get-query Person)}]
   :ident         (fn [] [:component/id ::PeoplePage])
   :initial-state (fn [_] {:people [(comp/get-initial-state Person {:id 1 :name "Alice"})]})}
  (dom/div
    (map ui-person people)))
```

The compiler does not validate that the query's shape matches what the server (or initial-state) produces. Adding a field to `:query` without updating `:initial-state` or the Pathom resolver produces a missing-data render, not a compile error or an exception. Cross-check the three shapes (query, initial-state, server response) after every component refactor and whenever something fails to display. Fulcro Inspect's DB tab is the fastest way to confirm that the normalized data matches.

## Mutations

`defmutation` from `com.fulcrologic.fulcro.mutations` (alias `m`). The full shape:

```clojure
(ns app.model.session
  (:require
   [com.fulcrologic.fulcro.mutations :as m :refer [defmutation]]
   [com.fulcrologic.fulcro.algorithms.normalized-state :as fns]))

(defmutation login [{:keys [email password]}]
  (action [{:keys [state]}]
    (swap! state assoc-in [:component/id ::Session :session/status] :pending))
  (remote [env]
    true)
  (ok-action [{:keys [state result]}]
    (swap! state assoc-in [:component/id ::Session :session/user] (:user result)))
  (error-action [{:keys [state]}]
    (swap! state assoc-in [:component/id ::Session :session/status] :failed)))
```

Sections (use the ones you need):

| Section | Runs |
|---------|------|
| `action` | Synchronously, before remote dispatch. Place optimistic state changes here. |
| `remote` | Returns `true` (send the mutation as-is), `false` (suppress), or an EQL `ast` to shape what the server sees, for example to strip large parameters or rename keys. Return an `ast` only when the wire payload must differ from the client invocation. Multiple remotes named after their keys (`:remote`, `:rest-remote`, …). |
| `ok-action` | After a successful remote response. `result` is the server's return value. |
| `error-action` | After a failed remote response. |
| `result-action` | Runs after every remote result (success or failure). Use it for cleanup that should not branch on outcome. |

Update the client database only inside a mutation section. The `state` key in the env is an atom, and `swap!` on it is the supported way to update normalized data: in `action` for optimistic changes, and in `ok-action`, `error-action`, or `result-action` for changes that depend on the server response, as in the example above. Calling `swap!` on the state atom from UI code, such as a button handler or a `useEffect`, bypasses `comp/transact!`. Fulcro Inspect's Transactions tab does not record the change, the remote pipeline does not run, and post-mutation hooks attached to the change do not fire. Transact a `defmutation` instead.

Transact a mutation from the UI through `comp/transact!`:

```clojure
(comp/transact! this [(login {:email "alice@example.com" :password "..."})])
```

For pessimistic flows (UI waits for the server before reflecting state), see [`com.fulcrologic.fulcro.algorithms.tx-processing`](https://book.fulcrologic.com/) and the pessimistic mutation patterns in the developer's guide.

## Loading Data

`com.fulcrologic.fulcro.data-fetch` (alias `df`).

```clojure
;; whole component, into a target
(df/load! this :all-people Person
          {:target               [:component/id ::PeoplePage :people]
           :marker               :people-load
           :post-mutation        `refresh-counts
           :post-mutation-params {:scope :people}})

;; one field on a known entity
(df/load-field! this :person/email)
```

Targeting helpers from `com.fulcrologic.fulcro.algorithms.data-targeting` (often aliased `targeting`):

| Helper | Use |
|--------|-----|
| `targeting/replace-at` | Replace the value at a path |
| `targeting/append-to` | Append an ident to a list |
| `targeting/prepend-to` | Prepend an ident to a list |
| `targeting/multiple-targets` | Send the same load result to multiple targets |

Loading markers (the `:marker` option) write a status entry into the DB that components can render as a spinner. Read the marker through `(df/get-loading-state state marker-id)` or by querying `[df/marker-table marker-id]`.

`:post-mutation` triggers a normal `defmutation` after the load merges. Use it for derived state, follow-on loads, or routing transitions that should fire only after data is present.

Do not hand-roll merges. `(swap! state assoc-in [:person/id 42] new-person)` bypasses the normalization that `df/load!` performs and easily produces a denormalized DB. If a merge cannot be expressed as a load, use `(merge/merge-component! app Person new-person)` from `com.fulcrologic.fulcro.algorithms.merge`.

## Routing

Dynamic routing from `com.fulcrologic.fulcro.routing.dynamic-routing` (alias `dr`).

```clojure
(ns app.ui.root
  (:require
   [com.fulcrologic.fulcro.components :as comp :refer [defsc]]
   [com.fulcrologic.fulcro.routing.dynamic-routing :as dr :refer [defrouter]]
   [app.ui.home :refer [Home]]
   [app.ui.people :refer [PeoplePage]]
   [app.ui.person :refer [PersonDetail]]))

(defrouter RootRouter [this {:keys [current-state]}]
  {:router-targets [Home PeoplePage PersonDetail]})

(defsc Root [this {:keys [router]}]
  {:query         [{:router (comp/get-query RootRouter)}]
   :initial-state {:router {}}}
  (ui-root-router router))
```

Each router target component declares a `:route-segment`:

```clojure
(defsc PersonDetail [this {:person/keys [id name]}]
  {:query         [:person/id :person/name]
   :ident         :person/id
   :route-segment ["person" :person-id]
   :will-enter    (fn [app {:keys [person-id]}]
                    (dr/route-deferred
                      [:person/id (uuid person-id)]
                      #(df/load! app [:person/id (uuid person-id)] PersonDetail
                                 {:post-mutation        `dr/target-ready
                                  :post-mutation-params {:target [:person/id (uuid person-id)]}})))}
  (dom/div name))
```

Navigate with `(dr/change-route! this ["person" id])`. Routes can defer rendering (`dr/route-deferred`) until a load completes, then call `dr/target-ready` to signal completion.

For HTML5 history integration, wire the [`kibu/pushy`](https://github.com/kibu-australia/pushy) library to bridge browser navigation events to `dr/change-route!`. The pattern is documented in the developer's guide.

## Forms

`com.fulcrologic.fulcro.algorithms.form-state` (alias `fs`).

```clojure
(ns app.ui.profile
  (:require
   [com.fulcrologic.fulcro.components :as comp :refer [defsc]]
   [com.fulcrologic.fulcro.algorithms.form-state :as fs]
   [com.fulcrologic.fulcro.mutations :as m :refer [defmutation]]))

(defsc ProfileForm [this {:profile/keys [name email] :as props}]
  {:query         [:profile/id :profile/name :profile/email fs/form-config-join]
   :ident         :profile/id
   :form-fields   #{:profile/name :profile/email}
   :initial-state (fn [params] (fs/add-form-config ProfileForm params))})

(defmutation save-profile [_]
  (action [{:keys [state ref]}]
    (swap! state fs/entity->pristine* ref))
  (remote [_] true))
```

Key helpers:

| Helper | Purpose |
|--------|---------|
| `fs/add-form-config` / `fs/add-form-config*` | Add form metadata to an entity |
| `fs/dirty?` | Has any field changed since pristine? |
| `fs/validity` | `:valid`, `:invalid`, or `:unchecked` for a form or field |
| `fs/mark-complete*` | Mark a field as "user has interacted" so validation can run |
| `fs/entity->pristine*` | Snapshot the current state as the new pristine baseline |
| `fs/pristine->entity*` | Revert to the pristine snapshot |

Validation runs through a predicate per field (`(defmethod fs/form-field-valid? :profile/email ...)`), with custom error rendering driven by `(fs/validity profile :profile/email)` in the component body.

Fulcro RAD layers `fo/*` (`com.fulcrologic.rad.form-options`) on top of `fs/*` for attribute-driven form generation. When working in a RAD project, prefer RAD's form rendering; outside RAD, build the form body in component render and reach for `fs/*` directly.

## UI State Machines (UISM)

`com.fulcrologic.fulcro.ui-state-machines` (alias `uism`).

A state machine declares states, events, and transitions. Use it when a UI flow has explicit phases that cannot be modeled cleanly as ad-hoc props:

- Authentication (`:initial`, `:awaiting-credentials`, `:authenticating`, `:authenticated`, `:failed`).
- Multi-step wizards.
- Network retry loops.
- Routing transitions that interleave loads, validation, and navigation.

```clojure
(ns app.machines.session
  (:require
   [com.fulcrologic.fulcro.ui-state-machines :as uism :refer [defstatemachine]]))

(defstatemachine session-machine
  {::uism/actor-names #{:actor/session}
   ::uism/aliases     {:status [:actor/session :session/status]}
   ::uism/states      {:initial {::uism/handler (fn [env] ...)}
                       :authenticating {::uism/events {:event/success ...
                                                       :event/failure ...}}}})
```

State machines coordinate `defmutation` invocations and `df/load!` calls; they do not replace either. Reach for them sparingly — a small UI flow rarely justifies the ceremony.

## Pathom Server

The current server-side EQL implementation is [Pathom 3](https://pathom3.wsscode.com/). The conventional alias is `pco` for `com.wsscode.pathom3.connect.operation`.

```clojure
(ns app.server.resolvers
  (:require
   [com.wsscode.pathom3.connect.operation :as pco]
   [com.wsscode.pathom3.connect.indexes :as pci]
   [com.wsscode.pathom3.interface.eql :as p.eql]
   [app.server.db :as db]))

(pco/defresolver person-by-id [{:person/keys [id]}]
  {::pco/output [:person/id :person/name :person/email]}
  (db/get-person id))

(pco/defresolver all-people [_]
  {::pco/output [{:all-people [:person/id]}]}
  {:all-people (db/list-people)})

(def env (pci/register [person-by-id all-people]))

(defn process [eql] (p.eql/process env eql))
```

Mutations on the server use `(pco/defmutation ...)`. Symbols must match the client side so EQL flows symmetrically: a client `defmutation` in `app.model.session/login` sends a transaction whose key is the fully-qualified symbol `app.model.session/login`; the server resolves that symbol to a Pathom mutation of the same name.

Pathom 2 (`com.wsscode.pathom.connect`) is still in widespread use. New projects should adopt Pathom 3; existing Pathom 2 codebases should consult the [Pathom 3 migration guide](https://pathom3.wsscode.com/docs/migrate-from-pathom2/) before rewriting incrementally.

Wire the parser into a transit endpoint (`/api`) using Ring middleware. Fulcro's HTTP remote sends a transit-encoded EQL request and expects a transit-encoded result. The exact Ring stack varies; the developer's guide chapter on servers walks through one canonical wiring.

## Fulcro Inspect

Fulcro Inspect is the single highest-leverage debugging tool. Since Fulcro 3.8, Inspect is a library the application adds as a development dependency and installs explicitly. The setup has four parts, following the [fulcro-inspect README](https://github.com/fulcrologic/fulcro-inspect):

1. Install the Chrome extension (`fulcro-inspect-chrome-<version>.zip`) or the Electron app (4.x) from the [fulcro-inspect releases page](https://github.com/fulcrologic/fulcro-inspect/releases). Open a new tab or restart Chrome after installing the extension.
2. Add `com.fulcrologic/fulcro-inspect` to the development alias in `deps.edn` (it brings in `com.fulcrologic/fulcro-devtools-remote`). Point shadow-cljs at that alias with `:deps {:aliases [:dev]}`.
3. Add the preload for the chosen front end:

   ```clojure
   ;; shadow-cljs.edn
   {:deps   {:aliases [:dev]}
    :builds
    {:main
     {:devtools
      {:preloads [com.fulcrologic.devtools.chrome-preload        ; or com.fulcrologic.devtools.electron-preload
                  com.fulcrologic.fulcro.inspect.dom-picker-preload]}}}}
   ```

4. Register the app with Inspect during client startup:

   ```clojure
   (ns app.client
     (:require
      [com.fulcrologic.devtools.common.target :refer [ido]]
      [fulcro.inspect.tool :as it]
      [app.application :refer [SPA]]))

   (defn ^:export init []
     (ido (it/add-fulcro-inspect! SPA))
     ...)
   ```

`ido` behaves like `do` in development and removes its body from release builds, so Closure dead-code elimination strips Inspect from production output. The DOM picker preload is optional but recommended. It lets the Inspect tab highlight a rendered component and jump to its props and ident.

The pre-3.8 preload `com.fulcrologic.fulcro.inspect.preload` still exists, but it only logs "Inspect NOT installed" to the console. A project that still lists it, or that relies on the old Chrome Web Store extension, has no working Inspect.

What Inspect provides:

- **DB tab** — browse the normalized client database, expand idents, jump from a value to the component that uses it.
- **Transactions tab** — every `comp/transact!` call, its action results, remote payload, and response.
- **Network tab** — every HTTP remote request and response in transit form.
- **EQL tab** — run an EQL query against the current state or a connected Pathom parser, with autocomplete.

Keep Chrome DevTools open during development so Binaryage devtools (`binaryage/devtools` in the `:dev` alias) can format Clojure data in the console. Closing and reopening DevTools forces a re-cache that occasionally drops the formatter.

### REPL flight recorder

Fulcro 3.9 added `com.fulcrologic.fulcro.inspect.repl-tool`, which records transactions, network activity, state changes, and state machine transitions into a log that can be queried from the REPL. It suits REPL-driven and agent-driven debugging where the Inspect UI is not available:

```clojure
(require '[com.fulcrologic.fulcro.inspect.repl-tool :as rt])

(rt/install! SPA)
(rt/clear!)
;; trigger the behavior under investigation
(rt/digest)   ; structured summary of what happened
(rt/tx-log)   ; transactions only
```

`rt/network-log` and `rt/uism-log` narrow the log further. The log lives in the app's runtime atom and has a bounded size.

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

### `with-redefs` is unreliable across the wire

Stubbing a server-side resolver from a client-side test does not work; they run in different processes. Reach for `fulcro-spec`'s component stubs on the client, and for ordinary `with-redefs` (JVM) or `with-redefs` + reload (CLJS) only within a single process.
