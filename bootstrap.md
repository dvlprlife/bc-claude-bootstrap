# BC Claude Code workflow — bootstrap

**Audience: Claude Code, on a new machine, in a new Business Central (AL) repository.**
This kit installs the issue-to-merge workflow: repo check, issue planner, issue worker, PR reviewer,
the `bc-review` skill (compile gate + BCQuality review), the AL MCP server, and sandbox publish and
test. The kit is self-contained. You do not need access to any other repository to use it.

Read this file from start to end before you change anything. Then do the steps in order.

## What the kit contains

| Path in this kit | Installs to | Purpose |
|---|---|---|
| `repo/CLAUDE.md` | `<repo>/CLAUDE.md` | Claude-specific wiring: build command, branching, CI, issue loop |
| `repo/.github/copilot-instructions.md` | `<repo>/.github/` | The full convention set. `CLAUDE.md` imports it |
| `repo/.claude/agents/README.md` | `<repo>/.claude/agents/` | The loop and the label lifecycle |
| `repo/.claude/agents/repo-check.md` | `<repo>/.claude/agents/` | Subagent: `gh` auth, the 8 labels, the base branch |
| `repo/.claude/agents/issue-planner.md` | `<repo>/.claude/agents/` | Subagent: posts an implementation plan on an issue |
| `repo/.claude/agents/pull-request-reviewer.md` | `<repo>/.claude/agents/` | Subagent: reviews the PR, moves the labels |
| `repo/.claude/commands/issue-worker.md` | `<repo>/.claude/commands/` | Slash command `/issue-worker`: implements an issue |
| `repo/.claude/skills/bc-review/SKILL.md` | `<repo>/.claude/skills/bc-review/` | Skill: compile gate + BCQuality review |
| `repo/.claude/settings.json` | `<repo>/.claude/` | Shared permissions; turns on the `al` MCP server |
| `repo/.mcp.json` | `<repo>/` | The AL MCP server (`al launchmcpserver`) |
| `repo/.github/workflows/CloseLinkedIssueOnMerge.yaml` | `<repo>/.github/workflows/` | Closes the issue and clears its labels when the PR merges |
| `repo/.gitignore` | `<repo>/` | Git ignore list for an AL-Go repo. Optional: the user chooses it in Step 2 |
| `repo/{{APP_NAME}}.code-workspace` | `<repo>/` | VS Code workspace: shared symbol cache, ruleset, analyzers |
| `repo/CHANGELOG.md` | `<repo>/` | Keep a Changelog, with `## [Unreleased]` |
| `repo/supporting-files/{{PREFIX}}.ruleset.json` | `<repo>/supporting-files/` | Analyzer ruleset |
| `repo/supporting-files/DEPRECATIONS.md` | `<repo>/supporting-files/` | Obsoletion tracker |
| `repo/supporting-files/release-notes/{{APP_NAME}}-Release-Updates.md` | same path | Customer release notes |
| `repo/supporting-files/documentation/documentation-style-ste100.md` | same path | STE writing standard for `docs/**` |
| `repo/supporting-files/documentation/release-process.md` | same path | Release runbook (release branch → main) |
| `repo/docs/README.md` | `<repo>/docs/` | End-user guide root |
| `repo/docs/appsource-listing.md` | `<repo>/docs/` | AppSource listing copy |
| `repo/docs/img/MANIFEST.md` | `<repo>/docs/img/` | Screenshot capture list |
| `user/CLAUDE.md` | `~/.claude/CLAUDE.md` | Global rules: writing standard, progress dashboard |
| `user/agents/dashboard-builder.md` | `~/.claude/agents/` | Subagent: live progress dashboard on claude.ai |
| `user/dashboard-builder/template.html` | `~/.claude/dashboard-builder/` | The dashboard page template |

## Step 1 — Check the prerequisites

Run each check. Stop and tell the user about each one that fails. Do not install a tool without
the user's approval.

1. **Git and the GitHub CLI.** Run `git --version` and `gh auth status`. If `gh` is not signed in,
   tell the user to type `! gh auth login`. The sign-in is interactive.
