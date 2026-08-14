---
name: srota-prd-design-audit
description: Audits a PRD (and any linked docs/attachments) against its design (Figma or screenshots) to find misalignments in both directions — things written in the PRD but missing/different in the design, and things shown in the design but missing from the PRD — plus a structured edge-case check (empty, error, loading, permission, boundary, offline states). Use this whenever the user asks to "check PRD vs Figma," "compare spec to design," "find gaps between requirements and screens," "audit the design against the PRD," mentions reviewing a PRD alongside design links/screenshots/weblinks/attachments, or wants a design-QA / requirements-traceability pass before handoff or dev kickoff. Trigger even if they only supply one side (e.g. just a Figma link) and ask you to flag gaps — proceed with what's available and call out what's missing. For a PRD-only quality pass with no design involved, use [[srota-prd-audit]] instead.
---

# PRD ↔ Design Alignment Audit

Cross-checks a product spec against its design so nothing silently falls through the cracks between writing and building. Produces a single HTML report with three kinds of findings: **PRD → design gaps**, **design → PRD gaps**, and **edge cases** neither side has clearly handled.

## When this triggers

- "compare this PRD to the Figma", "check if the design matches the spec"
- "find what's missing between the PRD and the screens"
- "audit this before we hand off to dev"
- User supplies a PRD (file/pasted text/doc link) plus a design (Figma link and/or screenshots) and wants alignment checked
- Only one side is supplied — still run it, and call out "design not provided, PRD-only edge-case check" (or vice versa) rather than refusing. If there's no design at all, consider [[srota-prd-audit]] instead — it's built for exactly that case.

## Overview of the flow

1. **Gather inputs** — collect and normalize the PRD, any linked docs/weblinks/attachments, and the design.
2. **Extract a requirements ledger from the PRD** — every feature, screen, field, state, flow, rule, and copy string it names.
3. **Extract a screen inventory from the design** — every screen, state, component, field, and copy string actually shown.
4. **Cross-check both directions** and list mismatches.
5. **Run the edge-case checklist** against both sources.
6. **Write the HTML report.**

Do all extraction *before* comparing — don't compare screen-by-screen live, or you'll miss cross-references (e.g. PRD describes a rule on page 3 that should show up as a screen state on a later frame).

---

## Step 1: Gather inputs

**PRD:**
- Local file (.md/.txt/.pdf) → read it with the `Read` tool directly. For `.docx` or other formats `Read` can't parse, ask the user to export/paste it as Markdown, plain text, or PDF instead.
- Pasted text in chat → use as-is.
- Weblink (Notion/Google Doc/Confluence/etc.) → fetch it with `WebFetch`. If it's behind auth and fetch fails, tell the user and ask them to paste the content instead.
- Multiple linked docs (tech spec, API doc, ticket) mentioned inside the PRD → fetch/read those too if reachable; they often carry requirements the main PRD only references in passing.

**Design:**
- Figma link → look for a Figma MCP connector (`ToolSearch` with `"figma"` — e.g. `mcp__claude_ai_Figma__*`). If found, use it to pull frame names, layers, and text content; authenticate first if it isn't connected yet.
- No Figma MCP available → ask the user for screenshots/exported images instead of guessing at the file's contents from the link alone.
- Screenshots/exported images uploaded → read them with the `Read` tool (images render visually) — read every screen, state, label, and piece of copy carefully. Don't skip modal/toast/tooltip states buried in a screenshot.
- If neither a live design connection nor screenshots are available, say so plainly and skip straight to a PRD-only edge-case pass (or point at [[srota-prd-audit]] for the full PRD-only treatment).

Only ask the user for clarification if scope is genuinely ambiguous (e.g. a Figma file with 40 frames and no clear indication which are in scope) — otherwise proceed with everything provided.

## Step 2: Build the PRD requirements ledger

Read the whole PRD (and linked docs) and pull out, as a flat list, every instance of:

