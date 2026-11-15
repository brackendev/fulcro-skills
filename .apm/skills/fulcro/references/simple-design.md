# Simple Design in Fulcro

Part of the `fulcro` skill, for Fulcro 3 applications. It shows how Fulcro's own constructs carry out Key Rule 20 of the host-neutral `clojure` skill ("Choose simple over easy"). Read `clojure/references/simple-design.md` in the `clojure` skill first. For code outside Fulcro's constructs, `clojurescript/references/simple-design.md` and `clojure-jvm/references/simple-design.md` give the host constructs. Their rules and exceptions apply here unchanged.

Fulcro already keeps apart most of the concerns that those references name. In a Fulcro project the rule is mostly this: use the Fulcro construct, and do not build a second mechanism beside it. The example below was run on the JVM against Fulcro 3.10.1.

## What Fulcro already keeps apart

| Concern | Fulcro construct | Do not add |
|---------|------------------|------------|
| The application's state | The normalized client database: one map of tables keyed by ident (`references/components.md`) | A second store, or a component-local atom, for data the database holds |
| What a component needs | Its `:query`, composed with `comp/get-query` (Iron Law FUL-2) | A read of the state atom from render |
| A change | A mutation in a transaction, which is data: `[(mark-read {:message/id 1})]` | A `swap!` on the application state from UI code (Iron Law FUL-1) |
| Where and when a change is carried out | The mutation's `action`, `remote`, and result sections, run by Fulcro's transaction processing | An HTTP call and its callbacks inside a component |
| What the server can answer | Pathom 3 resolvers that declare their input and output (`references/pathom-server.md`) | An endpoint for each screen |
| A flow with several steps | A UI state machine (Key Rule 7) | Status flags spread across mutations |
| Form state | The `fs/*` form-state functions (Key Rule 5) | Dirty and valid flags kept by hand in a component |

## Write each change as a function of the state map

The state atom in a mutation's environment is the edge. Keep the change itself in a plain function that takes the state map and returns the next one, and let the mutation apply it with `swap!`. The Fulcro Developer's Guide: "Fulcro adopts the general notation of suffixing functions with * when they operate on a normalized state map (not atom)." `fs/add-form-config*` is one of Fulcro's own.

```clojure
(ns app.model.message
  (:require [com.fulcrologic.fulcro.mutations :as m :refer [defmutation]]))

;; the change is a plain function from a state map to a state map
(defn mark-read* [state-map message-id]
  (assoc-in state-map [:message/id message-id :message/read?] true))

;; the mutation applies it, and says that the server runs the same mutation
(defmutation mark-read [{:message/keys [id]}]
  (action [{:keys [state]}]
    (swap! state mark-read* id))
  (remote [_env] true))
```

```clojure
;; the function is tested with a literal map, without an application
(deftest mark-read-test
  (is (= {:message/id {1 {:message/id 1 :message/read? true}}}
         (mark-read* {:message/id {1 {:message/id 1 :message/read? false}}} 1))))
```

- Several mutations can share one `*` function, and one mutation can thread several of them in a single `swap!`.
- Use the pure forms of Fulcro's own helpers inside a `swap!`. `merge/merge-component` takes a state map and returns a state map. `merge/merge-component!` is for code outside a mutation, such as server push (Iron Law FUL-1).
- Never read the state atom twice in one section to compute a change. Compute from the map that `swap!` passes in.

## Treat the transaction as the queue

`(comp/transact! this [(mark-read {:message/id 1})])` hands Fulcro a list that names the mutation and its parameters. The component does not call the server, does not wait, and does not know which remote handles the change. That is the separation the baseline asks a queue for, so do not build another one in the client.

- Put the call to the server in the mutation's `remote` section, not in the component.
- Put what happens after the server answers in `ok-action` and `error-action`, not in a callback that the component holds.
- Keep the parameters plain data that transit can encode (Iron Law FUL-4). An atom, a function, or a component instance in the parameters ties the mutation back to the UI.
- Load data with `df/load!` and its `:target` and `:post-mutation` options instead of merging a response into the state by hand (Key Rule 3).

## Declare data, and let Fulcro and Pathom work out how

- **Queries.** A component states the attributes it needs. It does not say where they come from or how they are joined. Compose child queries with `comp/get-query` so that normalization works (Iron Law FUL-2).
- **Idents.** An ident names an entity once. Every reference to the entity is that ident, so a change to the entity shows wherever the entity is shown (`references/components.md`).
- **Resolvers.** A Pathom resolver declares the attributes it needs and the attributes it provides, and Pathom finds the path from one to the other (`references/pathom-server.md`).
- **Flows.** A UI state machine holds a flow's states and events in one definition (`references/state-machines.md`).

## When the Fulcro construct is not the right choice

- State that no other component reads and that need not survive a remount, such as whether a menu is open, can stay in a React hook in a hooks component. Anything that a mutation, a load, or another component reads belongs in the database. This point is this package's guidance.
- Server push and timers have no component to transact from. They use `merge/merge-component!` or a transaction on the application itself (Iron Law FUL-1).
- Authorization is not declared once. Check it in each resolver and mutation on the server (Iron Law FUL-5).
