# Student Reading Progress Dashboard — Spec

Reference implementation: `index.html` (single self-contained HTML file, CSS and JS inline). Two external dependencies: Google Fonts (Roboto) and the hosted Ignite logo SVG.

This is a prototype dashboard for school administrators to track K-2 student reading progress across a district. It uses procedurally generated demo data. The goal is to validate the information architecture, visual design, and interaction model before building a production version.

## Context

Ignite Reading provides 1:1 virtual reading tutoring for K-2 students. This dashboard shows administrators how their students are progressing through the school year. The audience is district administrators, school principals, and literacy coordinators who need to quickly understand which students are on track and which need attention.

## Navigation hierarchy

Three levels with full drill-down. Each level is a single-page view swap (no routing, no URL changes).

1. **District view** — cards for each school, each showing three aggregate charts (additional months of growth, WCPM, high-frequency words) across all grades at that school. Schools are listed alphabetically.
2. **School view** — cards for each grade at that school, each showing the same three aggregate charts scoped to that grade.
3. **Grade view** — student-level table with sortable columns, a metric toggle (pill buttons on desktop, a dropdown on mobile), and per-student bar charts.

Back navigation uses a text button with a left arrow icon. The back button label is the name of the parent (school view back button says the district name, grade view back button says the school name).

## Data model

All data is generated client-side using a seeded PRNG (mulberry32-style) so results are deterministic across page loads. No external data source.

### Grade configuration (`GM` object)

All three grades have the same four metrics in the same order so the dashboard reads consistently as students progress. WCPM is the metric of truth across grades; NWF and LNF stay available as foundational checks for students who haven't yet mastered them.

**Kindergarten** — 4 metrics, 2 key (WCPM, HFW), 2 supplementary (NWF, LNF)
- Words correct per minute: target 10, BOY benchmark 2, max 15, unit "WCPM"
- High-frequency words: target 20, BOY benchmark 5, max 25, unit "high-frequency words"
- Nonsense word fluency: target 28, BOY benchmark 10, max 35, unit "correct letter sounds"
- Letter naming fluency: target 42, BOY benchmark 17, max 55, unit "letters per minute"

**1st Grade** — 4 metrics, 2 key (WCPM, HFW), 2 supplementary (NWF, LNF)
- Words correct per minute: target 42, BOY benchmark 17, max 58, unit "WCPM"
- High-frequency words: target 200, BOY benchmark 80, max 220, cap 200, unit "high-frequency words"
- Nonsense word fluency: target 58, BOY benchmark 27, max 75 (most 1st graders enter already mastered)
- Letter naming fluency: target 42, BOY benchmark 17, max 55 (most 1st graders enter already mastered)

**2nd Grade** — 4 metrics, 2 key (WCPM, HFW), 2 supplementary (NWF, LNF)
- Words correct per minute: target 90, BOY benchmark 40, max 110, unit "WCPM"
- High-frequency words: target 200, BOY benchmark 120, max 220, cap 200, unit "high-frequency words"
- Nonsense word fluency: target 75, BOY benchmark 54, max 100 (essentially all 2nd graders mastered)
- Letter naming fluency: target 42, BOY benchmark 17, max 55 (essentially all 2nd graders mastered)

Nonsense Word Fluency targets and BOY benchmarks are Acadience-typical defaults; the data team will confirm the production source values.

Each metric has a `key` flag (1 = key metric used in aggregate calculations, 0 = supplementary). Metrics with `cap` cannot exceed that value (HFW caps at 200).

**Mastery.** If a student's BOY score for a metric is already at or above the EOY target, that student has "mastered" the metric during initial assessment. Mastered students keep their BOY score as their current value, are never re-tested for ongoing progress on that metric, and appear in the grade-view student table as a single italic "Mastered at initial assessment" row (with a small check icon) in place of their numeric data. They count as "on track" for that metric in the projected-to-meet-target percentage but contribute zero growth to the average-growth calculation. Mastered rows always group at the bottom of the table regardless of sort column.

