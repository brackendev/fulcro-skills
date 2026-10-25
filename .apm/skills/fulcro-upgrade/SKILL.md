---
name: fulcro-upgrade
description: "Upgrade the Fulcro dependencies in deps.edn to the latest stable versions, list the upstream changes that affect the project, and verify the project compiles and tests pass"
argument-hint: "[--report] [all]"
user-invocable: true
disable-model-invocation: true
---

# Upgrade Fulcro

Upgrade the `com.fulcrologic/fulcro` dependency in `deps.edn` to the latest stable version on Clojars, read the upstream changes between the project's version and that version, and verify the project still compiles and passes tests. Also upgrades `com.fulcrologic/fulcro-rad`, `com.fulcrologic/guardrails`, `com.fulcrologic/fulcro-inspect`, and `fulcrologic/fulcro-spec` in the same pass when the project declares them. fulcro-spec's Clojars group is `fulcrologic`, not `com.fulcrologic`. ClojureScript itself is upgraded by `/cljs-upgrade` in clojurescript-skills, not by this skill.

The argument grammar follows `CONVENTIONS.md` in the fulcro-skills source repository. The table below is complete for this skill.

## Arguments

| Input         | Target                                                                       |
|---------------|------------------------------------------------------------------------------|
| (no argument) | Read the upstream changes, then update `:mvn/version` for `com.fulcrologic/fulcro` (and `fulcro-rad`, `guardrails`, `fulcro-inspect`, and `fulcrologic/fulcro-spec` if present) in `deps.edn`, refresh dependencies, compile, and run tests |
| `all`         | Same as (no argument); accepted for family consistency                       |
| `--report`    | Print the current and latest stable versions for each Fulcro artifact found in `deps.edn` and the upstream changes between them that affect the project. No writes, no compile, no tests |

The skill rewrites version strings in a single `deps.edn`. `<path>` rows are not part of the standard scope vocabulary for this skill because the target is fixed.

## Mutation

Mutates `deps.edn` by default, replacing the `:mvn/version` value for `com.fulcrologic/fulcro`, and for `com.fulcrologic/fulcro-rad`, `com.fulcrologic/guardrails`, `com.fulcrologic/fulcro-inspect`, and every `fulcrologic/fulcro-spec` entry when present. The project workflows layout declares fulcro-spec twice, in the `:dev` and `:test` aliases, and the skill updates and reverts both together. It then refreshes dependencies with `clojure -P`, compiles the ClojureScript client with `npx shadow-cljs compile main`, runs the test suites, and refreshes clj-kondo imports. On a dependency, compile, or test failure, the skill restores the saved copy of `deps.edn`, which reverts every version change. With `--report`, the skill writes nothing in the project and runs no compile or tests. It still clones the upstream repositories into a temporary directory to read their commit history.

## Prerequisites

Verify `deps.edn` exists in the current directory and contains a `com.fulcrologic/fulcro` dependency. If not, stop and tell the user.

## Steps

Parse `$ARGUMENTS` to determine whether `--report` is present.

### 1. Record Current Versions

Read `deps.edn` and note the current `:mvn/version` value for each of:

- `com.fulcrologic/fulcro` (required)
- `com.fulcrologic/fulcro-rad` (if present)
- `com.fulcrologic/guardrails` (if present)
- `com.fulcrologic/fulcro-inspect` (if present, usually in the `:dev` alias)
- `fulcrologic/fulcro-spec` (if present). Record every occurrence and its alias. In the project workflows layout it appears in the `:dev` alias, which the shadow-cljs client test builds read, and in the `:test` alias, which Kaocha runs. When the occurrences hold different versions, report both and ask the user which one to treat as current.
- `com.wsscode/pathom3` (if present). Report its version but do not update it unless the user explicitly asks. Pathom 3 publishes only alpha releases, and its `CHANGELOG.md` marks breaking changes in them with "BREAKING". When the user asks, read the entries newer than the project's version in `https://raw.githubusercontent.com/wilkerlucio/pathom3/main/CHANGELOG.md` and apply step 4's search rules and step 5's confirmation rules to them.

Unless `--report` is present, save a copy of the full file so that a later step can restore it:

```bash
cp deps.edn "${TMPDIR:-/tmp}/deps.edn.fulcro-upgrade"
```

### 2. Fetch the Latest Stable Versions

Query Clojars for each artifact found above. The group is part of the URL: `com.fulcrologic` for `fulcro`, `fulcro-rad`, `guardrails`, and `fulcro-inspect`, and `fulcrologic` for `fulcro-spec`. `https://clojars.org/api/artifacts/com.fulcrologic/fulcro-spec` returns 404.

```bash
curl -sSL -A 'Mozilla/5.0' \
  'https://clojars.org/api/artifacts/<group>/<artifact>' \
  | python3 -c "import json,re,sys; d=json.load(sys.stdin); s=[v['version'] for v in d['recent_versions'] if not re.search('[A-Za-z]', v['version'])]; print('latest_release:', d['latest_release']); print('newest stable:', max(s, key=lambda v: [int(n) for n in re.findall(r'[0-9]+', v)]) if s else None)"
```

