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

- Horizontal lines only → cleanest, works for all dataset sizes
- Full grid only for extremely dense spreadsheet-like tables
- No zebra striping — use hover highlight instead

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

In Filament: `->stickyHeader()` on the table, `->frozen()` on a column.

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

Always use explicit pagination with page numbers — never infinite scroll.
Pak Hendra has no sense of position in the data with infinite scroll.

✓ `< 1 2 3 ... 12 >` with per-page selector (25 / 50 / 100)

Always show total count: `Showing 1–25 of 847 records`.