Each grade also has `plural` labels: "Kindergartners", "1st Graders", "2nd Graders".

### District metric mapping (`DMET`)

`DMET` maps each grade to one representative metric (WCPM for every grade, since WCPM is the metric of truth). It is used by the school/district "projected to meet target" count so each student is counted once against a single metric rather than once per key metric. The card-level charts aggregate WCPM, HFW, and months of growth directly (see Aggregate metric charts).

### School configuration (`SCFG`)

Six schools, each with a unique seed, variable student counts per grade (some grades have 0 students, meaning that school doesn't serve that grade), and an open seats count. Not every school has every grade.

### Teacher assignment

Teachers are generated per grade per school from a pool of 18 names. One teacher per ~20 students (minimum 1). Students are round-robin assigned to teachers. Teacher name is stored on each student object.

### Student data generation

For each student, for each metric:
- BOY is drawn from a grade-specific range (`BOY_R`).
- If BOY is already at or above the EOY target, the student is marked **mastered** for that metric: current equals BOY and no further growth is generated.
- Otherwise ~68% of students are generated "on track" (current score between a threshold and 85% of target) and ~32% "below" (current just above BOY). Current is always at least BOY + 1.

Student names use full first + last name pairs drawn from `FN` and `LN` pools (e.g., "Alex Nguyen"), not initials.

Mastery falls out of the BOY ranges: WCPM and HFW BOY ranges sit well below their targets (no mastery), while NWF and LNF BOY ranges for 1st and 2nd grade sit at or above target for most students, so most upper-grade students are mastered on the foundational metrics.

### Projection formula

Linear projection from BOY to EOY:

```
projected = BOY + ((current - BOY) / monthsElapsed) * totalMonths
```

Currently configured for December (month 4 of a 10-month school year): `MO=4, MT=10`.

If a metric has a cap, the projection is clamped to that cap.

### Status determination

Binary. No intermediate/approaching state. Status drives both color **and** a non-color icon (check or warning), so the cue does not rely on color alone (WCAG 1.4.1).

- **On track** (bar `--ok-bar` #28d7a3, edge/text `--ok-bar-edge`/`--ok-fg` #188161): projected EOY ≥ target. Check icon.
- **Below target** (bar `--bad-bar` #ff3f6d, edge/text `--bad-bar-edge`/`--bad-fg` #cc3357): projected EOY < target. Warning-triangle icon.

Dark mode tokens shift the text/edge colors lighter so they retain AA contrast on the dark surface; bar fills stay the same hue.

Status applies consistently to the bar fill, the projected-growth slash pattern, the journey-pill stroke on aggregate charts, and the Projected-EOY status icon in the student table.

Mastered students are treated as "on track" for that metric (they began at or above target).

## Visual design

### Design system (IRDL tokens)

The dashboard is restyled to the Ignite brand system. All colors, radii, and shadows are CSS custom properties so the prototype tracks the design system as it evolves.

**Brand**: `--brand-500` `#573988` (primary purple), `--brand-600` `#462E6D` (hover), `--brand-300` `#B49DD3`, `--brand-100` `#F7F4FA`, `--brand-800` `#27004B`.

**Surfaces**: `--bg-page` (white), `--bg-surface` (white), `--bg-sunken` `#f5f5f5` (table headers, chart tracks, stats blocks), `--bg-hover` `#fafafa` (row + nav hover), `--bg-info` `#F7F4FA`. Dark-mode variants use a neutral dark scale (no tan tint).

**Text**: `--text-primary` `#1a1a1a`, `--text-body` `#3d3c3c`, `--text-secondary` `#5e5e5e` (AA on the sunken surface), `--text-tertiary` `#999999` (decorative only, fails AA as text), `--text-brand`.

**Borders**: strong, regular, soft, input, brand variants.

**Status**: `--ok-bar` / `--ok-bar-edge` / `--ok-fg` / `--ok-bg` and the matching `--bad-*` tokens. Warning is reserved as `--warn-*` for future use.

**Radius**: `--r-sm` 4px, `--r-md` 8px, `--r-lg` 12px, `--r-pill` 9999px.

**Shadows**: `--shadow-card` (subtle resting), `--shadow-elev` (hover).

Dark mode uses `prefers-color-scheme`; no toggle. The dashboard re-renders on color-scheme change so bar fills (read from CSS vars at render time) track the theme.

### Typography

Roboto, loaded from Google Fonts (`weight: 400, 500, 600, 700`). System font stack as fallback. The design system uses the full 400/500/600/700 range — primary text is 400 body, 500 medium for labels, 600 for headings and emphasized values, 700 for the largest stat values and the in-callout pills.

## App shell

The page is a flex shell: a persistent left sidebar plus a main column.

**Sidebar** (256px wide on desktop):
- Ignite logo (image, from the hosted brand URL)
- District/school name banner (purple band)
- `Reports` section with **Home**, **Student Progress** (active), **Session Attendance**, **Roster**
- `Help Center`
- User footer at the bottom: 32px circular avatar (initials), user name + role, sign-out button

On desktop the sidebar is sticky at `top: 0` for the full viewport height. Below 1024px it collapses to a 72px icon-only rail (labels, school banner, and user meta hide). Below 768px it hides entirely.

**Main column**: the page content (`#app`) plus a site-footer with the data-freshness note.

## Sticky behaviors

Three stacked sticky bands keep navigational and context controls visible while reading long content.

1. **Page-header band** (`.page-header-sticky`) — pins to the top of the viewport. Contains the eyebrow, back link (where applicable), title, subtitle, sub-counts, and the open-seats CTA. It breaks out to the full content width via negative horizontal margins so its background fills the column edge to edge.
2. **Grade-view toolbar** (`.grade-toolbar`, grade view only) — pins directly under the page-header band. Contains the metric toggle (pill buttons or dropdown) and the chart legend.
3. **Student-table header** (`.sticky-thead`, grade view only) — pins directly under the toolbar band. Because the table lives inside a horizontal-scroll container (which creates its own sticky context), the real `<thead>` cannot pin to the viewport. The fix is a hand-built duplicate header rendered as a sibling of `.table-wrap`, overlapped onto the real `<thead>` at `scroll=0` via a negative `margin-bottom` sized to the real header's height. Horizontal scrolling of the table translates the sticky header's inner table by the same `scrollLeft` so columns stay aligned.

A small JS helper (`applyStickyOffsets`) measures the heights of the first two bands and exposes them as CSS variables (`--page-header-h`, `--toolbar-h`) so the lower bands stack without hardcoded pixel math. It re-runs on window resize.

## Components

**View title** — 24px, weight 600, primary text. The page-header band shows: an uppercase eyebrow (`Reports · Student Progress`), the back link (school + grade views), the title in an `<h1>`, a small secondary-text subtitle (e.g., "School-level reading progress" or "K–2 reading progress"), and on the right the open-seats block.

**Open-seats block** — top-right of the view-head: when seats are open, "N open seats" plus a brand-purple "Fill Seats" primary button (the only brand-primary CTA on the page). When all seats are filled, "All seats filled" in secondary text. Wraps below the title on narrow viewports.

**Counts subtitle bar** — 14px secondary text, dot separators, light rules above and below. Numbers are bolded primary text so the values pop. Content varies by view level:
- District: total students, per-grade student counts
- School: total students, per-grade student counts
- Grade: student count with grade plural label

**Growth statement callout** — hero treatment, sits between the sub-counts bar and the metric cards on every view. White card with a 2px brand-purple border and a subtle brand shadow. A 44px brand-purple circle on the left holds a trending-up icon. The body has an uppercase brand eyebrow ("GROWTH HIGHLIGHT") above a 20px primary-text sentence. Key growth figures (months of progress, per-metric deltas) sit inline as brand-purple `<b>` pills with a 1.5px brand border and small shadow. The personalized sentence adapts to the scope:
- District and school views: aggregated across all students (K, 1st, 2nd), citing words correct per minute and high-frequency words.
- Grade view: scoped to that grade, same WCPM/HFW phrasing across all grades including kindergarten.

The months-of-progress figure uses a proxy formula until a real definition lands: per student, `(current − BOY) / (target − BOY) × total_school_months`, capped at total months, then averaged across all students and metrics in scope.

**Summary metric cards** (`.metrics-row`, `.mc`) — auto-fit grid of three cards (`minmax(220px, 1fr)`). White surface, soft border, card shadow, 16-20px padding. Each card shows a 14px secondary-text label (with a `min-height: 2.7em` reservation so 1- and 2-line labels align across the three cards), a 32px weight-700 value, a 12px "by year-end" subtitle, and a green trending-up icon plus "+N since last week" trend line in `--ok-fg`. Every view shows the same three cards: **Projected Additional Months of Growth · Projected Words Correct per Minute · Projected High-Frequency Words**. Scope is aggregate across all grades at the district and school levels, and grade-specific at the grade view. The "since last week" trend is a deterministic placeholder derived from the value (the prototype has no real time-series history).

**School / grade cards** (`.gc`) — white surface, soft border, card shadow, hover state (border shifts to brand-soft, shadow elevates). `role="button"` with `tabindex="0"` and Enter/Space keyboard handlers. Header has the unit name + count on the left and a chevron-right icon on the right. On district view, the count line shows total students plus a per-grade breakdown ("`35 students · K: 8 · 1st: 20 · 2nd: 7`"), stacked under the school name, with bold numbers.

**Aggregate metric charts** (on district school cards and school grade cards) — three charts per card in a responsive grid (3-across desktop, 2-across at ≤1024px, 1-across at ≤640px). Each card shows the same three charts in the same order: Additional Months of Growth, Words Correct per Minute, High-Frequency Words.

Each chart shares a common shape: a 14px primary-text label preceded by a status mark icon (check for on-track, warning for below target — non-color status cue) above a 20px tall bar with rounded ends. The WCPM and HFW charts contain a status-colored solid fill from average BOY to average current, a diagonal slash pattern from average current to average projected, a vertical target tick at the target position, and pill labels overlaying the bar at the BOY / Current / Projected positions. (The start is marked by its pill and the rounded left edge of the fill; there is no separate start dot on the aggregate charts — the student-table bars keep their start dot since they have no pills.) The journey pills (Start, Current, Projected) are small white chips with primary-text numbers and a 1px stroke matching the bar's status edge color. When they would collide horizontally, the lower-priority pill is hidden in this order: Start > Projected > Current. The target is a separate small pill sitting just below the bar, horizontally centered on the target tick (which extends down to meet it) and rendered in secondary text so it reads as a goal marker rather than a data point. Below the chart, a notes block (centered text on the sunken-color background, with padding) carries the stats on separate lines: "+N average growth" and "X% projected to meet target".

The Additional Months of Growth chart uses the same bar shape on a 0-to-15-months scale (room to show growth beyond a full school year). It fills from 0 to the average months achieved, then shows a diagonal projected-growth slash from current to the projected year-end months (current rate extrapolated: months ÷ months-elapsed × total-months). A target tick sits at 10 (a full year of growth). Pills mark current and projected. Bar color: green if projected ≥ a full year (10 months), red if behind. Reads "projected N months of growth by year-end" below.

The solid fill on every chart (aggregate and student-level) is rounded only on its left end; the right end is square so it reads as continuing into the projected-growth slash rather than terminating as a self-contained pill.

Aggregate math: WCPM and HFW averages use raw scores across all students in scope regardless of grade. The target tick uses the weighted-average target across the same students. Mastered students contribute their BOY/current as their values (no growth) and count as "projected to meet target".

**Chart legend** — small colored squares for On track / Below target, a line for EOY target, and the slash pattern swatch for Projected growth. Shared component, used on district, school, and grade views.

**Student table** (grade view only) — fixed table layout inside a rounded, shadowed `.table-wrap`. Uppercase 12px secondary header cells on the sunken-color background; hover state on body rows (`--bg-hover`). Minimum width 1000px (the wrap scrolls horizontally on narrower viewports). Columns:
- Student (name, weight 500, truncates with ellipsis)
- Teacher (secondary color, truncates with ellipsis)
- Start (BOY score, secondary color, right-aligned, sortable)
- Current (current score, primary color, right-aligned, sortable)
- Chart (inline bar chart from BOY to current, no header label, not sortable)
- Growth (signed, green positive / red negative)
- Projected EOY (status icon + value — non-color status cue)
- vs. Target (signed delta)

All four metrics now have targets, so the table is always 8 columns.

**Inline bar charts** (in student table) — 14px tall, 7px radius. Layers:
1. Track: sunken-color background spanning the full bar
2. Start marker: 12px white circle with a 2px status-colored stroke at the BOY (beginning-of-year) position, left edge aligned with the start of the colored fill
3. Current progress: solid fill in the status color, extending from BOY to current position (rounded left edge, square right edge)
4. Projected growth zone: diagonal slash pattern (45deg repeating gradient, 2px stripes at 6px intervals) in the status color, extending from current to projected position
5. Target marker: 2px wide, 20px tall tick mark at the target percentage, primary-text color

**Metric toggle buttons** (grade view) — horizontal row of pill buttons inside the sticky toolbar band. Inactive state is a neutral chip (border, body text); active state fills brand-purple with white text. All four metrics are available on every grade view, always in the same order: Words correct per minute, High-frequency words, Nonsense word fluency, Letter naming fluency. WCPM is selected by default for every grade. Below 768px viewport, the buttons collapse into a `<select>` dropdown with the same order. `role="tablist"`; `aria-selected` reflects the active state.

**Mastery row treatment** — when a student is mastered on the currently-viewed metric, the row collapses the numeric and chart columns into a single italic "Mastered at initial assessment" cell preceded by a small green check icon. Student name and teacher columns remain. Mastered rows always group together at the bottom of the table regardless of which column is sorted, so working students cluster at the top where attention should go.

**Data-freshness footer** — a centered note at the bottom of the page in the site-footer (under the main column, separated by a light top rule): "Data updates at the top of every hour · Last updated [time] · Refresh to see new data." The time is computed client-side to the top of the current hour.

### Icons

Inline SVG icons (Heroicons-style outline strokes). No icon font / CDN. Used for: back arrow, chevron-right (card affordance), check (on-track status, mastery), warning triangle (below-target status), arrow-down-on-square (sign out), trending-up (callout and trend lines), and the sidebar nav glyphs.

## Accessibility

- **Skip link** at the top jumps focus to `#main-content`.
- **Focus-visible** outlines on every interactive element using brand color.
- **Reduced motion**: transitions collapse to near-zero when `prefers-reduced-motion: reduce` is set.
- **Semantic structure**: each view uses `<h1>` for the title; `<nav>`, `<main>`, `<aside>`, `<footer>` landmarks.
- **ARIA**: `role="region"` on the callout with `aria-label`; `role="tablist"` and `aria-selected` on the metric toggle; `aria-current="page"` on the active sidebar item; `aria-hidden="true"` on decorative SVGs.
- **Non-color status cues**: bar status is always paired with a check or warning icon next to the chart title and on the Projected-EOY value in the student table (WCAG 1.4.1).
- **Keyboard**: school/grade cards are `role="button"` with `tabindex="0"` and Enter/Space handlers. Table columns are sortable via the header buttons.
- **Color contrast**: text and status colors selected to pass WCAG 2.1 AA on both light and dark sunken surfaces. The `--text-tertiary` token is reserved for decorative use (icons/dividers) and intentionally fails AA — never used for text.

## Responsive behavior

Layout adapts at three breakpoints:

**1024px and below** — sidebar collapses to a 72px icon-only rail; the aggregate-chart grid on district/school cards drops from three across to two across.

**768px and below** — sidebar hides entirely (the main column takes the full width); metric toggle buttons collapse into a `<select>` dropdown; the callout tightens its padding and the hero text drops to 16px.

**640px and below** — content padding tightens further; aggregate-chart grid drops to a single column; summary-metric cards stack to a single column.

The student table always lives inside a horizontal-scroll wrapper (1000px min-width) so the row's columns stay aligned. The sticky table-header mechanism translates with the table's `scrollLeft` so the column headers stay aligned during horizontal scroll.

## Interaction

- Clicking a school card navigates to that school's view (Enter/Space too).
- Clicking a grade card navigates to that grade's student table (Enter/Space too).
- Back button navigates up one level.
- Metric toggle buttons / dropdown switch which metric the student table displays (re-renders the grade view).
- Column headers are sortable (click to sort, click again to reverse). Sort state resets to name ascending when switching metrics or navigating. Mastered students always group at the bottom regardless of sort column.
- The "Fill Seats" CTA in the header is the only brand-primary CTA on the page. Destination is `[TBD]` for the prototype.
- No filtering.

## Key design decisions

1. **Binary status only.** On track or below target. No yellow / approaching state. Simpler for administrators to act on.
2. **Status based on projected EOY, not current score.** A student could have a low current score but strong growth trajectory and still be "on track." This rewards growth.
3. **Status is never carried by color alone.** A check or warning icon accompanies every status indicator (chart title, projected-EOY column) to satisfy WCAG 1.4.1.
4. **Three aggregate charts per card.** District and school cards show months of growth, WCPM, and HFW side by side (3-across desktop, 2-across tablet, 1-across mobile), each aggregating real scores across all students in scope.
5. **WCPM is the metric of truth.** Every grade exposes the same four metrics with WCPM as the default and the headline; NWF and LNF are supplementary and mostly resolve to mastery for upper grades.
6. **Mastery instead of emerging metrics.** All four metrics have targets. Students who began at or above a target are "mastered" and shown as a single row rather than a score.
7. **Sticky context bands.** The page header, grade-view toolbar, and table thead stack as sticky layers so the user keeps title, scope, controls, and column labels in view while scanning long content.
8. **Schools sort alphabetically.** District view orders schools A-Z while preserving each school's original index (so per-school seeds and deep links stay stable).
9. **Single-page app shell.** Persistent left sidebar plus a flex main column, no routing. Drill-down replaces `#app` contents in place.
10. **Demo data is seeded.** Same data every page load. Change a school's seed in `SCFG` to get different students.

## File structure

A single `index.html` file. Inline `<style>` for all CSS (with IRDL tokens), inline `<script>` for the data engine and rendering. External requests: Google Fonts (Roboto) and the hosted Ignite logo image. No build step.

## Getting started

The prototype lives at [github.com/Ignite-Reading/champion-facing-dashboard](https://github.com/Ignite-Reading/champion-facing-dashboard) and is deployed via GitHub Pages at [ignite-reading.github.io/champion-facing-dashboard](https://ignite-reading.github.io/champion-facing-dashboard/). Open `index.html` directly in a browser or visit the Pages URL. No build step required.

## What's next (not yet built)

These are potential next steps, not commitments. Discuss with Chris before implementing.

- Convert from static HTML prototype to a production framework (React, etc.)
- Connect to real student data via API; the months-of-growth formula and data lineage are owned by the data team
- Replace the placeholder week-over-week trend with a real time-series source
- Wire the "Fill Seats" CTA to its destination
- Wire the sidebar nav items (Home, Session Attendance, Roster, Help Center) to their respective destinations once they exist
- Replace the placeholder user (`Paige Chou — District Leader`) with the real session user
- Add time period selector (view progress at different points in the year)
- Add export / print functionality
- Full accessibility audit (keyboard nav across the table, screen-reader walkthrough, contrast spot-check across all theme states)
