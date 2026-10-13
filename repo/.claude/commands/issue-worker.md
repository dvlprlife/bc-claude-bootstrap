---
description: Implement the next agent-ready issue end-to-end — branch, code, compile gate, commits, PR
argument-hint: [issue-number]
---

Implement an issue end-to-end: `$1` if an issue number was given, otherwise the next eligible one.

This runs in the main loop on purpose — it is long, it must stay steerable, and it pauses for a human
sandbox sign-off. Do not delegate the implementation to a subagent.

> **Prerequisite:** `repo-check` has been run. Labels exist and the base branch is known.

## Step 1 — Find the issue

**If `$1` is set**, verify it qualifies:

```
gh issue view $1 --json number,title,labels,state
```

It must be `open`, carry `Agent` + `Status:Ready`, and **not** carry `Review:Plan` (that means a human
still needs to sign off on the plan). If any condition fails, report which and **stop**.

**Otherwise:**

```
gh issue list --label "Agent" --label "Status:Ready" --state open --sort created --json number,title,labels --limit 10
```

Exclude anything with `Review:Plan`, then take the **lowest number**. If nothing is eligible, report
"No issues ready for agent pickup" and **stop**.

## Step 2 — Read the plan and any rework context

```
gh issue view <NUMBER> --json title,body,comments,labels
```

**2a — The plan.** Find the `## Implementation Plan` comment (or a structured list of changes in the
body). If there is **no plan**:

```
gh issue edit <NUMBER> --remove-label "Status:Ready" --add-label "Status:Need Plan"
gh issue comment <NUMBER> --body "No implementation plan found. Marking as Status:Need Plan."
```

Report and **stop**.

**2b — Rework context.** Look for a previous "Agent review requested changes" comment or a PR review.
If present, read that review — those findings are the priority for this pass, with the plan as the
overall guide.

## Step 3 — Move the labels

```
gh issue edit <NUMBER> --remove-label "Status:Ready" --add-label "Status:In Progress"
```

## Step 4 — Branch

Find the base branch — the highest `release/X.X.X.X`, or `main` if none exists:

```
git ls-remote --heads origin "release/*"
git fetch origin <BASE_BRANCH>
```

Keep the base branch name; Step 7 needs it. Check whether a branch already exists (rework):

```
git branch -a --list "*<ISSUE_NUMBER>-*"
```

- **Exists** (rework): `git checkout <BRANCH>` then `git rebase origin/<BASE_BRANCH>`.
- **New:** `git checkout -b <ISSUE_NUMBER>-<slug> origin/<BASE_BRANCH>`

Slug: lowercase, spaces to hyphens, special characters removed, 50 characters max — e.g.
`239-credit-limit-check-on-sales-orders`. The number prefix matters: `CloseLinkedIssueOnMerge.yaml`
closes the issue matching it on merge.

## Step 5 — Implement

1. Follow the plan **step by step**. On a rework pass, lead with the reviewer's findings.
2. Write to the bar the reviewer applies — the repo-specific checklist in
   `.claude/agents/pull-request-reviewer.md` §4b, plus the conventions in `CLAUDE.md` and the
   `.github/copilot-instructions.md` it imports. Reading §4b before you start is cheaper than a bounce.
3. The upkeep the reviewer will check, restated only as a reminder of *when* each applies:
   - **User-visible change** → `CHANGELOG.md` under `## [Unreleased]` **and** a row in the top table of
     `supporting-files/release-notes/{{APP_NAME}}-Release-Updates.md`. Both, together.
   - **Forward-facing change** → the relevant `docs/**` page, **in this PR**. Add a page or section when
     none fits. Do not defer this because the `docs/` tree is in flight — fold it into the branch where
     those pages live.
   - **Obsoleting or removing anything** → `supporting-files/DEPRECATIONS.md`.
   - **New objects** → both permission sets (and the test permission set for test objects).
4. Commit in **Conventional Commits** form, referencing the issue:
   `feat(sales): add a credit limit check to sales orders (#239)`. Make focused, logical
   commits — do not lump the whole change into one.
