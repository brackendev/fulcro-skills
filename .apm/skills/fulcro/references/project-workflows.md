# Fulcro Project Workflows

Reference for the Fulcro project shape (`deps.edn`, `shadow-cljs.edn`, `dev/user.clj`), the development loop, Pathom 3 server wiring, the test stack, the `guardrails` toggle, and the surrounding ecosystem.

Authoritative sources: [The Fulcro Developer's Guide](https://book.fulcrologic.com/), the [`fulcro` GitHub repo](https://github.com/fulcrologic/fulcro), and the [`fulcro-template`](https://github.com/fulcrologic/fulcro-template) scaffold. The template's dependency pins and server code do not match the current releases, so take its directory layout and client setup from it, and take the versions and the Pathom 3 server from this file.

The ClojureScript / shadow-cljs / npm specifics covered here are framework-specific; the host-neutral CLI, REPL, lint, format, and test commands live in the [`clojure-jvm`](https://github.com/brackendev/clojure-jvm-skills) and [`clojurescript`](https://github.com/brackendev/clojurescript-skills) baselines.

## Project Layout

A typical Fulcro project organized for both client and server work:

```
my-app/
├── deps.edn                         Clojure CLI dependencies and aliases
├── shadow-cljs.edn                  ClojureScript build configuration
├── package.json                     npm dependencies (shadow-cljs, react, react-dom)
├── tests.edn                        Kaocha configuration (test paths)
├── karma.conf.js                    Karma configuration for the :ci client test build
├── build.clj                        tools.build for uberjar packaging
├── .clj-kondo/
│   └── config.edn                   Linter configuration (with Fulcro hooks)
├── .cljfmt.edn                      cljfmt configuration (the one /fulcro-fix recommends)
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

A minimal Fulcro project pulls in the framework, a server-side Pathom 3, and the usual dev / test / build aliases. The versions below were current on 2026-10-25 (Fulcro 3.10.0). Pathom 3 publishes only alpha releases, and the pin is the newest one. Check [Clojars](https://clojars.org/) and Maven Central for newer releases before pinning.

```clojure
{:paths ["src/main" "resources"]

 :deps  {org.clojure/clojure         {:mvn/version "1.12.6"}
         org.clojure/clojurescript   {:mvn/version "1.12.145"}
         com.fulcrologic/fulcro      {:mvn/version "3.10.0"}
         com.fulcrologic/guardrails  {:mvn/version "1.3.4"}
         com.wsscode/pathom3         {:mvn/version "2025.01.16-alpha"}
         metosin/muuntaja            {:mvn/version "0.6.12"}
         ring/ring-jetty-adapter     {:mvn/version "1.15.6"}
         ring/ring-defaults          {:mvn/version "0.7.1"}
         com.cognitect/transit-clj   {:mvn/version "1.1.363"}
         ch.qos.logback/logback-classic {:mvn/version "1.6.6"}}

 :aliases
 {:dev   {:extra-paths ["src/dev" "src/test"]
          :extra-deps  {thheller/shadow-cljs           {:mvn/version "3.5.7"}
                        com.fulcrologic/fulcro-inspect {:mvn/version "1.0.5"}
                        binaryage/devtools             {:mvn/version "1.0.7"}
                        org.clojure/tools.namespace    {:mvn/version "1.5.1"}
                        fulcrologic/fulcro-spec        {:mvn/version "3.2.11"}}}

  :test  {:extra-paths ["src/test"]
          :extra-deps  {lambdaisland/kaocha   {:mvn/version "1.91.1392"}
                        fulcrologic/fulcro-spec {:mvn/version "3.2.11"}}
          :main-opts   ["-m" "kaocha.runner"]}

  :cljfmt {:replace-deps {dev.weavejester/cljfmt {:mvn/version "0.16.6"}}
           :main-opts    ["-m" "cljfmt.main"]}

  :build {:deps       {io.github.clojure/tools.build {:mvn/version "0.10.14"}}
          :ns-default build}

  :dry4clj {:extra-deps {io.github.unclebob/dry4clj {:git/url "https://github.com/unclebob/dry4clj.git"
                                                     :git/sha "5994ef2bcaaae8d12bcd0afb4f091ae09083228c"}}
            :main-opts  ["-m" "dry4clj.core"]}}}
```

shadow-cljs 3.x requires Java 21 or later. Keep the `thheller/shadow-cljs` version in `deps.edn` equal to the `shadow-cljs` npm package version in `package.json`. `fulcro-spec` appears in both `:dev` and `:test`: shadow-cljs reads only the `:dev` alias, so the client test builds need it there, and Kaocha runs with `:test`. Keep the two versions equal. `/fulcro-upgrade` updates both. Its Clojars group is `fulcrologic`, not `com.fulcrologic`. `dry4clj` is not published to Clojars, so the alias uses a Git coordinate pinned to a commit. The `:cljfmt` alias uses `:replace-deps`, so the formatter runs without the project's dependencies, which it does not need.

## `package.json`, `tests.edn`, and `karma.conf.js`

```json
{
  "name": "my-app",
  "private": true,
  "devDependencies": {
    "shadow-cljs": "3.5.7"
  },
  "dependencies": {
    "react": "19.3.1",
    "react-dom": "19.3.1"
  }
}
```

The `:ci` build runs under Karma. Add its runner with `npm install --save-dev karma karma-chrome-launcher karma-cljs-test`, and point Karma at the build output:

```javascript
// karma.conf.js
module.exports = function (config) {
  config.set({
    browsers:   ['ChromeHeadless'],
    basePath:   'target',
    files:      ['karma-test.js'],
    frameworks: ['cljs-test'],
    plugins:    ['karma-cljs-test', 'karma-chrome-launcher'],
    client:     {args: ['shadow.test.karma.init']}
  })
};
```

Kaocha looks for tests in `test/` unless `tests.edn` says otherwise. Without this file, `clojure -M:test` finds no tests in `src/test`, prints a warning, and still exits 0:

```clojure
#kaocha/v1
{:tests [{:id           :unit
          :test-paths   ["src/test"]
          :source-paths ["src/main"]}]}
```

## `shadow-cljs.edn`

```clojure
{:deps         {:aliases [:dev]}             ; source paths and libraries come from deps.edn, including the :dev alias

 :nrepl        {:port 9000}

 :dev-http     {8000 {:root      "resources/public"
                      :proxy-url "http://localhost:3000"}} ; /api and other unknown paths go to the Jetty server

 :builds
 {:main {:target     :browser
         :output-dir "resources/public/js"
         :asset-path "/js"
         :modules    {:main {:init-fn app.client/init}}
         :dev        {:compiler-options {:external-config {:guardrails {}}}} ; guardrails checks in dev builds only
         :devtools   {:preloads [com.fulcrologic.devtools.chrome-preload
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

`:deps {:aliases [:dev]}` matters. shadow-cljs, `fulcro-inspect`, `binaryage/devtools`, and the client copy of `fulcro-spec` are declared in the `:dev` alias, and `:deps true` would build without them. With `:deps`, shadow-cljs also ignores `:jvm-opts` in `shadow-cljs.edn`, so JVM options belong in the `deps.edn` alias. With `:deps`, shadow-cljs ignores `:source-paths` and uses the `deps.edn` paths, so `src/dev` and `src/test` come from the alias's `:extra-paths`. The `chrome-preload` connects the app to the Fulcro Inspect Chrome extension. Use `com.fulcrologic.devtools.electron-preload` for the Electron app instead. The preload alone does not register the app: `app.client/init` must also call `(ido (it/add-fulcro-inspect! SPA))`, as shown in `references/inspect.md`. General shadow-cljs configuration is covered by the [`clojurescript`](https://github.com/brackendev/clojurescript-skills) skill.

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

Connect from an editor (CIDER, Calva, Cursive) to the nREPL port shadow-cljs prints on startup (9000, set by `:nrepl {:port 9000}` above). The JVM server runs in a different REPL session; the shadow-cljs REPL drives the browser-side code.

## Development Loop

```bash
npm install
npx shadow-cljs watch main           # terminal 1: ClojureScript watch + dev HTTP server
clj -A:dev                           # terminal 2: server REPL; (start) to launch Jetty
```

Open `http://localhost:8000` for the dev HTTP server (serving `resources/public`). The Fulcro client at `app.client/init` will mount. The dev HTTP server serves only files under `:root`. Its `:proxy-url` forwards every other request, including the client's `/api` calls, to the Jetty server on port 3000, so start the server with `(start)` before loading data. Inspect appears as a Chrome DevTools tab, or in the Electron app, once the page has loaded and `add-fulcro-inspect!` has run. `references/inspect.md` covers installing the extension or the Electron app.

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
clojure -T:build uber
```

The advanced build surfaces externs-inference warnings; see the [`clojurescript`](https://github.com/brackendev/clojurescript-skills) skill for the externs workflow. Fulcro's own code is externs-clean; warnings typically come from npm libraries used through string requires.

The uberjar wraps the server JAR with the compiled assets under `resources/public/js`. Deploy the uberjar to the target host and start it with `java -jar app.jar`.

### Removing Fulcro's log calls from the release build

Fulcro logs through [Timbre](https://github.com/taoensso/timbre). Each log call keeps its source location in a release build, so the bundle contains the absolute path of the Fulcro jar on the build machine, for example `jar:file:/Users/<name>/.m2/repository/com/fulcrologic/fulcro/<version>/fulcro-<version>.jar!/com/fulcrologic/fulcro/application.cljc`. Timbre removes every log call below a minimum level at compile time when the compiling JVM sees the `TAOENSSO_TIMBRE_MIN_LEVEL_EDN` environment variable or the `taoensso.timbre.min-level.edn` JVM property. The Fulcro Developer's Guide describes this under [Removing Debug Logging](https://book.fulcrologic.com/#_removing_debug_logging).

1. **Choose the level.** `:report` removes Fulcro's warnings and errors along with its debug logging, and every embedded path with them. It also removes the application's own Timbre warnings and errors, so the production bundle logs nothing to the console through Timbre. `:warn` keeps warnings and errors, and the paths in those calls stay in the bundle.
2. **Keep the release build out of the compiler cache.** shadow-cljs caches compiled namespaces, Fulcro's included, and a cached namespace keeps the log calls it was compiled with. Block caching for Timbre in release builds. A namespace that requires a blocked namespace is not cached either, so every Fulcro namespace that logs recompiles on each release build:

   ```clojure
   :builds
   {:main {:target  :browser
           ;; ...
           :release {:build-options {:cache-blockers #{taoensso.timbre}}}}}
   ```

3. **Build in a fresh JVM.** Timbre reads the level once, when the compiler loads it. The shadow-cljs CLI sends commands to a running server, including one that `watch` started, and that server does not see a variable set in another terminal. Stop the server, then run:

   ```bash
   TAOENSSO_TIMBRE_MIN_LEVEL_EDN=:report npx shadow-cljs release main
   ```

   `--force-spawn` (`npx shadow-cljs release main --force-spawn`) also starts a fresh JVM while a server keeps running.

Check the bundle with `grep -c "$HOME" resources/public/js/main.js`, which prints `0` when no path under the home directory remains.

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
    (concat person/resolvers session/resolvers)))

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
    (wrap-defaults (assoc-in site-defaults [:security :anti-forgery] false))))
```

The Fulcro HTTP remote on the client sends transit-encoded EQL with `Accept: application/transit+json`; the server reads transit (via muuntaja or hand-wired `transit-clj`), runs the parser, and returns the transit-encoded result. This handler sends every request to the parser and turns off anti-forgery for the whole stack. To keep CSRF protection on, leave `:anti-forgery` on, put the request's `:anti-forgery-token` into the HTML page the server renders, and set the HTTP remote's `:request-middleware` to `(-> (http/wrap-csrf-token token) (http/wrap-fulcro-request))`, as `fulcro-template` does. A custom `:request-middleware` replaces the default `wrap-fulcro-request`, so keep it in the chain, or the request is sent without transit encoding.

## Testing

| Layer | Tool | Notes |
|-------|------|-------|
| Server-side units | [`fulcro-spec`](https://github.com/fulcrologic/fulcro-spec) + [`kaocha`](https://github.com/lambdaisland/kaocha) | `clojure -M:test` |
| Client-side units | `fulcro-spec` + shadow-cljs `:test` build | `npx shadow-cljs watch test`, then open `http://localhost:8021`, which the build's `:devtools` HTTP server serves while shadow-cljs runs |
| CI client tests | shadow-cljs `:ci` (Karma) | `npx shadow-cljs compile ci && npx karma start --single-run` |
| Headless integration | `com.fulcrologic.fulcro.headless` with `fulcro-spec` | `clojure -M:test`, with components in `.cljc` files |

`fulcro-spec`'s `(specification ...)` / `(behavior ...)` / `(assertions ...)` macros are BDD-flavored wrappers over `clojure.test`. They produce structured output that Kaocha and the browser test runner both consume. `provided` and `when-mocking` stub functions inside a test.

Fulcro runs headless on the JVM for full-stack tests without a browser. `(h/build-test-app {:root-class Root :remotes {:remote (lr/sync-remote handler)}})` from `com.fulcrologic.fulcro.headless` and `com.fulcrologic.fulcro.headless.loopback-remotes` builds an app with synchronous transaction processing and captured render frames. `lr/pathom-remote` and `lr/fulcro-ring-remote` route the remote through a Pathom parser or the real Ring stack. With Pathom 3, `(lr/pathom-remote p.eql/process :env-fn (fn [] pathom/env))` calls `(p.eql/process env eql)` for each request. `h/last-frame` returns the latest render, and `com.fulcrologic.fulcro.headless.hiccup` (`rendered-tree->hiccup`, `find-by-id`, `click!`, `type-text!`) inspects and drives the rendered output. Components must be in `.cljc` files so the JVM can load them. Transaction and network capture (`h/captured-transactions`, `h/captured-network-events`) requires the JVM property `-Dcom.fulcrologic.fulcro.inspect=true`.

## `guardrails`

[`guardrails`](https://github.com/fulcrologic/guardrails) adds `(>defn name ...)` and friends for runtime function specs. It is disabled by default; enable with a JVM flag:

```bash
clj -J-Dguardrails.enabled=true -A:dev   # server REPL with guardrails checks on
```

For the ClojureScript build, the guardrails README enables checks through the compiler's external configuration: the `:main` build above sets `{:external-config {:guardrails {}}}` under `:dev :compiler-options`, so dev builds check and release builds do not. A `:jvm-opts` entry in `shadow-cljs.edn` has no effect here, because `:deps` hands JVM options to `deps.edn`. Guardrails is a development aid, not a production check; production builds should leave it off.

## Ecosystem

| Package | Purpose | Status |
|---------|---------|--------|
| [`fulcro-rad`](https://github.com/fulcrologic/fulcro-rad) | Rapid Application Development on top of Fulcro: attribute-based data modeling, generated forms and reports. | Active. Useful for CRUD-heavy apps; an application can mix RAD and hand-built UI. |
| [`fulcro-inspect`](https://github.com/fulcrologic/fulcro-inspect) | Development-time debugger: the library plus the Chrome extension or the Electron app. | Active. Fulcro depends on the library, and the app registers it with `add-fulcro-inspect!`. |
| [`fulcro-spec`](https://github.com/fulcrologic/fulcro-spec) | BDD testing macros for `clojure.test`. | Active. Standard in Fulcro test suites. |
| [`fulcro-i18n`](https://github.com/fulcrologic/fulcro-i18n) | Internationalization (string extraction, translations). | Active. Integrates with Fulcro 3. |
| [`statecharts`](https://github.com/fulcrologic/statecharts) | Standalone SCXML-based state machine library; can be used with or without Fulcro. | Active. More flexible than the built-in UISM for general state-machine work. |
| [`guardrails`](https://github.com/fulcrologic/guardrails) | Function specs with `(>defn ...)`; dev-only overhead. | Active. Standard in Fulcro projects that want spec-style contracts. |
| [`fulcro-garden-css`](https://github.com/fulcrologic/fulcro-garden-css) | Garden-based CSS-in-Clojure for Fulcro components. | Optional. Fulcro components work with any CSS approach, and `fulcro-template` uses this library. |

`references/forms.md` covers how RAD's `fo/*` form options relate to the `fs/*` form-state algorithms.
