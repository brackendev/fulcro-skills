# Fulcro Inspect

Part of the `fulcro` skill, for Fulcro 3 applications.

## Fulcro Inspect

Fulcro Inspect is the single highest-leverage debugging tool. Inspect is a library that the application installs explicitly at startup. The setup has four parts, following the [fulcro-inspect README](https://github.com/fulcrologic/fulcro-inspect):

1. Install the Fulcro Inspect Chrome extension from the Chrome Web Store or as `fulcro-inspect-chrome-<version>.zip` from the [fulcro-inspect releases page](https://github.com/fulcrologic/fulcro-inspect/releases), or install the Electron app from the same releases page. Open a new tab or restart Chrome after installing the extension.
2. Add `com.fulcrologic/fulcro-inspect` to the development alias in `deps.edn` (it brings in `com.fulcrologic/fulcro-devtools-remote`). Fulcro 3.10.1 already depends on `fulcro-inspect` 1.0.5, so the entry pins the version rather than adding a new library. Point shadow-cljs at that alias with `:deps {:aliases [:dev]}`.
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

`ido` behaves like `do` in development and removes its body from release builds, so Closure dead-code elimination strips all but a few kilobytes of Inspect from production output. The DOM picker preload is optional but recommended. It lets the Inspect tab highlight a rendered component and jump to its props and ident.

`com.fulcrologic.fulcro.inspect.preload` is not an Inspect preload. Loading it only logs "Inspect NOT installed" to the console, so replace it with the preloads above.

What Inspect provides:

- The **DB** and **DB Explorer** tabs browse the normalized client database. DB Explorer follows idents to the entities they point to.
- The **Transactions** tab lists every `comp/transact!` call, its action results, remote payload, and response.
- The **Network** tab lists every remote request and response. "Send to query" copies a request into the EQL tab.
- The **EQL** tab runs an EQL query or mutation through one of the app's remotes and shows the server's raw answer.
- The **Statecharts** tab shows the state of `com.fulcrologic/statecharts` machines.

Keep Chrome DevTools open during development so Binaryage devtools (`binaryage/devtools` in the `:dev` alias) can format Clojure data in the console. Closing and reopening DevTools forces a re-cache that occasionally drops the formatter.

### REPL flight recorder

`com.fulcrologic.fulcro.inspect.repl-tool` records transactions, network activity, state changes, and state machine transitions into a log that can be queried from the REPL. It suits REPL-driven and agent-driven debugging where the Inspect UI is not available:

```clojure
(require '[com.fulcrologic.fulcro.inspect.repl-tool :as rt])

(rt/install! SPA)
(rt/clear!)
;; trigger the behavior under investigation
(rt/digest)   ; structured summary of what happened
(rt/tx-log)   ; transactions only
```

`rt/network-log` and `rt/uism-log` narrow the log further. The log lives in the app's runtime atom and has a bounded size.
