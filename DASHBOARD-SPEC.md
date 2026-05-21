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

Three grades, each with their own metrics:

**Kindergarten** — 4 metrics, 2 key (PA, LNF), 2 emerging (WCPM, HFW)
- Phonological awareness: target 35, BOY benchmark 14, max 45, unit "correct sound segments"
- Letter naming fluency: target 42, BOY benchmark 17, max 55, unit "letters per minute"
- Words correct per minute: no target, max 15, unit "WCPM" (emerging)
- High-frequency words: no target, max 25, unit "high-frequency words" (emerging)

**1st Grade** — 2 metrics, both key
- Words correct per minute: target 42, BOY benchmark 17, max 58, unit "WCPM"
- High-frequency words: target 200, BOY benchmark 80, max 220, cap 200, unit "high-frequency words"

**2nd Grade** — 2 metrics, both key
- Words correct per minute: target 90, BOY benchmark 40, max 110, unit "WCPM"
- High-frequency words: target 200, BOY benchmark 120, max 220, cap 200, unit "high-frequency words"

Each metric has a `key` flag (1 = key metric used in aggregate calculations, 0 = emerging/supplementary). Metrics with `cap` cannot exceed that value (HFW caps at 200).

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

System font stack. Two weights: 400 (regular) and 500 (medium, used for headings, names, values). No bold (600/700).

### Components

**Growth statement callout** — sits between the sub-counts bar and the metric cards on every view. Sits in a secondary-background block with generous padding. The personalized sentence is rendered at 18px in primary text so the headline stat reads as a hero. The national benchmark sentence (13px secondary text) appears **only on the district view**; the school and grade views drop it because the audience there is already inside their own scope. Key figures in both sentences are bolded (weight 500). The personalized sentence adapts to the scope:
- District and school views: aggregated across 1st and 2nd graders only, citing words correct per minute and high-frequency words.
- Grade view (K): scoped to kindergartners, citing phonological awareness (letter sounds) and letter naming fluency.
- Grade view (1st / 2nd): scoped to that grade, same metrics as the district sentence.

The months-of-progress figure uses a proxy formula until a real definition lands: per student, `(current − BOY) / (target − BOY) × total_school_months`, capped at total months, then averaged across all students and metrics in scope. The benchmark percentage and benchmark months are hard-coded placeholders.

**View title** — 18px, weight 500. District view shows district name, school view shows school name, grade view shows "[School Name] — [Grade Name]".

**Counts subtitle bar** — 13px secondary text, light rules above and below, dot separators between items. Content varies by view level:
- District: total students, school count, per-grade student counts
- School: total students, per-grade student counts
- Grade: student count with grade plural label

**Summary metric cards** — 3-column grid. Background secondary, rounded corners. 12px label, 22px value, 12px subtitle. Content varies by view, but the first card on every view is "Avg months of growth" with the value computed via the months-of-progress proxy formula.

**School/grade cards** — white background, 0.5px tertiary border, 12px border radius. Clickable with hover state (border darkens). Header has name + student count on left, chevron-right icon on right.

**Metric summary bars** (on cards) — 2-column grid layout. Each metric shows: 12px label, 8px tall bar with status-colored fill from average BOY to average current position, a 10px white start dot with a 2px status-colored stroke at the average BOY position, a 2px tall benchmark tick mark at the target position, a start → current → target value triplet (BOY in tertiary, current in primary weight 500, target in tertiary, light arrow separators), and two stats lines (average growth, % projected to meet target).

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

**Metric toggle buttons** (grade view) — horizontal row of pill buttons. Active state uses info color scheme. All metrics for that grade are available (not just key metrics). No asterisks on button labels.

**Emerging metric note** — when viewing a non-key metric, italic text appears: "Emerging measure for [Grade]. No formal EOY target." Legend and table columns for projected/delta are hidden.

**Legend** — small colored squares for On track/Below target, a line for EOY target, and the slash pattern swatch for Projected growth. Only target-related items show when viewing a metric with a target.

### Icons

Tabler Icons webfont (outline style only). Used for: back arrow (`ti-arrow-left`), card chevron (`ti-chevron-right`).

### Responsive behavior

Below 640px viewport width, the layout adapts:
- Summary metric cards stack into a single column (instead of the 3-column grid).
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
