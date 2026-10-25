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
