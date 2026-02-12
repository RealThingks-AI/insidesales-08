

## Bug Fixes for Deals List View -- InlineEditCell and ListView

After a thorough audit of `InlineEditCell.tsx` and `ListView.tsx`, here are all the bugs found and the plan to fix them.

---

### Bug 1: Input field not visible in narrow columns (CRITICAL -- user-reported)

**Root cause:** When a cell enters edit mode for non-auto-save types (text, number, currency, textarea), the container renders the Input plus two 24px-wide save/cancel buttons side by side in a flex row. With cells constrained to `maxWidth` (e.g., 100px for Priority, 120px for Value), the `flex-1 min-w-0` input wrapper shrinks to near-zero width, making the input invisible.

**Fix in `InlineEditCell.tsx`:**
- Change the editing container layout from horizontal flex to a stacked layout for non-auto-save types: input on top, save/cancel buttons below (or overlaid to the right)
- Add `min-w-[60px]` to the input wrapper so it never collapses
- Move save/cancel buttons to overlay position (absolute, top-right) so they don't consume the input's width

### Bug 2: Currency `formatDisplayValue` crashes or shows wrong symbol

**Root cause (line 127-128):** `formatDisplayValue` for currency tries to access `value.currency_type` but the `value` prop is just a number (e.g., `deal.total_contract_value` is a number). The currency symbol lookup fails and defaults to `EUR`. Additionally, `Number(value).toLocaleString()` works on a number but the template string concatenation is fragile.

**Fix in `InlineEditCell.tsx`:**
- Remove the `value.currency_type` lookup from `formatDisplayValue`
- Instead, just format the number with a default currency symbol or accept a `currencyType` prop
- Or simplify to just show the number formatted

### Bug 3: `getFieldType` returns `'text'` for select-type fields

**Root cause (lines 191-198 in ListView.tsx):** Fields like `customer_challenges`, `relationship_strength`, `rfq_status`, `handoff_status`, `is_recurring` are enum/select fields in the Deal type, but `getFieldType` maps them as `'text'`. This means users see a plain text input instead of a dropdown when editing these fields.

**Fix in `ListView.tsx`:**
- Add select-type mappings for: `customer_challenges`, `relationship_strength`, `decision_maker_level`, `rfq_status`, `handoff_status`, `is_recurring`, `currency_type`
- Populate `getFieldOptions` with the correct options for each field

### Bug 4: `getFieldOptions` always returns empty array

**Root cause (lines 200-202 in ListView.tsx):** `getFieldOptions` returns `[]` for all fields, so even if a field were typed as `'select'`, the dropdown would render empty.

**Fix in `ListView.tsx`:**
- Return proper options arrays based on field name (e.g., `customer_challenges` returns `['Open', 'Ongoing', 'Done']`)

### Bug 5: Boolean switch uses stale `value` prop instead of `editValue`

**Root cause (line 226 in InlineEditCell.tsx):** The boolean Switch uses `Boolean(value)` from props, not from `editValue` state. While the auto-save fires immediately so this is cosmetically fine, it's inconsistent and would break if auto-save behavior changes.

**Fix:** Use `editValue` state for the Switch checked value.

### Bug 6: Table cell overflow clips Select dropdowns

**Root cause:** The table cells have strict `maxWidth` constraints and the table wrapper uses `overflow-scroll`. While `[&>div.relative]:!overflow-visible` attempts to fix this, Select dropdowns from Radix portal outside the cell -- this is actually fine since Radix uses portals. However, the SelectTrigger inside a narrow cell gets cut off visually.

**Fix:** For select/stage/priority types in edit mode, ensure the SelectTrigger has a readable minimum width by setting `min-w-[100px]` on the select trigger.

### Bug 7: Click-outside cancels instead of saving for text/number/currency

**Root cause (lines 86-95 in InlineEditCell.tsx):** When clicking outside a text/number/currency input, `handleCancel` is called, discarding the user's edits. Most CRMs auto-save on blur/click-outside.

**Fix:** Change click-outside behavior to call `handleSave` instead of `handleCancel` for text/number/currency types, matching user expectations.

---

### Technical Implementation

**File: `src/components/InlineEditCell.tsx`**

1. Change the editing container (line 290) to use relative positioning for the save/cancel overlay:
   - Wrap the whole edit area; place save/cancel as absolute-positioned buttons so they don't steal input width
   - Add `min-w-[60px]` to the input wrapper

2. Fix `formatDisplayValue` for currency (lines 126-129):
   - Remove `value.currency_type` access
   - Just format as number with a basic symbol

3. Fix boolean Switch (line 226): use `Boolean(editValue)` or pass through auto-save

4. Change click-outside handler (line 90): call `handleSave()` instead of `handleCancel()`

**File: `src/components/ListView.tsx`**

5. Update `getFieldType` (lines 191-198) to include select-type fields:
   ```
   if (['customer_challenges', 'business_value', 'decision_maker_level'].includes(field)) return 'select';
   if (['relationship_strength'].includes(field)) return 'select';
   if (['rfq_status'].includes(field)) return 'select';
   if (['handoff_status'].includes(field)) return 'select';
   if (['is_recurring'].includes(field)) return 'select';
   if (['currency_type'].includes(field)) return 'select';
   ```

6. Update `getFieldOptions` (lines 200-202) to return proper options:
   ```
   customer_challenges/business_value/decision_maker_level -> ['Open', 'Ongoing', 'Done']
   relationship_strength -> ['Low', 'Medium', 'High']
   rfq_status -> ['Drafted', 'Submitted', 'Rejected', 'Accepted']
   handoff_status -> ['Not Started', 'In Progress', 'Complete']
   is_recurring -> ['Yes', 'No', 'Unclear']
   currency_type -> ['EUR', 'USD', 'INR']
   ```
