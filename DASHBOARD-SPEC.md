# Student Reading Progress Dashboard — Spec

Reference implementation: `index.html` (single self-contained HTML file, all CSS and JS inline).

This is a prototype dashboard for school administrators to track K-2 student reading progress across a district. It uses procedurally generated demo data. The goal is to validate the information architecture, visual design, and interaction model before building a production version.

## Context

Ignite Reading provides 1:1 virtual reading tutoring for K-2 students. This dashboard shows administrators how their students are progressing through the school year. The audience is district administrators, school principals, and literacy coordinators who need to quickly understand which students are on track and which need attention.

## Navigation hierarchy

Three levels with full drill-down. Each level is a single-page view swap (no routing, no URL changes).

1. **District view** — cards for each school, each showing grade-level summary bars
2. **School view** — cards for each grade at that school, each showing key metric summary bars
3. **Grade view** — student-level table with sortable columns, metric toggle buttons, and per-student bar charts

Back navigation uses a text button with a left arrow icon. The back button label is the name of the parent (school view back button says the district name, grade view back button says the school name).

## Data model

All data is generated client-side using a seeded PRNG (mulberry32-style) so results are deterministic across page loads. No external data source.

### Grade configuration (`GM` object)

All three grades have the same four metrics in the same order so the dashboard reads consistently as students progress. WCPM is the metric of truth across grades; PA and LNF stay available as foundational checks for students who haven't yet mastered them.

**Kindergarten** — 4 metrics, 2 key (WCPM, HFW), 2 supplementary (PA, LNF)
- Words correct per minute: target 10, BOY benchmark 2, max 15, unit "WCPM"
- High-frequency words: target 20, BOY benchmark 5, max 25, unit "high-frequency words"
- Phonological awareness: target 35, BOY benchmark 14, max 45, unit "correct sound segments"
- Letter naming fluency: target 42, BOY benchmark 17, max 55, unit "letters per minute"

**1st Grade** — 4 metrics, 2 key (WCPM, HFW), 2 supplementary (PA, LNF)
- Words correct per minute: target 42, BOY benchmark 17, max 58, unit "WCPM"
- High-frequency words: target 200, BOY benchmark 80, max 220, cap 200, unit "high-frequency words"
- Phonological awareness: target 35, BOY benchmark 14, max 45 (most 1st graders enter already mastered)
- Letter naming fluency: target 42, BOY benchmark 17, max 55 (most 1st graders enter already mastered)

**2nd Grade** — 4 metrics, 2 key (WCPM, HFW), 2 supplementary (PA, LNF)
- Words correct per minute: target 90, BOY benchmark 40, max 110, unit "WCPM"
- High-frequency words: target 200, BOY benchmark 120, max 220, cap 200, unit "high-frequency words"
- Phonological awareness: target 35, BOY benchmark 14, max 45 (essentially all 2nd graders mastered)
- Letter naming fluency: target 42, BOY benchmark 17, max 55 (essentially all 2nd graders mastered)

Each metric has a `key` flag (1 = key metric used in aggregate calculations, 0 = supplementary). Metrics with `cap` cannot exceed that value (HFW caps at 200).

**Mastery.** If a student's BOY score for a metric is already at or above the EOY target, that student has "mastered" the metric during initial assessment. Mastered students keep their BOY score as their current value, are never re-tested for ongoing progress on that metric, and appear in the grade-view student table as a single "Mastered at initial assessment" row in place of their numeric data. They count as "on track" for that metric in the projected-to-meet-target percentage, but contribute zero growth to the average-growth calculation.

Each grade also has `plural` labels: "Kindergartners", "1st Graders", "2nd Graders".

### District metric mapping (`DMET`)

At district and school card levels, one representative metric is shown per grade: PA for Kindergarten, WCPM for 1st and 2nd grade.

### School configuration (`SCFG`)

