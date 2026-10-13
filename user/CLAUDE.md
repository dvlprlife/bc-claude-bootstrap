# Global instructions

## Writing standard for documentation and notes

All documentation and notes I write follow **Simplified Technical English (based on ASD-STE100)**
and the **no-absolutes** rule. This covers user guides, release notes and changelog entries,
AppSource copy, GitHub issues and PR descriptions, commit-message bodies, and dashboard text.

- **No absolutes, universal qualifiers, or warranty language** about how the software behaves:
  do not use *never*, *always*, *every*, *all*, *exactly*, *fully*, *ensures*, *guarantees*, or
  *100%* in a claim about behavior. Write "it does not delete it", not "it never deletes it".
  This rule outranks the others, including active voice.
- Two exemptions: a **directive to the reader** ("Use a sandbox, not production") and a
  **validation rule the software enforces** ("Splits must total 100%").
- STE basics: short sentences (20 words for an instruction, 25 for a description), one
  instruction for each sentence, active voice, imperative instructions, present tense, no *via*,
  *in order to*, *utilize*, *prior to*, *e.g.*, *i.e.*, or *etc.*
- Where a repo has its own style guide (for example
  `supporting-files/documentation/documentation-style-ste100.md` in a Business Central app repo), that guide
  is the full rule set and wins on detail.
- Quoted requirement text from a spec (an FRD's acceptance criteria) stays as written.

## Progress dashboard for long tasks

For any task with **more than 5 steps**, or that should take **longer than 30 minutes**:

1. **Before starting**, have the `dashboard-builder` subagent set up the project dashboard (or
   reuse the existing one — it keeps the URL in `~/.claude/dashboards/<repo>/dashboard.json`).
   Give it the task breakdown (each step as a task), the current phase, and any links. Tell me the
   dashboard URL once.
2. **Once per phase**, have `dashboard-builder` update it: the status of each task the phase
   closed, deliverables produced (commits, PRs, CI runs, files, artifacts), next steps, and a
   one-line activity entry. A phase is a coherent chunk of work with a result worth recording —
   an issue implemented, a review completed, a PR opened, a release published — not an
   individual step. Batch the steps of a phase into one update. Update sooner than the end of a
   phase only when the phase is long-running and something notable lands mid-way (a gate fails,
   a decision changes the plan). Within a session, continue the same `dashboard-builder` agent
   (SendMessage) instead of spawning a new one each time.
3. **When you need a decision from me**, add it to the dashboard's questions list with the
   default you are taking, and **keep going with that default** — don't stop and wait, and don't
   use AskUserQuestion for it. Check the `ANSWERS:` line of each dashboard-builder report: when I
   have answered a question on the page, act on my answer (and adjust any work done under the
   default).

**Exception:** actions that can't be undone or that leave this machine — pushing, merging,
deleting, force operations, publishing or sending anything externally — still need my explicit
go-ahead. For those, add the question to the dashboard, continue with everything that doesn't
depend on it, and ask me directly before doing the action itself. A default on the dashboard is
not approval for them.
