---
name: build-spacing-scale
description: >
  Builds, merges, renames, reorders, and prunes a numeric scale of FLOAT variables in Figma —
  spacing, sizing, or any value ramp. Use this skill whenever the user wants to consolidate two
  or more spacing/size scales into one, deduplicate values across differently-named scales, add a
  spacing primitive scale under a Primitives collection, rename a scale group (e.g. Space → Scale),
  enforce a step-name convention (e.g. name = value × 25), sort a scale by value, or remove unused
  steps. Trigger on phrases like "merge both spacing scales", "merge the values and ignore names",
  "add this scale under primitives", "rename Space to Scale", "order the scale", "there's no value
  N in the design system, remove it", or when the user pastes two variable tables (Name/Value) and
  asks to combine them. Covers: merge-by-value, naming conventions, placement, ordering, and
  pruning — with confirmation before any destructive (binding-breaking) write.
compatibility:
  tools:
    - figma:get_variable_defs   # quick read of a selected node's bound variables (optional)
    - figma:use_figma           # all inspection + writes via the Plugin API (read, create, rename, delete)
  figma_setup: >
    Official Figma MCP server connected to the target file open in Figma Desktop.
    No Desktop Bridge dev-plugin required — use_figma runs Plugin API code directly by fileKey.
    MANDATORY: load the figma-use skill before every use_figma call.
---

# build-spacing-scale

Takes one or more numeric scales (spacing, sizing, radius, any FLOAT ramp) and produces a single
clean primitive scale: merged by value, named to a consistent convention, ordered, and pruned of
unused steps. The same operations apply to any value ramp — "spacing" is just the common case.

The core insight that makes merges correct: **a scale's identity is its VALUE, not its name.** Two
scales built by different teams will use different name conventions (`50/100/200` vs `050/100/150`)
for the same underlying pixel values. Merge on the value, then apply one naming convention.

---

## Inputs

| Input | How to get it |
|---|---|
| Target Figma file URL | From the user — extract the `fileKey` from the URL and pass it to every `use_figma`/`get_variable_defs` call. Confirm the file is open in Figma Desktop with the Figma MCP server connected (no Desktop Bridge plugin needed). |
| Source scales | Which collections/groups to merge. The user may paste Name/Value tables as screenshots — match each to a real group by its **values**, not its names. |
| Target collection | Where the merged scale lives. For a design system, raw scales belong in the **Primitives** collection (alongside `Color/…`), enabling cross-type scalability (color + spacing + radius in one Primitives collection). |
| Naming convention | Default `name = value × 25` step-names unless the user specifies otherwise. Confirm if irregular. |

---

## Standing rules (apply throughout)

- **Merge by value, ignore names.** Dedupe on the resolved numeric value. When two source steps
  share a value but differ in name, the name from the user's referenced/primary table wins.
- **Renaming is binding-safe; deleting is not.** Renaming a collection, group, or variable never
  breaks bindings — they resolve by ID. **Deleting a variable detaches every binding** that pointed
  at it (the consuming property keeps its last raw number but loses the link). Any operation that
  deletes — pruning a step, or reordering (see below) — must be confirmed with the user first when
  the variables may be in use.
- **There is no reorder API.** `VariableCollection.variableIds` is read-only and `figma.variables`
  exposes no move/reorder method. Variables appear in the panel in creation order, and newly
  created variables always append to the **end**. To reorder a group you must **delete every
  variable and recreate them in the desired order** — preserving each one's value, scopes, and
  description. Before doing so, scan for inbound aliases/bindings and abort (or repoint) if any
  exist. Freshly-created primitives that nothing aliases yet are safe to delete+recreate.
- **Never invent values silently.** Don't interpolate missing steps or "fix" an off-pattern value
  without flagging it as an explicit, scoped exception with sign-off.
