---
name: fulcro-fix
description: >-
  Fix a Fulcro project with lint, format, test, advanced, and dry
  (duplicate-form) steps across the Clojure server and ClojureScript client.
  The format step writes by default. Use when asked to lint, format, test, or
  check a Fulcro project, to run its server and client test suites, or to
  verify a shadow-cljs release build. Use --report to replace the formatting
  write with cljfmt check.
argument-hint: "[lint|format|test|advanced|dry] [--report] [all|<path>|<glob>]"
allowed-tools: Bash, Read, Grep, Glob
user-invocable: true
---

# Fulcro Fix

Run lint, format, test, advanced-compilation, and duplicate-form checks on a Fulcro project. Fulcro projects are full-stack (Clojure server + ClojureScript client), so the `test` step aggregates server and client suites and the `advanced` step runs a shadow-cljs release build to surface production-only failures. The format step writes by default; the other four steps are pure-read of source.

The argument grammar follows `CONVENTIONS.md` in the fulcro-skills source repository. The table below is complete for this skill.

## Arguments

| Input             | Target                                                                       |
|-------------------|------------------------------------------------------------------------------|
| (no argument)     | Run all five steps (`lint`, `format`, `test`, `advanced`, `dry`) on the project (`src`, `test`) |
| `all`             | Same as (no argument); accepted for family consistency                       |
| `lint`            | Run lint only                                                                |
| `format`          | Run format only                                                              |
| `test`            | Run server (Kaocha) and client (shadow-cljs / Karma) test suites             |
| `advanced`        | Run a shadow-cljs release build as a production-build sanity check           |
| `dry`             | Run the dry4clj duplicate-form scan only                                     |
| `<path>` `<glob>` | Restrict lint, format, and dry steps to those files or directories. The `test` and `advanced` steps ignore paths and always run the configured suites and builds |
| `--report`        | Replace `cljfmt fix` with non-writing `cljfmt check` in the format step      |

Step keywords are combinable (for example, `/fulcro-fix lint test`). The `--report` flag may appear in any position. When `--report` is present without an explicit step keyword, every step still runs; only the format step's behavior changes.

## Mutation

Only the `format` step writes source. It runs `cljfmt fix` through the runner the Format step selects, rewriting files in place. With `--report`, the step runs `cljfmt check` through the same runner, which exits non-zero when files would change but does not write. The `lint`, `test`, and `dry` steps are pure-read of source regardless of `--report`. The `advanced` step writes shadow-cljs release output under `resources/public/js/` (build artifact, not source) and is unaffected by `--report`. The `lint` step's bootstrap writes the upstream clj-kondo exports into `.clj-kondo/imports/` and the lint cache into `.clj-kondo/.cache/`. Those are tooling files, not source, and `--report` does not change that write.

