

## Optimize "Add Accounts" Modal with Contact Counts and Expandable Contact Selection

### Problem
The current "Add Accounts to Campaign" modal shows a flat list of account names with industry/region but no indication of how many contacts exist under each account. Users cannot see or select individual contacts from this modal.

### Solution
Redesign the modal to show contact counts per account and allow expanding each account row to view/select individual contacts — essentially combining account and contact selection into one step.

### Changes

**File: `src/components/campaigns/CampaignAccountsContacts.tsx`**

1. **Fetch contacts when Add Account modal opens** — Modify the `allContacts` query to also be enabled when `addAccountModalOpen` is true (currently only enabled for `addContactModalOpen`). This gives us contact data to count and display.

2. **Show contact count badge on each account row** — For each available account, compute the count of contacts in the `contacts` table matching that `company_name`. Display it as a badge like `(12 contacts)` next to the account name.

3. **Add expandable contact list per account** — Use a `Collapsible` (already imported) on each account row. Clicking a chevron expands the row to show all contacts under that account with checkboxes. Selecting contacts here auto-includes them when the accounts are added.

4. **Track selected contacts alongside selected accounts** — Add a `selectedContactIds` state array. When an account is added, also insert any selected contacts from the expanded list into `campaign_contacts` in the same `handleAddAccounts` flow.

5. **Optimize the modal layout**:
   - Make the dialog wider (`sm:max-w-[700px]`) to accommodate the nested contact list
   - Add a summary bar showing "X accounts, Y contacts selected"
   - Group contacts under each account with indentation
   - Show contact email/phone/linkedin indicators as small icons

### UI Layout

```text
┌─────────────────────────────────────────────────┐
│ Add Accounts to Campaign                        │
├─────────────────────────────────────────────────┤
│ 🔍 Search accounts...                          │
│ ☐ Select All (45)                               │
├─────────────────────────────────────────────────┤
│ ▶ ☐ Bosch HQ, Germany          102 contacts    │
│ ▶ ☐ Magna International         100 contacts    │
│ ▼ ☑ Continental HQ, Germany      83 contacts    │
│    ├ ☑ John Smith · john@co.de  📧📞           │
│    ├ ☐ Jane Doe · jane@co.de    📧🔗           │
│    └ ☐ Max Müller               📞              │
│ ▶ ☐ Visteon Corporation          70 contacts    │
├─────────────────────────────────────────────────┤
│ 2 accounts, 1 contact selected    Cancel  [Add] │
└─────────────────────────────────────────────────┘
```

### Technical Details

- Contact counts computed via `useMemo` grouping `allContacts` by `company_name` matched against `account.account_name`
- Expanded state tracked with a `Set<string>` of account IDs (reuse existing pattern from `expandedAccounts`)
- On submit: insert selected accounts into `campaign_accounts`, then insert selected contacts into `campaign_contacts` with auto-matched `account_id`
- No database changes needed — all data already available from existing queries

### Files Modified
| File | Change |
|---|---|
| `src/components/campaigns/CampaignAccountsContacts.tsx` | Redesign Add Accounts modal with contact counts, expandable rows, and combined account+contact selection |

