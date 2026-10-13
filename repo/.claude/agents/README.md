# Issue workflow

The issue-to-merge lifecycle, wired into Claude Code. Conventions live in `CLAUDE.md` (which imports
`.github/copilot-instructions.md`); this file covers the loop itself.

## The four pieces

| Piece | Location | Shape | Makes code changes? |
|---|---|---|---|
| `repo-check` | `.claude/agents/repo-check.md` | subagent | no |
| `issue-planner` | `.claude/agents/issue-planner.md` | subagent | no |
| `/issue-worker [issue#]` | `.claude/commands/issue-worker.md` | slash command | **yes** |
| `pull-request-reviewer` | `.claude/agents/pull-request-reviewer.md` | subagent | no |

Subagents run in an isolated context and hand back a report — right for read-and-report work, and the
reason the three non-editing pieces have no edit tools at all.

**`issue-worker` is deliberately a slash command, not a subagent.** It is long and multi-step, it must
stay steerable while it runs, its deliverable is the branch/commits/PR rather than a report, and it has
to pause for the human sandbox sign-off — none of which a subagent can do.

The code-review half of `pull-request-reviewer` delegates to the **`bc-review` skill**
(`.claude/skills/bc-review/`), which owns two gates — the compile with all four analyzers, and the
BCQuality knowledge-base review — both required to approve. The reviewer keeps only what is
repo-specific: acceptance-criteria verification, the changelog / release-notes / docs / deprecations
checklist, label transitions, and the merged-issue reconcile.

## Label lifecycle

```
                                    ┌─────────────────────────────────────────────────────┐
                                    │                  (changes requested)                 │
                                    ▼                                                     │
┌──────────────────┐     ┌──────────────┐     ┌────────────────────┐     ┌──────────────────┐
│ Status:Need Plan │────▶│ Status:Ready │────▶│ Status:In Progress │────▶│ Status:In Review │
└──────────────────┘     └──────────────┘     └────────────────────┘     └──────────────────┘
        │                    │                                                    │
        │ (vague)            │ (no plan)                                          │ (approved)
        ▼                    ▼                                                    ▼
   ┌──────────┐       ┌──────────────────┐                                ┌──────────────┐
   │ follow up│       │ Status:Need Plan │                                │ Review:Merge │
   └──────────┘       └──────────────────┘                                └──────────────┘
                                                                                  │
                                                                                  │ (human merges)
                                                                                  ▼
                                                                            PR merged
                                                                           issue closed
```

| Label | Color | Meaning |
|-------|-------|---------|
| `Agent` | Purple (`#5319E7`) | Issue eligible for agent pickup |
| `Status:Need Plan` | Pink (`#E99695`) | Needs an implementation plan |
| `Status:Ready` | Green (`#0E8A16`) | Planned and ready to implement |
| `Status:In Progress` | Yellow (`#FBCA04`) | Actively being worked |
| `Status:In Review` | Blue (`#1D76DB`) | PR created, awaiting review |
| `Review:Plan` | Light purple (`#D4C5F9`) | Plan needs human review before pickup |
| `Review:Merge` | Green (`#C2E0C6`) | Approved — human performs the final merge |
| `follow up` | Teal (`#285E5D`) | Needs more information before it can be planned |

`repo-check` creates any of these that are missing.

## The loop

1. **`repo-check`** — verify `gh` auth, create missing labels, confirm the base branch (the active
   `release/X.X.X.X` branch, or `main` when the project has no release branch).
2. **New issue** — labelled `Agent` + `Status:Need Plan` by hand.
3. **`issue-planner`** — posts a plan, moves to `Status:Ready` + `Review:Plan`.
4. **Human reviews the plan** — removes `Review:Plan` to release it for pickup. The worker does
   not pick up an issue that still has that label.
