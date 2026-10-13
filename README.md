# bc-claude-bootstrap

A kit that installs the Claude Code issue workflow in a Business Central (AL-Go) repository:
issue planner, issue worker, PR reviewer, the `bc-review` compile + BCQuality gate, the AL MCP
server, and sandbox publish and test.

## Prerequisites

- **Windows**, with PowerShell 7 and Git for Windows (Git Bash). The build command calls `alc.exe`
  and uses Windows paths. The kit is not tested on macOS or Linux.
- **Claude Code**.
- **Git and the GitHub CLI** (`gh`), signed in to an account that can push to the target repo.
- **The AL command-line tools**: the .NET global tool
  `Microsoft.Dynamics.BusinessCentral.Development.Tools`.
- **A target repository built with AL-Go for GitHub**, with an app and a test app.
- **A Business Central sandbox** and an account with publish rights. For test runs, the sandbox
  needs the Microsoft test toolkit.
- **A clone of BCQuality** next to the target repo. Upstream `microsoft/BCQuality` is the default.
  A fork with your own rules is optional.

Optional:

- **The Admin role on the target repo**, to apply the branch protection ruleset.
- **claude.ai Artifacts in Claude Code**, for the progress dashboard.

## Use

1. Clone this repo to the machine:
   `git clone https://github.com/dvlprlife/bc-claude-bootstrap`
2. Open Claude Code in the target repository.
3. Tell Claude: `Read <path>\bc-claude-bootstrap\bootstrap.md and set up this repo.`

Claude collects the values (app name, prefix, ID ranges, sandbox), copies the templates, fills in
the placeholders, and checks the setup. It asks before it pushes or installs anything.

## What the kit changes

- **In the target repo:** it adds `CLAUDE.md`, `.github/copilot-instructions.md`, the files under
  `.claude/`, `.mcp.json`, one GitHub workflow, and documentation templates. It edits `app.json`,
  `AppSourceCop.json`, and `.gitignore`. The sandbox name and the tenant ID go into tracked files.
- **On GitHub:** it creates 8 workflow labels. If you choose it, it applies a branch protection
  ruleset to `main` and `release/*`.
- **On your machine, only if you choose it:** it adds a writing standard and a progress dashboard
  rule to `~/.claude/CLAUDE.md`. These rules apply to each project on the machine. The dashboard
  page loads its fonts from Google Fonts.

`repo/.claude/settings.json` pre-approves the read, label, comment, and PR-create commands that
the agents use. It does not pre-approve a merge, a delete, or a push.

## License

MIT. See [LICENSE](LICENSE).
