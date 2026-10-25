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

State machines coordinate `defmutation` invocations and `df/load!` calls; they do not replace either. Reach for them sparingly — a small UI flow rarely justifies the ceremony.
