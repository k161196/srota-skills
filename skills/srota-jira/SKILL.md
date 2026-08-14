---
name: srota-jira
description: Grill the user until a Jira ticket has everything another agent needs to implement it — issue identity, affected repo(s)/branch, concrete repro steps, a test plan, run/env instructions, and (for new features) a PRD/end-goal — then write the compiled findings back to the ticket. Makes no code changes. Use when the user wants to pick up a Jira issue, says things like "let's start on PROJ-123", "prep this ticket", "get this issue ready to hand off", or names/pastes a Jira ticket before any implementation work begins.
---

# srota-jira — fully spec a ticket before anyone touches code

Your job here is pure information-gathering. You never write, edit, or plan code changes in this skill — you make sure the Jira ticket itself contains everything a different agent (or the user, later) would need to implement the fix/feature without asking a single follow-up question. When the ticket is complete, you're done.

## 1. Identify the issue

Look for an issue key (e.g. `PROJ-123`) in: the user's message, the current branch name, recent commit messages, open PR title/description. If you can't confidently identify exactly one, stop and ask the user directly — this is the one question you ask before anything else, not part of the later interview:

> "Which Jira issue are we working on? (key or link)"

If two sources disagree (e.g. the branch name points to a different ticket than the user's message), don't guess — surface the conflict and ask which one is right.

## 2. Fetch the issue

Look for Jira/Atlassian tools first (e.g. `mcp__claude_ai_Atlassian__*`, or similarly named once connected — use `ToolSearch` with `"jira"` / `"atlassian"` to find them); if the connector isn't authenticated, use the `authenticate` tool before continuing. Never guess ticket contents from memory.

Pull the full issue: title, description, type (Bug/Task/Story/etc.), status, existing comments, and any custom fields that might already carry repo/branch info. Read what's already there — never ask the user for something the ticket already answers.

## 3. Pin down repo, branch, and affected service, then explore

Ask this before anything else in the interview, since exploring code depends on the answer. Skip only what the ticket already names:

> "Which repo(s) and branch is this on, and which service/API/script does it touch?"

Then, per the repo-wide codebase-memory-mcp instructions, explore that area of the code:

- If the current directory contains multiple service folders (monorepo-style), explore the relevant one(s) with `search_graph` / `trace_path` / `get_code_snippet` / `get_architecture` — don't fall back to grep unless the graph tools come up short.
- Use what you learn to ask sharper questions next (name the actual endpoint/file instead of asking "which endpoint"), and to avoid asking about anything you can just look up yourself.

## 4. Grill the user on everything else

Use the [[grilling]] skill's method: one question at a time, wait for the answer before asking the next, offer your best-guess/recommended answer when you have one (e.g. from step 3's exploration), and don't stop until every item below is concrete — not vague. "It's the checkout flow" is not a repro step; "call `POST /api/checkout` with an empty `cartId`, look at the `total` field in the response" is.

Cover every item below that the ticket doesn't already answer, skipping only what's already fully specified:

- **Reproduction steps (bug/task) — force specifics.** The exact API(s) to call (method, endpoint, payload/params), or the exact script/command to run, and which variables, fields, or log lines to look at. Keep pushing past "it just fails" until you have something another engineer could follow with zero prior context.
- **Test plan — force specifics.** How will we know this is actually fixed/built? Which test to run, which manual steps to follow, what the pass/fail criteria is. Don't accept "test it works."
- **How to run the service.** Ask how to start/run the affected service(s) locally. If required env vars or files aren't already in the repo (missing from `.env.example`, secrets not present), ask the user to provide an env file or the specific values — never guess or fabricate secrets, and never proceed on an assumed config.
- **New feature/story only:** ask for the PRD (link or pasted content) and the end goal / success criteria, in addition to everything above.

Keep grilling on any item that's still fuzzy — one unresolved fuzzy answer is enough reason to keep going, don't move to step 5 early.

## 5. Confirm, then write it back to the ticket — and only the ticket

Compile everything into a short structured summary (repo/branch, repro steps, test plan, run/env instructions, PRD link if applicable) and show it to the user for a quick confirm before posting — this lands on a ticket the whole team sees, so a final "does this look right?" beats a silent post. Once confirmed, add it to the Jira issue (comment or description update — whichever fits the existing ticket's conventions). Do not:

- touch any code, branch, or file in the repo
- update any other system (Slack, Linear, docs, etc.)
- start planning or drafting an implementation

The ticket being fully specified is the finish line. A different agent picks it up from there.
