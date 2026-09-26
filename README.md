# fulcro-skills

[Fulcro](https://github.com/fulcrologic/fulcro) full-stack framework skills packaged as an [APM](https://github.com/microsoft/apm) plugin. One install deploys the full set to every runtime in APM's default target set: Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, Windsurf, Kiro, and Grok Build. Antigravity is supported by naming it explicitly with `--target antigravity`.

Skills follow the [Agent Skills](https://agentskills.io) open standard. Two auto-trigger from conversation context (`fulcro`, `fulcro-lenses`); the rest appear as slash commands.

This package layers on top of three companions: the host-neutral [clojure-skills](https://github.com/brackendev/clojure-skills) baseline, [clojurescript-skills](https://github.com/brackendev/clojurescript-skills) for the ClojureScript client, and [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) for the JVM Clojure server. Install all four together so the `clojure` skill handles cross-dialect style, the `clojurescript` skill handles JavaScript interop and the shadow-cljs workflow, the `clojure-jvm` skill handles Java interop and the Clojure CLI / `tools.build` workflow, and this package adds Fulcro-aware components, mutations, queries, loads, routing, forms, UI state machines, and the Pathom 3 server.

## Companion packages

Six sibling APM packages. This package layers on top of [clojure-skills](https://github.com/brackendev/clojure-skills), [clojurescript-skills](https://github.com/brackendev/clojurescript-skills), and [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) and covers the Fulcro framework. Install all four together for Fulcro projects. The Biff and ClojureDart packages are not required for Fulcro work.

| Package | Focus | Layers on |
|---------|-------|-----------|
| [clojure-skills](https://github.com/brackendev/clojure-skills) | Host-neutral [Clojure](https://clojure.org/) family baseline: style, naming, threading, collections, atoms, dispatch, formatting, namespaces, and testing. Triggers on `.clj`, `.cljs`, `.cljc`, `.cljd`. | — |
| [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) | [Clojure](https://clojure.org/) on the JVM: Java interop, refs / agents / STM, `with-open`, JVM-typed exceptions, `alter-var-root`, and the Clojure CLI / `tools.build` / `clj-kondo` / `cljfmt` / `test-runner` / nREPL workflow. | `clojure-skills` |
| [clojurescript-skills](https://github.com/brackendev/clojurescript-skills) | [ClojureScript](https://clojurescript.org/) style and tooling: JavaScript interop, externs inference, macro stage separation, `catch :default`, JS-flavored numbers and truthiness, and the `cljs.main` workflow. Triggers on `.cljs`, `.cljc` compiled to JS, `shadow-cljs.edn`, `figwheel-main.edn`. | `clojure-skills` |
| [biff-skills](https://github.com/brackendev/biff-skills) | [Biff](https://biffweb.com/) web framework on the JVM: scaffolding, conventions, deployment. | `clojure-skills` + `clojure-jvm-skills` |
| [fulcro-skills](https://github.com/brackendev/fulcro-skills) (this package) | [Fulcro](https://github.com/fulcrologic/fulcro) full-stack framework: `defsc` components, idents and the normalized client database, mutations, `df/load!`, dynamic routing, forms, UI state machines, Fulcro Inspect, and the Pathom 3 server. Triggers on `com.fulcrologic.fulcro.*`, `com.fulcrologic.rad.*`, `com.wsscode.pathom3.*`, `defsc`, `defmutation`, `defrouter`, `df/load!`, and ident vectors. | `clojure-skills` + `clojurescript-skills` + `clojure-jvm-skills` |
| [clojuredart-skills](https://github.com/brackendev/clojuredart-skills) | [ClojureDart](https://github.com/Tensegritics/ClojureDart) on [Flutter](https://flutter.dev/): Dart interop, type hints, `cljd.flutter` directives, async, FFI, REPL, and the Flutter project workflow. Triggers on `.cljd`, `cljd.flutter`. | `clojure-skills` |

## Install

Install [APM](https://github.com/microsoft/apm) first if you don't already have it. Then, in a project:

```bash
apm install brackendev/fulcro-skills --target all
apm install brackendev/clojurescript-skills --target all
apm install brackendev/clojure-jvm-skills --target all
apm install brackendev/clojure-skills --target all
```

Globally for your user account:

```bash
apm install brackendev/fulcro-skills -g --target all
apm install brackendev/clojurescript-skills -g --target all
apm install brackendev/clojure-jvm-skills -g --target all
apm install brackendev/clojure-skills -g --target all
```

Update later with `apm update [-g]`. Remove with `apm uninstall brackendev/fulcro-skills [-g]`. A local filesystem path can replace the shorthand at either scope.

## Requirements

- [Clojure CLI](https://clojure.org/guides/install_clojure) and Java 17 or higher for any skill in this package.
- [Node and npm](https://nodejs.org/) for the ClojureScript build via [shadow-cljs](https://github.com/thheller/shadow-cljs) (recommended).
- [clojure-skills](https://github.com/brackendev/clojure-skills) installed alongside, for the host-neutral baseline plus `/clj-fix` and `/clj-smells-fix`.
- [clojurescript-skills](https://github.com/brackendev/clojurescript-skills) installed alongside, for the ClojureScript-specific guidance the `fulcro` skill defers to (JavaScript interop, externs, macro stage separation, JS numerics, `cljs.main` / `shadow-cljs` workflow).
- [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) installed alongside, for the JVM-specific guidance the Fulcro server defers to (Java interop, JVM exceptions, refs / agents / STM, the Clojure CLI / `tools.build` / `clj-kondo` / `cljfmt` / `test-runner` / nREPL workflow).
- [clj-kondo](https://github.com/clj-kondo/clj-kondo) for the lint steps in the user-invoked skills below.
- The `fulcro-fix` dry step requires a [dry4clj](https://github.com/unclebob/dry4clj) `:dry4clj` alias in `deps.edn`.
- [Fulcro Inspect](https://chrome.google.com/webstore/detail/fulcro-inspect) Chrome extension for development. The `/fulcro-skills:fulcro-new` scaffolding registers the preloads; the DevTools tab only appears when the extension is installed.

## Command guide

A quick guide to every slash command. The detailed entries under [Skills](#skills) cover arguments and examples.

| Command | Use it when | What it does |
|---------|-------------------|--------------|
| `/fulcro-new` | Starting a new Fulcro full-stack project | Scaffolds the ClojureScript client and Clojure JVM server with lint, format, and a development REPL |
| `/fulcro-fix` | Lint, format, tests, or the advanced build need attention | Runs the fix pipeline across client and server, rewriting files in the format step with `cljfmt fix` |
| `/fulcro-upgrade` | The Fulcro version is behind | Upgrades `com.fulcrologic/fulcro` to the latest release and verifies compile and tests |
| `/fulcro-smells-fix` | Reserved for a future Fulcro smells fix | Placeholder that prints a not-yet-implemented notice pointing to `/clj-smells-fix` |

## Skills

User-invocable skills in this package share an argument grammar, scope vocabulary, and mutation-as-default rule. See [CONVENTIONS.md](CONVENTIONS.md) for the full standard.

### Scaffolding and quality

#### `/fulcro-new <project-name>`

Scaffold a new Fulcro full-stack project with a ClojureScript client (shadow-cljs build, Fulcro Inspect preloads), a Clojure JVM server (Pathom 3 parser, Ring + Jetty), `clj-kondo` lint setup, `cljfmt` formatting, and a development REPL.

```bash
/fulcro-new my-app
```

#### `/fulcro-fix [lint|format|test|advanced|dry] [--report] [all]`

Fix a Fulcro project. Runs lint, format, test, advanced-compilation, and duplicate-form checks. Defaults to the full sequence and writes formatting in place via `cljfmt fix`. Each step is also addressable on its own. The `test` step aggregates server (Kaocha) and client (shadow-cljs / Karma) suites; the `advanced` step runs a shadow-cljs release build as a production-build sanity check. Pass `--report` to swap the format step for non-writing `cljfmt check`; lint, test, advanced, and dry are pure-read of source regardless.

```bash
/fulcro-fix
/fulcro-fix lint
/fulcro-fix advanced
/fulcro-fix --report
```

#### `/fulcro-upgrade [--report] [all]`

Upgrade the `com.fulcrologic/fulcro` dependency in `deps.edn` to the latest released version on Clojars. Optionally updates `com.fulcrologic/fulcro-rad` and `com.fulcrologic/guardrails` alongside. Refreshes dependencies, runs compile + tests, and refreshes clj-kondo imports. Surfaces the upstream `CHANGELOG.adoc` for major version jumps and asks for confirmation before applying them. Reverts the version on compile or test failure. Pass `--report` to print the current and latest versions without writing or running anything.

```bash
/fulcro-upgrade
/fulcro-upgrade --report
```

#### `/fulcro-smells-fix [path|all] [--report]` (placeholder)

Reserves the command name for a future Fulcro-specific smells fix pipeline. When implemented, will mirror the mutation contract of `/clj-smells-fix` in `clojure-skills`: auto-apply Stage 1 mechanical findings and the Stage 2 `DEFECT`-tier safety band; report `SMELL` and `HINT` findings; honor `--report` to disable all writes. Currently prints a "not yet implemented" notice that points users at `/clj-smells-fix` in `clojure-skills` for host-neutral smells in `.clj`, `.cljc`, and `.cljs` files.

### Auto-triggered

These skills activate from conversation context. They cannot be invoked directly.

| Skill | Triggers |
|-------|----------|
| **fulcro** | `.cljs` / `.cljc` files containing `com.fulcrologic.fulcro.*`, `com.fulcrologic.fulcro-css.*`, `com.fulcrologic.rad.*`, `com.wsscode.pathom3.*` (or legacy `com.wsscode.pathom.*`); the macros `defsc`, `defmutation`, `defrouter`, `defresolver`; the helpers `df/load!`, `df/load-field!`, `dr/change-route!`, `comp/get-query`, `comp/get-ident`, `comp/get-initial-state`, `comp/computed`, `comp/factory`, `fs/*`, `fo/*`, `uism/*`; ident vectors `[:table/id value]` and `[:component/id ::ComponentName]`; `shadow-cljs.edn` with a `:main` build that requires Fulcro; `deps.edn` containing `com.fulcrologic/fulcro`, `com.fulcrologic/fulcro-rad`, `com.fulcrologic/guardrails`, or `com.wsscode/pathom3`; transit endpoints `/api`; Fulcro Inspect preloads; or any mention of Fulcro, fulcrologic, Fulcro Inspect, or Fulcro RAD. Covers `defsc` component definition, idents and the normalized client database, EQL queries, mutations and their `action` / `remote` / `ok-action` / `error-action` sections, `df/load!` and load targeting, dynamic routing, forms, UI state machines, the Pathom 3 server resolver shape, and the Fulcro Inspect workflow. Defers to the [clojure](https://github.com/brackendev/clojure-skills) skill for host-neutral style, to the [clojurescript](https://github.com/brackendev/clojurescript-skills) skill for JavaScript interop and the CLJS workflow, and to the [clojure-jvm](https://github.com/brackendev/clojure-jvm-skills) skill for Java interop on the server and the JVM tooling workflow. |
| **fulcro-lenses** | Auto-triggers alongside the [code-lenses](https://github.com/brackendev/code-lenses) plugin in Fulcro work. Layers on top of `clojure-lenses` (in `clojure-skills`) and `clojurescript-lenses` (in `clojurescript-skills`) and records only the Fulcro-specific deltas: the normalized client database, ident composition, mutations as data, EQL queries, the `action` / `remote` / `ok-action` / `error-action` split, `df/load!` over hand-rolled merges, dynamic routing, form-state algorithms, UI state machines, and the Pathom 3 server. Covers the four default code-lenses philosophies (grug, Honest Code, Tidy First, Parse Don't Validate) and the two opt-in philosophies (APOSD, Legacy Code) that activate when their lens is added with `+aposd` or `+legacy-code` or invoked directly. |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT. See [LICENSE](LICENSE).
