

## Fix: Action Item "Not Found" from Email Reminder Links

### Root Cause

Two issues work together to cause this bug:

**Issue 1 — Email uses title, not ID:** In `daily-action-reminders/index.ts` line 138, the link is built as:
```
${appUrl}/action-items?highlight=${encodeURIComponent(item.title)}
```
This uses the item's **title** as the identifier, which is fragile (titles can have special characters, duplicates, or change over time).

**Issue 2 — Highlight logic only searches the filtered in-memory list:** In `ActionItems.tsx` line 68, the code does:
```
actionItems.find(a => a.id === highlightId || a.title === highlightId)
```
This only searches items already loaded by the current query (which excludes Completed/archived items by default). If the item doesn't match exactly or isn't in the current filter view, it shows "Item not found."

**Issue 3 — Race condition risk:** The `useEffect` fires once and immediately sets `highlightProcessed = true`, so if the data hasn't settled yet or filters change, the match opportunity is lost.

### Fix Plan

#### 1. `supabase/functions/daily-action-reminders/index.ts`
Change the email link from title-based to ID-based:
```typescript
// Before:
const itemUrl = `${appUrl}/action-items?highlight=${encodeURIComponent(item.title)}`;
// After:
const itemUrl = `${appUrl}/action-items?highlight=${item.id}`;
```
This is a UUID, guaranteed unique and stable.

#### 2. `src/pages/ActionItems.tsx`
Replace the current highlight `useEffect` with a more robust approach:
- First try to find the item in the already-loaded `actionItems` list (by id OR title for backward compatibility with old emails)
- If not found in the list, do a **direct Supabase query** using `.maybeSingle()` to fetch the item by `id` first, then by `title` as fallback
- If found via direct query, open the modal with that item (works even if the item is completed/archived/filtered out)
- Only show "Item not found" toast if the direct DB query also returns nothing
- Clean up the `highlightProcessed` state management to avoid race conditions

#### 3. Backward compatibility
Old email links still use title-based highlights (e.g., `?highlight=Test%20Email`). The new logic handles both:
- If `highlightId` looks like a UUID → query by `id`
- Otherwise → query by `title` using `ilike` for case-insensitive matching

### Files Changed

| File | Change |
|---|---|
| `supabase/functions/daily-action-reminders/index.ts` | Use `item.id` in email links instead of `item.title` |
| `src/pages/ActionItems.tsx` | Add direct Supabase fallback lookup with `.maybeSingle()` when item not in filtered list |

