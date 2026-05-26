---
name: fulcro-new
description: "Scaffold a new Fulcro full-stack project with shadow-cljs and Pathom 3"
argument-hint: "<project-name>"
user-invocable: true
disable-model-invocation: true
---

# Scaffold a Fulcro Project

Create a new [Fulcro](https://github.com/fulcrologic/fulcro) full-stack project with a ClojureScript client (shadow-cljs build), a Clojure JVM server (Pathom 3 parser, Ring + Jetty), `clj-kondo` lint setup, `cljfmt` formatting, and a development REPL. The layout mirrors the canonical [`fulcro-template`](https://github.com/fulcrologic/fulcro-template) structure.

## Arguments

| Input             | Target                                                                       |
|-------------------|------------------------------------------------------------------------------|
| `<project-name>`  | Required. Use hyphens (for example, `my-app`); the skill maps to underscores for file paths (`my_app/`) and keeps hyphens in namespace symbols. |
| (no argument)     | Prompt the operator for a project name.                                      |

This skill is exempt from the `all` and `<path>` rows of the standard scope vocabulary because scaffolding has no useful default scope. See `CONVENTIONS.md` in the repo root for the standard.

## Mutation

Mutates by default: creates the project directory and writes `deps.edn`, `shadow-cljs.edn`, `package.json`, the client entry files (`app.application`, `app.client`, `app.ui.root`, `app.model.session`), the server files (`app.server.main`, `app.server.pathom`, `app.server.middleware`), `src/dev/user.clj`, `.clj-kondo/config.edn` plus upstream imports, `.cljfmt.edn`, `resources/public/index.html`, `resources/dev.html`, and `.gitignore`. Also runs `npm install` and `npx shadow-cljs compile main` as a first-build sanity check. No `--report` flag; preview the side effects by reading this `SKILL.md`.

## Prerequisites

Verify these are installed before proceeding. If any are missing, stop and tell the user.

- Java 17 or higher: `java --version`
- Clojure CLI: `clj --version`
- Node + npm (for shadow-cljs): `node --version` and `npm --version`
- `clj-kondo` (linter): `clj-kondo --version`

Outbound network access to `clojars.org` and `npmjs.com` is required for dependency resolution.

## Steps

### 1. Get Project Name and Namespace

Use `$ARGUMENTS` as the project name. If empty, ask the user for one.

The project name uses hyphens (e.g., `my-app`). The main namespace mirrors the project name (`app` by default for `fulcro-template` parity; ask the user if they prefer a custom namespace like `com.example.my-app`). File paths use underscores.

Confirm the project name and main namespace with the user before continuing.

### 2. Resolve the Latest Fulcro Version

Fetch the latest released version from Clojars:

```bash
curl -sSL -A 'Mozilla/5.0' \
  'https://clojars.org/api/artifacts/com.fulcrologic/fulcro' \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['latest_release'])"
```

Do the same for `com.fulcrologic/guardrails` and `com.wsscode/pathom3`. If a fetch fails, fall back to the most recent versions known to work together (Fulcro 3.8.x with Pathom 3 alpha builds at the time of writing) and warn the user that the values may be stale.

### 3. Create Project Directory and Source Roots

```bash
mkdir -p <project-name>/{src/main/app/ui,src/main/app/model,src/main/app/server,src/dev,src/test,resources/public/js,resources/public/css}
cd <project-name>
```

### 4. Create `deps.edn`

```clojure
{:paths ["src/main" "resources"]

 :deps  {org.clojure/clojure          {:mvn/version "1.12.0"}
         org.clojure/clojurescript    {:mvn/version "1.12.42"}
         com.fulcrologic/fulcro       {:mvn/version "<latest-fulcro-version>"}
         com.fulcrologic/guardrails   {:mvn/version "<latest-guardrails-version>"}
         com.wsscode/pathom3          {:mvn/version "<latest-pathom3-version>"}
         metosin/muuntaja             {:mvn/version "0.6.10"}
         ring/ring-jetty-adapter      {:mvn/version "1.12.1"}
         ring/ring-defaults           {:mvn/version "0.5.0"}
         com.cognitect/transit-clj    {:mvn/version "1.0.333"}
         ch.qos.logback/logback-classic {:mvn/version "1.5.6"}}

 :aliases
 {:dev    {:extra-paths ["src/dev" "src/test"]
           :extra-deps  {thheller/shadow-cljs        {:mvn/version "2.28.18"}
                         binaryage/devtools          {:mvn/version "1.0.7"}
                         org.clojure/tools.namespace {:mvn/version "1.5.0"}}}

  :test   {:extra-paths ["src/test"]
           :extra-deps  {lambdaisland/kaocha       {:mvn/version "1.91.1392"}
                         com.fulcrologic/fulcro-spec {:mvn/version "3.1.31"}}
           :main-opts   ["-m" "kaocha.runner"]}

  :cljfmt {:extra-deps {dev.weavejester/cljfmt {:mvn/version "0.13.0"}}
           :main-opts  ["-m" "cljfmt.main"]}

  :build  {:deps       {io.github.clojure/tools.build {:mvn/version "0.10.5"}}
           :ns-default build}

  :dry4clj {:extra-deps {dry4clj/dry4clj {:mvn/version "0.1.1"}}
            :main-opts  ["-m" "dry4clj.core"]}}}
```

### 5. Create `shadow-cljs.edn`

```clojure
{:source-paths ["src/main" "src/dev" "src/test"]
 :deps         true
 :nrepl        {:port 9000}
 :dev-http     {8000 "resources/public"}

 :builds
 {:main {:target     :browser
         :output-dir "resources/public/js"
         :asset-path "/js"
         :modules    {:main {:init-fn app.client/init}}
         :devtools   {:preloads   [com.fulcrologic.fulcro.inspect.preload
                                   com.fulcrologic.fulcro.inspect.dom-picker-preload]
                      :after-load app.client/refresh}}

  :test {:target    :browser-test
         :test-dir  "resources/public/js/test"
         :ns-regexp "-test$"
         :devtools  {:http-port 8021
                     :http-root "resources/public/js/test"}}

  :ci   {:target    :karma
         :output-to "target/karma-test.js"
         :ns-regexp "-test$"}}}
```

### 6. Create `package.json`

```json
{
  "name": "<project-name>",
  "private": true,
  "devDependencies": {
    "shadow-cljs": "2.28.18",
    "karma": "6.4.4",
    "karma-chrome-launcher": "3.2.0",
    "karma-cljs-test": "0.1.0"
  },
  "dependencies": {
    "react": "18.3.1",
    "react-dom": "18.3.1"
  }
}
```

### 7. Create Client Entry Files

#### `src/main/app/application.cljs`

```clojure
(ns app.application
  (:require
   [com.fulcrologic.fulcro.application :as app]
   [com.fulcrologic.fulcro.networking.http-remote :as http]))

(defonce SPA
  (app/fulcro-app
    {:remotes {:remote (http/fulcro-http-remote {:url "/api"})}}))
```

#### `src/main/app/client.cljs`

```clojure
(ns app.client
  (:require
   [com.fulcrologic.fulcro.application :as app]
   [app.application :refer [SPA]]
   [app.ui.root :as root]))

(defn ^:export init []
  (app/mount! SPA root/Root "app")
  (js/console.log "Loaded"))

(defn ^:export refresh []
  (app/mount! SPA root/Root "app")
  (js/console.log "Refreshed"))
```

#### `src/main/app/ui/root.cljs`

```clojure
(ns app.ui.root
  (:require
   [com.fulcrologic.fulcro.components :as comp :refer [defsc]]
   [com.fulcrologic.fulcro.dom :as dom]))

(defsc Root [this {:root/keys [greeting]}]
  {:query         [:root/greeting]
   :initial-state (fn [_] {:root/greeting "Hello from Fulcro"})}
  (dom/div
    (dom/h1 greeting)))
```

#### `src/main/app/model/session.cljs`

A minimal example mutation, present so the project has a template to copy.

```clojure
(ns app.model.session
  (:require
   [com.fulcrologic.fulcro.mutations :as m :refer [defmutation]]))

(defmutation set-greeting [{:keys [greeting]}]
  (action [{:keys [state]}]
    (swap! state assoc :root/greeting greeting)))
```

### 8. Create Server Files

#### `src/main/app/server/main.clj`

```clojure
(ns app.server.main
  (:require
   [ring.adapter.jetty :as jetty]
   [app.server.middleware :as middleware]))

(defn start! [{:keys [port] :or {port 3000}}]
  (jetty/run-jetty middleware/app {:port port :join? false}))

(defn stop! [^org.eclipse.jetty.server.Server server]
  (.stop server))

(defn -main [& _]
  (start! {:port 3000}))
```

#### `src/main/app/server/pathom.clj`

```clojure
(ns app.server.pathom
  (:require
   [com.wsscode.pathom3.connect.indexes :as pci]
   [com.wsscode.pathom3.connect.operation :as pco]
   [com.wsscode.pathom3.interface.eql :as p.eql]))

(pco/defresolver server-greeting [_ _]
  {::pco/output [:root/greeting]}
  {:root/greeting "Hello from the Pathom server"})

(def env
  (pci/register [server-greeting]))

(defn process [eql-or-tx]
  (p.eql/process env eql-or-tx))
```

#### `src/main/app/server/middleware.clj`

```clojure
(ns app.server.middleware
  (:require
   [muuntaja.middleware :as muuntaja]
   [ring.middleware.defaults :refer [wrap-defaults site-defaults]]
   [app.server.pathom :as pathom]))

(defn api-handler [{:keys [body-params]}]
  {:status 200
   :body   (pathom/process body-params)})

(defn handler [{:keys [uri] :as req}]
  (if (= uri "/api")
    (api-handler req)
    {:status 404 :body "Not Found"}))

(def app
  (-> handler
      muuntaja/wrap-format
      (wrap-defaults (-> site-defaults
                         (assoc-in [:security :anti-forgery] false)))))
```

### 9. Create `src/dev/user.clj`

```clojure
(ns user
  (:require
   [clojure.tools.namespace.repl :refer [refresh]]
   [app.server.main :as server]))

(defonce system (atom nil))

(defn start []
  (when @system (throw (ex-info "Server already running" {})))
  (reset! system (server/start! {:port 3000}))
  :started)

(defn stop []
  (when @system
    (server/stop! @system)
    (reset! system nil))
  :stopped)

(defn restart []
  (stop)
  (refresh :after 'user/start))
```

### 10. Create `.clj-kondo/config.edn`

```clojure
{:linters {:unresolved-symbol    {:level :warning}
           :unresolved-namespace {:level :warning}}
 :lint-as {com.fulcrologic.fulcro.components/defsc clojure.core/defn
           com.fulcrologic.fulcro.mutations/defmutation clojure.core/defn
           com.fulcrologic.fulcro.routing.dynamic-routing/defrouter clojure.core/defn
           com.wsscode.pathom3.connect.operation/defresolver clojure.core/defn
           com.wsscode.pathom3.connect.operation/defmutation clojure.core/defn
           cljs.test/deftest clojure.test/deftest
           cljs.test/testing clojure.test/testing
           cljs.test/is      clojure.test/is}}
```

Import upstream exports so framework macros and library hooks are linted from their canonical configs:

```bash
clj-kondo --copy-configs --dependencies --lint "$(clj -A:dev -Spath)"
```

Commit `.clj-kondo/` to version control. Re-run `clj-kondo --copy-configs --dependencies` after Fulcro upgrades to pick up any new hook configuration.

### 11. Create `.cljfmt.edn`

```clojure
{:paths   ["src" "test"]
 :indents {ns          [[:inner 0]]
           defn        [[:inner 0]]
           fn          [[:inner 0]]
           defsc       [[:inner 0]]
           defmutation [[:inner 0]]
           defrouter   [[:inner 0]]
           defresolver [[:inner 0]]}}
```

### 12. Create `resources/public/index.html` and `resources/dev.html`

#### `resources/public/index.html` (production shell)

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title><Project Name></title>
    <link rel="stylesheet" href="/css/app.css" />
  </head>
  <body>
    <div id="app">Loading...</div>
    <script src="/js/main.js"></script>
  </body>
</html>
```

#### `resources/dev.html` (development shell)

The dev shell is identical to production for this minimal scaffold; copy the same content. Larger projects diverge here (e.g., to load source maps or skip CSP).

### 13. Create `.gitignore`

```
target/
out/
.cpcache/
.shadow-cljs/
.nrepl-port
node_modules/
resources/public/js/
*.log
```

### 14. First Compile and Test

```bash
npm install
npx shadow-cljs compile main
```

`shadow-cljs compile main` produces `resources/public/js/main.js` and validates the build configuration. If the compile fails, capture the output and report the error rather than continuing.

### 15. Report

Print the created files and next steps:

```
Created Fulcro project at <project-name>/.
Main namespace: app
Server namespace: app.server.main

Created:
  deps.edn
  shadow-cljs.edn
  package.json
  src/main/app/application.cljs
  src/main/app/client.cljs
  src/main/app/ui/root.cljs
  src/main/app/model/session.cljs
  src/main/app/server/main.clj
  src/main/app/server/pathom.clj
  src/main/app/server/middleware.clj
  src/dev/user.clj
  resources/public/index.html
  resources/dev.html
  .clj-kondo/config.edn
  .clj-kondo/imports/ (upstream exports)
  .cljfmt.edn
  .gitignore

Next steps:
  cd <project-name>
  npm install                                   # If not already run
  npx shadow-cljs watch main                    # Terminal 1: CLJS hot-reload
  clj -A:dev                                    # Terminal 2: server REPL
    user=> (start)                              # Start the Jetty server
  open http://localhost:8000                    # Dev HTTP server

Run /fulcro-fix to run the full quality pipeline.
Run /clj-fix for the host-neutral checks.
```

## Gotchas

- **Fulcro Inspect requires the Chrome extension.** Install [Fulcro Inspect](https://chrome.google.com/webstore/detail/fulcro-inspect) from the Chrome Web Store before running the dev build. The `:preloads` only register; the Inspect DevTools tab does not appear without the extension installed.
- **`binaryage/devtools` must be on the classpath.** The `:dev` alias adds it. Without it, the Inspect preload still works, but Clojure data in the browser console renders as opaque JS objects.
- **shadow-cljs build slot exhaustion.** `shadow-cljs watch main` holds a process; the second `compile` or `watch` invocation from a fresh shell may fail with "build target main is already running". Reach for `npx shadow-cljs stop` (or kill the watch process) before re-running.
- **`/api` route disables CSRF.** The middleware above sets `:anti-forgery false` for the API. Keep CSRF protection enabled elsewhere; an `/api` endpoint that authenticates user sessions through cookies needs additional protection (origin checks, token-bound requests).
- **Karma test build needs Chrome.** `npx shadow-cljs compile ci` writes a Karma test runner; running it requires `karma-chrome-launcher` and a Chrome (or Chromium) binary on `PATH`. The browser-test target (`:test`) only needs a browser to open the test page manually.
- **Namespace and directory naming.** Hyphens in namespaces, underscores in paths. `app.ui.root` lives at `src/main/app/ui/root.cljs`. The skill writes the right shapes; do not rename either by hand after generation.
- **Pathom 3 vs Pathom 2.** This scaffold uses Pathom 3 (`com.wsscode.pathom3.*`). Existing tutorials sometimes show Pathom 2 (`com.wsscode.pathom.*`); the APIs differ. When porting examples, translate against the [Pathom 3 migration guide](https://pathom3.wsscode.com/docs/migrate-from-pathom2/).
- **`fulcro-template` is the canonical reference scaffold.** When this skill drifts from upstream conventions, the template wins. Check [`fulcro-template`](https://github.com/fulcrologic/fulcro-template) before reporting a discrepancy as a bug.
