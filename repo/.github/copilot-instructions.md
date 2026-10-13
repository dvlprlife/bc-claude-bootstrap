# {{APP_NAME}} – Copilot Instructions

These notes tell Copilot (and Claude Code, which imports this file from `CLAUDE.md`) how to propose
changes in this repository. Prefer AL code that matches the patterns below and only suggest other
languages when explicitly asked.

## Project Overview

{{APP_DESCRIPTION}}

## Technology Stack
**Language:** AL (Business Central extension) for Microsoft Dynamics 365 Business Central.
**Frameworks/Libraries:** Business Central standard libraries and APIs, AL-Go for build automation.
**Platform:** Microsoft Dynamics 365 Business Central (Cloud and On-Premises).
**Source Control:** GitHub.
**Documentation:** Markdown files in the repository, Business Central documentation.

## Solution Context
- Business Central app in `{{APP_FOLDER}}/` plus its automated test app in `{{TEST_FOLDER}}/` and supplemental assets in `supporting-files/`.
- Functional areas live in their own module folders under `{{APP_FOLDER}}/`. Treat each folder as the source of truth for that domain.

## Object Ranges, Prefixes, and Metadata
- Production objects use IDs `{{ID_FROM}}..{{ID_TO}}`; test objects live in `{{TEST_ID_FROM}}..{{TEST_ID_TO}}`. Stay inside these ranges and avoid collisions.
- Every new object, field, enum, page action, and permission set needs the `{{PREFIX}}` prefix per `{{APP_FOLDER}}/AppSourceCop.json` (`mandatoryAffixes`). Keep captions business-friendly, but names must stay prefixed.
- Honor `NoImplicitWith` (set in `app.json`): always qualify fields with the record variable name.
- Respect `supporting-files/{{PREFIX}}.ruleset.json` suppression decisions; do not re-enable suppressed rules without a discussion.

## AL Coding Guidelines (source of truth: BCQuality)

The authoritative, machine-readable AL coding guidelines for this org live in the
**BCQuality** knowledge base — our fork at `{{BCQUALITY_REPO}}` (cloned next to
this repo as a sibling, e.g. `../BCQuality`). It is *remedial*: each rule encodes a
BC-specific pitfall an assistant would otherwise get wrong (CodeCop rule numbers,
ordering rules like `SetLoadFields` before filters, partial-load patterns, obsoletion
tags, and our own `custom/knowledge/reliability/` rules — e.g. load the GL Setup before
using its rounding precision in `Round`, and assign a return value on every path of a
result-returning procedure).

When proposing or writing AL, follow that guidance; the patterns below are a summary, not
a replacement. The `/bc-review` skill (`.claude/skills/bc-review/`) enforces these rules
at review time and compiles the project first — run it on a diff before opening a PR. Add
a new `custom/knowledge/` file in the fork whenever a real defect slips through, so the
guideline set compounds.

## Coding Patterns
- Tables: keep primary keys stable, add clustered keys as needed, and set `DataClassification = CustomerContent` unless another classification is justified. Reuse existing helper functions/codeunits before adding new ones.
- Pages & Page Extensions: Set `ApplicationArea = Basic, Suite` unless restricted, and keep `UsageCategory` accurate.
- APIs: follow a versioned folder pattern (`API/v1`, `API/v2`), keep entity names stable, and expose GUID-based primary keys when available. Never break an existing contract—add a new version instead.
- Events: prefer publisher events before modifying standard behavior. Subscribe in the relevant module folder and document the scenario with a short summary comment.
- Access: default new procedures to `internal` unless another app must call them. Do not narrow the access of a member that has already shipped as public — that is a breaking change.

