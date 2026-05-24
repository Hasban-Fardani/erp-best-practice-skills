# Visual Design: Typography & Color

## Font Family

Use screen-optimized sans-serif: **Inter, Roboto, or Nunito**. No decorative fonts.

## Font Size

| Element | Age 30–40 | Age 40–70 |
|---|---|---|
| Body / form input | 15–16px | **16–18px** |
| Form label | 14–15px | **16px** |
| Button text | 14–16px | **16–18px** |
| Section heading | 18–20px | **20–24px** |
| Page title | 22–24px | **24–28px** |
| Helper text / caption | 12px min | **14px min** |

Hard floor for Indonesian ERP: **16px body**. 1366×768 monitors are not retina.
Pak Hendra with reading glasses will struggle with anything smaller.

## Font Weight

Use exactly 4 values — no more:

```css
--fw-body:    400   /* body text, form field content */
--fw-label:   500   /* nav items, buttons, badges, form labels */
--fw-subhead: 600   /* section titles, table column headers */
--fw-title:   700   /* page titles, major headings */
```

Rules:
- Never use **300** for body text. Looks fine on Retina, disappears on cheap monitors.
- Never use **800–900** below 18px. Letter counters close up and become unreadable.
- `-webkit-font-smoothing: antialiased` renders fonts ~1 weight thinner. Avoid on body text.
- Skip a weight for intentional contrast: pair 400 body with 600 subheading, not 400 vs 500.
- If everything is bold, nothing is bold. Weight only works by contrast with regular.

✓ page-title `700` / section `600` / label `500` / body `400`
✗ All form labels at `600` — no hierarchy, entire page feels heavy

## Line Height & Line Length

| Context | Value |
|---|---|
| Body text / labels | `1.6 – 1.8` |
| Headings | `1.3 – 1.4` |
| Table rows | `1.5` minimum |

Max **65–70 characters** per line for 40+ users. Anything wider is tiring to read.

## Color Contrast (WCAG)

Severity: **HIGH.** Pak Hendra and Sri are 44+ — under-contrast text is
unreadable, not just suboptimal.

| Level | Ratio | Applies to |
|---|---|---|
| WCAG AA minimum | 4.5:1 | Normal text (< 18pt) |
| WCAG AA large text | 3:1 | Text ≥ 18pt or 14pt bold |
| WCAG AAA — ERP target for 40+ | **7:1** | All content text |

83.6% of websites fail contrast (WebAIM 2024).

✗ `#9ca3af` on white → ~2.8:1, fails AA
✓ `#4b5563` on white → ~7:1, passes AAA

Check: [webaim.org/resources/contrastchecker](https://webaim.org/resources/contrastchecker)

Additional rules for older users:
- Avoid similar shades of blue adjacent to each other — contrast perception degrades with age.
- Never rely on color alone. Always combine: **color + icon + text label**.
- No textured or patterned backgrounds in content areas.

## Status Badges

Severity: **HIGH.** Always use all three: **color + icon + text label**.
Color alone fails for color-blind operators (8% of male population) and on cheap
monitors with poor color rendition.

```
Active    → green    + ✓  + "Active"
Draft     → gray     + ○  + "Draft"
Pending   → amber    + ⏳ + "Awaiting Approval"
Rejected  → red      + ✗  + "Rejected"
Cancelled → red-muted + — + "Cancelled"
```

✓ Pak Hendra sees `⏳ Awaiting Approval` — clear without color perception
✗ Red badge only — Sri doesn't know if that's an error or a status

## Zero vs Null

Never render both as `0`.

| State | Display |
|---|---|
| Data exists, value is zero | `0` or `Rp 0` |
| No data / not applicable | `—` (em dash) |

## Touch & Click Targets

| Standard | Minimum size |
|---|---|
| WCAG 2.2 AA | 24×24px |
| WCAG AAA | 44×44px |
| ERP recommended for 40+ | **44–48px** |

Applies to buttons, row actions, icon buttons, and checkboxes.
