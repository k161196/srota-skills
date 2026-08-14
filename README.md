# srota-skills

Agent Skills for [Srota](https://github.com/k161196/srota), installable into any coding agent via the [Skills CLI](https://github.com/vercel-labs/skills).

## Install

```bash
npx skills add k161196/srota-skills
```

Install a specific skill directly:

```bash
npx skills add k161196/srota-skills@srota-mcp-orchestration
```

## Skills

| Skill | What it teaches |
| --- | --- |
| [`srota-mcp-orchestration`](skills/srota-mcp-orchestration/SKILL.md) | Using Srota's MCP server to spawn, watch, and coordinate sibling agent panes from inside a Srota pane. |
| [`srota-bro`](skills/srota-bro/SKILL.md) | `/srota-bro` — re-explain the previous assistant message in plain language. |
| [`srota-jira`](skills/srota-jira/SKILL.md) | Grill the user until a Jira ticket has full repro steps, test plan, repo/branch, and run instructions before anyone writes code. |
| [`srota-jira-context`](skills/srota-jira-context/SKILL.md) | Pull a Jira issue's description, comments, attachments (downloaded + auto-extracted), and linked Figma URLs onto disk as `context.md`. |
| [`srota-prd-audit`](skills/srota-prd-audit/SKILL.md) | Audit a PRD for ambiguous requirements, missing coverage, internal contradictions, and unaddressed edge cases; write an HTML report. |
| [`srota-prd-design-audit`](skills/srota-prd-design-audit/SKILL.md) | Cross-check a PRD against its Figma/design screens (both directions) plus an edge-case checklist, and write an HTML audit report. |

## Migration notes

- **`bro` → `srota-bro`** (2026-08-05): all skills in this repo are prefixed `srota-`. If you installed `bro` before this change, it's now orphaned under the old name and won't receive updates. Clean it up and reinstall under the new name:

  ```bash
  npx skills remove bro -g -y
  npx skills add k161196/srota-skills --skill srota-bro -g
  ```
