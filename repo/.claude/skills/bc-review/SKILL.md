---
name: bc-review
description: Compile-check and review AL/Business Central code. On a PR or branch diff it first compiles via the al CLI as a gate — with all four code analyzers enabled (CodeCop, AppSourceCop, PerTenantExtensionCop, UICop), and only using the AL MCP server's al_downloadsymbols to fetch global-source symbols when they're missing — then reviews against the BCQuality knowledge base (performance, security, privacy, upgrade, style, UI). Use when asked to review/validate a PR, a diff, the current branch changes, or a specific .al file, or to look up the BC coding guideline behind a finding.
---

# BC code review (via BCQuality)

Reviews Business Central AL code by consuming the **BCQuality** knowledge base — a
curated, machine-readable set of BC-specific coding guidelines and review skills.
The guidelines are *not* in this repo; they live in BCQuality and are referenced at
review time, so each BC project uses the same guidelines without copying docs around.

This skill is the entry point. It does not contain the guidelines — it points the
agent at BCQuality and follows BCQuality's own dispatch contract.

## Locate BCQuality

BCQuality is a sibling clone in the same parent folder as this project. Resolve it
in this order and use the first that exists:

1. `$BCQUALITY_PATH` environment variable, if set.
2. `../BCQuality` (sibling of this repo — the normal case).
3. Ask the user for the path.

If the folder is missing, tell the user to clone the project's BCQuality repo (`git clone
https://github.com/{{BCQUALITY_REPO}}` next to this repo) rather than guessing
— do not invent guideline content from memory.

Refer to the resolved location below as `<BCQ>`.

**The project's BCQuality repo is `{{BCQUALITY_REPO}}`.** That is upstream
`microsoft/BCQuality`, or a fork of it. A fork can hold organization-specific AL rules
in its `custom/` layer. Upstream has an empty `custom/`, so a clone of upstream reviews
with the `microsoft` and `community` layers, and that is a complete review. If the
project's repo is a fork but `<BCQ>` points at upstream (its `origin` remote is
`microsoft/BCQuality`), note in the report that custom rules were not applied.

## What to review (resolve the target from the user's request / args)

- **No argument, or "current branch" / "this branch"** → the diff of the current
  branch against its base. Resolve the base in this order: the open PR's base
  (`gh pr view --json baseRefName -q .baseRefName`); else this repo's default branch
  (`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`, or
  `git symbolic-ref --short refs/remotes/origin/HEAD` stripped of its `origin/`
  prefix). Do not hardcode a branch name — it differs per project. Use
  `git diff <base>...HEAD`. Pass it as `pr-diff`.
- **A PR number** (e.g. `/bc-review 333`) → `gh pr diff <n>`. Pass as `pr-diff`.
- **"staged"** → `git diff --staged`. Pass as `pr-diff`.
- **A file path ending in `.al`** → review that whole file. Pass as `file-path`.

## Phase 0 — Compile gate (always, before reviewing a PR or branch diff)

Before the BCQuality review, confirm the code **builds**. A review of code that doesn't
compile wastes effort and produces misleading findings. This gate is **CLI-first**:
`al` compiles directly in PowerShell, with no MCP dependency. The AL MCP server is needed
**only** to download symbols when they're missing (the CLI has no symbol-download
command) — see step 2.

**Determine which projects to compile (this is a multi-project workspace).** Each
directory containing an `app.json` (excluding any `.alpackages`) is a separate AL project.
Discover them dynamically — do **not** hardcode folder names:
`Get-ChildItem -Recurse -Filter app.json | Where-Object FullName -notmatch '\\.alpackages\\'`
(currently `{{APP_FOLDER}}/` plus {{DEPENDENT_FOLDERS}}). Then build the set of **affected projects**:
- **Diff / PR review:** map every changed `.al` file to its enclosing project (the nearest
  ancestor directory that contains an `app.json`) and compile **each distinct affected
  project**. A change in `{{APP_FOLDER}}` plus a change in `{{TEST_FOLDER}}` ⇒ compile both.
- **Dependency fan-out:** the other projects depend on `{{APP_FOLDER}}` (see their
  `app.json` `dependencies`), so a change under `{{APP_FOLDER}}/` can break a dependent
  project's compile even though its own files didn't change. When the diff touches
  `{{APP_FOLDER}}`, compile it first with `/out:` into the shared cache (see below), then
  compile every project that depends on it — or simply compile all of them when unsure.
- **Single-file review:** compile just that file's enclosing project.

`<PROJECT>` in the steps below means **each affected project in turn** — run steps 1–3
once per project and aggregate the results.