- **Always include a `1000` step.** Every scale this skill produces or touches must contain a raw
  value of `1000`, named `Scale/1000` (or the group's equivalent prefix) — a deliberate, permanent
  sentinel exception to the `name = value × 25` convention (25 × 1000 would be absurd). It exists so
  future Token-tier aliases (e.g. `Radius/Full`, pill/fully-rounded corner tokens) have a primitive
  to point at. Check for an existing step with value `1000` first (regardless of its name) — if one
  already exists, don't duplicate it; if none exists, add it. This check runs automatically in
  Phase 0/1 and does **not** need user confirmation — unlike other sentinels, it's a standing
  default, not a one-off judgment call. Give it the same scopes as the rest of the scale plus
  `CORNER_RADIUS` (it's the one step explicitly earmarked for radius aliasing).
- **Set scopes explicitly** on every created variable. For spacing/sizing use
  `["GAP", "WIDTH_HEIGHT"]` (covers auto-layout gap, padding, and width/height). Carry over the
  source variables' scopes when rebuilding.
- **Every phase ends in a proposal, not an action**, except the explicit execute step. Confirm
  before any delete.

---

## Phase 0 — Discover & match

Read the relevant FLOAT variables with a read-only `use_figma` script
(`getLocalVariableCollectionsAsync()` → `getVariableByIdAsync` per id, read `valuesByMode`):
- For each candidate collection/group: variable names, resolved values, modes, scopes.
- If the user pasted a Name/Value table, **match it to a real group by its set of values** — names
  often differ (the whole reason for the merge). If a pasted table has no matching values anywhere
  in the file, say so explicitly; it lives in another file or is just a reference (not a live group
  to delete).

Output: a plain value-keyed inventory of each source. No proposal yet. While inventorying, note
whether any source already has a step valued `1000` — needed for the standing `1000` rule below.

---

## Phase 1 — Merge proposal (read-only)

- Take the **union of values** across all sources; dedupe.
- Add a `1000` step if no source already has one (see Standing rules) — silently include it in the
  merged set rather than asking; only the unusual case (a 1000 already present, or the user
  rejecting it outright) needs to be called out.
- Decide inclusion of outliers/sentinels explicitly. A value like `1000` whose name breaks the
  ramp (every other step is `value × 25`, this one is `value × 1`) is a sentinel — surface it and
  ask whether to keep, drop, or correct it. Never assume.
- Drop steps the user says aren't used in the design system (e.g. "there's no value 1, remove it").
- Sort the merged values ascending.

Present the merged value list, which source each value came from, and any sentinel/outlier flags.

⛔ **Confirm** the merged value set (and sentinel handling) before writing.

---

## Phase 2 — Naming proposal (read-only)

- Apply ONE convention to the whole scale. Default: `name = value × 25` step-names
  (`2 → 50`, `4 → 100`, `8 → 200` …). Confirm if the user's reference shows irregularities.
- **Preserve the user's reference names where they exist**, even if slightly off-convention
  (e.g. a reference that names value 40 `900` rather than the strict `1000`). The pasted table is
  the source of truth for names; flag the deviation rather than silently "correcting" it.
- Use a group prefix for organization (e.g. `Scale/…` or `Space/…`) with the leaf = step-name.
  Renaming the group (e.g. `Space/` → `Scale/`) is a pure rename — binding-safe.

Present the full value → name table.

⛔ **Confirm** names before writing.

---

## Phase 3 — Execute & verify (writes)

Order the writes to minimize destructive churn:

1. **Add / merge into the target collection.** Create any missing values as new variables (skip
   values already present — idempotent). Name per Phase 2, set values, set scopes.
2. **Rename** in place where only names change (binding-safe — no delete needed).
3. **Prune** confirmed-unused steps (`.remove()`).
4. **Reorder** only if required, and only via delete+recreate (see Standing rules) — after the
   inbound-alias safety scan.

### Verify

Re-read the target group and assert:

| Assertion | Pass condition |
|---|---|
| No duplicate values | Each value appears once |
| Names follow the convention | Every step matches the agreed naming (sentinels explicitly excepted) |
| Order | Variables appear in the agreed order (by value, or by name) |
| Scopes | Every variable carries the intended scopes, not `ALL_SCOPES` |

Report name → value for the final group so the user can eyeball it.

---

## Common failure modes

**Merging by name instead of value.** Two scales naming the same pixel value differently
(`50` vs `050`) will be wrongly treated as distinct — always dedupe on the resolved value.

**Assuming there's a reorder API.** There isn't. `variableIds` is read-only. Reordering = full
delete+recreate of the group, which is destructive — scan for inbound aliases and confirm first.

**Forgetting that delete breaks bindings.** Unlike a rename, deleting a variable to prune or
reorder detaches every consumer. Confirm before deleting variables that may be in use, and warn
that spacing links will fall back to raw numbers.

**Silently "fixing" a sentinel.** An off-ramp value (name = value, not value × 25) can't be both
name-ordered and value-ordered at once. Surface it and let the user choose keep / drop / correct —
don't quietly rename it to something absurd (`value × 25` of a 1000 sentinel = `25000`).

**Duplicating the scale across collections.** Adding a primitive scale under Primitives while an
older standalone copy still exists creates two sources of truth. Flag it; offer to alias the old
collection to the new primitives or retire it.
