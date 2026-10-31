# Components, Idents, and Queries

Part of the `fulcro` skill, for Fulcro 3 applications.

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

A docstring goes between the component name and the argument list: `(defsc Person "A person row." [this props] {...} ...)`. A string placed after the argument list, as in `(defsc Person [this props] "A person." {:query [...]} ...)`, fails compilation with "Docstring is in the wrong position". Move it before the argument list.

| Option | Purpose |
|--------|---------|
| `:query` | EQL the component requires |
| `:ident` | Two-tuple table identity, or `(fn [] ...)` for a custom shape |
| `:initial-state` | Seed data, composable across the parent tree |
| `:pre-merge` | Transform a server response before normalization |
| `:use-hooks?` | `true` renders a function component that can use React hooks. `:pure` also memoizes it with React `memo`. Switching to or from `:pure` requires a browser reload. |
| `:componentDidMount` / `:componentWillUnmount` | Lifecycle hooks on the class form |
| `:initLocalState` | Component-local mutable state (rare; prefer the normalized DB, because local state is not part of the app database, so Fulcro Inspect and the transaction history do not show it) |

The render body is a function of props and must not have side effects: no `js/fetch`, no `localStorage` access, and no `swap!` on app state. React can call render many times. Start effects from a mutation, from `df/load!`, or from a lifecycle hook.

The factory wraps a component for rendering. Build it once per namespace:

```clojure
(def ui-person (comp/factory Person {:keyfn :person/id}))

;; in a parent render
(map ui-person (sort-by :person/name people))
```

`:keyfn` returns React's `key` for each rendered instance. A factory for a component that has an `:ident` defaults the key to the ident's second element as a string, so `{:keyfn :person/id}` above is redundant but harmless. Precedence runs from a `:react-key` entry in the props, to an explicit `:keyfn`, to the ident default. Components without an ident still need `:keyfn` when rendered in a list, or React logs a missing-key warning and re-renders more than necessary.

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

When the data is in the database but a component's props are `nil`, rebuild the props tree at the REPL and find where it stops. Jakub Holý's [Fulcro Troubleshooting Decision Tree](https://blog.jakubholy.net/2020/troubleshooting-fulcro/) recommends this check:

```clojure
(require
  '[com.fulcrologic.fulcro.algorithms.denormalize :as fdn]
  '[com.fulcrologic.fulcro.application :as app])

(let [state (app/current-state SPA)]
  (fdn/db->tree (comp/get-query Root) state state))
```

A missing branch usually means a join in some parent query was written by hand instead of with `comp/get-query`, or that a parent's `:initial-state` does not include the child. The two `:initial-state` forms compose children differently. In the template map, a join key's value (`{:root/router {}}`) is replaced by that child's initial state, with the map as its parameters. The lambda form does nothing automatically, so it must call `(comp/get-initial-state Child params)` for each join.