This skill excludes vendored, generated, and dependency-locked paths from the file set it walks. The filter combines `.gitignore` matches and a hardcoded floor (`node_modules/`, `vendor/`, `third_party/`, `.bundle/`, `target/`, `build/`, `dist/`, `out/`, `.shadow-cljs/`, `cljd-out/`, `*.lock`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Gemfile.lock`, `Cargo.lock`, `poetry.lock`, `composer.lock`). The `advanced` step's writes to `resources/public/js/` are build artifacts produced by the compiler itself and are outside the source-mutation scope of the rule. Naming a vendored source path directly through `<path>` or `<glob>` bypasses the filter for that target. When the filter excludes any path, add a single "skipped N vendored or generated paths" line to the report. Under `--report`, list every skipped path. This follows Rule 4 of the fulcro-skills `CONVENTIONS.md`.

## Steps

Parse `$ARGUMENTS` to determine which steps to run and whether `--report` is present. If no step keyword is supplied (or only `all` is supplied), run every step in order.

### 1. Lint

Bootstrap the upstream Fulcro and Pathom clj-kondo exports before every lint. clj-kondo copies configurations only into an existing `.clj-kondo/` directory, so create it first:

```bash
mkdir -p .clj-kondo
clj-kondo --copy-configs --dependencies --lint "$(clojure -A:dev -Spath)"
```

`--dependencies` skips each dependency jar that clj-kondo has already linted, so a repeat run is fast. After a Fulcro or Pathom upgrade, the new jar is linted and its exports are copied into `.clj-kondo/imports/`.

Then lint the project sources:

```bash
clj-kondo --lint src test
```

If the project has no `test/` directory, lint `src` only.

Report pass if exit code is 0 and there are no errors. Show the clj-kondo output regardless.

Without the upstream Fulcro exports, `defsc`, `defmutation`, `defrouter`, and Pathom `defresolver` forms generate hundreds of false positives. When the lint floods on those forms, confirm that the bootstrap ran and that `.clj-kondo/imports/com.fulcrologic/fulcro/` exists before reporting the flood as a genuine failure.

### 2. Format

Run cljfmt with the first runner the project supports, checked in this order. Check for the alias before running it: without one, `clojure -M:cljfmt fix` runs `fix` as a script path and fails with "fix (No such file or directory)".

| Runner | When | Fix | Check |
|--------|------|-----|-------|
| Project alias | `deps.edn` defines a `:cljfmt` alias | `clojure -M:cljfmt fix` | `clojure -M:cljfmt check` |
| Leiningen plugin | `project.clj` lists `dev.weavejester/lein-cljfmt` in `:plugins` | `lein cljfmt fix` | `lein cljfmt check` |
| Standalone binary | `command -v cljfmt` succeeds | `cljfmt fix` | `cljfmt check` |
| Clojure CLI tool | `clojure -Ttools list` lists a tool named `cljfmt` | `clojure -Tcljfmt fix` | `clojure -Tcljfmt check` |

Without `--report`, run the Fix command. With `--report`, run the Check command, which writes nothing. cljfmt's default file pattern covers `.clj`, `.cljs`, `.cljc`, `.cljx`, and `.bb` files.

Outside Leiningen, cljfmt formats `src`, `test`, and `project.clj` unless its configuration sets `:paths`. The Fulcro layout keeps `main`, `dev`, and `test` under `src/`, so the default covers it. When the project keeps source elsewhere, or when `<path>` or `<glob>` arguments narrow the run, pass the paths: as arguments to the alias, the binary, and `lein cljfmt` (`clojure -M:cljfmt fix src/main/app`), and as an option to the tool (`clojure -Tcljfmt fix :paths '["src/main/app"]'`). A path argument that does not exist stops cljfmt with "No such file:". Without path arguments, `lein cljfmt` formats the `:source-paths` and `:test-paths` directories that exist.

cljfmt loads `.cljfmt.edn` or `cljfmt.edn` from the project directory or the nearest parent directory that has one. It skips `.cljfmt.clj` and `cljfmt.clj` with the warning "detected but --read-clj-config-files is not set. Ignoring config file.", because it reads those files as Clojure code, and falls back to an `.edn` file or to its defaults. When the project has only a `.clj` configuration file, ask the user before adding `--read-clj-config-files` to the alias or binary command. The tool cannot read `.clj` configuration files. Never report a project as failing because of its configuration file name. `lein cljfmt` reads the `:cljfmt` map in `project.clj`, and it reads `.cljfmt.edn` only when that map sets `:load-config-file? true`.

When no runner is available, report `format: SKIPPED`, show the alias to add under `:aliases`, and name the cljfmt README (https://github.com/weavejester/cljfmt) for the other installation methods:

```clojure
:cljfmt {:replace-deps {dev.weavejester/cljfmt {:mvn/version "0.16.6"}}
         :main-opts    ["-m" "cljfmt.main"]}
```

`:replace-deps` keeps the project's own dependencies off the formatter's classpath, so a project dependency that conflicts with cljfmt or cannot be resolved does not stop the format step.

Report pass if exit code is 0, fail otherwise. Show any formatting changes (under `fix`) or the diff (under `check`).

Fulcrologic's own code, the Fulcro source and `fulcro-template`, indents every form body and every argument line by two spaces, which is Cursive's "Only Indent" setting, and that includes the libspecs under `(:require`. When the project has no cljfmt configuration, recommend this `.cljfmt.edn`:

```clojure
{:function-arguments-indentation :cursive
 :indents                        {#re ".*" [[:inner 0]]}}
```

The catch-all `:indents` rule gives every form that starts with a symbol a two-space body indent, including `defsc`, `defmutation`, `defrouter`, `defresolver`, the mutation sections, and the `fulcro-spec` macros. It replaces cljfmt's default rules, which is the intent. A pattern key matches only forms that start with a symbol, so keyword-headed forms such as `(:require ...)` fall back to cljfmt's list indentation, where `:cursive` gives two spaces. Without `:cursive`, cljfmt indents the libspecs by one space and rewrites every `ns` form.

In an `.edn` configuration file the pattern needs the `#re` tag. Fulcro's own `.cljfmt.edn` at `fulcro-3.10.1` writes `:indents ^:replace {#".*" [[:inner 0]]}`, which cljfmt cannot read from an `.edn` file, so the format step stops with "No dispatch macro for: \"". Report a project configuration with that form and show the `#re ".*"` version instead of editing it. cljfmt ignores the `^:replace` metadata, because `:indents` always replaces the default rules.

A project that keeps cljfmt's default community indentation lists the Fulcro macros under `:extra-indents` instead, each with `[[:inner 0]]`: `defsc`, `defmutation`, `defrouter`, `defresolver`, `action`, `remote`, `ok-action`, `error-action`, `result-action`, and the `fulcro-spec` macros `specification`, `behavior`, `component`, `assertions`, `provided`, and `when-mocking`. A project that lists only those macros under `:indents`, with no catch-all rule, loses the default rules for `def`, `if`, and `let`, and `cljfmt fix` rewrites much of the codebase. Report that configuration instead of running `fix` over it, and show the `:extra-indents` form to use.

### 3. Test

Run the server test suite first, then the client test suite, even when the server suite fails. Kaocha runs the server tests in the JVM, and the client tests run in Chrome through Karma or in Node, so a failure in one suite does not abort the other. Aggregate the exit codes.

#### Server tests (Kaocha)

```bash
clojure -M:test
```

The `:test` alias should drive Kaocha against `src/test/**/*_test.clj`, which needs a `tests.edn` that sets `:test-paths ["src/test"]`, because Kaocha's default test path is `test/`. Kaocha exits 0 when it finds no tests and prints "No tests were found", so report pass only if Kaocha exits 0, the summary shows at least one test, and it contains no failures or errors. Report a run with no tests as `FAIL` and show the `tests.edn` from the `fulcro` skill's project workflows reference.

#### Client tests (shadow-cljs `:test` build)

Detect the client test build from `shadow-cljs.edn` by its `:target`. When a build uses `:target :karma` (`:ci` in the project workflows layout) and the project has a `karma.conf.js`, compile it and run it in headless Chrome:

```bash
npx shadow-cljs compile ci && npx karma start --single-run
```

When a build uses `:target :node-test`, compile it and run the file named by its `:output-to`:

```bash
npx shadow-cljs compile <build> && node <output-to>
```

A `:browser-test` build only produces a page for a browser, so it cannot run unattended. If neither a `:karma` nor a `:node-test` build is configured, skip the client test step and record `test (client): SKIPPED`.

Report `test: PASS` only when both server and client suites pass. Report `test: FAIL` if either fails, and surface the failing summary.

### 4. Advanced

Run a shadow-cljs release build as a production-build sanity check:

```bash
npx shadow-cljs release main
```

This compiles under `:advanced` optimizations, runs externs inference, and surfaces warnings that do not appear in development builds. Report fail if exit code is non-zero, or if the output contains `WARNING:` lines from the Google Closure Compiler about property renaming or missing externs.

If the project does not define a `:main` (or equivalent) build, ask the user for the build name. Do not guess.

The advanced step is slow on first run (whole-program optimization). shadow-cljs caches across runs.

### 5. Dry

Scan for duplicate top-level forms with [dry4clj](https://github.com/unclebob/dry4clj). This step is advisory. It reports duplication candidates for review but never fails the run and never halts the pipeline.

Determine the project's production source directories from its build configuration instead of assuming a directory name:

- `deps.edn`: the top-level `:paths` (the `:dry4clj` alias is defined here). Tests usually live under a separate alias's `:extra-paths`, so scanning `:paths` excludes them.
- `shadow-cljs.edn`: `:source-paths`, unless `shadow-cljs.edn` sets `:deps`. With `:deps`, shadow-cljs takes its source paths from `deps.edn`, so use the `deps.edn` paths instead.

Scan the production source paths only and exclude test paths. If no configuration declares source paths, scan the source directories that exist in the repository, excluding any test directory. Do not assume `src`.

Check that `deps.edn` defines a `:dry4clj` alias before running the scan. When it does not, report `SKIPPED` and show the alias to add under `:aliases`:

```clojure
:dry4clj {:extra-deps {io.github.unclebob/dry4clj {:git/url "https://github.com/unclebob/dry4clj.git"
                                                   :git/sha "5994ef2bcaaae8d12bcd0afb4f091ae09083228c"}}
          :main-opts  ["-m" "dry4clj.core"]}
```

Run dry4clj with EDN output over the resolved paths:

```bash
clojure -M:dry4clj --edn <source-paths...>
```

Do not pass `--threshold`, `--min-lines`, or `--min-nodes`. Scoping to production source removes the dominant noise, which is repeated test scaffolding; tightening the score or size knobs either hides real matches or has no effect on exact-structure duplicates.

dry4clj exits 0 after a completed scan, and the EDN output never prints a clean-state message, so do not use the exit code or any text match to detect duplicates. A non-zero exit means the scan itself failed (for example, a file the reader cannot parse) and is reported as `ERROR`. Parse the EDN map `{:candidates [...]}` and classify the result:

| Result   | Condition                                                                                                                  |
|----------|----------------------------------------------------------------------------------------------------------------------------|
| `PASS`   | `:candidates` is empty.                                                                                                     |
| `REVIEW` | `:candidates` has one or more entries. List them and continue.                                                             |
| `ERROR`  | The command fails or its output cannot be parsed. Report the cause, and do not report `PASS`. |
| `SKIPPED` | No production source path can be identified, or `deps.edn` defines no `:dry4clj` alias. |

For `REVIEW`, sort candidates by exact matches first (`:score` equal to `1.0`), then by descending `min(:left-nodes, :right-nodes)`, then by descending `:score`. Report the scanned source paths, the candidate count, and the highest-priority candidates with their score, node counts, and both file ranges. Note that each candidate needs source inspection before extraction, and that test directories were excluded because repeated test scaffolding is often intentional.

Upstream dry4clj scans `.clj`, `.cljc`, and `.cljs` files by default; the Clojure server, the cljc shared layer, and the ClojureScript client are all covered.

## Report

After running all requested steps, print a summary:

```
Fulcro Fix Results:
  Lint:     PASS/FAIL/SKIPPED
  Format:   PASS/FAIL/SKIPPED
  Test:     PASS/FAIL/SKIPPED  (server: PASS, client: PASS)
  Advanced: PASS/FAIL/SKIPPED
  Dry:      PASS/REVIEW/ERROR/SKIPPED
```

If `--report` was passed, append `(report mode: format checked, not written)` after the summary. If a lint, format, test, or advanced step fails, stop and report the failure; do not continue to subsequent steps.

## Gotchas

- **`shadow-cljs release main` requires `node` and `npm` on `PATH`.** If either is missing, the advanced step is reported as SKIPPED rather than FAIL.
- **Karma needs a Chrome binary.** The `ChromeHeadless` launcher in `karma-chrome-launcher` reads `CHROME_BIN`, and otherwise looks for `google-chrome` or `google-chrome-stable` on Linux and the standard application path on macOS. On a CI image that has only Chromium, set `CHROME_BIN` to the Chromium binary.
- **A running shadow-cljs server takes over the release build.** While a server runs, including one that `watch` started, `npx shadow-cljs release main` runs inside that server's JVM, so it ignores environment variables set in the current terminal, such as `TAOENSSO_TIMBRE_MIN_LEVEL_EDN`. Stop the server (`npx shadow-cljs stop`, or Ctrl+C in the terminal running `watch`), or pass `--force-spawn` to run the release in a new JVM.
- **fulcro-spec includes no clj-kondo configuration.** Its `=>` arrows and the mocked call forms inside `when-mocking` and `provided` lint as unresolved symbols, so a project with fulcro-spec tests fails the lint step until `.clj-kondo/config.edn` excludes them. Report the errors and show this configuration:

  ```clojure
  {:linters {:unresolved-symbol {:exclude [(fulcro-spec.core/assertions [=> =fn=> =throws=> =check=>])
                                           (fulcro-spec.core/when-mocking)
                                           (fulcro-spec.core/provided)]}}}
  ```
- **Pathom resolvers can fail at test time without failing at compile time.** Invalid `::pco/output` declarations (for example, declaring an attribute the resolver does not actually return) only surface during query execution. The test suite is the place to catch this.
- **dry4clj cannot read auto-resolved namespaced keywords.** dry4clj reads source without resolving namespace aliases, so a keyword such as `::pco/output` stops the scan with "Invalid token" and a non-zero exit. Pathom 3 resolvers use this form routinely, so most Fulcro projects get `ERROR` from the dry step at the dry4clj commit pinned above. Report the file and the error, and keep it advisory. Do not rewrite source to work around the reader.
- **`dry` is a duplicate-form scan, not a dry-run mode.** The step keyword names the dry4clj tool. To preview the format step without writing, pass `--report`, not `dry`.
