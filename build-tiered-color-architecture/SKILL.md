---
name: build-tiered-color-architecture
description: >
  Restructures a Figma file's color variables into a 3-tier Primitives → Tokens → Semantic
  architecture, optionally matching the naming/numbering convention of a reference file. Use
  this skill any time the user wants to organize, restructure, or rebuild color variables into
  tiers — including phrases like "organize my color variables into Primitives/Tokens/Semantic",
  "restructure my Figma colors into a 3-tier system", "make my colors match [reference file]'s
  architecture", "set up a design token hierarchy", or "split primitives/tokens/semantics." Also
  trigger when the user shares a Figma file with flat or 2-tier color variables and asks to bring
  it up to a token-architecture standard, or references Material Design / Tailwind-style ramps
  for colors. Covers the full pipeline: discovery, direction/duplicate/granularity checks,
  Primitives proposal, Tokens proposal, Semantic rebinding, and execution — with a human review
  gate after every tier.
compatibility:
  tools:
    - figma-console:figma_get_variables   # collection inspection, alias resolution
    - figma-console:figma_execute         # variable/collection creation, binding (Phase 5 only)
  figma_plugin: figma-console-mcp (must be open and connected to the target file)
---

# build-tiered-color-architecture

Takes a file's existing color variables — however they're currently organized — and
restructures them into three explicit tiers: **Primitives** (hardcoded hue ramps, one mode),
**Tokens** (functional groups that alias Primitives, where multi-brand/multi-theme mode-switching
lives), and **Semantic** (component-purpose variables that alias Tokens only, used to build
screens). Five human review gates protect against destructive mistakes and silent invention of
color values.

---

## Inputs

| Input | How to get it |
|---|---|
| Target Figma file URL | From the user — confirm the Desktop Bridge is connected to this file |
| Reference file URL (optional) | If the user wants to match an existing file's convention — read its structure in Phase 0 too |
| Numbering convention | Default to Tailwind `50–950`. Confirm if the user specifies otherwise. |

---

## Phases at a glance

```
Phase 0 — Discovery                (read-only)
    └── Inventory target file's color variables: collections, groups, modes, steps, hex
    └── If a reference file is given, inventory its structure too (naming, step range, direction)
         ↓
Phase 1 — Pre-Flight Checks        (read-only)
    └── 1a. Direction check  (light→dark vs dark→light as numbers increase)
    └── 1b. Duplicate-hue check  (compare representative hex across groups at matching steps)
    └── 1c. Granularity check  (enough real hex values to fill the target step count?)
         ↓
    ⛔ HUMAN GATE A — confirm direction, duplicate resolution, interpolation exceptions
         ↓
Phase 2 — Tier 1: Primitives proposal     (read-only)
    └── Unroll mode-varying groups into separate single-mode groups, named by literal hue
    └── Apply confirmed numbering scheme
         ↓
    ⛔ HUMAN GATE B — approve Primitive groups + numbering before continuing
         ↓
Phase 3 — Tier 2: Tokens proposal         (read-only)
    └── Group by function (Primary/Secondary/Tertiary/Information/Success/Error + project-specific)
    └── Each Token group aliases its ENTIRE source Primitive ramp — not just steps in current use
    └── Multi-mode switching lives here, never at Tier 1
         ↓
    ⛔ HUMAN GATE C — approve Token groups, sources, and full-ramp inclusion
         ↓
Phase 4 — Tier 3: Semantic proposal       (read-only)
    └── Recreate the file's existing component-purpose collection variable-for-variable
    └── Translate every old (Primitive group, old step) reference → new (Token group, new step)
    └── Match-by-function where the semantic name names a function; else preserve original intent
         ↓
    ⛔ HUMAN GATE D — approve full binding table before any writes
         ↓
Phase 5 — Execute & Verify         (writes)
    └── Build all three tiers in Figma only after every gate has passed
    └── Verify: no Semantic→Primitive direct refs, no orphans, no dead values, full ramps
```

---

## Phase 0 — Discovery

Read every COLOR variable across all collections in the target file:
- Collection names, modes (names + IDs), variable names, step counts per group, hardcoded vs.
  aliased, resolved hex per mode.
- Note any non-color variables that share a name pattern with color groups (e.g. a "Scale"
  primitive for spacing) — don't conflate them with color ramps.

