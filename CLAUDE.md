# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Entropy Lab": an interactive explainer for entropy, Information Gain and decision-tree splitting. The whole app is one self-contained file, `index.html` (named so GitHub Pages serves it at the site root) (inline CSS + one inline `<script>` IIFE, vanilla JS, no framework, no build step, no dependencies). `prompt.md` is empty. Do not turn this into a Claude Artifact or split it into a bundler project; the user explicitly wants a plain local HTML/JS page.

The repo has no package manager, tests, or linter. This folder is its own git repo (`origin` = github.com/ypanjwani/EntropyExplainer, branch `main`), published with GitHub Pages at https://ypanjwani.github.io/EntropyExplainer/. It sits inside the user's home directory, which is a separate, remote-less git repo: always run git from this folder and never push from the home directory. Only `index.html` and this file matter.

## Running

- Open `index.html` directly in a browser (`Start-Process` on Windows), or serve the folder: `python -m http.server 8000` from this directory, then http://localhost:8000/index.html.
- Only external request is the Google Fonts `<link>`; the page works offline with fallback fonts.
- There is nothing to run for verification except loading the page and exercising the interactions. Pure logic (e.g. `bestSplit`, entropy math) can be sanity-checked by extracting functions from the file into a `node -e` script.

## Architecture

The page is an unnumbered "What is entropy?" intro card (`.section-what`; plain-language explanation plus three example mixes drawn by `buildEntropyExamples()` using `h(p)`) followed by three numbered sections (`.card`s), all joined by `.flow-connector` dividers, and one script that wires them up. Order inside the script matters because later sections reuse earlier helpers via shared closure scope:

1. **Section 1, "How impure is a set?"** (`// Section 1` in the script)
   - `h(p)` is the binary entropy function used by everything else.
   - The `p(Class A)` slider (step 0.02, N=50 points, so p = k/50 exactly) drives `renderSlider()`, which updates the entropy curve dot, the formula readout, the naive-`p` comparison overlay (toggle), and `renderScatter(p)`.
   - The scatter uses fixed seeded points (`mulberry32`, min-distance rejection sampling). Class A is assigned to the top-`k` points by a latent score (`0.7x + 0.3y + noise`), so changing `p` grows/shrinks a region rather than reshuffling. `bestSplit(k)` scans all axis-aligned thresholds on x and y for max Information Gain and draws the single dashed `#splitLine`. The noise factor (currently `0.14`) controls how separable the classes are.
2. **Section 2, dataset + split scoring** (`DATASETS`, `setDataset`)
   - `DATASETS` holds two selectable datasets, the single source of truth for Sections 2 and 3: `categorical` (Quinlan's 14-row Play Tennis, `values` per feature) and `numerical` (a made-up 18-row Loan Approval set with exactly two numeric columns, Income and Credit, so it fits a 2D plot; it carries `plot2d: true` and per-feature `axis` {min,max,ticks}). A segmented control (`#datasetSwitch`) calls `setDataset(id)`, which reassigns the active-dataset vars (`DS`, `DATA`, `TARGET`, `NOUN`, `FEATURES`, `FEATURE_LABELS`, `VALUE_ORDER`, `IS_NUM`), rebuilds the table/picker/leaderboard/group cards, and resets the tree. Targets are always `'Yes'`/`'No'` (Yes = blue) whatever the dataset's column name.
   - `computeSplit(rows, feature)` returns a scored split object `{feature, kind:'cat'|'num', title, groups, entropyBefore, weightedH, gain}`. Categorical: one group per value. Numeric: `thresholdSplits` tries midpoints between neighbouring distinct values only where the class flips, and the best cut is returned with `threshold` and the full `thresholds` list (shown in `#scanBlock`); each cut also carries the neighbouring values it sits between (`lo`/`hi`, with `loClass`/`hiClass`), which drive the "why the midpoint" text in Section 2, the why-card and the step log; it returns `null` if no cut exists. `selectFeature(f)` re-renders the section and highlights the matching table column.
   - The raw-data table (`buildDataTable`) has sortable numeric headers (`.th-sort`; cycles ascending / descending / original order via `sortKey`/`sortDir`; the `#` column keeps each row's original number). Category columns are not sortable.
   - `drawPlot(svg, {regions, lines})` is the shared 2D renderer (points as blue circles / orange stars, tinted pure regions, cut lines). Section 2 uses it in `#scanPlot` for the selected column's best cut; Section 3 uses it in `#partitionSvg`. Keep a numeric dataset to two features if it should be plottable.
3. **Section 3, tree builder** (`// Section 3`)
   - ID3-style. `candidateSplits(rows, used)` scores every usable column on a node's rows (a category is used up once asked on a path; a numeric column can be asked again with a new cut). State is `treeNodes` (id to node), `queue` (FIFO of unsplit non-leaf nodes), `stepIndex`. Each click of "Build next split" (`buildNextSplit`) pops one node, picks the max-IG remaining feature, creates children (`makeNode` marks them leaf if pure / no candidate left / empty, and stores `node.candidates`), uses `split.title` (e.g. `Income ≤ 47.5`) for node/edge/step-log labels, appends a step-log entry, and calls `renderTree()`.
   - For `plot2d` datasets each node carries `bounds` (its rectangle in data units) plus `splitAxis`/`splitThreshold`, and `drawPartition()` (called from `renderTree`) redraws the partition (regions = childless nodes), the next-to-split outline (`queue[0]`), and the "weighted entropy of all regions" meter with per-step history (`stepH`). Clicking a plot region opens the same `#whyCard`; clicking a tree node outlines its region (`focusId` / `setPartitionFocus`).
   - `renderTree` rebuilds the DOM as nested `.tree-node-block` / `.children-row` flex containers, then `drawConnectors` (via `requestAnimationFrame`, and on window resize) measures real box positions with `getBoundingClientRect` and draws elbow connectors in the `#treeLines` SVG overlay plus HTML `.edge-label` chips. Do not replace this with CSS pseudo-element connectors.
   - Clicking any node opens one shared fixed-position `#whyCard` (`showWhy` / `whyHtml` / `hideWhy`) explaining the split, leaf, or pending state; it closes on outside click, Esc, scroll, or re-render. `TOTAL_STEPS` is computed by simulating the whole build (`countTotalSplits`), not hardcoded.

## Conventions worth knowing

- Theme: forced white (no `prefers-color-scheme` block). Colors are CSS tokens on `:root` (`--series-a` blue = Class A / "Yes" / circle, `--series-b` orange = Class B / "No" / star, `--inset` for tinted inner surfaces). Reuse these tokens; keep the A=blue/circle, B=orange/star mapping consistent across all sections.
- Charts follow a thin-mark, recessive-grid style; the same two-hue mapping is used in the leaderboard bars, dot clusters, tree leaves, and table Yes/No pills.
- Respect `prefers-reduced-motion` (a single block near the top of the CSS lists transitions to disable; add new animated selectors there).
- The tree diagram is `role="img"` with the step log as its text alternative; keep the step log in sync with any tree changes.