2. **PowerShell 7.** Run `pwsh --version`. The compile command and the BCQuality index build use it.
3. **The AL CLI.** Run `al --version`. If it is missing, the user installs the .NET global tool:
   `dotnet tool install --global Microsoft.Dynamics.BusinessCentral.Development.Tools`. The tool
   gives the `alc.exe` compiler, the four code analyzers, and the MCP server
   (`al launchmcpserver`).
4. **The repository.** The repo must be an AL-Go for GitHub repository. If it is new, tell the user
   to create it from the AL-Go AppSource template (`microsoft/AL-Go-AppSource`) and to add the app
   and the test app with the AL-Go **Create a new app** and **Create a new test app** workflows.
   This kit does not replace the AL-Go workflows. It adds to them.
5. **BCQuality.** Clone the BCQuality fork as a sibling of the repo, in the same parent folder:
   `git clone https://github.com/{{BCQUALITY_REPO}} ../BCQuality`. Ask the user for the fork when
   it is not known. A clone of upstream `microsoft/BCQuality` works, but it has no `custom/` rules.
6. **The sandbox.** The user needs a BC sandbox and an account with publish rights. For test runs,
   the Microsoft test toolkit must be on the sandbox: Test Runner, Library Assert, Application
   Test Library, and Any.

## Step 2 — Collect the values

Get each value from the repo first. Ask the user only for the values you cannot find.

| Placeholder | Meaning | Where to find it |
|---|---|---|
| `{{APP_NAME}}` | App name, no spaces (for example `ContosoApp`) | `name` in the app's `app.json` |
| `{{APP_DESCRIPTION}}` | One or two sentences about what the app does | `brief` in `app.json`, or ask |
| `{{PUBLISHER}}` | Publisher display name (for example `Contoso, Ltd.`) | Ask; `publisher` in `app.json` is the short form |
| `{{PUBLISHER_FILE}}` | Publisher as it shows in a package filename (for example `Contoso`) | `publisher` in `app.json` |
| `{{PREFIX}}` | Mandatory affix (for example `CTS`) | `mandatoryAffixes` in `AppSourceCop.json`, or ask |
| `{{ID_FROM}}` / `{{ID_TO}}` | Production object range | `idRanges` in the app's `app.json` |
| `{{TEST_ID_FROM}}` / `{{TEST_ID_TO}}` | Test object range | `idRanges` in the test app's `app.json` |
| `{{APP_FOLDER}}` | Folder of the main app (for example `app`) | Folder that holds the main `app.json` |
| `{{TEST_FOLDER}}` | Folder of the test app (for example `test`) | Folder that holds the test `app.json` |
| `{{DEPENDENT_FOLDERS}}` | Projects that depend on the app, as text (for example `` `test/` ``) | `dependencies` in each other `app.json` |
| `{{BC_APPLICATION}}` | Minimum Business Central application version (for example `28.0.0.0`) | Ask. Do not trust the value that AL-Go generated |
| `{{BC_PLATFORM}}` | Minimum platform version (for example `1.0.0.0`) | Ask. Do not trust the value that AL-Go generated |
| `{{BC_RUNTIME}}` | AL runtime version that matches the application version (for example `17.0`) | Ask, or get it from the application version |
| `{{SANDBOX_ENV}}` | Sandbox environment name | Ask |
| `{{TENANT_ID}}` | Entra tenant GUID of the sandbox | Ask |
| `{{BCQUALITY_REPO}}` | BCQuality fork as `owner/repo` (for example `contoso/BCQuality`) | Ask |

Ask three more questions in this step. They have no placeholder, but Step 4 and Step 8 use the answers.

| Question | Choices | Used in |
|---|---|---|
| **Release branch strategy** — which branch does development target? | `main`, or a release branch `release/X.X.X.X` (ask for the version) | Step 4.6, Step 4.8, Step 8.4 |
| **CI gate** — is a green GitHub CI run necessary before the sandbox publish? | `No` (the local compile and the sandbox are the gates), or `Yes` (a green `CICD.yaml` run comes first) | Step 4.11 |
| **.gitignore** — does the repo use the kit's `.gitignore` or its own? | `Kit` (the kit's file replaces the repo's file), or `Own` (the repo keeps its file, and the kit adds the lines it needs) | Step 4.2 |