## Business Central Development Patterns
- Record lifecycle: call `Init`, set key fields, and use `Validate`/`TestField` instead of direct assignments so field triggers run. Use `Insert(true)`/`Modify(true)` when you expect OnInsert/OnModify logic to execute; reserve `false` only when you intentionally skip triggers.
- Filtering and iteration: compose filters with `SetRange`, `SetFilter`, and `SetLoadFields`; prefer `FindSet` (with `ForUpdate` when writing) or `FindFirst` instead of `Find('-')`. Release record references when finished to avoid table locks.
- Transactions and posting: keep transactions short, avoid `Commit` inside loops, and reuse existing posting codeunits or buffer patterns (`Gen. Jnl.-Post Line`, etc.) instead of duplicating logic. Guard preconditions with `TestField` before performing irreversible work.
- Temporary/buffer records: use `Temporary` tables or in-memory buffers for staging calculations rather than writing scratch data to persistent tables. Clear buffers explicitly (`DeleteAll`) after use.
- Enums and options: prefer enums over options; extend existing enums with the `{{PREFIX}}` prefix and reuse captions from `Translations/` when possible. Never shrink option/enumeration values in production tables.
- Error handling: throw `Error` or `FieldError` with label-based messages; only use `TryFunction` when you need rollback semantics and return simple success/failure booleans. Avoid swallowing errors silently.
- Permissions and security: update the app's `*.PermissionSet.al` files when new objects or events are added, honor `SecurityFiltering` on queries/pages, and only add `Scope = OnPrem` when absolutely required.

## Testing Expectations
- always ask before creating tests
- Add or update AL tests in `{{TEST_FOLDER}}/src/` for every non-trivial change. Put canonical test-data creation in `{{PREFIX}} Library - *` codeunits and extend them instead of reimplementing bootstrap logic.
- Use `Assert` from the Microsoft test libraries referenced in `{{TEST_FOLDER}}/app.json`. Follow the `[GIVEN]/[WHEN]/[THEN]` comment pattern.
- Remember to adjust the test app's permission set if new objects need test permissions.

## Tooling & Build
- This repo is built with AL-Go for GitHub; keep pipelines green by maintaining `app.json`, `AppSourceCop.json`, and `{{TEST_FOLDER}}/app.json` in sync — `Major.Minor` and dependencies. The `Build` segment is deliberately **not** kept in lockstep: it is the sandbox publish counter on `{{APP_FOLDER}}/app.json` (see **Sandbox Publish & Live Testing**), while `{{TEST_FOLDER}}/app.json`'s dependency pin is a minimum version that a higher app build still satisfies.
- Assets such as logos live under `logo/` or `supporting-files/documentation`. Reuse existing media rather than checking in duplicates.

## Sandbox Publish & Live Testing (AL MCP)

Beyond compile + BCQuality, changes get a **live human-test pass** in a BC sandbox via the AL MCP server, around the merge. GitHub CI is **not** a gate in this project.

**Gate — when to publish:** publish to the sandbox when **both `bc-review` gates pass** for that build: the compile with the four analyzers (zero errors, no `AL1003`), and the BCQuality review. The publish is part of the process; do not wait for a CI run.

- **When a PR passes both `bc-review` gates** → build the branch `.app`, publish it to the **{{SANDBOX_ENV}}** sandbox, and ask the user to test the change live. Merge only after they sign off.
- **After merge** → rebuild from the merged branch and publish again so the sandbox tracks it. Then delete the merged branch locally and on `origin`.

**Target** (from `{{APP_FOLDER}}/.vscode/launch.json`, which is local-only/untracked): env `{{SANDBOX_ENV}}`, `environmentType` `Sandbox`, tenant `{{TENANT_ID}}`.

**Recipe** — build the package yourself, then publish it prebuilt (the MCP's own build reads only `<project>/.alpackages` and can't see the out-of-tree `supporting-files/.alpackages` symbol cache):
1. **Bump the build number first.** Increment the **Build** segment of `{{APP_FOLDER}}/app.json` `version` (`Major.Minor.Build.Revision`) by one — `1.0.0.0` → `1.0.1.0` → `1.0.2.0`. Every package published to the sandbox carries its own version, so the install is a real upgrade and the version in **Extensions** tells you which build you are testing. Bump `{{APP_FOLDER}}/app.json` only: the test app is not published by this recipe, and its `dependencies` pin on {{APP_NAME}} is a *minimum* version, which a higher app version still satisfies — leave that pin at its baseline.
2. Build the `.app` in **PowerShell** with native Windows paths (the build command in `CLAUDE.md`). Do **not** use Bash with `MSYS_NO_PATHCONV=1` — it leaves `/out:` as a unix path that Windows `alc` silently fails to write.
3. Publish via `al_publish`: `{ appPath:"<scratch>\X.app", skipBuild:true, environmentName:"{{SANDBOX_ENV}}", environmentType:"Sandbox", tenant:"{{TENANT_ID}}", schemaUpdateMode:"ForceSync", forceUpgrade:true }`. Keep `forceUpgrade` even though the version now moves: testing branches one after another can send the sandbox version backwards, and it is the safety net after a reverted bump.
4. Discard any `.docx` layout churn the compile left behind (`git status`, then `git checkout --` each one), then **commit the bump**.

