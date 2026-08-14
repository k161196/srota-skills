# Edge-Case Checklist

Use this as a menu, not a mandate — walk through each section and only report on items that actually apply to the product being audited. Skip irrelevant sections silently.

## 1. Empty / zero states
- First-time use with no data yet (empty dashboard, empty list, empty inbox)
- A filter/search that returns zero results
- A deleted/removed item that other screens still reference

## 2. Loading & latency
- Initial load state (skeleton/spinner) for every screen with async data
- Slow network / timeout — what does the user see, is there a retry?
- Partial load (some data arrived, some didn't)

## 3. Error states
- Form validation errors — per field, and on submit
- Server/network error on any action (save, submit, delete, upload)
- Payment/transaction failure specifically, if money is involved
- 404 / not-found for deep links to deleted or nonexistent items
- Permission-denied (user navigates to something they can't access)

## 4. Boundary & limit conditions
- Minimum values (0 items, empty string, first day of a range)
- Maximum values (character limits, max file size, max items in cart/list, pagination limits)
- Exactly-at-the-limit vs. one-past-the-limit behavior
- Very long text (names, titles, comments) — truncation/wrapping
- Very large numbers (currency, counts) — formatting at scale

## 5. Concurrency & sync
- Two users/sessions editing or acting on the same resource
- Optimistic UI update that later fails to persist
- Stale data shown after another party changed it elsewhere
- Duplicate submission (double-tap/double-click a submit button)

## 6. Auth & session
- Session expiry mid-flow (especially mid-form or mid-checkout)
- Logged out in one tab, still open in another
- First login vs. returning login differences
- Account in a restricted/suspended/unverified state

## 7. Permissions & roles
- Each named role's view of every relevant screen (not just the primary role)
- What a lower-permission user sees when they lack access to a feature others have
- Ownership transfer / shared-resource edge cases (who can edit/delete what)

## 8. Offline / connectivity
- Action taken while offline — queued, blocked, or silently failed?
- Reconnection behavior — does queued/local data sync, and what if it conflicts?

## 9. Input & device variability
- Very small vs. very large screens (does the described/designed layout hold up?)
- Platform differences if multi-platform (iOS vs Android vs web quirks)
- Copy-paste, autofill, or voice input into structured fields
- Localization: longer translated strings, RTL languages, different date/number formats

## 10. Notifications & async side-effects
- What happens if a triggered notification/email fails to send
- Duplicate or delayed notifications
- Notification for an event that's since become stale (e.g. "item back in stock" after it sold out again)

## 11. Lifecycle & destructive actions
- Cancel/undo availability for irreversible actions (delete, cancel order, remove member)
- Confirmation step present for destructive actions matching their severity
- What happens to dependent data when a parent object is deleted (cascading vs orphaned)

## 12. Cross-flow interactions
- User backs out mid-flow (browser back, app backgrounding) — is state preserved or lost?
- Deep link into the middle of a multi-step flow
- Two features touching the same data disagreeing on rules (e.g. one flow enforces a limit the other doesn't)