5. Do **not** add tests unless the plan calls for them. If you think they are warranted, say so in the
   PR body instead of adding them unprompted.

## Step 6 — Compile gate + BCQuality

This is the mechanical enforcement of the AL guidelines and **must** pass before you push.

Invoke the **`bc-review` skill** on your working diff. It is **two gates**: the compile with all four
analyzers — CodeCop, AppSourceCop, PerTenantExtensionCop, UICop — against
`supporting-files/{{PREFIX}}.ruleset.json`, *then* the BCQuality knowledge-base review. Use the single build
command in `CLAUDE.md`: the `al` CLI in **PowerShell** with `/packagecachepath` pointed at
`supporting-files\.alpackages`. Not the MCP's own build, which cannot see that cache.

- Compile every project you changed **plus its dependents** — a change under `{{APP_FOLDER}}/` can break
  every project that depends on it ({{DEPENDENT_FOLDERS}}).
- **Zero errors.** Fix every `ALxxxx` and analyzer error before continuing.
- Triage every new warning (`AAxxxx`, `ASxxxx`, `PTExxxx`, `AWxxxx`): fix it, or justify it in the PR
  body. Do not leave new unexplained warnings in changed files.
- **Clear every knowledge-backed BCQuality blocker and major** before pushing — the reviewer treats
  them exactly like a build error and will bounce the PR. Fix each minor or justify it in the PR body.
  Findings with empty `references[]` are advisory; use judgment.
- A failed compile stops `bc-review` before the BCQuality passes. After fixing the errors, **re-run
  it** so that half actually runs — a green compile is not the gate on its own.
- Run existing tests covering the changed areas and fix failures. Add tests only if the plan asked.
- **Before committing**, revert the compile's side effect:
  run `git status` and `git checkout --` each `.docx` report layout you did not change.

## Step 7 — Push and open or update the PR

```
git push -u origin <BRANCH_NAME>
gh pr list --head <BRANCH_NAME> --state open --json number,url --limit 1
```

**No PR yet:**

```
gh pr create --base <BASE_BRANCH> --title "<type>(<scope>): <description> (#<ISSUE_NUMBER>)" --body "<BODY>"
```

The body needs `Closes #<ISSUE_NUMBER>` on the first line, a **Summary**, and the plan steps as a
checklist with completed items ticked. Add the `Agent` label to the PR. Note in the body anything you
deliberately left out — skipped tests, justified warnings.

**PR exists** (rework): the push updates it. Comment what changed:

```
gh pr comment <PR_NUMBER> --body "Rework pushed addressing review findings from the previous pass."
```

Then:

```
gh issue edit <NUMBER> --remove-label "Status:In Progress" --add-label "Status:In Review"
```

## Step 8 — Publish to the {{SANDBOX_ENV}} sandbox

This project does not use a GitHub CI run as a gate. The gates are the two `bc-review` gates from Step 6
and the sandbox. Do not start a `CICD.yaml` run unless the user asks.

Publish the branch build with the recipe in the **Sandbox Publish & Live Testing** section of
`.github/copilot-instructions.md`: bump the Build segment in `{{APP_FOLDER}}/app.json`, build with the
command in `CLAUDE.md`, publish the prebuilt package to {{SANDBOX_ENV}} with `al_publish`, then commit the bump
as `chore(sandbox): bump build to X.Y.N.0` and push it to the PR branch. If the publish fails, revert
the bump and report the error.

## Step 9 — Report, and stop for the human gate

Report the PR URL, what landed, and the version now in the sandbox. Then flag what is **not** yours to do:

- **Sandbox sign-off.** Merging waits on the user's live test in {{SANDBOX_ENV}}. Say plainly what a clean
  compile did *not* prove — anything only a running system can verify.

Do not merge the PR. After the user merges it, publish the merged build to {{SANDBOX_ENV}}, then delete the
merged branch locally and on `origin`.
