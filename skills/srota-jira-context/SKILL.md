---
name: srota-jira-context
description: Pull a Jira issue's full context onto disk — description, comments, every attachment (downloaded and auto-extracted if it's an archive), and any linked Figma/design URL — then write a context.md summary. Makes no code changes. Use when the user wants to pull down a Jira ticket's attachments, says "grab the attachments for PROJ-123", "pull context for this ticket", "get everything from this Jira issue locally", or before design/spec work that depends on files attached to a Jira issue.
---

# srota-jira-context — pull a Jira issue's attachments and links to disk

Your job is to materialize everything attached to a Jira issue into a local folder so it can actually be read/opened, plus a `context.md` that summarizes what's there. You make no code changes and no ticket updates — this is a one-way pull from Jira to disk.

Safe to re-run: if `<ISSUE-KEY>/` already exists, sync it — download only what's new or changed, don't re-fetch everything from scratch, and never silently delete local files.

## 0. Inputs

You need an issue key (e.g. `PROJ-123`). Look for it in the user's message, current branch name, or recent commits before asking. You may also need a `cloudId`/site if the Atlassian connector spans multiple sites — the MCP tools below will tell you if the call is ambiguous; only ask for it then, don't ask upfront.

## 1. Fetch the issue

Find the Atlassian MCP tools (`ToolSearch` with `"jira"` / `"atlassian"` if not already loaded — e.g. `mcp__claude_ai_Atlassian__getJiraIssue`). Authenticate first if the connector isn't already connected.

Fetch the issue with all fields (attachments, comments, description, custom fields included — don't ask for a narrow field list). Pull out:
- Description and comments — this is the "Notes" material for step 5.
- The attachments list: filename, id, and its `content` URL (Jira returns a full download URL per attachment — you don't need to reconstruct it from a site/cloudId).

## 2. Fetch remote issue links

Call `mcp__claude_ai_Atlassian__getJiraIssueRemoteIssueLinks` (or equivalent) for the same issue. Scan the results for any Figma or other design-tool URL (`figma.com`, or similarly obvious design links). Keep every one you find — an issue can have more than one.

## 3. Download attachments — new or changed only

Create `<ISSUE-KEY>/attachments/` if it doesn't exist yet (e.g. `PROJ-123/attachments/`) — namespacing by issue key avoids collisions if this runs for multiple tickets in the same repo checkout.

If `attachments/` already has files in it (re-run), diff Jira's current attachment list against what's on disk by filename before downloading anything:
- **Filename not on disk** → new attachment, download it.
- **Filename on disk, same size as Jira reports** → already have it, skip.
- **Filename on disk, different size** → it's been replaced in Jira since the last pull — re-download and overwrite, and note the update in context.md's Notes.
- **File on disk whose filename is no longer in Jira's attachment list** → never delete automatically. If the stale file is an archive with an extracted subfolder (step 4), treat that subfolder as stale too — always check extracted archive contents, not just the top-level files. Once the whole diff is done, list every stale file/subfolder and ask the user a single yes/no question: "These local files are no longer attached to the ticket: `<list>`. Delete them? (yes/no)". Delete only what they confirm; whatever they decline (or if they say no to all) stays, flagged in context.md's Notes as removed from Jira.

Attachment downloads need Basic Auth (email + API token), which the MCP fetch tools don't expose — use `curl` directly against each attachment's `content` URL:

```bash
curl -L -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -o "attachments/<filename>" "<content-url>"
```

- **Email:** call `mcp__claude_ai_Atlassian__atlassianUserInfo` to get the authenticated user's email rather than asking — it's already known to the connector.
- **API token:** check `$JIRA_API_TOKEN` in the environment first. If it's not set, ask the user which env var holds it (don't guess or fabricate a value, and don't ask them to paste the token itself into chat).
- Use `-L` (follow redirects) — Jira's attachment endpoint typically redirects to a signed media URL.

## 4. Auto-extract archives

For every attachment ending in `.zip`, `.tar`, `.tar.gz`, or `.tgz` that's new or was just re-downloaded: extract it into a subfolder named after the archive (minus extension) under `attachments/`, overwriting that subfolder if it already exists, then strip any `__MACOSX` directory the extraction produced. Leave already-extracted, unchanged archives alone — don't redo work. Leave non-archive attachments as-is.

## 5. Write context.md

Regenerate `<ISSUE-KEY>/context.md` from scratch each run (it reflects current state, not history) with:

- **Jira link** — the issue's browse URL (`<site>/browse/<ISSUE-KEY>`).
- **Figma / design links** — every URL found in step 2, or "none found" if there were none.
- **Attachment inventory** — a list of what's in `attachments/`: filename, and for extracted archives, a one-line summary of what's inside (don't just say "extracted" — glance at the contents and note what kind of thing it is, e.g. "design export, 12 PNGs + 1 Sketch file").
- **Notes** — anything non-obvious a future reader would otherwise have to rediscover: description is empty and the real spec lives in an attachment, comments contradict the description, an attachment was updated or removed since the last pull (per step 3), a Figma link 404s or looks private, etc. Skip this section only if there's genuinely nothing non-obvious.

Report the folder path to the user when done, and call out anything that changed since the last pull (new/updated/removed attachments). Don't summarize the ticket further in chat beyond that — the point of `context.md` is that it's the summary.
