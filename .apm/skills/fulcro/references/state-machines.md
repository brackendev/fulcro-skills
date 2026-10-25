# UI State Machines

Part of the `fulcro` skill, for Fulcro 3 applications.

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

Start a machine with `(uism/begin! app-or-component session-machine ::session {:actor/session Session})`, which takes the machine, an instance id, and the actor map, and send events with `(uism/trigger! app-or-component ::session :event/success)`.

A handler receives the state machine `env` and must return the updated `env`. Express its work through the env helpers, which record it in the `env` that the handler returns:

- `uism/apply-action` runs a function of the state map for local changes, and `uism/assoc-aliased` writes through an alias.
- `uism/trigger-remote-mutation` sends a remote mutation and can name the events to trigger on success and failure.
- `uism/load` issues a load, and `uism/transact` queues an ordinary transaction.

Defer transactions with `uism/transact` instead of calling `comp/transact!` in a handler. The `uism/transact` docstring warns that a synchronous transaction run inside a handler does not work as expected, because the handler finishes afterward and overwrites the state changes that the transaction made.

State machines coordinate `defmutation` invocations and `df/load!` calls; they do not replace either. Reach for them sparingly — a small UI flow rarely justifies the ceremony.
