# Mutations and Loading Data

Part of the `fulcro` skill, for Fulcro 3 applications.

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
  (ok-action [{:keys [state mutation-return-value]}]
    (swap! state assoc-in [:component/id ::Session :session/user] (:user mutation-return-value)))
  (error-action [{:keys [state]}]
    (swap! state assoc-in [:component/id ::Session :session/status] :failed)))
```

Sections (use the ones you need):

| Section | Runs |
|---------|------|
| `action` | Synchronously, before remote dispatch. Place optimistic state changes here. |
| `remote` | Returns `true` (send the mutation as-is), `false` (suppress), the `env` returned by `m/returning`, `m/with-target`, or `m/with-params`, or an EQL `ast` to shape what the server sees, for example to strip large parameters or rename keys. Return an `ast` only when the wire payload must differ from the client invocation. Multiple remotes named after their keys (`:remote`, `:rest-remote`, …). |
| `ok-action` | After a successful remote response. `result` is the network result map (`:status-code`, `:body`), and `mutation-return-value` is the server mutation's return value. |
| `error-action` | After a failed remote response. By default any status code other than 200 is a failure. |
| `result-action` | Runs after every remote result in place of the default result handler, `m/default-result-action!`. The default rewrites tempids, merges a `m/returning` result, and then calls `ok-action` or `error-action`. A custom `result-action` that does not call `(m/default-result-action! env)` loses all of that, and its `ok-action` and `error-action` never run. |

To merge the server's return value into the client database as an entity, return `(m/returning env Person)` from the `remote` section. Fulcro sends a mutation join with `Person`'s query and normalizes the result. Thread `m/with-target` after it to add the returned ident to a path, for example `(-> env (m/returning Person) (m/with-target (targeting/append-to [:component/id ::PeoplePage :people])))`. A Pathom 3 server returns only the attributes that a mutation join asks for, so a mutation that creates an entity with a tempid also needs the `:tempids` transform in `references/pathom-server.md`.

Update the client database only inside a mutation section. The `state` key in the env is an atom, and `swap!` on it is the supported way to update normalized data: in `action` for optimistic changes, and in `ok-action`, `error-action`, or `result-action` for changes that depend on the server response, as in the example above. Calling `swap!` on the state atom from UI code, such as a button handler or a `useEffect`, bypasses `comp/transact!`. Fulcro does not watch the state atom, so the UI does not refresh. Fulcro Inspect's Transactions tab does not record the change, the remote pipeline does not run, and post-mutation hooks attached to the change do not fire. Transact a `defmutation` instead.

Transact a mutation from the UI through `comp/transact!`:

```clojure
(comp/transact! this [(login {:email "alice@example.com" :password "..."})])
```

For pessimistic flows, where the UI waits for the server before reflecting state, put the follow-on work in `ok-action` and `error-action`, or pass `{:optimistic? false}` as the options to `comp/transact!`. `comp/ptransact!` is deprecated, and its docstring points to `result-action` for chaining.

## Custom Types over the Network

Fulcro uses transit between client and server, so anything a mutation, a load, or a response carries must be transit-encodable. For a custom type, register a handler in the global registry from `com.fulcrologic.fulcro.algorithms.transit` (alias `transit`) on both client and server:

```clojure
(deftype Point [x y])

(defn install! []
  (transit/install-type-handler!
    (transit/type-handler Point "geo/point"
      (fn [^Point p] [(.-x p) (.-y p)])
      (fn [[x y]] (Point. x y)))))
```

The Developer's Guide says to install type handlers "as early as possible," because some parts of the stack close over the registry when they start. It recommends `deftype` over `defrecord`, because a record "looks like a map to Fulcro's internals" and can lose its type when a query runs against it.

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

Loading markers (the `:marker` option) write a status entry into the DB that components can render as a spinner. Add the ident `[df/marker-table marker-id]` to the component's query, read the marker from the props under that same key, and test it with `df/loading?`:

```clojure
(defsc PeoplePage [this {:keys [people] :as props}]
  {:query [{:people (comp/get-query Person)} [df/marker-table :people-load]]
   ...}
  (let [marker (get props [df/marker-table :people-load])]
    (if (df/loading? marker)
      (dom/p "Loading...")
      (map ui-person people))))
```

`:post-mutation` triggers a normal `defmutation` after the load merges. Use it for derived state, follow-on loads, or routing transitions that should fire only after data is present.

Do not hand-roll merges. `(swap! state assoc-in [:person/id 42] new-person)` bypasses the normalization that `df/load!` performs and easily produces a denormalized DB. If a merge cannot be expressed as a load, use `(merge/merge-component! app Person new-person)` from `com.fulcrologic.fulcro.algorithms.merge`.