Upgrade only to the `newest stable` version, the highest version whose string contains no letters. `latest_release` never names a `-SNAPSHOT` build, but it can name a release candidate: Fulcro published `3.10.0-RC1` through `3.10.0-RC3` on Clojars before `3.10.0`. `recent_versions` also lists `-SNAPSHOT` builds (fulcro-spec lists `3.2.9-SNAPSHOT`) and older prereleases such as `3.9.0-rc14`, and the command skips them all. When the two lines differ, report both. When `newest stable` is `None`, ask the user.

If a network call fails, ask the user for the target versions rather than guessing. Do not invent a version number.

### 3. Compare Versions

Run these checks with and without `--report`. Compare versions by their numeric parts, not as strings (`3.9.5` is older than `3.10.0`). When the numeric parts are equal, the stable version is newer than a `-RC`, `-alpha`, `-beta`, or `-SNAPSHOT` version.

- **Already at the latest stable version.** When the current version equals the latest stable version, report "already at the latest stable version" for that artifact and leave it out of the remaining steps.
- **No downgrade.** When the current version is newer than the latest stable version (for example, a SNAPSHOT or release candidate of a later version), report "newer than the latest stable release <latest>, not changed" and leave it out of the remaining steps. Never lower a version.

When no artifact remains, print the report from step 4 with "none found" on the Changes line and stop.

### 4. Read the Upstream Changes

Fulcro releases breaking changes in minor versions, so read what changed between the project's version and the latest stable version at run time instead of relying on remembered migration notes. Read the changes for every remaining artifact before printing the `--report` output and before writing `deps.edn`. Fulcrologic records its changes in two places, and neither is complete:

- **`CHANGELOG.md` at the latest release tag.** In `fulcro` and `fulcro-rad` the file opens with "See git commit history for further change log entries. This file is often not updated." In `fulcro-inspect`, `guardrails`, and `fulcro-spec` its newest entries are older than the current releases. Print every entry for a version newer than the project's version. When the newest entry is older than the project's version, report that the changelog has no entries for the range.
- **The commit history between release tags.** Fulcrologic publishes no GitHub releases. Each repository tags its releases as `<repository>-<version>`: `fulcro-3.10.0` in `fulcrologic/fulcro`, `fulcro-inspect-1.0.5` in `fulcrologic/fulcro-inspect`, `guardrails-1.3.4` in `fulcrologic/guardrails`, `fulcro-rad-1.6.24` in `fulcrologic/fulcro-rad`, and `fulcro-spec-3.2.11` in `fulcrologic/fulcro-spec`. fulcro-spec uses that form from `fulcro-spec-3.1.6` on, and its tags for 3.1.5 and older are the bare version (`3.1.5`).

Fetch both for each artifact that changes, replacing `<repo>` with the repository name (`fulcro`, `fulcro-inspect`, `guardrails`, `fulcro-rad`, `fulcro-spec`):

```bash
curl -sSL "https://raw.githubusercontent.com/fulcrologic/<repo>/<repo>-<latest>/CHANGELOG.md"

tmp="$(mktemp -d)"
git clone --quiet --filter=blob:none "https://github.com/fulcrologic/<repo>.git" "$tmp/<repo>"
git -C "$tmp/<repo>" log --no-merges --format='%cs %s' '<repo>-<current>..<repo>-<latest>' \
  | grep -v 'maven-release-plugin'
```

If either tag is missing (for example, the project pins a SNAPSHOT build, which has no tag), list the tags with `git -C "$tmp/<repo>" tag --sort=-creatordate` and ask the user which range to use.

Find what the changes require. For every changelog entry or commit subject that names a breaking change, a removed or renamed namespace, function, macro, or option, a new requirement (such as a React or Java version), or a changed default, search the project for the names it mentions (the source and test directories, `shadow-cljs.edn`, `deps.edn`, and `package.json`) and list the matches with file and line. When a commit subject is too terse to tell what changed, read that commit with `git -C "$tmp/<repo>" show --stat <sha>` before deciding. Collect the entries that affect the project, with their matches, for the Changes line.

If `--report` is present, print the report and stop without writing `deps.edn`, refreshing dependencies, or running the compile and tests:

```
Fulcro upgrade report:
  com.fulcrologic/fulcro:         <current> -> <latest>
  com.fulcrologic/fulcro-rad:     <current> -> <latest>   (or "not present")
  com.fulcrologic/guardrails:     <current> -> <latest>   (or "not present")
  com.fulcrologic/fulcro-inspect: <current> -> <latest>   (or "not present")
  fulcrologic/fulcro-spec:        <current> -> <latest>   (or "not present")
  com.wsscode/pathom3:            <current>               (informational only)
  Changes: <each change that affects the project, with its artifact, source, and file matches, or "none found">
```

For an artifact that step 3 left out, print its current version followed by "(already at the latest stable version)" or "(newer than the latest stable release <latest>, not changed)" instead of an arrow.

### 5. Confirm and Update

When the minor or major version of an artifact changes, show the changes and the project matches from step 4, and ask the user whether to proceed before writing the new version into `deps.edn`. Treat any entry or commit subject that contains "BREAKING" as requiring explicit confirmation. Patch-level updates within the same minor version proceed without asking, but still report any matches. Report the edits that the matches call for outside `deps.edn` and ask before making them.

