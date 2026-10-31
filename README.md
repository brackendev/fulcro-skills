# fulcro-skills

[Fulcro](https://github.com/fulcrologic/fulcro) 3 full-stack framework skills packaged as an [APM](https://github.com/microsoft/apm) plugin. One install deploys the full set to every runtime in APM's default target set: Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, Windsurf, Kiro, and Grok Build. Antigravity is supported by naming it explicitly with `--target antigravity`.

Skills follow the [Agent Skills](https://agentskills.io) open standard. One auto-triggers from conversation context (`fulcro`). The rest appear as slash commands. The agent can also run `/fulcro-fix` from a natural-language request. `/fulcro-upgrade` runs only as a typed slash command. Runtimes differ in how they offer these commands: Codex lists them under `/skills` and runs them as `$name`, OpenCode attaches them with `@name`, and Gemini CLI does not read the user-only setting, so its agent can start any of them.

This package layers on top of three companions: the host-neutral [clojure-skills](https://github.com/brackendev/clojure-skills) baseline, [clojurescript-skills](https://github.com/brackendev/clojurescript-skills) for the ClojureScript client, and [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) for the JVM Clojure server. Install all four together so the `clojure` skill handles cross-dialect style, the `clojurescript` skill handles JavaScript interop and the shadow-cljs workflow, the `clojure-jvm` skill handles Java interop and the Clojure CLI / `tools.build` workflow, and this package adds Fulcro-aware components, mutations, queries, loads, routing, forms, UI state machines, and the Pathom 3 server.

Pathom 3 is published only as alpha releases, so the skills target the newest published Pathom 3 release.

## Companion packages

Eight sibling APM packages. This package layers on top of [clojure-skills](https://github.com/brackendev/clojure-skills), [clojurescript-skills](https://github.com/brackendev/clojurescript-skills), and [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) and covers the Fulcro framework. Install all four together for Fulcro projects. The Biff, Electric, ClojureDart, and Babashka packages are not required for Fulcro work.

| Package | Focus | Layers on |
|---------|-------|-----------|
| [clojure-skills](https://github.com/brackendev/clojure-skills) | Host-neutral [Clojure](https://clojure.org/) family baseline: style, naming, threading, collections, atoms, dispatch, formatting, namespaces, and testing. Triggers on `.clj`, `.cljs`, `.cljc`, `.cljd`. | — |
| [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) | [Clojure](https://clojure.org/) on the JVM: Java interop, refs / agents / STM, `with-open`, JVM-typed exceptions, `alter-var-root`, and the Clojure CLI / `tools.build` / `clj-kondo` / `cljfmt` / `test-runner` / nREPL workflow. | `clojure-skills` |
| [clojurescript-skills](https://github.com/brackendev/clojurescript-skills) | [ClojureScript](https://clojurescript.org/) style and tooling: JavaScript interop, externs inference, macro stage separation, `catch :default`, JS-flavored numbers and truthiness, and the `cljs.main` workflow. Triggers on `.cljs`, `.cljc` compiled to JS, `shadow-cljs.edn`, `figwheel-main.edn`. | `clojure-skills` |
| [babashka-skills](https://github.com/brackendev/babashka-skills) | [Babashka](https://babashka.org/) scripting host: built-in namespaces (`babashka.fs`, `babashka.process`, `babashka.http-client`), `bb.edn` tasks, subprocess exit checks, threads and process exit, records and types without Java classes, testing, pods, and packaging. Triggers on `.bb`, `bb.edn`. | `clojure-skills` |
| [biff-skills](https://github.com/brackendev/biff-skills) | [Biff](https://biffweb.com/) 2 web framework on the JVM: conventions and deployment. | `clojure-skills` + `clojure-jvm-skills` |
| [electric-skills](https://github.com/brackendev/electric-skills) | [Electric](https://github.com/hyperfiddle/electric) v3 full-stack reactive framework: `e/defn` components, `e/client` / `e/server` site transfer, `e/for` over `e/diff-by`, `e/watch` state, `e/Token` transactions, `e/Offload`, the `electric-dom3` and `electric-forms5` APIs, and the single-JVM shadow-cljs workflow. Triggers on `hyperfiddle.electric3`, `hyperfiddle.electric-dom3`, `e/defn`, `e/client`, `e/server`, and `com.hyperfiddle/electric`. | `clojure-skills` + `clojurescript-skills` + `clojure-jvm-skills` |
| [fulcro-skills](https://github.com/brackendev/fulcro-skills) (this package) | [Fulcro](https://github.com/fulcrologic/fulcro) 3 full-stack framework: `defsc` components, idents and the normalized client database, mutations, `df/load!`, dynamic routing, forms, UI state machines, Fulcro Inspect, and the Pathom 3 server. Triggers on `com.fulcrologic.fulcro.*`, `com.fulcrologic.rad.*`, `com.wsscode.pathom3.*`, `defsc`, `defmutation`, `defrouter`, `df/load!`, and ident vectors. | `clojure-skills` + `clojurescript-skills` + `clojure-jvm-skills` |
| [clojuredart-skills](https://github.com/brackendev/clojuredart-skills) | [ClojureDart](https://github.com/Tensegritics/ClojureDart) on [Flutter](https://flutter.dev/): Dart interop, type hints, `cljd.flutter` directives, async, FFI, REPL, and the Flutter project workflow. Triggers on `.cljd`, `cljd.flutter`. | `clojure-skills` |

## Install

Install [APM](https://github.com/microsoft/apm) 0.34.0 first. Homebrew offers only the newest release, so remove a Homebrew copy with `brew uninstall apm` and install from PyPI with [uv](https://docs.astral.sh/uv/getting-started/installation/): `uv tool install apm-cli==0.34.0`. APM 0.35.0 is not supported because it can delete installed skill files ([microsoft/apm#3372](https://github.com/microsoft/apm/issues/3372)). Then, in a project:

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

- [Clojure CLI](https://clojure.org/guides/install_clojure) and Java 21 or higher. shadow-cljs, which builds the Fulcro client, requires Java 21.
- [Node and npm](https://nodejs.org/) for the ClojureScript build via [shadow-cljs](https://github.com/thheller/shadow-cljs) (recommended).
- [clojure-skills](https://github.com/brackendev/clojure-skills) installed alongside, for the host-neutral baseline plus `/clj-fix` and `/clj-smells-fix`.
- [clojurescript-skills](https://github.com/brackendev/clojurescript-skills) installed alongside, for the ClojureScript-specific guidance the `fulcro` skill defers to (JavaScript interop, externs, macro stage separation, JS numerics, `cljs.main` / `shadow-cljs` workflow).
- [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) installed alongside, for the JVM-specific guidance the Fulcro server defers to (Java interop, JVM exceptions, refs / agents / STM, the Clojure CLI / `tools.build` / `clj-kondo` / `cljfmt` / `test-runner` / nREPL workflow).
- [clj-kondo](https://github.com/clj-kondo/clj-kondo) for the lint steps in the user-invoked skills below.
- The `fulcro-fix` dry step requires a [dry4clj](https://github.com/unclebob/dry4clj) `:dry4clj` alias in `deps.edn`. dry4clj is not published to Clojars, so the alias uses a Git coordinate. Without the alias, the dry step reports that it was skipped and shows the alias to add.
- The Fulcro Inspect Chrome extension or Electron app for development. The extension is available from the Chrome Web Store and from the [fulcro-inspect releases page](https://github.com/fulcrologic/fulcro-inspect/releases), and the Electron app from the releases page. The project also needs the `fulcro-inspect` library, its preload, and the call that registers the app, which the `fulcro` skill's Fulcro Inspect reference describes. The Inspect tab only appears when the extension or app is installed.
- [Babashka](https://babashka.org) (`bb`) to run the `fulcro-rules` hook. Without it, the hook does nothing and prints a notice on standard error.

## Command guide

A quick guide to every slash command. The detailed entries under [Skills](#skills) cover arguments and examples.

| Command | Use it when | What it does |
|---------|-------------------|--------------|
| `/fulcro-fix` | Lint, format, tests, or the advanced build need attention | Runs the fix pipeline across client and server, rewriting files in the format step with `cljfmt fix` |
| `/fulcro-upgrade` | The Fulcro version is behind | Reads the upstream changelog and commit history for the version range, upgrades `com.fulcrologic/fulcro` and the related Fulcro libraries, including `fulcro-spec`, to their latest stable releases, and verifies compile and tests |

## Skills

User-invocable skills in this package share an argument grammar, scope vocabulary, and mutation-as-default rule. See [CONVENTIONS.md](CONVENTIONS.md) for the full standard.

### Quality and upgrades

#### `/fulcro-fix [lint|format|test|advanced|dry] [--report] [all|<path>|<glob>]`

Fix a Fulcro project. Runs lint, format, test, advanced-compilation, and duplicate-form checks. Defaults to the full sequence and writes formatting in place via `cljfmt fix`. Each step is also addressable on its own, and a path or glob limits the lint, format, and dry steps to those files. The format step runs cljfmt through a `:cljfmt` alias, the `lein-cljfmt` plugin, a `cljfmt` binary on `PATH`, or a `cljfmt` Clojure CLI tool, whichever it finds first, and reports that it was skipped, with the alias to add, when none is installed. The `test` step aggregates server (Kaocha) and client (shadow-cljs / Karma) suites, and the `advanced` step runs a shadow-cljs release build as a production-build sanity check. Pass `--report` to replace the format step's write with non-writing `cljfmt check`. The lint, test, advanced, and dry steps are pure-read of source regardless.

```bash
/fulcro-fix
/fulcro-fix lint
/fulcro-fix format src/main/app/ui
/fulcro-fix advanced
/fulcro-fix --report
```

#### `/fulcro-upgrade [--report] [all]`

Upgrade the `com.fulcrologic/fulcro` dependency in `deps.edn` to the latest stable version on Clojars. Also updates `com.fulcrologic/fulcro-rad`, `com.fulcrologic/guardrails`, `com.fulcrologic/fulcro-inspect`, and `fulcrologic/fulcro-spec` when the project declares them, including both `fulcro-spec` entries in the `:dev` and `:test` aliases. It skips an artifact that is already at the latest stable version and never lowers a version. Before it writes a new version, it reads the upstream `CHANGELOG.md` and the commit history between the project's release tag and the latest release tag, and it lists the places in the project that the reported changes affect. For minor and major version jumps, it asks for confirmation before applying the upgrade. It then refreshes dependencies, runs compile + tests, and refreshes clj-kondo imports, and it restores the original `deps.edn` on a dependency, compile, or test failure. Pass `--report` to print the current and latest versions and the changes that affect the project without writing `deps.edn` or running the compile and tests.

```bash
/fulcro-upgrade
/fulcro-upgrade --report
```

### Auto-triggered

These skills activate from conversation context. They cannot be invoked directly.

| Skill | Triggers |
|-------|----------|
| **fulcro** | `.cljs` / `.cljc` files containing `com.fulcrologic.fulcro.*`, `com.fulcrologic.fulcro-css.*`, `com.fulcrologic.rad.*`, `com.wsscode.pathom3.*`; the macros `defsc`, `defmutation`, `defrouter`, `defresolver`; the helpers `df/load!`, `df/load-field!`, `dr/change-route!`, `comp/get-query`, `comp/get-ident`, `comp/get-initial-state`, `comp/computed`, `comp/factory`, `fs/*`, `fo/*`, `uism/*`; ident vectors `[:table/id value]` and `[:component/id ::ComponentName]`; `shadow-cljs.edn` with a `:main` build that requires Fulcro; `deps.edn` containing `com.fulcrologic/fulcro`, `com.fulcrologic/fulcro-rad`, `com.fulcrologic/guardrails`, or `com.wsscode/pathom3`; transit endpoints `/api`; Fulcro Inspect preloads; or any mention of Fulcro, fulcrologic, Fulcro Inspect, or Fulcro RAD. Covers `defsc` component definition, idents and the normalized client database, EQL queries, mutations and their `action` / `remote` / `ok-action` / `error-action` sections, `df/load!` and load targeting, dynamic routing, forms, UI state machines, the Pathom 3 server resolver shape, and the Fulcro Inspect workflow. Defers to the [clojure](https://github.com/brackendev/clojure-skills) skill for host-neutral style, to the [clojurescript](https://github.com/brackendev/clojurescript-skills) skill for JavaScript interop and the CLJS workflow, and to the [clojure-jvm](https://github.com/brackendev/clojure-jvm-skills) skill for Java interop on the server and the JVM tooling workflow. The skill opens with Iron Laws (FUL-1 through FUL-5), rules that prevent stale UIs, silent data loss, and security holes. The agent does not write code that breaks them, and it reports a violation it finds in existing code before changing it. |

## Hooks

The package includes one hook, `fulcro-rules`.

### `fulcro-rules`

Agents seldom load an auto-triggered skill on their own, so this hook gives the agent the FUL-1 through FUL-5 Iron Laws of the `fulcro` skill as text. In a `deps.edn`, `shadow-cljs.edn`, or `project.clj` that names `com.fulcrologic/fulcro`, it puts the condensed laws into the agent's context when a session starts, and again after `/clear` or a compaction, with an instruction to load the `fulcro` skill along with the `clojure`, `clojurescript`, and `clojure-jvm` skills. When the session starts outside such a project, for example in a parent directory, the hook delivers the laws on the first edit of a `.clj`, `.cljs`, `.cljc`, or `.edn` file in a matching project instead, once per session. The hook never edits a file. Without Babashka it does nothing and prints a notice on standard error.

The hook has been run only under Claude Code, where it registers `SessionStart` and `PostToolUse`. The package ships empty Copilot and Cursor hook files, so APM writes no Copilot hook and no native Cursor hook. APM also installs the generic copy for Codex, Gemini CLI, Kiro, and Windsurf, where its effect has not been tested. OpenCode and Grok Build have no hook support.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT. See [LICENSE](LICENSE).
