

## MART Section — Comprehensive Improvements

### Issues Found

1. **No MART progress bar or summary at the top** — Users jump straight into sections without seeing overall progress
2. **No validation before marking done** — Users can mark Message "Done" with zero email templates, zero scripts, etc.
3. **Sections don't show content counts** — No indication of how many templates/scripts/regions exist without expanding
4. **No delete confirmation** — Email templates, phone scripts, LinkedIn messages, and materials delete instantly without confirmation
5. **Audience section doesn't reinitialize when campaign data changes** — `useState` initializer runs once; if campaign updates externally, audience form is stale
6. **Region section same stale-state issue** — Doesn't sync with updated campaign data
7. **Timing section doesn't sync `timingNotes` changes** — Initial `useState(timingNotes || "")` never updates when prop changes
8. **Email template name not shown in list** — Only subject line is displayed; template_name is hidden
9. **No "Expand All / Collapse All" toggle** — Users must click each section individually
10. **Message section has nested Cards inside CardContent** — Creates visual clutter with card-in-card nesting
11. **LinkedIn char counter shows wrong count for saved templates** — Displays length but no visual progress bar
12. **No reorder or duplicate for templates/scripts** — Common workflow actions missing
13. **Script audience segment is free text** — Unlike email templates which use checkboxes, scripts use a plain text input for segments
14. **Materials lack file size display** — No indication of uploaded file sizes

### Plan

**File: `src/components/campaigns/CampaignMARTStrategy.tsx`**
- Add an overall MART progress bar with percentage at the top of the section
- Add content count badges next to each section label (e.g., "Message · 3 templates, 1 script")
- Pass content counts from `useCampaignDetail` data (emailTemplates, phoneScripts, materials) to display inline
- Add "Expand All / Collapse All" button
- Add validation gate on "Save & Mark Done" — check that meaningful content exists before allowing marking done:
  - Message: at least 1 email template OR 1 phone script OR 1 LinkedIn template
  - Audience: at least 1 field populated (job titles, departments, etc.)
  - Region: at least 1 region card defined
  - Timing: start_date and end_date must be set on the campaign
- Show a warning toast if validation fails instead of silently marking done

**File: `src/components/campaigns/CampaignMARTMessage.tsx`**
- Show template_name prominently in the email template list cards
- Add delete confirmation dialog (AlertDialog) for email templates, phone scripts, LinkedIn messages, and materials
- Add duplicate button for email templates and phone scripts
- Fix script audience segment to use multi-checkbox (same SEGMENTS array as email templates) instead of free text
- Remove nested Card wrappers — use simple bordered sections to reduce visual noise
- Add a "Copy to clipboard" button for LinkedIn messages

**File: `src/components/campaigns/CampaignMARTAudience.tsx`**
- Add `useEffect` to sync state when `campaign.target_audience` changes externally
- Remove outer Card wrapper (parent already wraps in CardContent)

**File: `src/components/campaigns/CampaignMARTRegion.tsx`**
- Add `useEffect` to sync state when `campaign.region` changes externally
- Remove outer Card wrapper (parent already wraps in CardContent)
- Add delete confirmation before removing a region card

**File: `src/components/campaigns/CampaignMARTTiming.tsx`**
- Add `useEffect` to sync `notes` state when `timingNotes` prop changes
- Remove outer Card wrapper (parent already wraps in CardContent)
- Show "best outreach window" hint based on region timezones (if regions are set)

**File: `src/pages/CampaignDetail.tsx`**
- Pass emailTemplates, phoneScripts, materials counts to CampaignMARTStrategy so it can display content summaries per section

### Technical Details

- Validation logic lives in `CampaignMARTStrategy` — it reads the existing query data (emailTemplates, phoneScripts, etc.) from `useCampaignDetail` which is already fetched
- No database changes needed — all data is already available
- Content counts passed as new props: `emailTemplateCount`, `phoneScriptCount`, `linkedinTemplateCount`, `materialCount`, `regionCount`, `audienceData`
- Delete confirmation uses existing `AlertDialog` component already imported elsewhere in the app

### Files Modified
| File | Change |
|---|---|
| `src/components/campaigns/CampaignMARTStrategy.tsx` | Progress bar, content counts, expand all, validation gates |
| `src/components/campaigns/CampaignMARTMessage.tsx` | Delete confirmation, show template name, duplicate, fix script segments, copy LinkedIn |
| `src/components/campaigns/CampaignMARTAudience.tsx` | Sync state on prop change, remove nested Card |
| `src/components/campaigns/CampaignMARTRegion.tsx` | Sync state on prop change, remove nested Card, delete confirm |
| `src/components/campaigns/CampaignMARTTiming.tsx` | Sync notes state, remove nested Card |
| `src/pages/CampaignDetail.tsx` | Pass additional data props to MARTStrategy |