Write the values down and show them to the user in one table before Step 3.

## Step 3 — Copy and fill in the templates

1. Copy each file under `repo/` to the same relative path in the target repo. Rename the three
   files that have a placeholder in the filename. Do not copy `.gitignore` in this step. Step 4.2
   handles it.
2. Do not overwrite a file that already exists in the target repo. Merge into it instead:
   - `CHANGELOG.md` — keep the existing entries. Add `## [Unreleased]` if it is missing.
   - `.github/copilot-instructions.md` — keep project-specific content. Add the kit's sections.
   - A `.code-workspace` file — add the `al.*` settings from the kit's workspace file.
   - `.claude/settings.json` — merge the `permissions.allow` lists and `enabledMcpjsonServers`.
3. Replace each placeholder with its value in each copied file.
4. In `.mcp.json`, list each AL project folder after `launchmcpserver`, one for each argument.
5. Search the target repo for `{{`. The result must be empty, except in code that uses `{{`
   on purpose.

## Step 4 — Wire the repo

1. **AppSourceCop.json.** Make sure `{{APP_FOLDER}}/AppSourceCop.json` and
   `{{TEST_FOLDER}}/AppSourceCop.json` have `"mandatoryAffixes": ["{{PREFIX}}"]`. Create a file
   if it is missing. Without the file, AppSourceCop stops the compile of that project with
   `AS0054`. The AL-Go sample objects (`HelloWorld.al`, `HelloWorld.Test.al`) have no affix.
   Add `{{PREFIX}}` to their object names, or the compile fails with `AS0011`.
2. **.gitignore.** Use the answer to the `.gitignore` question from Step 2.
   - **Kit:** copy the kit's `repo/.gitignore` to the root of the target repo. If the repo
     already has a `.gitignore`, first show the user the lines that are only in the repo's file.
     Ask which of those lines to keep, and add them to the end of the new file.
   - **Own:** keep the repo's `.gitignore`. Create the file if it is missing.

   For each choice, make sure `.gitignore` has these lines: `.alpackages/`, `*.app`, `.vscode/`,
   and `.claude/settings.local.json`. The symbol cache, the build output, `launch.json`, and
   personal Claude settings stay out of git.
3. **Symbol cache.** Create the folder `supporting-files/.alpackages`. It is git-ignored, so each
   clone starts empty.
4. **Ruleset.** Read the kit's `{{PREFIX}}.ruleset.json` with the user. The kit's file is for
   an AppSource app with its own ID range. Get the app type from `type` in
   `.github/AL-Go-Settings.json` (`PTE` or `AppSource App`).
   - **AppSource app:** keep the file as it is.
   - **Per-tenant extension:** remove the `PTE0001` and `PTE0002` suppressions, so that
     PerTenantExtensionCop checks the `50000..99999` range. Keep `AS0084` hidden, and change
     its justification: AppSourceCop rejects the per-tenant range by design. Also hide
     `AS0015`, `AS0051`, and `AS0052`. These are AppSource submission rules (translation
     file, marketplace manifest properties, manifest URL), and each one is an error on a
     per-tenant extension. Write a per-tenant justification for each rule.
5. **launch.json.** Add a sandbox configuration to `{{APP_FOLDER}}/.vscode/launch.json` (local
   only): `"environmentType": "Sandbox"`, `"environmentName": "{{SANDBOX_ENV}}"`,
   `"tenant": "{{TENANT_ID}}"`, `"schemaUpdateMode": "ForceSync"`.
6. **CloseLinkedIssueOnMerge.yaml.** A `pull_request` workflow runs from the PR's **base** branch.
   Put the file on `main` and on each active `release/X.X.X.X` branch.
7. **CI triggers.** Read `.github/workflows/CICD.yaml` and `PullRequestHandler.yaml`. Compare the
   `on:` blocks with the **Branching and CI** section of `CLAUDE.md`. Correct `CLAUDE.md` where
   this repo's AL-Go version is different.
