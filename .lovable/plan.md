
## Remove Leads as a Standalone Module — Full Cleanup

Since Leads have been moved under Deals, multiple UI sections, settings, backup tools, notifications, and dropdowns still reference "Leads" as an independent module. Here is every affected file and the exact change needed.

---

### 1. Backup & Restore Settings (`src/components/settings/BackupRestoreSettings.tsx`)

**Two issues:**
- Line 59: The `MODULES` array includes `{ id: 'leads', name: 'Leads', ... }` — this shows Leads in the **Module Backup** card grid and the **Scope** dropdown.
- Line 166: `fetchModuleCounts` queries the `leads` table to display the count badge.

**Fix:** Remove the `leads` entry from the `MODULES` array. Remove `'leads'` from the `fetchModuleCounts` tables array. This will hide the Leads card from Module Backup and remove Leads from the Scheduled Backup Scope dropdown — consistent with the screenshot shown.

---

### 2. Notifications Page (`src/pages/Notifications.tsx`)

**Three issues:**
- Lines 71–72: `module_type === 'leads'` navigates to `/leads?highlight=...` (dead route).
- Line 76: `notification.lead_id` navigates to `/leads?highlight=...` (dead route).
- Line 82: `notification_type === 'lead_update'` navigates to `/leads` (dead route).
- Line 220: Empty state text says "action items and leads" — Leads no longer a standalone module.

**Fix:** Change all `/leads` navigations to `/deals`. Update empty state text to "action items and deals".

---

### 3. Notification Bell (`src/components/NotificationBell.tsx`)

Already partially fixed (`leads` module → `/deals`) but `lead_id` fallback also navigates to `/deals` — this is already correct. No change needed here.

---

### 4. Profile Section — Default Module Dropdown (`src/components/settings/account/ProfileSection.tsx`)

**Issue:** Line 282: `<SelectItem value="leads">Leads</SelectItem>` — shows Leads as a startup page option which no longer exists.

**Fix:** Remove the `leads` SelectItem. The `/leads` route already redirects to `/deals`, but it's confusing UX to show it as an option. Remove it from the dropdown.

---

### 5. CRMSidebar (`src/components/CRMSidebar.tsx`)

**Issue:** Line 27: `{ icon: UserPlus, label: 'Leads', path: '/leads' }` — this is a legacy sidebar still showing Leads as a navigation item.

**Fix:** Remove the Leads nav item from this sidebar. (Note: The active `AppSidebar.tsx` already doesn't include Leads — this file appears to be a legacy component but should still be cleaned up.)

---

### 6. Audit Log Utils (`src/components/settings/audit/auditLogUtils.ts`)

**Issue:** Line 100: `leads: 'Leads'` in the readable resource type map. This is **intentionally kept** — old audit log entries still reference `resource_type: 'leads'` and must remain displayable in the Audit Logs section. No change needed here (historical logs must still render).

---

### 7. Action Items Table (`src/components/ActionItemsTable.tsx`)

**Issue:** Line 452: Icon logic `if (t === 'lead' || t === 'leads')` renders a `UserPlus` icon for action items linked to the legacy leads module. Since leads still exist in the database as a `module_type`, this display logic should be kept for backward compatibility. No change needed.

---

### 8. Import/Export Hooks (internal logic — keep as-is)

Files like `leadsCSVExporter.ts`, `leadsCSVProcessor.ts`, `genericCSVProcessor.ts`, `columnConfig.ts`, `headerMapper.ts`, `duplicateChecker.ts` still reference `leads` as a table-level concept. These are **backend data processing utilities** that interact directly with the `leads` table in the database (which still exists). These should NOT be changed — removing them would break any Leads backup/restore and import/export operations.

---

### Summary of Files to Change

| File | Change |
|---|---|
| `src/components/settings/BackupRestoreSettings.tsx` | Remove `leads` from `MODULES` array and `fetchModuleCounts` tables |
| `src/pages/Notifications.tsx` | Fix 3 `/leads` navigations → `/deals`; update empty state text |
| `src/components/settings/account/ProfileSection.tsx` | Remove `<SelectItem value="leads">Leads</SelectItem>` |
| `src/components/CRMSidebar.tsx` | Remove Leads nav item |

### Files NOT changed (by design)

- `src/components/AppSidebar.tsx` — Already correct, no Leads item
- `src/App.tsx` — Already has `/leads` → `/deals` redirect, keep it
- `src/components/NotificationBell.tsx` — Already routes leads to `/deals`
- `src/components/settings/audit/auditLogUtils.ts` — Keeps `leads: 'Leads'` for historical audit log display
- Import/Export hooks — Internal data layer, directly uses the `leads` DB table
- `src/components/ActionItemsTable.tsx` — Keeps leads icon for backward-compatible display of old linked tasks
