---
name: srota-prd-audit
description: Audits a PRD (and any linked docs/attachments) on its own terms — finds requirements that are ambiguous or underspecified, things a spec at this scope should cover but doesn't mention, internal contradictions between sections or linked docs, and a structured edge-case check (empty, error, loading, permission, boundary, offline states) the PRD hasn't clearly addressed. Use this whenever the user asks to "audit this PRD," "review this spec for gaps," "check this PRD for holes," mentions wanting a requirements-quality pass before handoff or dev kickoff, or hands over a PRD and asks what's missing or unclear. If the user also has a Figma/design to check the PRD against, use [[srota-prd-design-audit]] instead.
---

# PRD Audit

Reads a product spec closely and reports where it would leave an engineer guessing. Produces a single HTML report with four kinds of findings: **ambiguous/underspecified** requirements, things **missing** that a spec at this scope should cover, **internal contradictions**, and **edge cases** the PRD hasn't addressed.

## When this triggers

- "audit this PRD", "review this spec for gaps"
- "what's missing from this PRD", "is this spec ready for dev"
- "check this PRD before we hand off to dev"
- User supplies a PRD (file/pasted text/doc link) and wants it checked for holes, ambiguity, or contradictions

## Overview of the flow

1. **Gather inputs** — collect and normalize the PRD and any linked docs/attachments it references.
2. **Extract a requirements ledger** — every feature, screen, field, state, flow, rule, and copy string it names.
3. **Audit the ledger** for ambiguity, missing coverage, and internal contradictions.
4. **Run the edge-case checklist** against the PRD.
5. **Write the HTML report.**

Do the extraction *before* auditing — don't judge requirements one at a time as you read, or you'll miss cross-references (e.g. a rule stated on page 3 that a later section silently assumes doesn't apply).

---

## Step 1: Gather inputs

- Local file (.md/.txt/.pdf) → read it with the `Read` tool directly. For `.docx` or other formats `Read` can't parse, ask the user to export/paste it as Markdown, plain text, or PDF instead.
- Pasted text in chat → use as-is.
- Weblink (Notion/Google Doc/Confluence/etc.) → fetch it with `WebFetch`. If it's behind auth and fetch fails, tell the user and ask them to paste the content instead.
- Multiple linked docs (tech spec, API doc, ticket) mentioned inside the PRD → fetch/read those too if reachable; they often carry requirements the main PRD only references in passing, and are also where contradictions tend to hide.

Only ask the user for clarification if scope is genuinely ambiguous (e.g. the PRD covers three unrelated features and it's unclear which is in scope) — otherwise proceed with everything provided.

## Step 2: Build the requirements ledger

Read the whole PRD (and linked docs) and pull out, as a flat list, every instance of:

- **Screens/pages** named or implied ("settings screen", "checkout flow — step 2")
- **Fields & inputs** with their constraints (required/optional, max length, format, default value)
- **States** explicitly or implicitly required: empty, loading, error, success, disabled, read-only, per-role variants
- **Flows & sequencing** — what happens after each action, redirect targets, confirmation steps
- **Business rules** — validation logic, limits, permissions, pricing/quantity rules, conditional visibility
- **Copy/microcopy** explicitly specified (exact button text, error messages, tooltips)
- **Out-of-scope items** the PRD explicitly excludes
- **Success/acceptance criteria** listed at the end of most PRDs

Give each item a short ID (e.g. `PRD-014`) so it can be referenced later. Don't skip this structuring step even for a short PRD — it's what prevents a shallow, impressionistic read.

## Step 3: Audit the ledger

Go through the ledger and sort findings into three buckets:

1. **Ambiguous or underspecified** — stated, but not concretely enough to build or test from. E.g. "resend should be rate-limited" with no cooldown duration or attempt cap; "errors should be handled gracefully" with no specific error copy or recovery path; a field marked "validated" with no rule for what counts as valid.
2. **Missing or not addressed** — a thing a spec at this scope should cover but never mentions at all. E.g. no permissions/role model for a multi-user feature; no mention of what happens on failure for a payment or destructive action; a flow with a start and success state but no described error path; no rollout/migration plan for a change affecting existing data.
3. **Internal contradictions** — two parts of the PRD (or a linked doc) disagree. E.g. one section caps an action at 3 attempts, another says 5; the summary says a feature is admin-only, a later section describes a self-serve flow for all users; a linked tech spec assumes a constraint the PRD doesn't mention or contradicts.

For each finding, note: ID(s) involved, a one-line description, and severity — **Blocker** (an engineer literally cannot implement this correctly without guessing, e.g. an undefined error state on a money-moving action), **Should fix** (real gap, not implementation-blocking), or **Minor** (wording/cosmetic ambiguity unlikely to cause a wrong build).

## Step 4: Edge-case check

Run the checklist in `references/edge-case-checklist.md` against the PRD. For each relevant item mark **Covered** (cite where), **Partially covered**, or **Not addressed**. Only include items relevant to this product's actual flows — skip irrelevant categories silently rather than padding the report.

## Step 5: Write the report

Use `assets/report_template.html` as the base (self-contained HTML/CSS, no external dependencies). Fill in:
- Header: product/feature name, date, sources used (PRD file/link, any extra docs fetched)
- Summary counts (X blockers, Y should-fix, Z minor, W edge cases not addressed)
- The three findings tables from Step 3, sorted by severity
- The edge-case table from Step 4
- A short "Open questions for the team" list for anything genuinely unresolvable from the text alone

Write the filled HTML to a local file (e.g. `<feature-name>-prd-audit.html` in the current working directory, or wherever the user prefers) using the `Write` tool, then tell the user the path so they can open it themselves — don't try to publish or present it any other way. Keep the in-chat summary brief (a few lines plus the counts) — the report carries the detail; don't restate every finding inline in chat.

## Notes

- For a large PRD, don't hold everything in memory across one pass — build the ledger into a scratch file in the session's scratchpad directory as you go, then audit from those notes.
- Never invent a finding without a source to point to. If something is genuinely unresolvable from the text (not clearly missing, just unclear which way it should go), put it under "Open questions," not a findings table.
- This is a QA pass, not a redesign — flag the gap and its severity; don't propose new features or requirements beyond what's needed to close the gap.