8. **Release branch.** Use the release branch strategy from Step 2.
   - **Release branch:** the workflow targets the highest release branch. If the branch from
     Step 2 does not exist, cut it (`git checkout -b release/X.X.X.X origin/main`). Pushing it
     needs the user's approval.
   - **`main`:** do not cut a branch. In `CLAUDE.md` and `.claude/agents/README.md`, change the
     text that says development targets a release branch, so that it names `main`.
9. **Application and platform version.** In the `app.json` of each AL project, set
   `"application": "{{BC_APPLICATION}}"`, `"platform": "{{BC_PLATFORM}}"`, and
   `"runtime": "{{BC_RUNTIME}}"`. The AL-Go **Create a new app** workflow writes an old application
   version and no runtime. Set the version of each Microsoft dependency in the test app
   (Library Assert, Any, and the others) to `{{BC_APPLICATION}}` also.
10. **Test app dependency.** Make sure `{{TEST_FOLDER}}/app.json` has the main app in
    `dependencies`: the `id`, `name`, `publisher`, and `version` from `{{APP_FOLDER}}/app.json`.
    The AL-Go **Create a new test app** workflow does not add this dependency. Without it, the
    test app cannot use the app's objects.
11. **CI gate.** Use the CI gate answer from Step 2. The kit's files are written for the answer
    `No`: GitHub CI is not a gate, and Claude publishes to the sandbox when both `bc-review` gates
    pass.
    - **No:** keep the files as they are.
    - **Yes:** change the installed files so that a green CI run comes before the sandbox publish.
      A PR into a release branch gets no automatic build, so the run starts with
      `gh workflow run CICD.yaml --ref <branch>`. Change these passages:
      - `CLAUDE.md`, **Branching and CI**: replace the two bullets "GitHub CI is not a gate in
        this project" and "The gates are the local compile and the sandbox". The new bullet says
        that the sandbox publish is allowed only after CI is green, or on the user's request.
      - `.github/copilot-instructions.md`, **Sandbox Publish & Live Testing**: the gate is "CI is
        green for that build". The pre-merge publish follows a green PR-branch run. The
        post-merge publish follows a green run on the base branch. Remove the two sentences
        that say CI is not a gate.
      - `.claude/agents/README.md`, **The gates around the merge**: replace the "No GitHub CI
        gate" and "Compile and publish" bullets with a "CI" bullet that gives the
        `gh workflow run` command. Change loop step 8 so that CI comes before the publish.
      - `.claude/commands/issue-worker.md`: remove Step 8 (publish). In the report step, tell
        the user to start the CI run, and say that the publish follows a green run.
    The cleanup rule (delete the merged branch after the merge) applies to both answers.

## Step 5 — Install the user-level files

Do this once on each machine. Compare each file with the copy that is already there first.

1. If `~/.claude/CLAUDE.md` exists, merge the kit's `user/CLAUDE.md` into it. Do not replace it.
2. Copy `user/agents/dashboard-builder.md` to `~/.claude/agents/`.
3. Copy `user/dashboard-builder/template.html` to `~/.claude/dashboard-builder/`.

The dashboard needs claude.ai Artifacts in the Claude Code session. When Artifacts are not
available, skip the dashboard. The issue workflow does not use it.

## Step 6 — Get the symbols and do the first compile

1. Restart Claude Code in the repo, so it loads `.mcp.json`, the agents, the command, and the skill.
   Approve the `al` MCP server when Claude Code asks.
2. Download the Microsoft symbols for each project with the AL MCP server:
   `al_downloadsymbols { "projectPath": "<repo>\\{{APP_FOLDER}}", "globalSourcesOnly": true }`.
   Then move or copy the downloaded `.app` files from `<project>/.alpackages` into
   `supporting-files/.alpackages`.

   Make sure the cache has a package for each Microsoft dependency of the test app (Library
   Assert, Any, and the others). With AL tools 18.0.43, the download for the test project
   can return only the packages of the first project. For each missing package, get the
   symbols from the Microsoft public feed
   (`https://dynamicssmb2.pkgs.visualstudio.com/DynamicsBCPublicFeeds/_packaging/MSSymbols/nuget/v3/index.json`).
   The package ID is `Microsoft.<NameWithoutSpaces>.symbols.<app id>`, for example
   `Microsoft.LibraryAssert.symbols.dd0be2ea-f733-4d65-bb34-a28f4624fb14`. Use the highest
   version with the same major and minor as the downloaded Base Application. The `.nupkg` is
   a zip file; copy the `.app` in it to `supporting-files/.alpackages`.
