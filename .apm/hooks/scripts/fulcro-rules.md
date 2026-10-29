# Fulcro rules from fulcro-skills

This is a Fulcro 3 project. The Iron Laws below come from the `fulcro` skill and add to the `clojure` baseline laws. Never write code that breaks one. Before writing, reviewing, or refactoring code in this project, load the `fulcro` skill along with the `clojure`, `clojurescript`, and `clojure-jvm` skills. The skills hold the full laws, their exceptions, and the references.

## Iron Laws

- FUL-1: Change app state only through Fulcro transactions (`defmutation` with `comp/transact!`, `df/load!`, `m/set-value!`, `uism/trigger!`, `dr/change-route!`, or `merge/merge-component!`), never by touching the state atom. A custom `result-action` must call `m/default-result-action!`.
- FUL-2: Compose joins with `comp/get-query`, and give every entity of one type the same ident shape across query, ident, and initial state.
- FUL-3: Compose child initial state into every parent up to the root with `comp/get-initial-state`, or load the data before showing the component.
- FUL-4: Send only transit-encodable data over the network, and register a handler for each custom type.
- FUL-5: Authorize every resolver and mutation on the server, with the authenticated user placed in the Pathom environment by middleware.
