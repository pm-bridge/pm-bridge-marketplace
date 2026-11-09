# quick-tools plugin

A PM Bridge plugin with **skills only** - no MCP server, no process, no configuration. It is a
standard Claude Code plugin with no `.mcp.json`:

```
.claude-plugin/plugin.json          name, version, description, author
skills/one-off-task/SKILL.md        quick-tools:one-off-task
test/skills.test.mjs                the drift test
```

A one-off job that no tool covers does not need a plugin of its own: a persona with the Workspace
permission level (or higher) writes a small script, runs it once in its command sandbox and sends
the result back. This skill tells it how to do that well.

## Skills

| skill | what it is for |
|---|---|
| `quick-tools:one-off-task` | A job the owner needs now: sign or edit a PDF, convert documents and images, pull a table out of a file. Where the files go (`.pmb-quick/<job>/` in the workspace) and the throwaway Python environment (`$TMPDIR/pmb-quick/<job>/`, deleted afterwards), the tools macOS already has (`textutil`, `sips`), libraries that suit common jobs (PyMuPDF, openpyxl, python-docx, pdfplumber), a signature example, how to hand the result back and clean up, and when to offer keeping a script. |

## What it needs

- The persona's permission level is **Workspace** or **Full access** (read only cannot write files).
- `python3` for the jobs that need a library (macOS ships it with the command line developer
  tools: `xcode-select --install`). PyPI is on the usual developer sites, so installs need no
  approval.

Nothing is uploaded anywhere: every job runs on this Mac.