- **Screens/pages** named or implied ("settings screen", "checkout flow — step 2")
- **Fields & inputs** with their constraints (required/optional, max length, format, default value)
- **States** explicitly or implicitly required: empty, loading, error, success, disabled, read-only, per-role variants
- **Flows & sequencing** — what happens after each action, redirect targets, confirmation steps
- **Business rules** — validation logic, limits, permissions, pricing/quantity rules, conditional visibility
- **Copy/microcopy** explicitly specified (exact button text, error messages, tooltips)
- **Out-of-scope items** the PRD explicitly excludes (flag if the design builds them anyway)
- **Success/acceptance criteria** listed at the end of most PRDs

Give each item a short ID (e.g. `PRD-014`) so it can be referenced later. Don't skip this structuring step even for a short PRD — it's what prevents a shallow, impressionistic comparison.

## Step 3: Build the design screen inventory

For every screen/frame available, extract:

- Screen name and its apparent purpose/flow position
- Every visible state on that frame (empty? loading? error? success? a specific role's view?)
- Every field, its label, placeholder, and any visible validation/error copy
- Every interactive element and where it visibly navigates (if annotated) or appears to lead
- Any copy, tooltips, badges, or microcopy visible
- Any states the design shows that the PRD never mentions (designers often add states — e.g. "verified badge," "3+ items in cart" — that never made it into the written spec)

Tag each with a short ID too (e.g. `DES-Checkout-02`).

## Step 4: Cross-check both directions

Go through the PRD ledger and design inventory and sort findings into three buckets:

1. **In PRD, missing/different in design** — e.g. PRD requires a "resend OTP after 30s" cooldown but no screen shows a disabled/cooldown state; PRD marks a field required but the design doesn't show it as such; PRD caps cart at 5 items but no screen shows a "limit reached" message.
2. **In design, missing/undocumented in PRD** — e.g. the design has a "verified seller" badge with no PRD mention; a settings toggle exists with no described behavior; an entire screen (e.g. "onboarding step 4") exists in the design but isn't in the PRD's flow description.
3. **Mismatches** (both exist but disagree) — different field names, different copy than specified, different flow order, different role permissions, numeric limits that don't match (PRD says "max 3 photos," design implies 4 slots).

For each finding, note: ID(s) involved, a one-line description, and severity — **Blocker** (breaks the flow or leaves a state undefined, e.g. payment error state), **Should fix** (real gap, not flow-breaking), or **Minor** (wording/cosmetic).

## Step 5: Edge-case check

Independently of the direct cross-check, run the checklist in `references/edge-case-checklist.md` against both the PRD and the design. For each relevant item mark **Covered** (cite where), **Partially covered**, or **Not addressed** (in neither source). Only include items relevant to this product's actual flows — skip irrelevant categories silently rather than padding the report.

## Step 6: Write the report

Use `assets/report_template.html` as the base (self-contained HTML/CSS, no external dependencies). Fill in:
- Header: product/feature name, date, sources used (PRD file/link, design link/screenshots, any extra docs fetched)
- Summary counts (X blockers, Y should-fix, Z minor, W edge cases not addressed)
- The three gap tables from Step 4, sorted by severity
- The edge-case table from Step 5
- A short "Open questions for the team" list for anything ambiguous or unverifiable (e.g. Figma link inaccessible, PRD section unclear)

Write the filled HTML to a local file (e.g. `<feature-name>-alignment-audit.html` in the current working directory, or wherever the user prefers) using the `Write` tool, then tell the user the path so they can open it themselves — don't try to publish or present it any other way. Keep the in-chat summary brief (a few lines plus the counts) — the report carries the detail; don't restate every finding inline in chat.

## Notes

- For a large PRD or design file, don't hold everything in memory across one pass — build the ledgers into a scratch file in the session's scratchpad directory as you go, then compare from those notes.
- Never invent a finding without a source to point to. If something is ambiguous rather than clearly missing, put it under "Open questions," not a gap table.
- This is a QA pass, not a redesign — flag the gap and its severity; don't propose new features or UI beyond that.
