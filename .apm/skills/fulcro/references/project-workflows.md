# Fulcro Project Workflows

Reference for the Fulcro project shape (`deps.edn`, `shadow-cljs.edn`, `dev/user.clj`), the development loop, Pathom 3 server wiring, the test stack, the `guardrails` toggle, and the surrounding ecosystem.

Authoritative sources: [The Fulcro Developer's Guide](https://book.fulcrologic.com/), the [`fulcro` GitHub repo](https://github.com/fulcrologic/fulcro), and the [`fulcro-template`](https://github.com/fulcrologic/fulcro-template) scaffold.

The ClojureScript / shadow-cljs / npm specifics covered here are framework-specific; the host-neutral CLI, REPL, lint, format, and test commands live in the [`clojure-jvm`](https://github.com/brackendev/clojure-jvm-skills) and [`clojurescript`](https://github.com/brackendev/clojurescript-skills) baselines.

## Project Layout

A typical Fulcro project organized for both client and server work:

```
my-app/
├── deps.edn                         Clojure CLI dependencies and aliases
├── shadow-cljs.edn                  ClojureScript build configuration
├── package.json                     npm dependencies (shadow-cljs, react, react-dom)
├── build.clj                        tools.build for uberjar packaging
├── .clj-kondo/
│   └── config.edn                   Linter configuration (with Fulcro hooks)
├── .cljfmt.edn                      Formatter indentation for Fulcro macros
├── resources/
│   ├── public/
│   │   ├── index.html               Production HTML shell
│   │   └── js/                      Compiled JS output (gitignored)
│   ├── dev.html                     Development HTML shell
│   └── config.edn                   Public configuration (committed)
├── src/
│   ├── main/                        Production sources (shipped in JAR)
│   │   └── app/
│   │       ├── application.cljs     Fulcro app instance (default network, remotes)
│   │       ├── client.cljs          Mount + hot-reload entry point
│   │       ├── ui/
│   │       │   ├── root.cljs        Root component, router
│   │       │   └── *.cljs           Other UI components
│   │       ├── model/
│   │       │   ├── session.cljs     Client mutations
│   │       │   └── *.cljs           Domain mutations grouped by feature
│   │       └── server/              Clojure (JVM) sources
│   │           ├── main.clj         Server entry, main function
│   │           ├── pathom.clj       Pathom 3 parser + resolver registry
│   │           ├── resolvers/       Domain resolvers grouped by entity
│   │           ├── middleware.clj   Ring middleware (transit, csrf, sessions)
│   │           └── http.clj         HTTP server (Jetty / http-kit / Aleph)
│   ├── dev/                         Development-only sources (REPL, fixtures)
│   │   └── user.clj                 (start) / (stop) / (restart) helpers
│   └── test/
│       └── app/
│           └── *_test.cljs / *_test.clj
└── target/                          Build outputs (gitignored)
```

File naming uses underscores; namespace declarations use hyphens. `src/main/app/ui/root.cljs` corresponds to namespace `app.ui.root`.

## `deps.edn`

A minimal Fulcro project pulls in the framework, a server-side Pathom 3, and the usual dev / test / build aliases. The version numbers below are placeholders — check [Clojars](https://clojars.org/) for current releases before pinning.

```clojure
{:paths ["src/main" "resources"]

 :deps  {org.clojure/clojure         {:mvn/version "1.12.0"}
         org.clojure/clojurescript   {:mvn/version "1.12.42"}
         com.fulcrologic/fulcro      {:mvn/version "3.8.0"}
         com.fulcrologic/guardrails  {:mvn/version "1.2.9"}
         com.wsscode/pathom3         {:mvn/version "2025.01.16-alpha"}
         ring/ring-jetty-adapter     {:mvn/version "1.12.1"}
         ring/ring-defaults          {:mvn/version "0.5.0"}
         com.cognitect/transit-clj   {:mvn/version "1.0.333"}
         ch.qos.logback/logback-classic {:mvn/version "1.5.6"}}

 :aliases
 {:dev   {:extra-paths ["src/dev" "src/test"]
          :extra-deps  {thheller/shadow-cljs {:mvn/version "2.28.18"}
                        binaryage/devtools   {:mvn/version "1.0.7"}
                        org.clojure/tools.namespace {:mvn/version "1.5.0"}}}

  :test  {:extra-paths ["src/test"]
          :extra-deps  {lambdaisland/kaocha   {:mvn/version "1.91.1392"}
                        fulcrologic/fulcro-spec {:mvn/version "3.2.10"}}
          :main-opts   ["-m" "kaocha.runner"]}

  :cljfmt {:extra-deps {dev.weavejester/cljfmt {:mvn/version "0.13.0"}}
           :main-opts  ["-m" "cljfmt.main"]}

  :build {:deps       {io.github.clojure/tools.build {:mvn/version "0.10.5"}}
          :ns-default build}

  :dry4clj {:extra-deps {dry4clj/dry4clj {:mvn/version "0.1.1"}}
            :main-opts  ["-m" "dry4clj.core"]}}}
```

## `shadow-cljs.edn`

```clojure
{:source-paths ["src/main" "src/dev" "src/test"]
 :dependencies []                            ; deps come from deps.edn via :deps true
 :deps         true

 :nrepl        {:port 9000}

 :dev-http     {8000 "resources/public"}

 :builds
 {:main {:target     :browser
         :output-dir "resources/public/js"
         :asset-path "/js"
         :modules    {:main {:init-fn app.client/init}}
         :devtools   {:preloads [com.fulcrologic.fulcro.inspect.preload
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

`:devtools/:preloads` are critical for Fulcro Inspect. The `binaryage/devtools` dependency must also be on the classpath (declared in the `:dev` alias of `deps.edn`).

## `dev/user.clj`

```clojure
(ns user
  (:require
   [clojure.tools.namespace.repl :refer [refresh refresh-all]]
   [app.server.main :as server]
   [app.server.pathom :as pathom]))

(defonce system (atom nil))

(defn start []
  (when @system (throw (ex-info "Already running" {})))
  (reset! system (server/start! {:port 3000 :env pathom/env}))
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

Connect from an editor (CIDER, Calva, Cursive) to the nREPL port shadow-cljs prints on startup (default 9000). The JVM server runs in a different REPL session; the shadow-cljs REPL drives the browser-side code.

## Development Loop

```bash
npm install
npx shadow-cljs watch main           # terminal 1: ClojureScript watch + dev HTTP server
clj -A:dev                           # terminal 2: server REPL; (start) to launch Jetty
```

Open `http://localhost:8000` for the dev HTTP server (serving `resources/public`). The Fulcro client at `app.client/init` will mount; Inspect appears as a Chrome DevTools tab once the page has loaded.

The default file-save loop:

1. Edit a `.cljs` file under `src/main/`.
2. shadow-cljs recompiles and triggers `:after-load app.client/refresh`.
3. The browser hot-reloads without losing state.

Edits to `.clj` server files require either `(restart)` from the REPL or `(clojure.tools.namespace.repl/refresh)` followed by `(start)`. Reach for `(restart)` when components or the system map change; smaller edits often work without it.

## Production Build

```bash
# ClojureScript advanced compilation
npx shadow-cljs release main

# Uberjar (server + compiled CLJS in resources)
clj -T:build uber
```

The advanced build surfaces externs-inference warnings; see the [`clojurescript`](https://github.com/brackendev/clojurescript-skills) skill for the externs workflow. Fulcro's own code is externs-clean; warnings typically come from npm libraries used through string requires.

The uberjar wraps the server JAR with the compiled assets under `resources/public/js`. Deploy the uberjar to the target host and start it with `java -jar app.jar`.

## Pathom 3 Server Wiring

A minimal Pathom 3 parser and Ring handler:

```clojure
(ns app.server.pathom
  (:require
   [com.wsscode.pathom3.connect.indexes :as pci]
   [com.wsscode.pathom3.interface.eql :as p.eql]
   [app.server.resolvers.person :as person]
   [app.server.resolvers.session :as session]))

(def env
  (pci/register
    (concat person/resolvers
            session/resolvers)))

(defn process [eql-or-tx]
  (p.eql/process env eql-or-tx))
```

```clojure
(ns app.server.middleware
  (:require
   [muuntaja.middleware :as muuntaja]
   [ring.middleware.defaults :refer [wrap-defaults site-defaults]]
   [app.server.pathom :as pathom]))

(defn api-handler [{:keys [body-params]}]
  {:status 200
   :body   (pathom/process body-params)})

(def app
  (-> api-handler
      muuntaja/wrap-format
      (wrap-defaults (-> site-defaults
                         (assoc-in [:security :anti-forgery] false)))))
```

The Fulcro HTTP remote on the client sends transit-encoded EQL; the server reads transit (via muuntaja or hand-wired `transit-clj`), runs the parser, and returns the transit-encoded result. Set `:anti-forgery false` on the `/api` route only — keep CSRF protection elsewhere.

## Testing

| Layer | Tool | Notes |
|-------|------|-------|
| Server-side units | [`fulcro-spec`](https://github.com/fulcrologic/fulcro-spec) + [`kaocha`](https://github.com/lambdaisland/kaocha) | `clj -M:test` |
| Client-side units | `fulcro-spec` + shadow-cljs `:test` build | `npx shadow-cljs compile test`, open the test page |
| CI client tests | shadow-cljs `:ci` (Karma) | `npx shadow-cljs compile ci && npx karma start` |
| Integration | `fulcro-spec` with `fulcro/checksum-state` and component stubs | Run alongside the unit suites |

`fulcro-spec`'s `(specification ...)` / `(behavior ...)` / `(assertions ...)` macros are BDD-flavored wrappers over `clojure.test`. They produce structured output that Kaocha and the browser test runner both consume.

## `guardrails`

[`guardrails`](https://github.com/fulcrologic/guardrails) adds `(>defn name ...)` and friends for runtime function specs. It is disabled by default; enable with a JVM flag:

```bash
clj -J-Dguardrails.enabled=true -A:dev
```

For shadow-cljs, set the flag in `shadow-cljs.edn` under `:jvm-opts`. Guardrails is a development aid, not a production check; production builds should leave the flag off.

## Ecosystem

| Package | Purpose | Status |
|---------|---------|--------|
| [`fulcro-rad`](https://github.com/fulcrologic/fulcro-rad) | Rapid Application Development on top of Fulcro: attribute-based data modeling, generated forms and reports. | Active. Useful for CRUD-heavy apps; an application can mix RAD and hand-built UI. |
| [`fulcro-spec`](https://github.com/fulcrologic/fulcro-spec) | BDD testing macros for `clojure.test`. | Active. Standard in Fulcro test suites. |
| [`fulcro-i18n`](https://github.com/fulcrologic/fulcro-i18n) | Internationalization (string extraction, translations). | Active. Integrates with Fulcro 3. |
| [`fulcro-incubator`](https://github.com/fulcrologic/fulcro-incubator) | Experimental features that mature before joining core (additional state-machine patterns, pessimistic mutations, DB helpers). | Active, experimental. |
| [`statecharts`](https://github.com/fulcrologic/statecharts) | Standalone SCXML-based state machine library; can be used with or without Fulcro. | Active. More flexible than the built-in UISM for general state-machine work. |
| [`guardrails`](https://github.com/fulcrologic/guardrails) | Function specs with `(>defn ...)`; dev-only overhead. | Active. Standard in Fulcro projects that want spec-style contracts. |
| [`fulcro-garden-css`](https://github.com/fulcrologic/fulcro-garden-css) | Garden-based CSS-in-Clojure for Fulcro components. | Legacy. Most projects use Tailwind, CSS modules, or hand-written CSS instead. |

When using `fulcro-rad`, generated forms use `fo/*` (`com.fulcrologic.rad.form-options`) on top of the `fs/*` form-state algorithms; outside RAD, build the form body in component render and reach for `fs/*` directly.
