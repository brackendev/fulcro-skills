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

This skill is exempt from the `all` and `<path>` rows of the standard scope vocabulary because scaffolding has no useful default scope, as documented in `CONVENTIONS.md` in the fulcro-skills source repository.

## Mutation

Mutates by default: creates the project directory and writes `deps.edn`, `shadow-cljs.edn`, `package.json`, the client entry files (`app.application`, `app.client`, `app.ui.root`, `app.model.session`), the server files (`app.server.main`, `app.server.pathom`, `app.server.middleware`), `src/dev/user.clj`, `tests.edn`, a sample server test (`app.server.pathom-test`), `.clj-kondo/config.edn` plus upstream imports, `.cljfmt.edn`, `resources/public/index.html`, `resources/dev.html`, and `.gitignore`. Also runs `npm install`, `npx shadow-cljs compile main`, and `clj -M:test` as a first-build sanity check. No `--report` flag; preview the side effects by reading this `SKILL.md`.

## Prerequisites

Verify these are installed before proceeding. If any are missing, stop and tell the user.

- Java 21 or higher (required by shadow-cljs 3.x): `java --version`
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

Do the same for `com.fulcrologic/guardrails`, `com.fulcrologic/fulcro-inspect`, and `com.wsscode/pathom3`. If a fetch fails, ask the user for the versions to use. Do not invent a version number.

### 3. Create Project Directory and Source Roots

```bash
mkdir -p <project-name>/{src/main/app/ui,src/main/app/model,src/main/app/server,src/dev,src/test,resources/public/js,resources/public/css}
cd <project-name>
```

### 4. Create `deps.edn`

```clojure
{:paths ["src/main" "resources"]

 :deps  {org.clojure/clojure          {:mvn/version "1.12.6"}
         org.clojure/clojurescript    {:mvn/version "1.12.145"}
         com.fulcrologic/fulcro       {:mvn/version "<latest-fulcro-version>"}
         com.fulcrologic/guardrails   {:mvn/version "<latest-guardrails-version>"}
         com.wsscode/pathom3          {:mvn/version "<latest-pathom3-version>"}
         metosin/muuntaja             {:mvn/version "0.6.12"}
         ring/ring-jetty-adapter      {:mvn/version "1.15.5"}
         ring/ring-defaults           {:mvn/version "0.7.1"}
         com.cognitect/transit-clj    {:mvn/version "1.1.363"}
         ch.qos.logback/logback-classic {:mvn/version "1.6.5"}}

 :aliases
 {:dev    {:extra-paths ["src/dev" "src/test"]
           :extra-deps  {thheller/shadow-cljs           {:mvn/version "3.5.4"}
                         com.fulcrologic/fulcro-inspect {:mvn/version "<latest-fulcro-inspect-version>"}
                         binaryage/devtools             {:mvn/version "1.0.7"}
                         org.clojure/tools.namespace    {:mvn/version "1.5.1"}}}

  :test   {:extra-paths ["src/test"]
           :extra-deps  {lambdaisland/kaocha     {:mvn/version "1.91.1392"}
                         fulcrologic/fulcro-spec {:mvn/version "3.2.11"}}
           :main-opts   ["-m" "kaocha.runner"]}

  :cljfmt {:extra-deps {dev.weavejester/cljfmt {:mvn/version "0.16.6"}}
           :main-opts  ["-m" "cljfmt.main"]}

  :build  {:deps       {io.github.clojure/tools.build {:mvn/version "0.10.14"}}
           :ns-default build}

  :dry4clj {:extra-deps {io.github.unclebob/dry4clj {:git/url "https://github.com/unclebob/dry4clj.git"
                                                     :git/sha "5994ef2bcaaae8d12bcd0afb4f091ae09083228c"}}
            :main-opts  ["-m" "dry4clj.core"]}}}
```

The fixed versions above were current on 2026-10-04. Keep `thheller/shadow-cljs` equal to the `shadow-cljs` npm version in `package.json`. `dry4clj` is not published to Clojars, so the alias uses a Git coordinate pinned to a commit.

### 5. Create `shadow-cljs.edn`

```clojure
{:deps         {:aliases [:dev]}             ; source paths and libraries come from deps.edn
 :nrepl        {:port 9000}
 :dev-http     {8000 "resources/public"}

 :builds
 {:main {:target     :browser
         :output-dir "resources/public/js"
         :asset-path "/js"
         :modules    {:main {:init-fn app.client/init}}
         :devtools   {:preloads   [com.fulcrologic.devtools.chrome-preload
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
    "shadow-cljs": "3.5.4",
    "karma": "6.4.4",
    "karma-chrome-launcher": "3.2.0",
    "karma-cljs-test": "0.1.0"
  },
  "dependencies": {
    "react": "19.3.0",
    "react-dom": "19.3.0"
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
    [com.fulcrologic.devtools.common.target :refer [ido]]
    [com.fulcrologic.fulcro.application :as app]
    [fulcro.inspect.tool :as it]
    [app.application :refer [SPA]]
    [app.ui.root :as root]))

(defn ^:export init []
  (ido (it/add-fulcro-inspect! SPA))
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

(defsc Root [_this {:root/keys [greeting]}]
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

### 9. Create `src/dev/user.clj`, `tests.edn`, and a Sample Test

#### `src/dev/user.clj`

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

#### `tests.edn`

Kaocha looks in `test/` by default. Point it at `src/test`, where this layout keeps tests:

```clojure
#kaocha/v1
{:tests [{:id         :unit
          :test-paths ["src/test"]}]}