**If the user references a reference/style-model file:**
Connect to it (`figma_list_open_files` / `figma_navigate` if Desktop Bridge needs switching) and
inventory the same things:
- Naming pattern, e.g. `Color/[Group]/[Step]`
- Step range and increment style (e.g. Tailwind `50,100,200,300,...,950` vs. Material
  `0,5,10,...,100`)
- Whether modes exist at each tier, and which tier (if any) introduces multi-mode switching
- Whether Tier 2 sources are full ramps or only specific consumed steps

Output: a plain inventory table. No proposal yet.

---

## Phase 1 — Pre-Flight Checks

Run all three before proposing anything — each one changes the plan downstream.

### 1a. Direction check

Find the lightest and darkest step in a sample ramp. Compare against the target numbering
convention's direction (Tailwind: `50` = lightest, `950` = darkest). If the file's current
convention runs the opposite way, flag it explicitly and confirm the flip before assigning any
new numbers. Get this wrong and every subsequent renumbering table is backwards.

### 1b. Duplicate-hue check

For every primitive group (including ones about to be unrolled from modes), pull 2–3
representative hex values at matching steps and compare across groups. Two groups that resolve
to near-identical color are a flag, not a silent merge and not a silent "keep both" — surface the
specific hex evidence and ask how to resolve (merge into one shared primitive, keep both despite
similarity, or rename one to disambiguate).

### 1c. Granularity check

Count the real, distinct hex values backing each group that needs to hit a target step count.
If a group has fewer real values than the step count requires (e.g. 4 flat semantic values where
an 18-step ramp is expected), do **not** silently interpolate. Flag it as a rule conflict against
"never invent variables," and get explicit per-group sign-off before generating any interpolated
values. Always check whether more real values for that hue exist elsewhere in the file before
proposing interpolation.

### Human Gate A

Present: the direction-flip recommendation, any duplicate-hue findings with hex evidence, and
any granularity shortfalls needing an interpolation exception. Wait for explicit answers to each
before drafting Tier 1.

---

## Phase 2 — Tier 1: Primitives proposal

- **Unroll, don't preserve, mode-variance.** Any primitive that currently differs across modes
  (e.g. one "Primary" variable with 5 different hex sets) becomes **N separate single-mode
  groups**, one per mode, named for what that group's hue actually looks like — not for its old
  functional role. Groups that don't vary by mode collapse from N modes down to 1.
- Apply the Gate A duplicate-hue resolution (e.g. a merged group absorbs both sources).
- Apply the confirmed direction and step-numbering scheme to every group uniformly. Groups with
  different real step-counts may need slightly different spacing to land on the same start/end
  bounds — show the exact old→new mapping table per group, don't just describe it.
- Any tier explicitly excluded from renumbering (e.g. a Neutral ramp matching a reference file's
  own convention) keeps its original numbering — call this out explicitly so it isn't missed
  downstream.

### Human Gate B

Present: full group list (final names), step count and hex-per-step source (real vs.
interpolated), and the complete renumbering table. No execution until approved.

---

## Phase 3 — Tier 2: Tokens proposal

- **This is where multi-mode switching lives — never at Tier 1, never skipped at Tier 2.**
- Group by **function**, not hue. Default functional vocabulary unless the user specifies
  otherwise: `Primary`, `Secondary`, `Tertiary`, `Information`, `Success`, `Error` — `Tertiary` is
  optional; only include it if the source file actually has a distinct hue for it.
- Each functional Token group aliases its source Primitive's **entire ramp**, not just the steps
  currently consumed by downstream semantic bindings. Partial-ramp tokens silently limit future
  use; always carry the full range forward.
- It's valid for two or more functional names to share one source while remaining separate Token
  groups (e.g. `Secondary` and `Information` both aliasing the same hue) — don't collapse them
  into one just because they're redundant; ask if it looks unintentional, but don't assume.
- For brand/theme-variant groups (the ones unrolled in Phase 2), the corresponding Token group
  (often named `Brand` or `Primary`) mode-switches across those Primitive groups — one Token
  variable per step, with a different alias target per mode.

### Human Gate C

Present: every Token group, its full step range, and its source mapping (per mode, where
applicable). No execution until approved.

---

## Phase 4 — Tier 3: Semantic proposal

- Tier 3 recreates the file's **existing component-purpose collection** (Icon/Text/
  Background/Border/Navigation, or whatever the file calls these) variable-for-variable — same
  names, same mode structure, unless the user has asked for a category rename (e.g.
  Background → Surface).
