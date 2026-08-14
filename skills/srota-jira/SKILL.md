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

## 3. Pin down repo, branch, and affected service — ask first, explore only what's left

If the ticket doesn't already name which service/API/script this is (check the description, comments, and custom fields first), ask before doing any codebase searching — don't go hunting through folders trying to guess it, that's exactly the wasted work asking up front avoids:

> "Which repo(s) and branch is this on, and which service/API/script does it touch?"

The moment you have that answer and it wasn't already on the ticket, post it back to Jira immediately (a short comment or field update) — don't wait for step 5. That way the mapping is captured even if this session ends before the interview finishes, and next time anyone (human or agent) opens this ticket, that search is already done.

**Ask before you explore, for anything the user can plausibly answer from memory.** The user is often the assignee or the person who wrote the relevant code — a one-line answer from them is faster and more authoritative than reconstructing the same fact through several `search_graph`/`trace_path` round-trips, and unlike a guessed route prefix or file path, it can't be subtly wrong. This applies not just to "which service" but to any concrete specific the interview needs: the exact endpoint(s)/route(s) involved, which of several similar/duplicate implementations is the one in question, test data (symbol/ID/account to repro with), env values. Ask for these directly, one at a time, before reaching for graph tools.

Reach for codebase exploration only when:

- The user doesn't know or isn't sure (explicitly says so, or their answer is still vague after one follow-up).
- The ticket or user's answer leaves a genuine fork unresolved (e.g. two parallel implementations, old vs. new page) — explore enough to name the fork precisely, then ask the user which side it's on rather than guessing.
- You need to verify a fact before writing it into the ticket — never assert a file path, route, or line number in the step-5 write-back that you haven't actually seen in the code.

When you do explore, per the repo-wide codebase-memory-mcp instructions: if the current directory contains multiple service folders (monorepo-style), explore the relevant one(s) with `search_graph` / `trace_path` / `get_code_snippet` / `get_architecture` — don't fall back to grep unless the graph tools come up short. Use what you learn to ask sharper questions (name the actual endpoint/file instead of asking "which endpoint") — but treat that exploration as the exception for gaps the user couldn't fill, not the default first move.

Net effect: asking becomes the default for anything the user (often the assignee/author) could just tell you — exact endpoints, which of several duplicate implementations applies, test data, env values. Exploration is demoted to three specific fallback cases: user doesn't know, a genuine fork needs naming before you can ask a sharp question, or verifying a fact before it's written into the ticket.

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
