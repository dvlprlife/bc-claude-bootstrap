# {{APP_NAME}} — Claude Code instructions

Business Central AL extension (**{{APP_NAME}}**, publisher {{PUBLISHER}}), built with AL-Go for GitHub.

## Project conventions

The full convention set — object ranges, `{{PREFIX}}` prefix rules, coding patterns, BC development
patterns, testing expectations, the PR checklist, and the changelog / release-notes / docs /
deprecations maintenance rules — lives in one file and is imported here:

@.github/copilot-instructions.md

Read it as authoritative. Everything below is Claude-Code-specific wiring that is *not* in that file.

## Building and compiling

**One build command.** Compile in **PowerShell** with native Windows paths, using the out-of-tree
symbol cache and all four analyzers:

```powershell
$repo = (git rev-parse --show-toplevel) -replace '/', '\'
$alDir = Split-Path (Get-Command al).Source
$az = Split-Path (Get-ChildItem "$alDir\.store" -Recurse -Filter Microsoft.Dynamics.Nav.CodeCop.dll | Select-Object -First 1).FullName
& "$az\alc.exe" /project:"$repo\{{APP_FOLDER}}" /packagecachepath:"$repo\supporting-files\.alpackages" `
  /out:"$env:TEMP\{{APP_NAME}}.app" /ruleset:"$repo\supporting-files\{{PREFIX}}.ruleset.json" `
  /analyzer:"$az\Microsoft.Dynamics.Nav.CodeCop.dll" /analyzer:"$az\Microsoft.Dynamics.Nav.AppSourceCop.dll" `
  /analyzer:"$az\Microsoft.Dynamics.Nav.PerTenantExtensionCop.dll" /analyzer:"$az\Microsoft.Dynamics.Nav.UICop.dll"
```

- Call **`alc.exe` directly**, not `al compile --`. As observed with AL tools 18.0.43, the
  `al compile` wrapper does not load the analyzers (one `AL1003` warning for each rule,
  `System.Collections.Immutable` not found) and still exits 0, so the build looks clean while no
  analyzer ran. Check this again after an AL tools update. An `AL1003` in the output means the
  gate did not run.
- Do **not** use the AL MCP server's own build (`al_build`) to produce a package — as observed with
  AL tools 18.0.43, it reads `<project>/.alpackages` and does not see `supporting-files/.alpackages`.
  Use `al_downloadsymbols` when symbols are genuinely missing, copy the downloaded `.app` files from
  `<project>/.alpackages` to `supporting-files/.alpackages`, then re-run the command above.
- **Where `/out:` goes.** For a check of `{{APP_FOLDER}}` alone, `$env:TEMP` is fine. When a dependent
  project compiles next, or the app's objects changed, set `/out:` to
  `$repo\supporting-files\.alpackages\{{PUBLISHER_FILE}}_{{APP_NAME}}_<version>.app` and delete the
  superseded package (see **Symbol cache** below). For a dependent project, `$env:TEMP` is fine.
- Do **not** run this through Bash with `MSYS_NO_PATHCONV=1` — that leaves `/out:` as a unix path
  and Windows `alc` silently fails to write the file.
- A change under `{{APP_FOLDER}}/` can break every project that depends on it ({{DEPENDENT_FOLDERS}});
  compile those too.
- All four analyzers — **CodeCop, AppSourceCop, PerTenantExtensionCop, UICop** — against
  `supporting-files/{{PREFIX}}.ruleset.json`. Zero errors is the gate.
- The `bc-review` skill wraps this compile gate plus the BCQuality review; prefer invoking it over
  hand-rolling a review.

**Symbol cache:** `supporting-files/.alpackages` is git-ignored, so a fresh clone starts empty. It
holds the Microsoft symbols plus the {{APP_NAME}} package that the dependent projects resolve
`{{PREFIX}}` objects from. That package's filename carries the version, so it moves with every
sandbox build-counter bump (`{{PUBLISHER_FILE}}_{{APP_NAME}}_1.0.0.0.app` → `…_1.0.1.0.app`). `alc`
takes the **highest** version present, so when you refresh the symbol, write it under the current
version and **delete the superseded file** — leave one {{APP_NAME}} package in the cache rather than
relying on the compiler to choose between two.

**Gotcha — report layouts:** when the app has Word (`.docx`) report layouts, `al compile` can rewrite
them as a side effect. Run `git status` after a compile and `git checkout --` any `.docx` you did not
change before committing.

## Branching and CI

- Active development targets the highest **`release/X.X.X.X`** branch, not `main`. Feature branches are
  `<issue#>-<slug>` with **no `feature/` prefix**.
- **AL-Go's `PullRequestHandler.yaml` only builds PRs based on `main`.** A PR into `release/*` gets
  **no pre-merge build**. To CI-test one first: `gh workflow run CICD.yaml --ref <branch>` — a
  `workflow_dispatch` ignores the path filters below. Confirm the `branches:` filter in your copy of
  `PullRequestHandler.yaml`.
- **AL-Go's `CICD.yaml` runs on push to `main` / `release/*` / `feature/*`, with
  `paths-ignore: ['**.md', '.github/workflows/*.yaml', '!.github/workflows/CICD.yaml']`** in the
  default template (a change to `CICD.yaml` itself does trigger a run). Confirm the `on:` block in
  your copy. Two consequences:
  - Our feature branches carry no `feature/` prefix, so pushing to one does not auto-trigger it.
  - A **docs-only change triggers no run at all**, even merged straight into a release branch.
- **`CloseLinkedIssueOnMerge.yaml` auto-closes the issue matching the branch-name number prefix**
  (`372-…` closes #372), regardless of Refs-vs-Closes wording. For a change that should *not* close an
  issue, don't prefix the branch with a number. The workflow runs from the PR's **base** branch, so it
  must exist on `main` and on each active `release/X.X.X.X`.
- **Clean up after a merge.** When a PR is merged, delete its branch locally and on `origin`
  (`git branch -d <branch>`, `git push origin --delete <branch>`), switch to the base branch, and
  pull. Do this as part of the process, without a separate request. Delete only a branch that
  GitHub reports as merged; do not delete `main` or a `release/*` branch.
- **GitHub CI is not a gate in this project.** Do not start a `CICD.yaml` run and do not wait for
  one, unless the user asks. The CI facts above stay as reference.
- **The gates are the local compile and the {{SANDBOX_ENV}} sandbox.** When both `bc-review` gates pass
  (the compile with the four analyzers, and BCQuality), build the branch package and publish it to
  the {{SANDBOX_ENV}} sandbox as part of the process. The merge waits for the user's live test there. See
  the **Sandbox Publish & Live Testing** section of the imported instructions for the recipe.

## Issue workflow

The issue-to-merge lifecycle is wired into this harness. Use it rather than working issues ad hoc:

| Stage | Invoke | Shape |
|---|---|---|
| Verify prerequisites (auth, labels, base branch) | `repo-check` | subagent |
| Plan an issue | `issue-planner` | subagent |
| Implement an issue | `/issue-worker [issue#]` | slash command (runs in the main loop) |
| Review the resulting PR | `pull-request-reviewer` | subagent |

`issue-worker` is deliberately **not** a subagent: it is long-running, must stay steerable mid-flight,
its deliverable is the branch/commits/PR rather than a report, and it has to pause for your sandbox
sign-off. See `.claude/agents/README.md` for the label lifecycle and the full loop.
