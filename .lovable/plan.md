

## Increase Details Panel Width by 10%

The details panel width is controlled by CSS grid column definitions in two files that must stay in sync.

### Current State
- Details panel: `minmax(750px, 3fr)`
- Expanded stage column: `minmax(300px, 300px)`

### Changes

**Increase details panel from `3fr` to `3.5fr` and from `750px` min-width to `825px`** (roughly a 10% increase).

### Files to Update

1. **`src/components/KanbanBoard.tsx`** (line 508)
   - Change `'minmax(750px, 3fr)'` to `'minmax(825px, 3.5fr)'`

2. **`src/components/kanban/AnimatedStageHeaders.tsx`** (line 54)
   - Change `'minmax(750px, 3fr)'` to `'minmax(825px, 3.5fr)'`

Both files use identical grid column logic and must remain in sync for the stage headers to align with the content grid below.

