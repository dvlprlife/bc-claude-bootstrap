# {{APP_NAME}} – Release Process (release/X.X.X.X → main)

This is the repeatable runbook for promoting a finished release branch to `main`,
producing the artifact for signing, publishing the GitHub Release, and prepping the
repo for the next cycle.

> **Scope:** AppSource *submission* is **out of scope** — it is performed separately.
> This runbook stops at producing the signed-ready artifact and the GitHub Release.

## Branching model

- Active development happens on the current **`release/X.X.X.X`** branch. Feature/fix
  PRs target it (see `.claude/commands/issue-worker.md`).
- The branch name is the **full app version** that the branch releases
  (`Major.Minor.Build.Revision`): `release/1.2.0.0` releases version `1.2.0.0`. The tag, the
  GitHub Release title, the CHANGELOG heading, and the release-notes header use the same
  four-segment version.
- **`main`** is the release-integration branch (the AppSource line). It receives the
  release branch when a version is ready, and separately receives AL-Go system-file
  updates (the `Update AL-Go System Files` PRs) and AL-Go settings changes.
- Because both sides move, the release→main merge is a normal **bidirectional merge**,
  not a fast-forward. main keeps its newer AL-Go infra; the release branch brings the
  product code.

## Opening a release branch

Cut the next branch from `main` (or from the previous release branch when it has not merged yet):

```bash
git checkout -b release/X.X.X.X origin/main
git push -u origin release/X.X.X.X
```

Copy `.github/workflows/CloseLinkedIssueOnMerge.yaml` onto the new branch if it is not there —
a `pull_request` workflow runs from the PR's **base** branch.

## Pre-flight checks

1. Working tree clean; `git fetch origin`.
2. **Dry-run the merge** (no working-tree changes):
   ```bash
   git merge-tree --write-tree --name-only origin/main origin/release/X.X.X.X
   ```
   Exit 0 = clean. Non-zero prints conflicted paths — resolve before proceeding.
3. **Reconcile versions** — confirm what main will hold after merge:
   ```bash
   for b in origin/main origin/release/X.X.X.X; do
     for p in {{APP_FOLDER}} {{TEST_FOLDER}}; do
       git show "$b:$p/app.json" | grep -o '"version"[^,]*'
     done
   done
   ```
   The release branch's `{{APP_FOLDER}}/app.json` version is the version being released — but read
   `Major.Minor` when you compare branches. The `Build` segment is the **sandbox publish
   counter** (see *Sandbox Publish & Live Testing* in `.github/copilot-instructions.md`), so
   it moves during the cycle for reasons that have nothing to do with the release, and
   `Revision` is unused. `{{TEST_FOLDER}}/app.json` trails at its baseline by design — its
   dependency pin is a minimum, not a lockstep.
   Before the merge, set `{{APP_FOLDER}}/app.json` `version` to the version in the branch name.
   The sandbox counter can leave a higher `Build` there during the cycle.
4. **CI awareness** — check the latest CICD run on the release branch. Note any known red
   test and whether branch protection on `main` blocks the merge.

## Step 1 — Merge release/X.X.X.X → main (PR)

Open a PR and merge with a merge commit (keeps full history and a record):

```bash
gh pr create --base main --head release/X.X.X.X \
  --title "release: merge X.X.X.X into main" --body "<summary>"
gh pr merge <num> --merge
```

After merge, `main` holds the product code at version `X.X.X.X` and its own newer
AL-Go infra. The push to `main` triggers a CICD build automatically.

## Step 2 — Build the artifact for signing

When the CICD run on `main` completes, download the apps artifact from that run:

```bash
gh run list --branch main --workflow CICD.yaml --limit 1
gh run download <run-id> -n "<artifact-name>"
```

This `.app` is the unsigned release artifact. **Hand it to the external signing
step.** AppSourceCop in this build validates against the **previous** published
baseline (`{{APP_FOLDER}}/AppSourceCop.json` `version`) — leave that baseline untouched
until after publish (see Step 5).

> **If CICD can't produce the artifact** (for example, the build aborts compiling the
> **test app** on a symbol-download failure): build the production app out-of-band
> against the correct BC symbols and use that `.app`. The test app is not part of the
> AppSource submission. Resolve the test-app problem separately so the pipeline heals.

## Step 3 — Cut the GitHub Release

Run the **Create release** workflow (`.github/workflows/CreateRelease.yaml`). It
promotes the latest CICD artifact into a GitHub Release and tags it — it does **not**
rebuild:

```bash
gh workflow run CreateRelease.yaml -f appVersion=latest \
  -f name="{{APP_NAME}} X.X.X.X" -f tag="X.X.X.X" -f releaseType=Release \
  -f createReleaseBranch=false -f updateVersionNumber=""
```

It fails with "No artifacts found" if Step 2's build hasn't produced artifacts on
the same branch, so run it after the CICD build completes.

**Manual fallback** (no CICD artifact to promote). Create the release directly and
attach the signed `.app`:

```bash
# GitHub rejects a .app file extension as a release asset — zip it first.
# Create the release WITHOUT the asset first (a failed asset upload rolls the whole
# release back), then upload the zip. Use the FULL commit SHA for --target.
gh release create <tag> --target <full-sha> --title "Version X.X.X.X" --notes-file notes.md
gh release upload <tag> <name>_signed.zip
```

## Step 4 — AppSource submission

Performed outside this runbook.

## Step 5 — Post-release prep (next cycle)

Done as one prep commit/PR on `main` (or the next `release/` branch) **after** the
release is published:

1. **AppSourceCop baseline** — set `{{APP_FOLDER}}/AppSourceCop.json` `version` to the
   just-released version (`X.X.X.X`), so the next cycle's breaking-change checks
   compare against it.
2. **App version** — bump `{{APP_FOLDER}}/app.json` `version` to the next development version, and
   **reset the `Build` segment to `0`** (`1.2.6.0` → `1.3.0.0`): whatever Build reached
   during the cycle is the sandbox publish counter, not a patch number, and the new line
   starts its own count. Set Build deliberately when you *are* shipping a patch — and
   pick a number the counter has not already burned. For a next-major cycle also bump the
   `application` platform target. The test app rides its own version line; BC dependency
   minimums tolerate a higher installed app, so it needs no change unless retargeting.
3. **Sequence the CHANGELOG** (`CHANGELOG.md`, Keep a Changelog format):
   - Rename the `## [Unreleased]` heading to `## [X.X.X.X] - YYYY-MM-DD`.
   - Add a fresh empty `## [Unreleased]` section above it for the next cycle.
4. **Finalize the release-notes table**
   (`supporting-files/release-notes/{{APP_NAME}}-Release-Updates.md`):
   - Fill the `Title / Release date / Build No.` header row for this release
     (date + `X.X.X.X`), and open a fresh empty placeholder section above it.
   - The per-issue rows accumulated during the cycle become this release's entry list.
   - **ID convention:** the issue/PR number goes **only** in the ID column — do not
     repeat it parenthetically in the Title. Any change without its own issue uses the
     release→main PR number as its ID.

## Notes & gotchas

- `al compile` can rewrite `.docx` report layouts as a side effect —
  `git checkout --` them before committing.
- Keep `supporting-files/DEPRECATIONS.md` honest at release time — items past their
  removal cohort become eligible to hard-remove in the next major.
