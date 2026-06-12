---
name: dataviz
description: Turn data into one honest, clear chart that makes a single point — and judge whether a chart is any good. Use when building a chart, critiquing an existing one, visualizing data, or presenting analytics to stakeholders.
---

# Data Visualization

Every chart should make **one honest point clearly**. The chart follows the message — not the other way around. Picking the right chart is only the start; most charts that fail are technically correct but say nothing. Work the steps in order; don't return a chart until it passes the rubric.

## Workflow

### 1. Find the message
State the single point in one sentence. If you can't, you're not ready to chart — and the answer may be a table, not a chart. Everything downstream hangs on this sentence.

### 2. Pick the chart
Decide what relationship in the data carries the message, then pick from its row. You choose a *relationship*, not a chart.

| Relationship | When the point is… | Use | Avoid |
| --- | --- | --- | --- |
| **Change over time** | a trend across time | line; area; small multiples | bars for long series |
| **Comparison / magnitude** | how categories rank or differ in size | sorted horizontal bar; lollipop | pie; 3D |
| **Ranking** | the order matters more than the values | sorted bar; dot-strip; slope | alphabetical sort |
| **Part-to-whole** | components of one total | stacked bar; treemap; waffle; pie (≤4 slices) | pie >4 slices; comparing pies |
| **Deviation** | +/− vs a reference (target, zero, average) | diverging bar; line vs. reference line | hiding the reference |
| **Distribution** | spread, shape, outliers | histogram; box; violin; beeswarm | bar of raw values |
| **Correlation** | how two variables move together | scatter; bubble (3rd var = size) | dual-axis tricks |
| **Spatial** | location/geography *is* the point | choropleth; proportional-symbol map | a map when geography is incidental |
| **Flow** | movement/volume between states | Sankey; chord; waterfall | — |

## What makes the chart useful

The chart type is the easy part. These three steps are what separate a chart that communicates from a dashboard tile that doesn't.

### 3. Encode it accurately
Read off the most accurate channel first — **position > length > angle > area > color** (Cleveland & McGill). Put the key comparison in the highest channel available, then keep it honest:

- **Bars must start at zero** (length is read from the baseline). **Lines may use a non-zero axis** (position) — use it to reveal real day-to-day variation honestly.
- **Time on the x-axis → line, not bars.** Bars are for categorical comparison.
- **Pie only** for part-to-whole, ≤4 slices, **never to compare**. A sorted bar almost always beats it.
- **Sort bars by value** unless the category order is inherent (age bands, dates).
- A **distribution** question needs a distribution chart — never a bar of raw observations.
- **Lie Factor ≈ 1:** visual change proportional to data change. No truncated bars, no extra dimensions, no 3D.
- **Stay credible:** show the full relevant range (no cherry-picked window), include the baseline/sample size, never use a dual axis to imply correlation, drop spurious precision (decimals on noisy data), and label the source.

### 4. Focus attention
A useful chart controls what the eye does first.

- **Maximize data-ink:** delete gridlines, borders, drop shadows, and any legend you can replace with a direct label. Nothing should remain that you could remove without losing information.
- **One focal point:** gray everything, then color the *single* mark that is the message. Color is preattentive — used everywhere it says nothing; reserved for one mark it points the eye exactly where you want it.
- **Keep it accessible:** never rely on color alone (pair it with a label, position, or shape), use a colorblind-safe palette, and keep text–background contrast high. ~8% of men can't separate red from green.
- **Use grouping, not chrome:** direct-label lines at their ends instead of a detached legend; separate groups with whitespace rather than borders (Gestalt).
- **Add a reference line** when the claim is "vs. target / average / zero," and **many series → small multiples** beat one crowded chart.

### 5. Make it land
- **Title = the takeaway as a sentence** ("Churn doubled after the May price change"), not the topic ("Monthly churn"). Include units and source.
- **Be explanatory, not exploratory:** you're making one point, not exposing all the data. Cut everything that doesn't serve it.
- **Give it a shape:** lead with the "so what," show the tension (vs. baseline / target / expectation), end with the implication or next step.
- **Match the audience:** their vocabulary, no unexplained jargon. Audience trumps every other rule.

### 6. Run the rubric
Don't ship until every check passes. To **critique** an existing chart, run the same rubric and report which checks fail + the fix.

## Rubric

1. **Message** — can I state the one point in a sentence? If not, stop.
2. **Right form** — does the chart match the relationship? (Or is it really a table?)
3. **Credible** — Lie Factor ≈ 1; bars start at zero; full range, baseline/sample shown; source labeled.
4. **Accurate channel** — the key comparison is on position/length, not angle/area/color.
5. **Clear focus** — exactly one thing draws the eye, and it's the point; nothing left to remove.
6. **Takeaway title** — the title states the conclusion, with units and source.
7. **Audience** — explanatory, fits who's reading, no unexplained jargon.
