

## Campaign Module - Merge Accounts/Contacts and Fix Issues

### Current Problems

1. **Accounts and Contacts are separate tabs** -- User must switch between tabs to add accounts then contacts. Should be one unified "Accounts & Contacts" tab where selecting an account expands to show its contacts.
2. **No channel-readiness validation** -- Email/Call/LinkedIn outreach buttons don't check if the contact has the required field (email, phone_no, linkedin).
3. **Contact loading capped at 1000** -- `allContacts` query fetches all contacts with no pagination. Supabase default limit is 1000 rows. With 4,432 contacts in DB, many are missing.
4. **Contact details incomplete** -- The Add Contacts modal query doesn't fetch `phone_no` or `linkedin`. The contacts table display doesn't show email.

### Plan

#### 1. Merge Accounts & Contacts into One Tab

**File: `src/pages/CampaignDetail.tsx`**
- Replace the 7-tab layout (Overview, MART, Accounts, Contacts, Outreach, Tasks, Analytics) with 6 tabs: **Overview, MART Strategy, Accounts & Contacts, Outreach, Tasks, Analytics**
- The merged tab renders a new `CampaignAccountsContacts` component

**New file: `src/components/campaigns/CampaignAccountsContacts.tsx`**
- Top section: Account list (existing `CampaignAccounts` logic) with an "Add Accounts" button
- Each account row is expandable -- clicking it shows contacts linked to that account
- Inside the expanded section: contact list with checkboxes for selection, plus "Add Contacts" button scoped to that account
- The "Add Contacts" modal shows contacts where `company_name` matches the selected account
- Keep existing contact actions (Email, Call, LinkedIn, Convert to Deal) inline
- Standalone contacts (no account) shown in a separate "Unlinked Contacts" section at the bottom

#### 2. Channel-Readiness Validation

**File: `src/components/campaigns/CampaignAccountsContacts.tsx`** (in the merged component)
- Before opening Email slide-over: check `contact.email` exists. If missing, show toast: "This contact has no email address."
- Before opening Call slide-over: check `contact.phone_no` exists. If missing, show toast.
- Before opening LinkedIn slide-over: check `contact.linkedin` exists. If missing, show toast.
- Visually dim/disable the action icon when the field is missing, with a tooltip explaining why

#### 3. Fix 1000-Record Limit

**File: `src/components/campaigns/CampaignAccountsContacts.tsx`**
- When fetching contacts for the "Add Contacts" modal, use paginated fetching:
  ```typescript
  // Fetch all contacts in batches of 1000
  let allData = [];
  let from = 0;
  const batchSize = 1000;
  while (true) {
    const { data } = await supabase.from("contacts")
      .select("id, contact_name, email, position, company_name, phone_no, linkedin")
      .range(from, from + batchSize - 1);
    allData.push(...(data || []));
    if (!data || data.length < batchSize) break;
    from += batchSize;
  }
  ```
- This ensures all 4,432+ contacts are loaded

#### 4. Show Contact Details (Email, Phone, LinkedIn)

- In the contacts table within the merged component, add Email column
- In the "Add Contacts" modal, show email beneath position info so users can verify contact data before adding
- Fetch `phone_no` and `linkedin` fields in all contact queries

### Files to Modify

| File | Change |
|------|--------|
| `src/pages/CampaignDetail.tsx` | Merge Accounts + Contacts tabs into one "Accounts & Contacts" tab |
| `src/components/campaigns/CampaignAccountsContacts.tsx` | **New file** -- Unified component with expandable accounts, nested contacts, channel validation, paginated loading |
| `src/components/campaigns/CampaignOverview.tsx` | Update any references to separate accounts/contacts tab names |
| `src/components/campaigns/CampaignAccounts.tsx` | Keep for reference but no longer imported directly in CampaignDetail |
| `src/components/campaigns/CampaignContacts.tsx` | Keep for reference; logic absorbed into new component |

### UI Layout

```text
┌─ Accounts & Contacts ─────────────────────────────────────┐
│ [+ Add Accounts]  [Search...]  [Filter: Stage ▾]          │
│                                                            │
│ ▼ Realthingks (5 contacts)           Not Contacted         │
│   ├─ Deepak Dongare    deepak@...    📧 📞 🔗  [Deal]     │
│   ├─ Lukas Schleicher  oliver@...    📧 ⚠📞 ⚠🔗           │
│   ├─ Peter Jakobsson   peter@...     📧 ⚠📞 ⚠🔗           │
│   └─ [+ Add Contacts from Realthingks]                     │
│                                                            │
│ ▶ Another Account (2 contacts)       Contacted             │
│                                                            │
│ ── Unlinked Contacts ──                                    │
│   └─ (contacts with no matching account)                   │
└────────────────────────────────────────────────────────────┘

⚠ = field missing, icon dimmed with tooltip
```

### Technical Notes
- The `contacts` table has `email`, `phone_no`, and `linkedin` columns -- all needed for channel validation
- All existing slide-over logic (Email, Call, LinkedIn) and Convert-to-Deal logic will be moved from `CampaignContacts.tsx` into the new component
- Paginated fetch solves the Supabase default 1000-row limit
- No database migrations needed

