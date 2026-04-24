# Ingredient Flavor Graph Visualization — Plan

## Overview
A simple static website inside `ingredient_chart_viz/` that renders ingredients as a directed graph. Nodes are ingredients (e.g. soy sauce, salt), edges are labeled flavor contributions (e.g. `salt → soy sauce` labeled `umami`) with arrows. When zoomed out, major flavor categories (umami, salty, sweet, sour, bitter, spicy, aromatic) are visible as groupings.

## Stack
- **Plain static site** — `index.html` + `style.css` + `app.js` (no build step).
- **[Cytoscape.js](https://js.cytoscape.org/)** for the graph — native support for labeled directed edges, zoom/pan, and compound (parent) nodes.
- Loaded via CDN, so no npm/bundler setup.

## File layout
```
ingredient_chart_viz/
├── index.html        # page shell + Cytoscape CDN
├── style.css         # page + graph container styles
├── app.js            # loads data, configures Cytoscape, style rules
└── data/
    └── ingredients.json   # nodes + edges + flavor groups
```

## Data model (`ingredients.json`)
Three kinds of entries:
1. **Flavor nodes** (compound parents): `umami`, `salty`, `sweet`, `sour`, `bitter`, `spicy`, `aromatic`. Act as containers.
2. **Ingredient nodes**: e.g. `soy sauce`, `salt`, `miso`, `lemon`, `sugar`. Each has a `parent` pointing to its dominant flavor group.
3. **Directed labeled edges**: e.g. `{ source: "salt", target: "soy sauce", label: "umami" }`.

Seed dataset: ~15–20 ingredients and ~20 edges so the graph is meaningful on first load.

## Visualization behavior
- **Two-level zoom switch** — the app maintains two graph views and swaps between them based on zoom level:
  - **Overview view (default, zoomed out):** a small graph of just the ~7 dominant flavor nodes (umami, salty, sweet, sour, bitter, spicy, aromatic), connected by aggregated edges (an edge between two flavors exists if any ingredient-level edge crosses between them, weighted by count). Big labels, thick edges, no ingredient detail.
  - **Detail view (zoomed in):** the full ingredient graph with directed labeled edges (e.g. `salt → soy sauce` labeled `umami`), ingredients grouped inside their flavor parents.
- **Transition logic:** a `zoom` event handler watches Cytoscape's zoom level against a threshold (e.g. `zoom < 0.6` = overview, `zoom >= 0.6` = detail). Crossing the threshold swaps the elements collection and re-runs layout, with a short fade so the switch feels continuous rather than jarring. Pan position is preserved across the swap so the user stays centered on the flavor they were exploring.
- **Entry point:** app boots in overview mode so the first thing the user sees is the major-flavors map; scrolling to zoom in reveals the ingredients.
- **Directed arrows** with edge labels along the edge in both views (`curve-style: bezier`, `target-arrow-shape: triangle`, `label: data(label)`).
- **Layout**: `cose` (force-directed) for the detail view so related ingredients cluster inside their flavor groups; a fixed circular/preset layout for the overview so the flavor map is stable across loads.
- **Interactions**: drag nodes, scroll to zoom, click a flavor node in overview to auto-zoom into the detail view focused on that cluster; click an ingredient in detail to highlight its neighborhood (dim the rest).

## Styling
- One color per flavor category, applied to parent node background and ingredient node border — instant visual read of flavor.
- Readable edge labels with subtle backgrounds so they don't blur into edges.

## How to run
Open `index.html` in a browser, or run `python3 -m http.server` from the folder if the JSON fetch needs HTTP.

## Implementation phases

Work ships in three phases. Each phase is independently runnable — don't start phase N+1 until phase N works end-to-end in the browser.

### Phase 1 — MVP: basic ingredient graph

Goal: a single-view directed graph of ingredients with labeled edges, rendered from JSON. No multi-level, no editor.

1. **Scaffold files.** Create `index.html` (loads Cytoscape from CDN, has one `<div id="cy">` container), `style.css` (fills viewport, gives `#cy` 100vh height), `app.js` (empty entry point), and `data/ingredients.json`.
2. **Seed data.** Write ~15 ingredients and ~20 edges into `ingredients.json` using the flat schema: `{ nodes: [{ id, label, flavor }], edges: [{ source, target, label }] }`. Pick one cuisine (e.g. East Asian pantry) so the graph tells a coherent story.
3. **Render baseline.** In `app.js`, `fetch('data/ingredients.json')`, instantiate Cytoscape with a `cose` layout, wire the nodes/edges into its format. Verify nodes appear.
4. **Style nodes and directed edges.** One color per flavor category (store the palette in a `FLAVOR_COLORS` constant), `target-arrow-shape: triangle`, `curve-style: bezier`, node `label: data(label)`, edge `label: data(label)`. Verify arrows point source → target.
5. **Polish interactions.** Scroll-zoom and pan enabled by default; add click-to-highlight-neighborhood (select a node, dim non-neighbors via a `.faded` class toggle).
6. **Legend.** Add a static HTML legend in a corner mapping color → flavor. Hardcoded, no interactivity.
7. **Run + verify.** `python3 -m http.server`, open in browser, confirm: labels readable, arrows correct, click-highlight works, no console errors.

**Exit criterion:** someone unfamiliar can open the page and understand "these ingredients are connected by these flavor relationships."

### Phase 2 — MVP: multi-level graph (overview ↔ detail)

Goal: add the zoom-driven switch between the dominant-flavors overview and the full ingredient detail view. Builds on Phase 1's file structure.

1. **Derive overview data.** Add a `buildOverview(data)` function in `app.js` that computes flavor→flavor aggregated edges from the ingredient edges. Returns `{ nodes: [{ id: flavor, count }], edges: [{ source, target, weight }] }`. Weight = number of ingredient edges crossing that flavor pair.
2. **Two element collections.** Build both `detailElements` and `overviewElements` up front. Keep them in closure, don't re-derive on every swap.
3. **View state machine.** Introduce `currentView: 'overview' | 'detail'` and a `setView(next)` function that calls `cy.elements().remove()` then `cy.add(nextElements)` then runs the view's layout. Overview uses `circle` layout (stable); detail uses `cose`.
4. **Zoom threshold with hysteresis.** Pick two thresholds: zoom-in trigger `> 0.8` switches overview→detail; zoom-out trigger `< 0.4` switches detail→overview. Wire to `cy.on('zoom', ...)`. The gap prevents flicker near the boundary.
5. **Boot into overview.** App loads in overview mode. Render `<h2>Hover a flavor, scroll in to see ingredients</h2>` as a one-time hint.
6. **Click-to-drill.** In overview, clicking a flavor node calls `setView('detail')` and programmatically pans/zooms to that flavor's cluster (`cy.fit(flavorCluster, 50)`). Faster than asking users to scroll.
7. **Transition polish.** Fade the container (`opacity: 0` → `1` over 200ms) during `setView` so the swap doesn't feel abrupt. Cache detail-view node positions after first layout so re-entering detail reuses them instead of re-running `cose`.
8. **Style overview edges.** Thicker edges sized by `weight`, no arrows in overview (aggregation makes direction meaningless — call this out in the legend).
9. **Run + verify.** Confirm: default view is overview; zooming in crosses threshold and swaps to detail; zooming out swaps back; no flicker when hovering near threshold; clicking a flavor drills in.

**Exit criterion:** someone unfamiliar boots the page, sees the 7 flavors, scrolls in, sees the ingredients, scrolls out, sees flavors again — with no jank.

### Phase 3 — UI to edit ingredients

Goal: in-browser editor for adding/editing/deleting ingredients and edges, with export back to `ingredients.json`. Graph stays read-only until the user opens the editor.

1. **Storage layer.** Wrap the in-memory dataset in a small `store.js` module with `getState()`, `addIngredient()`, `updateIngredient()`, `deleteIngredient()`, `addEdge()`, `deleteEdge()`. Persist to `localStorage` on every change; on boot, prefer `localStorage` over the shipped JSON if present. Add a "Reset to defaults" button that clears `localStorage`.
2. **Editor panel.** Add a slide-in side panel (`<aside>`, toggled by an "Edit" button in the corner). Panel has two tabs: **Ingredients** and **Edges**.
3. **Ingredients tab.** List all ingredients with inline edit (name, flavor category dropdown from the fixed 7). "+ Add ingredient" button opens a blank row. Delete button per row with a confirm.
4. **Edges tab.** List all edges as `source → target : label` rows. "+ Add edge" opens a form: source dropdown (all ingredients), target dropdown, label text field (free-form initially; consider a flavor dropdown later). Delete per row.
5. **Live re-render.** Every store mutation calls `rebuildGraph()` which recomputes both element collections and re-runs `setView(currentView)`. Keep the current zoom/pan so the user doesn't lose context mid-edit.
6. **Validation.** Reject edges where source === target, or where either endpoint doesn't exist, or duplicates of an existing (source, target, label) triple. Show inline error text, don't alert().
7. **Export / import.** "Download JSON" button in the editor serializes current state to a file download. "Load JSON" accepts a file upload. This is the durable save path; `localStorage` is just convenience.
8. **Run + verify.** Confirm: add an ingredient, it appears in the graph; add an edge, it appears with the right label and arrow; delete propagates; refresh persists; download + reload round-trips losslessly.

**Exit criterion:** a user can build a small graph from scratch without hand-editing JSON and walk away with a file they can commit.

## Out of scope (future)
- Search / filter across large graphs, multi-flavor ingredients (one ingredient contributing to multiple parents), collaborative editing, undo/redo in the editor, exporting the graph as an image, seed datasets beyond the initial cuisine.

---

## Pros & Cons

### Pros

**Overview-first UX matches the mental model**
Booting into the dominant-flavors map makes the "big picture" the default answer. Users who only want the gist get it in one glance; users who want detail earn it by zooming.

**Two explicit views beat zoom-hiding labels**
Swapping element collections at a threshold gives each view its own layout, styling, and label density — cleaner than trying to make one graph readable at every zoom level.

**Zero setup friction**
No npm, bundler, or build step. Open `index.html` and it works.

**Cytoscape.js fits the problem natively**
Labeled directed edges, compound parent nodes, and per-element styling are first-class; swapping collections and re-running layout is a supported pattern.

**Aggregated overview edges are computable from detail data**
The overview graph is derived (flavor→flavor edges summed from ingredient-level edges), so there's still one source of truth in JSON.

**Cheap to throw away**
If the split-view approach feels wrong, falling back to single-graph-with-zoom-dependent-labels is a small refactor.

### Cons

**View-switch threshold is a UX landmine**
A single zoom cutoff causes flicker if the user hovers near it. Needs hysteresis (different thresholds for zoom-in vs zoom-out) or a hard snap animation — either way, more code than a single graph.

**Two layouts to curate instead of one**
The overview needs a stable, hand-friendly layout (preset/circle); the detail view uses `cose`. That's two layout configs to tune and two sets of styles to keep visually consistent.

**Preserving pan position across the swap is fiddly**
Cytoscape's viewport math differs between element sets. Getting the user to stay "centered on umami" through the transition requires mapping coordinates between the two graphs, not just keeping `pan()` constant.

**Aggregated edges hide directionality**
Flavor→flavor edges summed from many ingredient edges may have mixed directions. Either pick a convention (net direction, or drop arrows in overview) or accept that overview arrows will sometimes mislead.

**"Dominant flavor" as a single parent is still lossy**
Soy sauce is umami *and* salty. Compound nodes allow only one parent per child. The overview aggregation inherits this — an ingredient only contributes to one flavor's node size.

**Cytoscape edge labels get crowded in detail view**
With ~20 edges fine; at 100+ they overlap. The split-view buys some relief (overview has few edges) but the detail view still needs label-collision handling eventually.

**Force-directed layouts are non-deterministic**
`cose` re-runs produce different layouts, disorienting on every view switch. Caching positions after first run (or pinning) mitigates this but adds state.

**CDN dependency, JSON-as-database ceiling carry over — but the editor softens the second one**
Offline still breaks without vendoring. The editor (Phase 3) removes the hand-edit-JSON ceiling for authoring, but `localStorage` isn't a real database — multi-device state and collaboration are still out.

**Editor + live re-render compounds the non-determinism problem**
Every mutation re-runs layout if positions aren't cached. Users who drag nodes into place will lose their layout when they add a new ingredient unless position-caching is carefully preserved through mutations — more state to manage than the read-only version had.

**Three-phase plan risks scope creep on Phase 3**
Phase 3 is larger than Phases 1–2 combined. Realistic risk: shipping a half-finished editor that's worse than no editor. Discipline: Phase 3 is not "start" until Phases 1 and 2 both pass their exit criteria and sit for a day without revisions.

### Bottom line
Split into three phases, each independently valuable: Phase 1 proves the graph idea, Phase 2 proves the multi-level switch, Phase 3 makes the dataset self-serve. Decisions to lock before Phase 2: zoom threshold *with hysteresis*, overview edge arrows yes/no, pan-position mapping. Decision to lock before Phase 3: `localStorage`-vs-JSON-file as primary durability. Ship Phases 1 and 2 before touching the editor — the hardest bugs live in the multi-level transition, not the CRUD.
