# bc-claude-bootstrap

A kit that installs the Claude Code issue workflow in a Business Central (AL-Go) repository:
issue planner, issue worker, PR reviewer, the `bc-review` compile + BCQuality gate, the AL MCP
server, and sandbox publish and test.

## Use

1. Copy this folder to the new machine.
2. Open Claude Code in the target repository.
3. Tell Claude: `Read <path>\bc-claude-bootstrap\bootstrap.md and set up this repo.`

Claude collects the values (app name, prefix, ID ranges, sandbox), copies the templates, fills in
the placeholders, and checks the setup. It asks before it pushes or installs anything.