5. **`/issue-worker`** — implements, compiles, commits, opens the PR, moves to `Status:In Review`.
6. **`pull-request-reviewer`** — reviews against conventions and acceptance criteria.
7. **Changes requested** → back to `Status:Ready`; `/issue-worker` reworks the same branch (step 5).
8. **Approved** → `Review:Merge`. Claude publishes the branch build to the {{SANDBOX_ENV}} sandbox. The live test and the merge are the human's.
9. **Merged** → `CloseLinkedIssueOnMerge.yaml` strips the status labels and closes the issue.
10. **Cleanup** → Claude publishes the merged build to the {{SANDBOX_ENV}} sandbox, then deletes the merged
    branch locally and on `origin`, and switches the checkout to the base branch.

## The gates around the merge

- **No GitHub CI gate.** This project does not use a CI run as a gate. Do not start a `CICD.yaml` run
  and do not wait for one, unless the user asks.
- **Compile and publish.** When both `bc-review` gates pass, Claude builds the branch package and
  publishes it to the {{SANDBOX_ENV}} sandbox. This is part of the process, not an optional step.
- **Sandbox sign-off.** The merge waits on a live human test in the {{SANDBOX_ENV}} sandbox. A clean compile is
  not this gate.

## Run the loop automatically

The `/loop` command makes Claude check GitHub on an interval and move one issue forward one stage in
each run. Without it, the user starts each stage with one sentence.

Before the loop starts:

1. Keep one Claude Code session open in the repo, on the base branch. The loop runs in that session
   and stops when the session closes.
2. Use a permission mode that does not stop for prompts. If not, the loop waits at the first push,
   compile, or publish prompt.

Then type this command. Change `10m` to the interval you want.

```
/loop 10m Check this repo on GitHub for agent work and move one issue forward one stage. In order: (1) a merged PR whose branch still exists: publish the merged build to {{SANDBOX_ENV}}, then delete the branch; (2) an open issue with Agent whose Review:Merge label the user removed after the agent review (the issue events show an unlabeled event for Review:Merge, the issue has no Status:* or Review:* label, and its PR is open with the agent review posted): the removal is the user's sign-off after their sandbox test, so merge that PR with `gh pr merge <number> --merge`; (3) an open issue with Agent + Status:In Review and no review yet: run pull-request-reviewer; (4) an open issue with Agent + Status:Ready and no Review:Plan: run /issue-worker on the lowest number, including its publish to {{SANDBOX_ENV}}; (5) an open issue with Agent + Status:Need Plan: run issue-planner. Do one stage for each run. The user's removal of Review:Plan approves the plan and the whole worker stage, including the publish to the {{SANDBOX_ENV}} sandbox when the publish changes or drops sandbox data. Do not stop to ask about plan-review decisions or the publish; use the plan's defaults and report what you did. Do not merge a PR while its issue still has Review:Merge; say that it is ready and wait for the user to remove the label. If nothing is ready, say so in one line.
```

With the loop in operation, the user's part for each issue is:

1. Open the issue with the labels `Agent` and `Status:Need Plan`.
2. Remove `Review:Plan` to accept the plan. This also approves the worker stage, and that includes a
   publish to {{SANDBOX_ENV}} that changes or drops sandbox data. To change a plan decision, comment on
   the issue before you remove the label.
3. Test the build in {{SANDBOX_ENV}}. Then remove `Review:Merge` from the issue. The removal is the sign-off
   after the test, and the loop merges the PR on its next run. A merge on GitHub also works.

The merge stage needs permission for an unattended merge. The kit's `.claude/settings.json` does not
pre-approve `gh pr merge`, so Claude Code asks before each merge, and in some permission modes it
can refuse to schedule a loop that merges. The user decides whether to allow the merge in their own
Claude Code settings. Without that, use the command with stage (2) removed, and merge on GitHub or
tell Claude to merge.

The loop works on one issue at a time, because the repo has one working folder and one sandbox
version counter. When CI is a gate in the project, add a stage to the command that waits for a
green run before the publish.