3. Compile the app with the PowerShell command in `CLAUDE.md`. The command calls `alc.exe`
   directly. If the output has an `AL1003` warning, the analyzers did not load and the result
   is not a pass. Set `/out:` to
   `supporting-files\.alpackages\{{PUBLISHER_FILE}}_{{APP_NAME}}_<version>.app`. The dependent
   projects resolve the app's objects from this package.
4. Compile each dependent project with the same command and the same cache.
5. Zero errors is the gate. Report the warning count for each project to the user.
6. Run `git status`. Revert any `.docx` layout that the compile changed.

## Step 7 — Do a dry run of the loop

1. Run the `repo-check` subagent. It creates the 8 labels. Do not create them by hand.
2. Run `/bc-review` on one `.al` file. Confirm that Gate 1 (compile) and Gate 2 (BCQuality) both run,
   and that the BCQuality `outcome` is `completed`.
3. Optional: ask the user for a small test issue. Label it `Agent` + `Status:Need Plan`, then run
   `issue-planner` on it.

## Step 8 — Commit

1. Show the user the list of changed files.
2. With the user's approval, create a branch with no number prefix (for example
   `chore-claude-workflow`). A number prefix makes `CloseLinkedIssueOnMerge.yaml` close that issue.
3. Commit as `chore: add Claude Code issue workflow`.
4. Push and open a PR only when the user approves. The PR goes to the active release branch, or to
   `main` when there is no release branch.

## Verification checklist

- [ ] `CLAUDE.md`, `.github/copilot-instructions.md`, `.mcp.json`, and `.claude/**` are in the repo.
- [ ] A search for `{{` in the installed files gives no result.
- [ ] `.gitignore` is the file the user chose, and it has the four lines from Step 4.2.
- [ ] Each `app.json` has the application, platform, and runtime versions that the user gave.
- [ ] The test app's `app.json` has a dependency on the main app.
- [ ] `al --version`, `gh auth status`, and `pwsh --version` succeed.
- [ ] `../BCQuality` exists, and its `origin` is the fork.
- [ ] `supporting-files/.alpackages` has the Microsoft symbols and one `{{APP_NAME}}` package.
- [ ] Each AL project compiles with zero errors and all four analyzers.
- [ ] `repo-check` reports 8 labels and an active base branch.
- [ ] `/bc-review` reports Gate 2 outcome `completed`.
- [ ] The `al` MCP server shows as connected in `/mcp`.

## The workflow, in short

1. `repo-check` — verify the prerequisites.
2. The user opens an issue with the labels `Agent` + `Status:Need Plan`.
3. `issue-planner` — posts the plan. Labels move to `Status:Ready` + `Review:Plan`.
4. The user reviews the plan and removes `Review:Plan`.
5. `/issue-worker` — branch, code, `bc-review` gates, commits, PR. Label moves to `Status:In Review`.
6. `pull-request-reviewer` — approves (`Review:Merge`) or requests changes (`Status:Ready`).
7. Only when CI is a gate (Step 2): `gh workflow run CICD.yaml --ref <branch>`, then wait for a
   green run.
8. Bump the build, publish to the sandbox, and the user tests the change live.
9. The user merges. `CloseLinkedIssueOnMerge.yaml` closes the issue.
10. Publish the merged build to the sandbox again. When CI is a gate, wait for a green run on the
    base branch first.
11. Delete the merged branch locally and on `origin`.

To run these steps on an interval without a prompt for each stage, use the `/loop` command in the
**Run the loop automatically** section of `.claude/agents/README.md`. Tell the user about it at
the end of the bootstrap.

The full rules are in `.claude/agents/README.md`, `CLAUDE.md`, and `.github/copilot-instructions.md`.
