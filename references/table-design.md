# Table Design

> **Scope.** Owns: visual presentation of tables — column alignment, tabular
> numerals, row style and spacing, multi-value cells, fixed header/columns,
> row actions on hover, empty states, pagination UI (page numbers, total
> count, infinite-scroll exceptions). **See also:** [[performance]] §
> Pagination Doctrine for server-side cursor/offset strategy and indexing
> · [[visual-design]] for color and typography tokens · [[navigation]] for
> row-click target rules.

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

For this ERP family, primary row actions are always visible. Do not require
horizontal scrolling, hover discovery, or opening a menu for the action the
operator is expected to take most often. This is a **HIGH** usability rule for
40+ operators and keyboard users.

Use compact icon buttons only when the icon has a stable meaning and provide a
visible tooltip/title plus an accessible label. Use text buttons for ambiguous
or high-consequence actions. Keep the action column sticky when the table is
wide.

Secondary actions may live in a dropdown, but the menu must:

- remain inside the viewport or use a collision-aware placement;
- open upward or into a portal when the row is near the table footer;
- never be clipped by `overflow: hidden`, the table wrapper, or a sticky footer;
- have a keyboard path and an explicit close path.

The last visible row is a required test fixture. A screenshot with an open menu
on the final row is not optional evidence.

✓ Primary `Lihat` and `Edit` stay visible; `Arsipkan`, `Duplikat`, and `Lainnya`
  can be secondary
✗ Every action is hidden behind `Lainnya`, or a dropdown opens below the footer

## Sorting Contract

Default sorting is a business rule, not an accidental database order. Before
building a data table, record:

1. The primary recency field. Start with the newest relevant date descending,
   not automatically `created_at`.
2. All equivalent date fields that can represent recency in that domain
   (`issued_at`, `updated_at`, `submitted_at`, `effective_at`, etc.) and the
   precedence when more than one exists.
3. A stable tie-breaker such as an immutable ID or document number so refreshes
   do not reshuffle equal timestamps.
4. Allowed user sorts: multi-column ordering, direction for every column, and
   A–Z/Z–A for text fields.
5. The URL/request representation and authorization allow-list for sortable
   columns. Never accept a raw user-supplied SQL column.

Minimum evidence for a new table: Pest cases for newest-first default, date
precedence, stable ties, multi-column ordering, A–Z/Z–A text ordering, invalid
sort rejection, and query-string persistence.

For bulk operations: show bulk action bar only after at least one row is selected.
Cursor and background must change on hover to signal the row is interactive.
If clicking a row does nothing, remove the pointer cursor.

## Empty State

Never leave a blank table — Sri will assume the system is broken.

✓ "No quotations found for the selected filters. [+ Create Quotation]"
✓ "No results match your search. Try adjusting the date range."
✗ Table with column headers and no rows, no message

## Pagination (Visual Presentation)

This section owns the **UX presentation** of pagination. Server-side strategy
(cursor vs offset, page size by volume, indexing, count caching) belongs to
[[performance]] § Pagination Doctrine.

**Default for transactional lists: explicit pagination with page numbers.**
Severity: HIGH. Pak Hendra has no sense of position in the data with infinite
scroll. He cannot say "I'm on page 4" or "let me go back to page 2."

✓ `< 1 2 3 ... 12 >` with per-page selector (25 / 50 / 100)

Always show total count: `Showing 1–25 of 847 records`. For huge tables where
exact count is expensive, fall back to `Showing 1–25 of many` — the count
caching pattern is in [[performance]] § Query Patterns to Avoid.

**Infinite scroll is acceptable for:**
- Activity feeds and audit timelines (chronological browsing, no position needed)
- Notification lists (transient, dismissable)
- Comment threads on a single record

**Never use infinite scroll for:**
- Invoice / quotation / contract lists (operator needs position, count, filter persistence)
- Approval queues (each record needs deliberate handling, not casual scrolling)
- Anything Sri filters and returns to — back navigation loses position
- Any list operators reference by "the third one from the top of page 4"
