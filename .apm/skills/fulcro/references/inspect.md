# Fulcro Inspect

Part of the `fulcro` skill, for Fulcro 3 applications.

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