Then update the `:mvn/version` value in `deps.edn` for each remaining artifact, including every `fulcrologic/fulcro-spec` occurrence. Preserve formatting and surrounding content.

### 6. Refresh Dependencies

Download the new versions, naming every alias that declares an upgraded artifact (`:dev` and `:test` in the project workflows layout):

```bash
clojure -P -M:dev:test
```

Use plain `clojure -P` when every upgraded artifact is in the top-level `:deps`. If dependency resolution fails, restore the saved file over `deps.edn`, report the error, and stop:

```bash
cp "${TMPDIR:-/tmp}/deps.edn.fulcro-upgrade" deps.edn
```

### 7. Compile and Test

Run the ClojureScript build (development compile is enough as a sanity check; the `/fulcro-fix advanced` step covers the production build separately):

```bash
npx shadow-cljs compile main
```

Then run the test suites:

```bash
clojure -M:test
npx shadow-cljs compile ci && npx karma start --single-run
```

Run the Karma line only when `shadow-cljs.edn` has a `:target :karma` build (`:ci` in the project workflows layout) and the project has a `karma.conf.js`. When it has a `:target :node-test` build instead, run `npx shadow-cljs compile <build> && node <output-to>`, using the build's `:output-to`. If neither is configured, skip the client test and record `test (client): SKIPPED`.

If compilation or any test suite fails, restore the saved file over `deps.edn` with the command in step 6, report the error together with any change from step 4 that explains it, and stop.

### 8. Refresh clj-kondo Imports

Fulcro's clj-kondo exports may have changed across the upgrade:

```bash
clj-kondo --copy-configs --dependencies --lint "$(clojure -A:dev -Spath)"
```

clj-kondo copies configurations only into an existing `.clj-kondo/` directory. When the project has none, skip this step and report `clj-kondo imports: SKIPPED (no .clj-kondo/ directory)`.

### 9. Report

Print a summary:

```
Fulcro upgraded:
  com.fulcrologic/fulcro:         <old> -> <new>
  com.fulcrologic/fulcro-rad:     <old> -> <new>   (or "not present")
  com.fulcrologic/guardrails:     <old> -> <new>   (or "not present")
  com.fulcrologic/fulcro-inspect: <old> -> <new>   (or "not present")
  fulcrologic/fulcro-spec:        <old> -> <new>   (or "not present")
  com.wsscode/pathom3:            <unchanged>      (informational only)
  Changes: <each change from step 4 that affects the project, with its file matches, or "none found">

  Compile:           PASS
  Tests (server):    PASS
  Tests (client):    PASS / SKIPPED
  clj-kondo imports: refreshed / SKIPPED

Recommended next steps:
  /fulcro-fix advanced    # Production-build sanity check with externs inference
  Smoke test Fulcro Inspect in the browser
  Re-run any forms / load flows that exercise Fulcro internals
```

For an artifact that step 3 left out, print the same note as in the `--report` output.

## Gotchas

- **Clojars `latest_release` skips SNAPSHOT builds but not release candidates.** Clojars excludes versions ending in `-SNAPSHOT` from `latest_release` and returns `null` when an artifact has only SNAPSHOT builds. A release candidate still counts, so apply the stable-version rule in step 2. Pathom 3 is the exception: it publishes only alpha releases, so its newest alpha is the latest release.
- **fulcro-spec's group is `fulcrologic`.** `com.fulcrologic/fulcro-spec` does not exist on Clojars, so a Clojars query with that group returns 404 and a `deps.edn` entry with it fails `clojure -P` with "Could not find artifact com.fulcrologic:fulcro-spec". Report such an entry instead of rewriting its version.
- **Check the React and Inspect setup against the `fulcro` skill after the upgrade.** Compare `package.json`'s `react` and `react-dom` with the React requirement the `fulcro` skill states, and compare the `shadow-cljs.edn` preloads and the client startup call with its Fulcro Inspect reference. Report any difference and ask before editing files other than `deps.edn`.
- **`shadow-cljs compile main` requires `node` and `npm` on `PATH`.** If either is missing, the compile step cannot run; report the missing prerequisite and stop.
- **Some projects pin Fulcro via Git** (`{:git/url "https://github.com/fulcrologic/fulcro" :git/tag "fulcro-<version>" :git/sha "..."}`) rather than `:mvn/version`. This skill only handles the `:mvn/version` path; if a Git dependency is present, stop and tell the user.
- **Fulcro RAD's companion libraries release separately.** When the project declares `com.fulcrologic/fulcro-rad-datomic`, `com.fulcrologic/fulcro-rad-sql`, or `com.fulcrologic/fulcro-rad-semantic-ui`, report their current and latest versions next to `fulcro-rad` so the user can update them together. This skill does not rewrite them.
- **clj-kondo imports occasionally break across upgrades.** If the lint step floods after the refresh, delete `.clj-kondo/.cache/` and `.clj-kondo/imports/` and re-run `clj-kondo --copy-configs --dependencies`.