```

#### `src/test/app/server/pathom_test.clj`

A server test so `clj -M:test` exercises the Pathom parser from the first run:

```clojure
(ns app.server.pathom-test
  (:require
    [clojure.test :refer [deftest]]
    [fulcro-spec.core :refer [assertions]]
    [app.server.pathom :as pathom]))

(deftest server-greeting-test
  (assertions
    "resolves the root greeting"
    (pathom/process [:root/greeting]) => {:root/greeting "Hello from the Pathom server"}))
```

### 10. Create `.clj-kondo/config.edn`

```clojure
{:linters {:unresolved-symbol    {:level   :warning
                                  :exclude [(fulcro-spec.core/assertions [=> =fn=> =throws=> =check=>])]}
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

fulcro-spec ships no clj-kondo configuration, so the `:exclude` entry keeps its `=>` arrows inside `assertions` from being reported as unresolved symbols.

Import upstream exports so framework macros and library hooks are linted from their canonical configs:

```bash
clj-kondo --copy-configs --dependencies --lint "$(clj -A:dev -Spath)"
```

Commit `.clj-kondo/` to version control. Re-run `clj-kondo --copy-configs --dependencies` after Fulcro upgrades to pick up any new hook configuration.

### 11. Create `.cljfmt.edn`

```clojure
{:paths                          ["src"]
 :function-arguments-indentation :cursive
 :extra-indents                  {defsc         [[:inner 0]]
                                  defmutation   [[:inner 0]]
                                  defrouter     [[:inner 0]]
                                  defresolver   [[:inner 0]]
                                  action        [[:inner 0]]
                                  remote        [[:inner 0]]
                                  ok-action     [[:inner 0]]
                                  error-action  [[:inner 0]]
                                  result-action [[:inner 0]]}}
```

Use `:extra-indents`, not `:indents`. In current cljfmt, `:indents` replaces every default rule, so `def`, `if`, `when`, and `let` lose their standard indentation. `:cursive` gives the two-space argument indentation that Fulcro's own source and `fulcro-template` use. The mutation section names (`action`, `remote`, and the result sections) keep `defmutation` bodies at two spaces.

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
clj -M:test
```

`shadow-cljs compile main` produces `resources/public/js/main.js` and validates the build configuration. `clj -M:test` runs the sample server test through Kaocha. If either command fails, capture the output and report the error rather than continuing.

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
  src/test/app/server/pathom_test.clj
  tests.edn
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

- **Fulcro Inspect needs the extension or the Electron app.** Install the Chrome extension (`fulcro-inspect-chrome-<version>.zip`) or the Electron app from the [fulcro-inspect releases page](https://github.com/fulcrologic/fulcro-inspect/releases), then open a new tab or restart Chrome. The old Chrome Web Store extension does not work with Fulcro 3.8 or later. The scaffold uses `com.fulcrologic.devtools.chrome-preload`. Switch it to `com.fulcrologic.devtools.electron-preload` for the Electron app.
- **Inspect needs both the preload and the install call.** `app.client/init` calls `(ido (it/add-fulcro-inspect! SPA))`. `ido` removes the call from release builds. Removing the call leaves the preload connected but shows no app in Inspect. The pre-3.8 preload `com.fulcrologic.fulcro.inspect.preload` only logs "Inspect NOT installed".
- **shadow-cljs must read the `:dev` alias.** shadow-cljs, `fulcro-inspect`, and `binaryage/devtools` live in the `:dev` alias, so `shadow-cljs.edn` sets `:deps {:aliases [:dev]}`. With `:deps true`, the build cannot find those libraries.
- **shadow-cljs build slot exhaustion.** `shadow-cljs watch main` holds a process; the second `compile` or `watch` invocation from a fresh shell may fail with "build target main is already running". Reach for `npx shadow-cljs stop` (or kill the watch process) before re-running.
- **`/api` route disables CSRF.** The middleware above sets `:anti-forgery false` for the API. Keep CSRF protection enabled elsewhere; an `/api` endpoint that authenticates user sessions through cookies needs additional protection (origin checks, token-bound requests).
- **Karma test build needs Chrome.** `npx shadow-cljs compile ci` writes a Karma test runner; running it requires `karma-chrome-launcher` and a Chrome (or Chromium) binary on `PATH`. The browser-test target (`:test`) only needs a browser to open the test page manually.
- **Namespace and directory naming.** Hyphens in namespaces, underscores in paths. `app.ui.root` lives at `src/main/app/ui/root.cljs`. The skill writes the right shapes; do not rename either by hand after generation.
- **Pathom 3 vs Pathom 2.** This scaffold uses Pathom 3 (`com.wsscode.pathom3.*`). Existing tutorials sometimes show Pathom 2 (`com.wsscode.pathom.*`); the APIs differ. When porting examples, translate against the [Pathom 3 migration guide](https://pathom3.wsscode.com/docs/migrate-from-pathom2/).
- **`fulcro-template` is the canonical reference scaffold.** When this skill drifts from upstream conventions, the template wins. Check [`fulcro-template`](https://github.com/fulcrologic/fulcro-template) before reporting a discrepancy as a bug.