**The invariant:** `{{APP_FOLDER}}/app.json` `version` on a branch **is the last build published to the sandbox from that branch**. So bump → build → publish → discard the `.docx` churn → commit `{{APP_FOLDER}}/app.json` alone, as `chore(sandbox): bump build to X.Y.N.0`. If the publish **fails**, revert the bump and commit nothing — an unpublished version number breaks the invariant. Never bump without publishing.

- **Pre-merge publish** → the bump is committed on the feature branch and rides into the PR with the rest of the work. A branch numbers from where it was **cut**, so two branches in flight both claim the same Build and **conflict on the `version` line** at merge. Resolve by keeping the higher number; the post-merge publish re-establishes the invariant on the release branch.
- **Post-merge publish** → the bump is committed directly on `release/X.X.X.X`. Put `[skip ci]` in that commit message — `app.json` is **not** covered by `CICD.yaml`'s `paths-ignore`, so the push would otherwise start a full run, and this project does not use that run as a gate.

**The Build segment is not free — it is part of the released version.** {{APP_NAME}} ships the version that is in `{{APP_FOLDER}}/app.json` at release time, and a patch release uses a non-zero Build. Two consequences: the release process **resets Build to `0`** when it opens the next line (see `supporting-files/documentation/release-process.md`), and a mid-cycle hotfix cannot reuse a Build number the sandbox counter has already burned — pick the next free one.

**A CI artifact's name is not the app's version.** Unless `repoVersion` is set in AL-Go settings, AL-Go names artifacts from its default `1.0` plus the run number and attempt (`…-Apps-1.0.265.0`) whatever `app.json` says. Read an artifact name as a build identifier, never as the version that ships.

**Auth:** `useInteractiveLogin` (the default) opens a browser for Entra sign-in on the first publish — sign in with an account in tenant `{{TENANT_ID}}` that has publish rights; the token caches, so subsequent publishes are silent. No secret is stored.

**Note:** this **dev-endpoint** publish accepts the app's `{{ID_FROM}}–{{ID_TO}}` object range — unlike the Extension Management **UI upload**, which rejects ISV ranges. This project does not use CI as a gate, so this sandbox publish and the user's live test are the check on a running system.

### Running AL tests on the sandbox

AL tests can be **run** against the cloud sandbox, not just compiled — this is how a change gets verified when a clean compile can't prove it (an API round-trip, a posting result). Same MCP server.

**Ask the user which environment before publishing** — publishing force-syncs the schema on a shared environment. Tenant `{{TENANT_ID}}` has `{{SANDBOX_ENV}}` (the app-named dev sandbox). Never use a customer or demo tenant for dev test runs.

1. `al_auth_login { tenant:"{{TENANT_ID}}", environmentType:"Sandbox", environmentName:"{{SANDBOX_ENV}}", usernameHint:"<user>@<domain>" }` — opens a browser and caches the token. A returned `authRequired:false` means it is already good.
2. Build and publish **the app first, then the test app**, each with the prebuilt recipe above (`skipBuild:true`). The Microsoft test toolkit — Test Runner, Library Assert, Application Test Library, Any — must already be installed on the environment.
3. `al_run_tests { codeunitId, testMethods?:[...], environmentType:"Sandbox", environmentName:"{{SANDBOX_ENV}}", tenant:"{{TENANT_ID}}" }`. The result string lists PASS/FAIL per method; an extra blank-named entry is the codeunit-level result.

**Gotcha — outbound HTTP in tests.** The test runner blocks outbound HTTP ("HTTP Client Requests not enabled"). If a code path calls out (a license check, an external API), seed whatever cache that path reads in test setup so it does not call out, and record the pattern here once you have one.

## Pull Request Checklist
- Schema changes include upgrade codeunits or upgrade logic where necessary.
- Data model, captions, and translations stay synchronized.
- Tests added/updated and passing locally before pushing.
- Sandbox live-test pass: branch build published to the **{{SANDBOX_ENV}}** sandbox (AL MCP) and user-confirmed before merge; the merged branch re-published after the merge (see **Sandbox Publish & Live Testing**).
- Release notes under `supporting-files/release-notes/` are updated when user-visible behavior changes.
- `CHANGELOG.md` updated under `[Unreleased]` when user-visible behavior changes.
- End-user guide (`docs/**`) updated in the same PR for **forward-facing enhancements** (required), or when an internal change leaves an existing page inaccurate (conditional) — see **Documentation Maintenance**.