- **Every alias must point only at Tier 2.** Never leave (or recreate) a direct reference to
  Tier 1.
- For each existing semantic variable, in each of its modes: identify the OLD (Primitive group,
  old step) it currently resolves to, then translate through the Phase 2/3 mappings into the NEW
  (Token group, new step). This exactly preserves the original visual result — it's a mechanical
  translation, not a redesign.
- **Match-by-function rule:** when a semantic variable's own name names a function (`Success`,
  `Error`, `Tertiary`, etc.), it must source only from the Tier 2 group of that same name — e.g.
  a Success-named text color comes only from Token `Success`, never from `Brand` or `Neutral`
  even if the hex happens to match. When the semantic category is *not* itself a function (e.g.
  `Default`, `Disabled`), follow the original file's per-mode intent instead — the correct source
  is whichever Tier 2 group fits that specific mode's role, even if it differs mode-to-mode.
  Tier 3 is the layer screens are actually built from, so this distinction matters for downstream
  consistency.
- If a category-wide source-preference rule is specified (e.g. "Text should prioritize
  Information over Secondary when both apply"), apply it only where it's actually applicable —
  state plainly if no current binding triggers the rule, rather than forcing it.
- **Flag, don't silently fix,** any inconsistency uncovered during remapping — e.g. one mode of a
  variable sourcing from an unexpected group while every other mode follows the pattern. Preserve
  the original behavior by default; only change it on explicit instruction.

### Human Gate D

Present the full binding table, grouped by semantic category, old reference → new reference.
This is the last gate before any writes — wait for explicit go-ahead.

---

## Phase 5 — Execute & Verify

Only after all four gates have passed. Build in this order — never reorder:

1. Create Tier 1 Primitive collection(s) and variables (hardcoded hex, single mode each except
   where the source genuinely needs more — but multi-mode at Tier 1 should not occur post-restructure).
2. Create Tier 2 Token collection, modes, and variables — each value an alias into Tier 1.
3. Create or repoint Tier 3 Semantic collection — each value an alias into Tier 2 only.

### Verify

Re-fetch all three collections and assert:

| Assertion | Pass condition |
|---|---|
| Semantic→Primitive direct refs | Zero — every Semantic alias resolves through Tokens |
| Orphaned aliases | Zero — no alias pointing to a deleted/nonexistent variable ID |
| Dead Primitive values | Zero unused, unless intentionally kept for a non-color-binding reason |
| Token ramp completeness | Every Token group includes its full source ramp, not a subset |
| Duplicate hues | None remaining unless explicitly approved to keep as separate groups |

Report failures with variable names and IDs — never silently pass or auto-correct. Surface to
the user for a decision.

---

## Standing rules (apply throughout all phases)

- Primitives are hardcoded; Tokens alias only Primitives; Semantics alias only Tokens.
- Never invent a new hex value without flagging it as an explicit, scoped exception with
  sign-off — granularity shortfalls are a conversation, not a silent workaround.
- Always check before editing — every phase ends in a proposal, not an action. Phase 5 is the
  only phase that writes anything.
- When a reference file is provided, match its naming convention and numbering *structure*, not
  just its general spirit — confirm exact step values and direction rather than assuming.
- Default numbering convention is Tailwind `50–950` unless told otherwise.

---

## Common failure modes

**Assuming direction matches without checking** — a file using `0=darkest→100=lightest` will
produce an inverted, confusing ramp if relabeled directly onto `50=lightest→950=darkest` without
an explicit flip. Always check Phase 1a before assigning any new numbers.

**Mistaking "same family" for "same hue."** Two colors can sit in the same general hue family
(e.g. both blue-ish) without being a true duplicate — only flag genuine near-identical matches at
matching steps, not just thematic similarity.

**Forcing full-ramp parity where the data doesn't support it.** Not every functional group will
have enough real hex values to match other groups' step counts — don't force interpolation
silently; that's a Gate A decision, every time.

**Partial-ramp Tokens.** Only including the steps a Semantic variable currently uses at Tier 2
looks efficient but limits future flexibility — always carry the full Primitive ramp forward into
its Token group.

**Treating Tier 2 mode-switching as automatic.** Cross-collection mode resolution in Figma
generally requires the consuming context to apply matching modes to both the Tokens and
Primitives collections — preserve the existing alias structure (same target pattern per mode)
rather than assuming hue-switching "just works" without verifying it.
