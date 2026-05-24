# Table Design

## Column Alignment

Column header always matches its data alignment — mismatched alignment creates visual noise.

| Data Type | Alignment | Reason |
|---|---|---|
| Names, labels, descriptions | Left | Brain reads left→right |
| Monetary values, quantities, percentages | Right | Compare digits right to left |
| Dates as strings (`2025-01-01`) | Left | Identifier, not a numeric value |
| IDs, document codes | Left | Not compared by magnitude |
| Checkboxes, icons, avatars | Center | No reading direction |
| Action buttons (last column) | Right | Universal convention |

✓ "Total" header right-aligned, values right-aligned
✗ "Total" header left-aligned, values right-aligned → looks like a layout bug

## Numbers

Use tabular numerals so digits are equal-width and columns scan vertically:

```css
.col-number {
  font-variant-numeric: tabular-nums;
}
```

Keep decimal places consistent across rows. If one row shows `1,500.00`,
all rows must show two decimal places. Mixed precision looks like data errors.

## Row Style & Spacing

**Default: horizontal lines only.** Severity: PREFERENCE. Cleanest pattern,
works for all dataset sizes, doesn't compete visually with the data.

**Use full grid when:**
- Spreadsheet-like dense data where column boundaries matter for scanning
  (financial statements, payroll line items, journal entries)
- Many narrow numeric columns where visual containment helps

**Use zebra striping when:**
- Very wide tables (15+ columns) where horizontal eye-tracking breaks down
- Mixed-content rows (text + numbers + dates + actions) where row identity helps
- The audience has explicitly requested it (Pak Hendra was raised on Excel)

In any case: hover highlight is mandatory. Zebra without hover is worse than
lines without zebra. Zebra colors must be subtle (≤ 5% luminance difference)
or they overwhelm the data.

| Property | Value |
|---|---|
| Row height (general) | 44px minimum |
| Row height (40+ users) | 48px |
| Column padding (each side) | 16px |
| Hover background | `#EFF6FF` |
| Selected row background | `#DBEAFE` |
| Row background | `#FFFFFF` or `#F8F9FA` |

## Stacking Multiple Values in One Cell

Stack when secondary info is purely supplementary:
- Name + email (email as sub-text below name)
- Date + time
- Amount + currency unit

Never stack values that need to be compared, or long text that forces row height up.
For long text: truncate with a hover tooltip showing the full value.

✓ `"PT Maxximum Digit..." [hover: PT Maxximum Digital Indonesia]`
✗ Text wrapping to 3 lines → breaks alignment across the entire row

## Fixed Header & Columns

- Fixed header when table scrolls vertically — Sri shouldn't scroll up to recall column names
- Fixed first column when table scrolls horizontally — never lose the row identifier
- Fixed last column when table is wide and has totals or actions

Implementation: `position: sticky` for HTML tables, or built-in
"stickyHeader" / "frozen column" options on most table components (AG Grid,
TanStack Table, MUI DataGrid, Ant Design Table, Filament tables, etc.).

## Row Actions

Show actions on hover only — permanently visible buttons on every row fill the table with noise.

✓ Hover row → `Edit | View | Delete` appear at the right edge
✗ 3 buttons on every row always → Sri can't scan the actual data

For bulk operations: show bulk action bar only after at least one row is selected.
Cursor and background must change on hover to signal the row is interactive.
If clicking a row does nothing, remove the pointer cursor.

## Empty State

Never leave a blank table — Sri will assume the system is broken.

✓ "No quotations found for the selected filters. [+ Create Quotation]"
✓ "No results match your search. Try adjusting the date range."
✗ Table with column headers and no rows, no message

## Pagination

**Default for transactional lists: explicit pagination with page numbers.**
Severity: HIGH. Pak Hendra has no sense of position in the data with infinite
scroll. He cannot say "I'm on page 4" or "let me go back to page 2."

✓ `< 1 2 3 ... 12 >` with per-page selector (25 / 50 / 100)

Always show total count: `Showing 1–25 of 847 records`. For huge tables where
exact count is expensive: `Showing 1–25 of many` is acceptable —
see [[performance]] § Query Patterns to Avoid.

**Infinite scroll is acceptable for:**
- Activity feeds and audit timelines (chronological browsing, no position needed)
- Notification lists (transient, dismissable)
- Comment threads on a single record

**Never use infinite scroll for:**
- Invoice / quotation / contract lists (operator needs position, count, filter persistence)
- Approval queues (each record needs deliberate handling, not casual scrolling)
- Anything Sri filters and returns to — back navigation loses position
- Any list operators reference by "the third one from the top of page 4"

For deep pagination at scale (10k+ rows), use cursor pagination not offset.
See [[performance]] § Pagination Doctrine.