## Changelog Maintenance

`CHANGELOG.md` follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and is the public record of what users receive in each version. Every PR that lands a **user-visible change** must update it.

### What counts as user-visible (entry required)

- New tables, fields, pages, page extensions, reports, or API endpoints
- New or changed actions on pages
- Behavior changes to existing calculation, posting, or payment logic
- Bug fixes to user-facing features (pages, reports, processing)
- Changes to `app.json` metadata that affect the AppSource listing (displayName, description, version, keywords) — **except** the sandbox build-counter bump described in **Sandbox Publish & Live Testing**. That bump is transient dev-loop state: the release process sets the shipped version deliberately and resets Build before anything reaches a customer, so the bump gets no CHANGELOG entry and no release-notes row.

### What does NOT count (no entry needed)

- Internal refactors with no behavior change
- Test-only changes
- CI workflow or AL-Go updates
- Contributor-facing docs (`CLAUDE.md`, `copilot-instructions.md`, `.claude/**`)
- `.gitignore`, `.vscodeignore`, or ruleset changes

### How to update during development

- Maintain a `## [Unreleased]` section at the top of `CHANGELOG.md`.
- Inside `[Unreleased]`, use the Keep-a-Changelog subsections as needed: `### Added`, `### Changed`, `### Fixed`, `### Removed`, `### Deprecated`, `### Security`.
- Each entry is one concise line describing what shipped from the user's point of view.
- Always include the issue and/or PR number at the end of the entry, e.g.: `- Add a credit limit check to sales orders (#253)` or `- Fix journal posting for multi-currency lines (#237, PR #240)`.

### At release time

- Rename `## [Unreleased]` → `## [X.X.X.X] - YYYY-MM-DD` (use the actual publish date).
- Add a fresh empty `## [Unreleased]` section above it for the next cycle.

## Release Notes Maintenance

`supporting-files/release-notes/{{APP_NAME}}-Release-Updates.md` is the **customer-facing** release notes published with the AppSource update. It must be updated alongside `CHANGELOG.md` — the same **user-visible change** criteria above apply (an internal refactor, test-only, or contributor-doc change needs neither file touched).

### How to update during development