Six schools, each with a unique seed, variable student counts per grade (some grades have 0 students, meaning that school doesn't serve that grade), and an open seats count. Not every school has every grade.

### Teacher assignment

Teachers are generated per grade per school from a pool of 18 names. One teacher per ~20 students (minimum 1). Students are round-robin assigned to teachers. Teacher name is stored on each student object.

### Student data generation

For each student, for each metric:
- If the metric has no target (emerging): BOY is 0, current is random 0 to max.
- If the metric has a target: BOY is drawn from a grade-specific range. ~68% of students are generated "on track" (current score between a threshold and 85% of target), ~32% are generated "below" (current just above BOY). Current is always at least BOY + 1.

### Projection formula

Linear projection from BOY to EOY:

```
projected = BOY + ((current - BOY) / monthsElapsed) * totalMonths
```

Currently configured for December (month 4 of a 10-month school year): `MO=4, MT=10`.

If a metric has a cap, the projection is clamped to that cap.

### Status determination

Binary. No intermediate/approaching state.

- **On track** (green `#639922`): projected EOY >= target
- **Below target** (red `#E24B4A`): projected EOY < target

Dark mode uses lighter variants: green `#97C459`, red `#F09595`.

Status applies consistently to bar fill color, projected growth slash pattern, and projected EOY dot in the student table. All use the same color for a given student.

Metrics without a target (emerging) default to "on track" status.

## Visual design

### Theming

CSS custom properties for all colors, with light and dark mode via `prefers-color-scheme` media query. No toggle, follows system preference.

Key colors:
- Page background: `--color-background-tertiary` (#fafaf8 light, #2c2c2a dark)
- Card background: `--color-background-primary` (#ffffff light, #2c2c2a dark)
- Metric card background: `--color-background-secondary` (#f5f4f0 light, #363633 dark)
- Active toggle button: info colors (blue tint background, blue text, blue border)

Max width: 780px, centered.

### Typography

System font stack. Two weights for the body of the design system: 400 (regular) and 500 (medium, used for headings, names, values). The growth-statement callout is a deliberate exception that uses 700 for the hero metrics — see the Components section.

### Components

**Growth statement callout** — sits between the sub-counts bar and the metric cards on every view. Sits in a secondary-background block with generous padding. A single personalized sentence rendered at 18px in primary text. Key growth figures inside the sentence (months of progress, per-metric deltas) sit inline at the same 18px size in weight 700, wrapped in a white pill with a 0.5px tertiary border and small horizontal padding — a quieter "highlight" treatment that emphasizes the numbers without changing the text size or introducing additional color. (Weight 700 is a deliberate exception to the "no 600/700 weight" rule in the Typography section.) The personalized sentence adapts to the scope:
- District and school views: aggregated across 1st and 2nd graders only, citing words correct per minute and high-frequency words.
- Grade view (K): scoped to kindergartners, citing phonological awareness (letter sounds) and letter naming fluency.
- Grade view (1st / 2nd): scoped to that grade, same metrics as the district sentence.

The months-of-progress figure uses a proxy formula until a real definition lands: per student, `(current − BOY) / (target − BOY) × total_school_months`, capped at total months, then averaged across all students and metrics in scope. The benchmark percentage and benchmark months are hard-coded placeholders.

**View title** — 18px, weight 500. District view shows district name, school view shows school name, grade view shows "[School Name] — [Grade Name]". On the district and school views the title sits in a flex header with an **open-seats block** pinned to the top right: when seats are open it shows "N open seats" plus a "Fill Seats" call-to-action button (info color scheme); when all seats are filled it shows "All seats filled" in subtle tertiary text. The block stacks below the title on viewports under 640px. Open seats is no longer a summary metric card — the top metric row on district and school views is now two cards (Avg months of growth, Projected to meet target).

**Counts subtitle bar** — 13px secondary text, light rules above and below, dot separators between items. Content varies by view level:
- District: total students, per-grade student counts
- School: total students, per-grade student counts
- Grade: student count with grade plural label

**Summary metric cards** — 3-column grid. Background secondary, rounded corners. 12px label, 22px value, 12px subtitle. Content varies by view, but the first card on every view is "Avg months of growth" with the value computed via the months-of-progress proxy formula.

**School/grade cards** — white background, 0.5px tertiary border, 12px border radius. Clickable with hover state (border darkens). Header has name + student count on left, chevron-right icon on right.

**Aggregate metric charts** (on district school cards and school grade cards) — three charts per card laid out in a responsive grid (3-across desktop, 2-across tablet, 1-across mobile). Each card shows the same three charts in the same order: Total months of growth, Words correct per minute, High-frequency words.

Each chart shares a common shape: a 13px primary-text label above a 28px tall bar with rounded ends. The WCPM and HFW charts contain a status-colored solid fill from average BOY to average current, a diagonal slash pattern from average current to average projected, a white start dot with status-colored stroke at the BOY position, a vertical target tick at the target position, and pill labels overlaying the bar at the BOY / Current / Projected positions. The journey pills (Start, Current, Projected) are small white chips with primary-text numbers and a 1px stroke matching the bar's status color (green on-track / red below), overlaying the bar. When they would collide horizontally, the lower-priority pill is hidden in this order: Start > Projected > Current. The target is a separate small pill sitting just below the bar, horizontally centered on the target tick (which extends down to meet it) and rendered in secondary text so it reads as a goal marker rather than a data point. Below the chart, a notes block (centered text on a secondary-color background, matching the bar track, with padding) carries the stats on separate lines: "+N average growth" and "X% projected to meet target".

The Months of growth chart uses the same bar shape on a 0-to-15-months scale (room to show growth beyond a full school year). It fills from 0 to the average months of growth achieved, then shows a diagonal projected-growth slash from current to the projected year-end months (current rate extrapolated: months ÷ months-elapsed × total-months). A target tick sits at 10 (a full year of growth). Pills mark current and projected. Bar color: green if projected ≥ a full year (10 months), red if behind. Reads "projected N months of growth by year-end" below.

The solid fill on every chart (aggregate and student-level) is rounded only on its left end; the right end is square so it reads as continuing into the projected-growth slash rather than terminating as a self-contained pill.

Aggregate math: WCPM and HFW averages use raw scores across all students in scope regardless of grade. The target tick uses the weighted-average target across the same students. Mastered students contribute their BOY/current as their values (no growth) and count as "projected to meet target".

**Student table** (grade view only) — fixed table layout. Sortable columns with arrow indicators. Columns:
- Student (name, weight 500, truncates with ellipsis)
- Teacher (secondary color, truncates with ellipsis)
- Start (BOY score, secondary color, right-aligned, sortable)
- Current (current score, primary color, right-aligned, sortable)
- Chart (inline bar chart from BOY to current, no header label, not sortable)
- Growth (signed, green positive / red negative)
- Projected EOY (colored dot + value, only if metric has target)
- vs. EOY target (signed delta, only if metric has target)

The numeric Current value sits to the right of Start so the two reference numbers are adjacent, and the chart sits to the right of Current. Column widths adjust based on whether the metric has a target (7 or 8 columns).

**Inline bar charts** (in student table) — 12px tall, rounded. Layers:
1. Track: light gray background spanning the full bar
2. Start marker: 12px white circle with a 2px status-colored stroke, positioned so its left edge sits at the BOY (beginning-of-year) position, overlaying the start of the colored fill
3. Current progress: solid fill in the status color, extending from BOY to current position
4. Projected growth zone: diagonal slash pattern (45deg repeating gradient, 2px stripes at 6px intervals) in the status color, extending from current to projected position
5. Target marker: 2px wide, 18px tall tick mark at the target percentage

**Metric toggle buttons** (grade view) — horizontal row of pill buttons. Active state uses info color scheme. All four metrics are available on every grade view, always in the same order: Words correct per minute, High-frequency words, Phonological awareness, Letter naming fluency. WCPM is selected by default for every grade. Below 640px viewport, the buttons collapse into a `<select>` dropdown with the same order. No asterisks on button labels.

**Mastery row treatment** — when a student is mastered on the currently-viewed metric, the row collapses the numeric and chart columns into a single italic "Mastered at initial assessment" cell. Student name and teacher columns remain. Mastered rows always group together at the bottom of the table regardless of which column is sorted, so working students (the ones who still need progress monitoring) cluster at the top where attention should go.

**Legend** — small colored squares for On track/Below target, a line for EOY target, and the slash pattern swatch for Projected growth. Only target-related items show when viewing a metric with a target.

### Icons

Tabler Icons webfont (outline style only). Used for: back arrow (`ti-arrow-left`), card chevron (`ti-chevron-right`).

### Responsive behavior

Below 767px viewport width:
- The chart column in the student-level table is hidden so the row only carries the numeric fields. The Current column is right-aligned to compensate.

Below 640px viewport width:
- Summary metric cards stack into a single column (instead of the 3-column grid).
- The per-grade/per-metric grid on district and school cards also stacks into a single column so the bars and triplets stay readable.
- Metric toggle buttons in the grade view become a `<select>` dropdown. Both elements are always rendered; CSS shows one or the other based on viewport.
- The student table wraps in a horizontal-scroll container with a 640px minimum width. The first column (Student) is sticky so the name stays visible while the rest of the row scrolls.
- Body padding tightens from 2rem to 1rem to claim more usable width.

## Interaction

- Clicking a school card navigates to that school's view
- Clicking a grade card navigates to that grade's student table
- Back button navigates up one level
- Metric toggle buttons switch which metric the student table displays (re-renders the full grade view)
- Column headers are sortable (click to sort, click again to reverse). Sort state resets to name ascending when switching metrics or navigating.
- No filtering. No dropdown selectors.

## Key design decisions

1. **Binary status only.** On track or below target. No yellow/approaching state. Simpler for administrators to act on.
2. **Status based on projected EOY, not current score.** A student could have a low current score but strong growth trajectory and still be "on track." This rewards growth.
3. **Consistent status coloring.** Bar, slash pattern, and projected dot all use the same color per student. No mixed signals.
4. **2-column metric grid on cards.** Third metric wraps to a new row at the same width rather than cramming 3 across.
5. **No filter dropdown on grade view.** The metric toggle buttons are sufficient. Keeps the interface simple.
6. **Teacher column in student table.** Students from multiple classes can appear in one grade view, so the teacher name disambiguates.
7. **Emerging metrics available but clearly secondary.** Toggle buttons show all metrics, but the note and missing target columns make it clear these are supplementary.
8. **Demo data is seeded.** Same data every page load. Change a school's seed in `SCFG` to get different students.

## File structure

Currently a single `index.html` file (the district view). Everything is inline: CSS in a `<style>` block, JS in a `<script>` block, no external dependencies except Tabler Icons CDN.

A separate `school-dashboard.html` exists from an earlier iteration (single-school view only). It is out of sync with the district dashboard and should not be used as a reference.

## Getting started

1. Create a new GitHub repo (e.g., `student-progress-dashboard`) under the Ignite Reading org or Chris's personal account.
2. Add `index.html` and this spec to the repo root.
3. The prototype is a single self-contained HTML file. Open it directly in a browser to see the current state. No build step required.

## What's next (not yet built)

These are potential next steps, not commitments. Discuss with Chris before implementing.

- Convert from static HTML prototype to a production framework (React, etc.)
- Connect to real student data via API
- Add time period selector (view progress at different points in the year)
- Add export/print functionality
- Add responsive design for tablet/mobile
- Consider whether school-level and grade-level views need different summary metrics
- Accessibility audit (keyboard navigation, screen reader support, ARIA labels)