The symbol cache is shared across all projects at `<repo>/supporting-files/.alpackages`
(don't assume — if a compile can't find symbols, check `al.packageCachePath` in the
project `settings.json` / the `.code-workspace`). Use that same `<cache>` for every
project. Note `{{APP_FOLDER}}`'s own symbols are NOT on global sources, so a dependent
project's compile depends on the app package being present in this shared cache — compile
`{{APP_FOLDER}}` with `/out:"$repo\supporting-files\.alpackages\<Publisher>_{{APP_NAME}}_<version>.app"`
when it is missing or stale, and keep one {{APP_NAME}} package there. The MCP
`al_downloadsymbols globalSourcesOnly` route cannot supply it; the CLI with this explicit
cache is the reliable path for the dependent projects.

1. **Compile with code analysis (PowerShell — the default).** Always run **all four code
   analyzers** (CodeCop, AppSourceCop, PerTenantExtensionCop, UICop) — the project ruleset
   filters/hides what shouldn't fire. The analyzer DLLs ship with the `al` dotnet global
   tool under its `.store` tree; resolve their folder portably (don't hardcode the version).
   Run it in **PowerShell** with native Windows paths — not Bash with `MSYS_NO_PATHCONV=1`,
   which leaves `/out:` as a unix path that Windows `alc` silently fails to write:

   ```powershell
   $repo = (git rev-parse --show-toplevel) -replace '/', '\'
   $alDir = Split-Path (Get-Command al).Source
   $az = Split-Path (Get-ChildItem "$alDir\.store" -Recurse -Filter Microsoft.Dynamics.Nav.CodeCop.dll | Select-Object -First 1).FullName
   & "$az\alc.exe" /project:"$repo\<PROJECT>" /packagecachepath:"$repo\supporting-files\.alpackages" `
     /out:"$env:TEMP\bc-review-<PROJECT>.app" /ruleset:"$repo\supporting-files\{{PREFIX}}.ruleset.json" `
     /analyzer:"$az\Microsoft.Dynamics.Nav.CodeCop.dll" /analyzer:"$az\Microsoft.Dynamics.Nav.AppSourceCop.dll" `
     /analyzer:"$az\Microsoft.Dynamics.Nav.PerTenantExtensionCop.dll" /analyzer:"$az\Microsoft.Dynamics.Nav.UICop.dll"
   ```

   Call `alc.exe` directly, not `al compile --`: as observed with AL tools 18.0.43, the wrapper
   does not load the analyzers (an `AL1003` warning for each rule) and still exits 0. Check
   this again after an AL tools update. Treat an `AL1003` in the output as "the analyzers did
   not run", not as a pass.

   Exit code 0 = builds. Delete the throwaway `.app` after. If the compile rewrote any
   `.docx` report layout as a side effect, `git checkout --` it. (The CLI
   `/analyzer:<dll>` flags are the equivalent of the MCP `al_compile` params
   `enableCodeAnalysis: true` + `codeAnalyzers: ["${CodeCop}","${AppSourceCop}","${PerTenantExtensionCop}","${UICop}"]`.)

2. **Only if the compile fails on MISSING SYMBOLS** (e.g. "symbol ... could not be
   found" — fresh clone, BC-version bump, or a new `app.json` dependency), fetch them via
   the AL MCP server, then re-run step 1. The CLI cannot download symbols; this is the
   one job the MCP server (`al` in `.mcp.json`) exists for:
   `al_downloadsymbols { "projectPath": "<PROJECT>", "globalSourcesOnly": true }` —
   platform/base symbols from Microsoft NuGet/AppSource, **no BC server or auth**, skips
   already-cached. The download lands in `<PROJECT>/.alpackages`, and step 1 reads only the
   shared cache: copy the downloaded `.app` files to `<repo>/supporting-files/.alpackages`
   before you re-run step 1. If the `al` server isn't approved/connected, tell the user to
   approve it or run VS Code's "AL: Download Symbols" — do not guess at symbols.

3. **Capture diagnostics (always — this is a required output, not optional).** Every run
   of this gate must surface the compiler diagnostics, by whichever path produced the
   compile:
   - **CLI compile (the default):** parse `alc`'s stdout. Each diagnostic is
     `path(line,col): error|warning <CODE>: message`, where `<CODE>` is a compiler code
     (`ALxxxx`) **or** an analyzer code — `AAxxxx` (CodeCop), `ASxxxx` (AppSourceCop),
     `PTExxxx` (PerTenantExtensionCop), `AWxxxx` (UICop). Collect all errors **and**
     warnings with their code and `file:line`.
   - **MCP compile:** if you compiled via the MCP server (`al_compile`), read diagnostics
     with `al_getdiagnostics { "projectPath": "<PROJECT>", "severities": ["error","warning"] }`
     (raise `limit` toward its 500 max on a noisy project) for structured JSON.
     ⚠️ `al_getdiagnostics` reflects the **MCP server's own `al_compile` session**, not a
     CLI compile — so only use it when the compile itself was `al_compile` in that same
     session (e.g. right after the step-2 symbol download). Do not CLI-compile and then
     call `al_getdiagnostics` expecting the CLI's results.
   Either way, produce a per-project diagnostics summary (counts by severity + the list:
   code, `file:line`, message), then an **aggregate** summary across all affected projects.
   This is part of the gate's output every time, with each finding attributed to its project.

**Gate behavior (aggregated across every affected project):**
- **Errors in ANY affected project** (non-zero exit / any error-severity diagnostic) →
  STOP. Report the compiler errors first (project + code + file:line + message) and do
  **not** run the BCQuality review yet — the change doesn't build. Offer to fix, then
  re-run Phase 0. One project failing fails the gate even if the others compile.
- **All affected projects clean / warnings only** → report the per-project warning counts,
  then proceed to the BCQuality review below. Feed the compiler diagnostics into the report
  alongside the BCQuality findings (compiler diagnostics are authoritative and outrank
  advisory findings).

Fully-MCP variant (when the `al` server is connected and you prefer structured output
end-to-end): `al_downloadsymbols { globalSourcesOnly: true }` (if needed) →
`al_compile { onlyErrors: false, enableCodeAnalysis: true, codeAnalyzers: ["${CodeCop}","${AppSourceCop}","${PerTenantExtensionCop}","${UICop}"] }`
→ `al_getdiagnostics { severities: ["error","warning"] }`. Same gate behavior; all three
steps share one MCP session so diagnostics line up with the compile. (Note: as observed
with AL tools 18.0.43, the MCP `al_compile` reads symbols from the default
`<project>/.alpackages` and does **not** honor `al.packageCachePath`, so the symbols must
be there — `al_downloadsymbols` puts them there.)

## How to run BCQuality

BCQuality's only hardcoded convention is: **invoke its entry-point skill first**, then
follow the dispatch record it returns. Do exactly that — do not pre-pick review skills.

1. **Invoke Entry.** Read `<BCQ>/skills/entry.md` and execute it against this
   task-context (omit dimensions you cannot determine — omitted means "unconstrained").
   Entry's own preparation step rebuilds `<BCQ>/knowledge-index.json` (via
   `pwsh ./tools/Build-KnowledgeIndex.ps1`) over the current clone, so do not rebuild
   it separately — just follow Entry. Task-context:

   ```yaml
   task-context:
     goal: "review BC AL code changes for quality and coding-guideline compliance"
     inputs-available: [pr-diff]      # or [file-path] for a single file
     technologies: [al]
     enabled-layers: [microsoft, community, custom]
   ```

   Entry returns a **dispatch record** naming the action skill(s) to run (normally
   `microsoft/skills/review/al-code-review.md`).

2. **Run each dispatched skill.** For each entry in the dispatch list, read the
   referenced skill file under `<BCQ>` and execute its Source → Relevance → Worklist →
   Action steps, reading `<BCQ>/skills/read.md` and `<BCQ>/skills/do.md` on demand for
   the knowledge-file format and the output contract. The super-skill runs each leaf
   domain (performance, security, privacy, upgrade, style, UI) as its own pass, then a
   cross-cutting self-review pass — keep them as discrete passes, do not collapse them.

   For a large diff this is genuinely parallel work — one self-contained pass per leaf
   domain — so prefer fanning the leaf passes out across subagents (the Agent tool),
   each given the BCQuality leaf-skill path, its knowledge-domain folder under `<BCQ>`,
   and the target. Then roll the sub-reports up yourself.

## How to report findings

Present the rolled-up findings grouped by severity (blocker → major → minor → info).
For each finding give: `file:line`, the message, and — for knowledge-backed findings —
the BCQuality guideline path from `references[]` so the user can open the rule. Clearly
separate the two kinds of finding:

- **Knowledge-backed** — cite the guideline file. These are the enforceable ones.
- **Agent findings** (`from-sub-skill: "agent"`, empty `references`) — flag as advisory
  / lower-confidence; never treat them as gating.

End with the BCQuality `outcome` (`completed` / `no-knowledge` / `partial` / etc.) and a
one-line summary. Do not modify code unless the user asks — this skill reviews and
reports; offer to apply mechanical `suggested-code` fixes if any are present.

## When a real issue has no guideline behind it

If the review (or a bug that was just fixed) is something BCQuality has **no** knowledge
file for, mention the gap. When the project's BCQuality repo is a fork, that is the signal
to add a file to its `custom/` layer — that is how the guideline set compounds over time.
Offer to draft the `custom/knowledge/<domain>/<slug>.md` file (plus `.good.al` / `.bad.al`
samples) per `<BCQ>/skills/write.md`. With upstream `microsoft/BCQuality`, there is no
`custom/` layer to add to: tell the user that a fork is the place for such a rule.