- The file is a series of per-build tables, newest at the top. The **top table** is the in-progress release that has not shipped yet.
- Add one row per user-visible change to that top table: `| #<ISSUE_NUMBER> | <title> |`.
- Keep the title customer-readable (what changed from the user's point of view). The issue number goes **only** in the ID column — never repeat it, or the PR number, parenthetically in the Title. A change without its own issue uses the PR number as its ID.
- Both `CHANGELOG.md` and this file are updated in the same PR; don't update one without the other.

### At release time

- The top table's header row gets the published `Update X.X.X.X`, release date (UTC), and build number filled in, and a fresh empty in-progress table is added above it for the next cycle.

## Documentation Maintenance

The `docs/` tree is the **end-user guide** — setup steps, usage procedures, and use-case walkthroughs. Two rules govern updates:

- **Forward-facing enhancements and changes are documented in the same PR — required, not optional.** Any change that adds, alters, or removes customer-facing behavior (a new feature or capability, a new or changed setup field/default, a new page/action/report, a changed procedure or workflow) **must** update the guide: extend the page that covers the area, or **add a new page/section when none exists yet**. Do not defer a forward-facing enhancement's documentation to a follow-up just because the relevant page — or the docs tree itself — is still in flight; coordinate the update into the owning docs branch/PR if needed, but the docs ship with the feature.
- **Everything else is conditional.** For changes that are *not* forward-facing (internal fixes, refactors), update the guide only when the change makes existing documented content inaccurate, incomplete, or misleading. The test is: *does any page under `docs/` now describe the product wrongly because of this change?* If yes, fix it in the same PR.

This is a *different* axis from the **Changelog Maintenance** "user-visible" test. A user-facing **bug fix** that merely restores already-documented behavior is *user-visible* (it needs a CHANGELOG / release-notes entry) but is **not** *forward-facing* (the guide already describes the intended behavior), so it falls under the conditional rule. "Forward-facing" means the customer-facing behavior itself is new or changed — which is what makes documenting it mandatory.

### Writing standard — ASD-STE100 Simplified Technical English

Write all end-user documentation in **ASD-STE100 Simplified Technical English**. This applies to every page under `docs/**` and every documentation enhancement (new pages, updated procedures, use-case walkthroughs). Apply it to customer-facing release-notes rows and CHANGELOG entries where practical. Core rules:

- Use the active voice and the present tense. Write procedures in the imperative mood ("Select the record").
- Write one instruction per sentence. Keep procedural sentences to 20 words or fewer, and descriptive sentences to 25 words or fewer.
- Use one word for one meaning across the docs; prefer the simple word ("use", not "utilize"; "start", not "initiate").
- Keep the articles ("the", "a") — do not drop them telegraph-style.
- Put a warning or caution before the instruction it protects, as its own sentence.
- Keep paragraphs to six sentences or fewer, with one topic per paragraph.

**No absolutes — this rule outranks the others, including active voice.** It applies to **all** documentation the company publishes — every application, every website. Do not make an absolute claim about how the software behaves. Documentation is read as a promise, and an absolute is a warranty the product must then honour in every edge case. Drop *never*, *always*, *every*, *exactly*, *fully*, *ensures*, and *guarantees* from statements about product behavior: write "it **does not** delete it", not "it **never** deletes it"; "An entry is created when…", not "**Every** entry is created when…". Two exemptions: a **directive to the reader** ("Use a sandbox — never production") makes no claim about the software, and a **factual product rule** ("A split must total 100%") is validation the app enforces, not a reliability promise.

The full standard — scope table, sentence limits, the project dictionary, the Technical-Name exemption, and the docs-PR review checklist — lives in `supporting-files/documentation/documentation-style-ste100.md`. Read it before a docs pass.

**The AppSource copy is in scope too.** `docs/appsource-listing.md` and the `app.json` `description` / `brief` are customer-facing product copy, so the no-absolutes rule binds there (rules 8d–8f). The *procedural* rules — sentence limits, imperative mood, the dictionary — do **not**: the listing sells, it does not instruct, so marketing voice is correct there. A competitive or positioning claim ("the only complete solution for X") is a commercial and legal decision owned by the product owner — raise it, do not rewrite it in a style pass.

### When a docs update is needed

- A change to a **setup step, setup field, default, permission, or install flow** that `docs/setup/**` walks through.
- A change to a **procedure or workflow** described in `docs/usage/**`.
- A **new or removed feature** that a `docs/use-cases/**` scenario covers — update the affected use-case, or add one if the feature is significant and unscenarioed.
- A **caption, action name, page layout, or screenshot** that a `docs/` page references by name or image.
- A change to `app.json` metadata or the feature set that the AppSource listing (`docs/appsource-listing.md`) describes.

### When no docs update is needed

- The change isn't reflected on any `docs/` page (e.g., an internal calc fix that preserves the documented behavior, or a tweak to a **pre-existing** field/action that isn't documented and isn't worth documenting — a *new* customer-facing field/action is forward-facing and required above).
- Internal refactors, test-only changes, CI/AL-Go changes, or contributor-facing docs.

Deferring to a docs follow-up issue is only for changes that are **not** forward-facing and genuinely don't fit the current tree — a customer-facing enhancement ships its documentation with the feature rather than being filed for later. Markdown-only docs edits need no compile, CHANGELOG, or release-notes entry of their own.

## Deprecation Tracking

`supporting-files/DEPRECATIONS.md` is the inventory of all obsoleted code, grouped by `ObsoleteTag` removal cohort, so deprecations are removed on a deliberate cadence instead of lingering. Keep it in sync with the code:

- **Obsoleting** an element (`ObsoleteState = Pending` or `[Obsolete(reason, tag)]`) ⇒ add it to the matching tag cohort with its `file:line` and reason.
- **Removing** an element (`ObsoleteState = Removed`) ⇒ move its entry to the Removed-placeholder section. **Deleting** a placeholder ⇒ drop the entry.
- Follow the staging policy documented in that file (Pending → Removed → Deleted) and the `(major − 1)` removal cadence. The change that touches obsoletions and the `DEPRECATIONS.md` edit belong in the same PR.
