---
name: fulcro-fix
description: >-
  Fix a Fulcro project with lint, format, test, advanced, and dry
  (duplicate-form) steps across the Clojure server and ClojureScript client.
  The format step writes by default. Use when asked to lint, format, test, or
  check a Fulcro project, to run its server and client test suites, or to
  verify a shadow-cljs release build. Use --report to replace the formatting
  write with cljfmt check.
argument-hint: "[lint|format|test|advanced|dry] [--report] [all]"
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

Only the `format` step writes source. It runs `clj -M:cljfmt fix` by default, rewriting files in place. With `--report`, the step runs `clj -M:cljfmt check`, which exits non-zero when files would change but does not write. The `lint`, `test`, and `dry` steps are pure-read of source regardless of `--report`. The `advanced` step writes shadow-cljs release output under `resources/public/js/` (build artifact, not source) and is unaffected by `--report`. The `lint` step's bootstrap writes the upstream clj-kondo exports into `.clj-kondo/imports/` and the lint cache into `.clj-kondo/.cache/`. Those are tooling files, not source, and `--report` does not change that write.

This skill excludes vendored, generated, and dependency-locked paths from the file set it walks. The filter combines `.gitignore` matches and a hardcoded floor (`node_modules/`, `vendor/`, `third_party/`, `.bundle/`, `target/`, `build/`, `dist/`, `out/`, `.shadow-cljs/`, `cljd-out/`, `*.lock`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Gemfile.lock`, `Cargo.lock`, `poetry.lock`, `composer.lock`). The `advanced` step's writes to `resources/public/js/` are build artifacts produced by the compiler itself and are outside the source-mutation scope of the rule. Naming a vendored source path directly through `<path>` or `<glob>` bypasses the filter for that target. When the filter excludes any path, add a single "skipped N vendored or generated paths" line to the report. Under `--report`, list every skipped path. This follows Rule 4 of the fulcro-skills `CONVENTIONS.md`.

## Steps

Parse `$ARGUMENTS` to determine which steps to run and whether `--report` is present. If no step keyword is supplied (or only `all` is supplied), run every step in order.

### 1. Lint

Bootstrap the upstream Fulcro and Pathom clj-kondo exports before every lint. clj-kondo copies configurations only into an existing `.clj-kondo/` directory, so create it first:

```bash
mkdir -p .clj-kondo
clj-kondo --copy-configs --dependencies --lint "$(clj -A:dev -Spath)"
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

Without `--report`, run cljfmt to fix formatting. `cljfmt` covers `.clj`, `.cljs`, and `.cljc`:

```bash
clj -M:cljfmt fix
```

With `--report`, run cljfmt in check mode (no writes):

```bash
clj -M:cljfmt check
```

Report pass if exit code is 0, fail otherwise. Show any formatting changes (under `fix`) or the diff (under `check`).

The project's `.cljfmt.edn` should include indentation entries for Fulcro macros (`defsc`, `defmutation`, `defrouter`, `defresolver`) and the mutation sections under `:extra-indents`. Current cljfmt treats `:indents` as a replacement for every default rule, so a project that lists Fulcro macros under `:indents` loses standard indentation for `def`, `if`, and `let`, and `cljfmt fix` rewrites much of the codebase. The `/fulcro-new` skill writes the `:extra-indents` form. Report an `:indents`-only configuration instead of running `fix` over it.

### 3. Test

Run the server test suite first, then the client test suite, even when the server suite fails. Kaocha runs the server tests in the JVM, and the client tests run in Chrome through Karma or in Node, so a failure in one suite does not abort the other. Aggregate the exit codes.

#### Server tests (Kaocha)

```bash
clj -M:test
```

The `:test` alias should drive Kaocha against `src/test/**/*_test.clj`. Report pass if Kaocha exits 0 and the summary contains no failures or errors.

#### Client tests (shadow-cljs `:test` build)

If `shadow-cljs.edn` defines a `:test` build, compile it and run the resulting test page in headless Chrome via Karma:

```bash
npx shadow-cljs compile ci && npx karma start --single-run
```

Alternatively, when a Node-friendly test entry exists:

```bash
npx shadow-cljs compile test && node target/test.js
```

Detect the test target from `shadow-cljs.edn`. If neither path is configured, skip the client test step and record `test (client): SKIPPED`.

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
- `shadow-cljs.edn`: `:source-paths`.

Scan the production source paths only and exclude test paths. If no configuration declares source paths, scan the source directories that exist in the repository, excluding any test directory. Do not assume `src`.

Run dry4clj with EDN output over the resolved paths:

```bash
clj -M:dry4clj --edn <source-paths...>
```

Do not pass `--threshold`, `--min-lines`, or `--min-nodes`. Scoping to production source removes the dominant noise, which is repeated test scaffolding; tightening the score or size knobs either hides real matches or has no effect on exact-structure duplicates.

dry4clj exits 0 after a completed scan, and the EDN output never prints a clean-state message, so do not use the exit code or any text match to detect duplicates. A non-zero exit means the scan itself failed (for example, a file the reader cannot parse) and is reported as `ERROR`. Parse the EDN map `{:candidates [...]}` and classify the result:

| Result   | Condition                                                                                                                  |
|----------|----------------------------------------------------------------------------------------------------------------------------|
| `PASS`   | `:candidates` is empty.                                                                                                     |
| `REVIEW` | `:candidates` has one or more entries. List them and continue.                                                             |
| `ERROR`  | The command cannot run or its output cannot be parsed (for example, the `:dry4clj` alias is missing). Report the cause; do not report `PASS`. |
| `SKIP`   | No production source path can be identified.                                                                               |

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
- **Karma requires Chrome on `PATH`.** `karma-chrome-launcher` looks for a `chrome` (or `chromium`) binary. On headless CI, install the Chrome runtime first (for example, `apt-get install -y chromium-browser` on Debian-based images).
- **shadow-cljs holds a daemon process.** `shadow-cljs watch` from another terminal can lock the `:main` build slot. Stop it (`npx shadow-cljs stop`) before running `release main`.
- **Pathom resolvers can fail at test time without failing at compile time.** Invalid `::pco/output` declarations (for example, declaring an attribute the resolver does not actually return) only surface during query execution. The test suite is the place to catch this.
- **dry4clj cannot read auto-resolved namespaced keywords.** dry4clj reads source without resolving namespace aliases, so a keyword such as `::pco/output` stops the scan with "Invalid token" and a non-zero exit. Pathom 3 resolvers use this form routinely, so most Fulcro projects get `ERROR` from the dry step at the dry4clj commit pinned by `/fulcro-new`. Report the file and the error, and keep it advisory. Do not rewrite source to work around the reader.
- **`dry` is a duplicate-form scan, not a dry-run mode.** The step keyword names the dry4clj tool. To preview the format step without writing, pass `--report`, not `dry`.
