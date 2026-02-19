
## Fix Backup Record Count Bloat & All System Section Issues

### Root Cause Analysis

The full backup jumped from 5,227 → 30,777 records because the `security_audit_log` table has **25,551 rows** — accounting for 83% of the backup. Here's why:

**`src/components/SecurityProvider.tsx`** logs four event types on every session:
- `SESSION_START` — fires every time `userRole` is set (11,167 entries)
- `SESSION_END` — fires on component unmount (9,070 entries)
- `SESSION_INACTIVE` — fires every time a browser tab is hidden (2,202 entries)
- `SESSION_ACTIVE` — fires every time a browser tab becomes visible again (2,103 entries)

This means every single tab switch writes two rows to the DB. With normal browsing the table fills up rapidly.

---

### Issues & Fixes

**Issue 1 — `security_audit_log` bloating full backup (PRIMARY)**

The `create-backup` and `scheduled-backup` edge functions include `security_audit_log` in `BACKUP_TABLES`. This is audit/operational data — not business data — and it should not be part of the standard backup. Likewise `user_sessions` and `keep_alive` are ephemeral operational tables.

**Fix:** Remove `security_audit_log`, `user_sessions`, and `keep_alive` from `BACKUP_TABLES` in both edge functions.

**File:** `supabase/functions/create-backup/index.ts`
**File:** `supabase/functions/scheduled-backup/index.ts`

---

**Issue 2 — `SecurityProvider` creates excessive SESSION_INACTIVE / SESSION_ACTIVE noise**

Every browser tab switch generates two DB writes. The `SESSION_INACTIVE` and `SESSION_ACTIVE` events are already in the `EXCLUDED_ACTIONS` list in `auditLogUtils.ts` (hidden from Audit Logs UI), but they still write to the DB and bloat the `security_audit_log` table.

**Fix:** Remove `SESSION_INACTIVE` and `SESSION_ACTIVE` logging from `SecurityProvider`. Keep `SESSION_START` and `SESSION_END` since those are meaningful security events showing logins/logouts.

**File:** `src/components/SecurityProvider.tsx` — Remove the `handleVisibilityChange` listener and its two `logSecurityEvent` calls.

---

**Issue 3 — "leads" backups still appear in Backup History**

The `getBackupLabel` function in `BackupRestoreSettings.tsx` looks up `backup.module_name` in the `MODULES` array. Since `leads` was removed from `MODULES`, it falls through to the raw `backup.module_name` value ("leads") and displays it as-is. This means old leads backups in the history now show the raw string "leads" instead of a friendly label.

**Fix:** Update `getBackupLabel` to handle the legacy `leads` module gracefully — display it as "Leads (Legacy)" or map it through a fallback label lookup that includes `leads`.

**File:** `src/components/settings/BackupRestoreSettings.tsx`

---

**Issue 4 — Edge functions still have `leads` in MODULE_TABLES**

Both `create-backup` and `scheduled-backup` still define `leads: ['leads', 'lead_action_items']` in `MODULE_TABLES`. This means if someone somehow triggers a leads module backup (e.g., directly via API), it still works as a standalone module. Since leads is now part of deals at the UI level, the edge functions' `MODULE_TABLES` should be updated so the deals module backup also includes the leads-related tables for completeness.

**Fix:**
- Remove `leads` key from `MODULE_TABLES` in both edge functions
- Add `leads` and `lead_action_items` to the `deals` module backup tables so historical leads data is captured when backing up Deals

**File:** `supabase/functions/create-backup/index.ts`
**File:** `supabase/functions/scheduled-backup/index.ts`

---

**Issue 5 — Full backup still includes `leads` table (correct to keep)**

The `leads` table still exists in the database and contains real data (14 records). The full backup SHOULD include the `leads` table for data safety. This is correct behavior — keep it in `BACKUP_TABLES`. Only the UI module card and module-scoped backup for "leads" as a standalone module should be removed.

---

### Summary of Files to Change

| File | Change |
|---|---|
| `supabase/functions/create-backup/index.ts` | Remove `security_audit_log`, `user_sessions`, `keep_alive` from `BACKUP_TABLES`; remove `leads` from `MODULE_TABLES`; add `leads`/`lead_action_items` to deals module tables |
| `supabase/functions/scheduled-backup/index.ts` | Same changes as create-backup |
| `src/components/SecurityProvider.tsx` | Remove `SESSION_INACTIVE` and `SESSION_ACTIVE` event listeners; keep `SESSION_START` and `SESSION_END` |
| `src/components/settings/BackupRestoreSettings.tsx` | Fix `getBackupLabel` to handle legacy `leads` module name gracefully |

### Files NOT changed (by design)

- `supabase/functions/restore-backup/index.ts` — The restore function correctly excludes `security_audit_log` from DELETE_ORDER and INSERT_ORDER (it already handles this properly for tables present in the backup file only)
- `src/components/settings/audit/auditLogUtils.ts` — `SESSION_INACTIVE`/`SESSION_ACTIVE` already correctly excluded from UI display
- `src/components/settings/AuditLogsSettings.tsx` — Audit log viewer is correct; the problem is data volume not display logic

### Expected Impact After Fix

- Future full backups will exclude ~25,000+ audit log rows, bringing record counts back to ~5,200 range
- `security_audit_log` will no longer grow at ~100+ rows/day from session noise
- Backup History will display legacy leads backups with a readable label instead of raw "leads"
- Deals module backup will now include historical leads data
