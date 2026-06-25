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
  gate after every tier, and a hardened execute-and-verify protocol learned from a real
  near-loss of source data during execution.
compatibility:
  tools:
    - figma:get_variable_defs   # quick read of a selected node's bound variables (optional)
    - figma:use_figma           # all inspection + writes via the Plugin API (read collections, create, rename, bind)
  figma_setup: >
    Official Figma MCP server connected to the target file open in Figma Desktop.
    No Desktop Bridge dev-plugin required — use_figma runs Plugin API code directly.
    MANDATORY: load the figma-use skill before every use_figma call.
---

# build-tiered-color-architecture

Takes a file's existing color variables — however they're currently organized — and
restructures them into three explicit tiers: **Primitives** (hardcoded hue ramps, one mode),
**Tokens** (functional groups that alias Primitives, where multi-brand/multi-theme mode-switching
lives), and **Semantic** (component-purpose variables that alias Tokens only, used to build
screens). Five human review gates protect against destructive mistakes and silent invention of
color values. Phase 5 carries extra hardening — read it carefully, it is not boilerplate; it
exists because a real run of this skill deleted a tier's only remaining source data mid-script.

---

## Example prompts

Real prompts this skill handles:

- **Full restructure:** "Organize my color variables into Primitives/Tokens/Semantic" / "restructure my Figma colors into a 3-tier system" / "make my colors match [reference file]'s architecture." → Run all five phases.
- **Naming-only pass:** "The new groups Color Primitives, Color Semantic, and Color Tokens are respectively 1. Primitives, 2. Tokens, and 3. Semantic — rename these collections accordingly. And for scalability, in the future we'll also add Typography, spacing, and other design-system variables." → The tiers already exist and are correctly bound; the user only wants the **Tier collection naming** convention applied (see Standing rules). This is a rename-only operation — no Primitives/Tokens/Semantic restructuring, no rebinding. Map by concept, not list position (`Color Primitives → 1. Primitives`, `Color Tokens → 2. Tokens`, `Color Semantic → 3. Semantic`), confirm the mapping, then rename. Collection renames don't touch bindings (those are by ID), so this is safe and reversible.
- **Post-build correction pass:** "Neutral is supposed to be 0-100, light to dark" / "some ramps only go up to 900 when I want 950" / "Tokens must be function-named, so Green must be Success instead." → These are corrections to an *already-built* architecture, not a request to redo it. Diagnose whether each fix is a pure relabel (same color, wrong label — fix via rename, see Phase 5) or an actual rebinding change (see "Renaming vs. rebuilding"). Re-verify live state first; don't assume the prior build matches what's being described.

---

## Inputs

| Input | How to get it |
|---|---|
| Target Figma file URL | From the user — extract the `fileKey` from the URL and pass it to every `use_figma`/`get_variable_defs` call. Confirm the file is open in Figma Desktop with the Figma MCP server connected (no Desktop Bridge plugin needed). |
| Reference file URL (optional) | If the user wants to match an existing file's convention — read its structure in Phase 0 too |
| Numbering convention | Default to Tailwind `50–950`. Confirm if the user specifies otherwise. |

---

## Phases at a glance

```
Phase 0 — Discovery                (read-only)
    └── Inventory target file's color variables: collections, groups, modes, steps, hex
    └── Check for pre-existing partial/draft work under any name — don't assume a clean slate
    └── If a reference file is given, inventory its structure too (naming, step range, direction)
         ↓
Phase 1 — Pre-Flight Checks        (read-only)
    └── 1a. Direction check  (light→dark vs dark→light, per group — confirm separately for any excepted group)
    └── 1b. Duplicate-hue check  (compare representative hex across groups at matching steps)
    └── 1c. Granularity check  (enough real hex values to fill the target step count AND bounds?)
         ↓
    ⛔ HUMAN GATE A — confirm direction, duplicate resolution, interpolation exceptions
         ↓
Phase 2 — Tier 1: Primitives proposal     (read-only)
    └── Unroll mode-varying groups into separate single-mode groups, named by literal hue
    └── Apply confirmed numbering scheme — verify actual achievable bounds against real data
         ↓
    ⛔ HUMAN GATE B — approve Primitive groups + numbering before continuing
         ↓
Phase 3 — Tier 2: Tokens proposal         (read-only)
    └── Group by function (Primary/Secondary/Tertiary/Information/Success/Error + project-specific)
    └── Each Token group aliases its ENTIRE source Primitive ramp — not just steps in current use
    └── Multi-mode switching lives here, never at Tier 1. Function names only — zero literal hue names.
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
Phase 5 — Execute & Verify         (writes — see hardened protocol below)
    └── Snapshot → verify snapshot → delete (isolated) → verify empty → build (isolated) → verify built
    └── Never combine delete and create in a single script call
    └── Prefer renaming over rebuilding whenever only a label is wrong, not the color
```

---

## Phase 0 — Discovery

Read every COLOR variable across all collections in the target file. Use a read-only `use_figma`
script (`figma.variables.getLocalVariableCollectionsAsync()`, then `getVariableByIdAsync` per id,
resolving aliases per mode) and `return` the inventory — collect:
- Collection names, modes (names + IDs), variable names, step counts per group, hardcoded vs.
  aliased, resolved hex per mode.
- Note any non-color variables that share a name pattern with color groups (e.g. a "Scale"
  primitive for spacing) — don't conflate them with color ramps.

**Check for pre-existing partial work before assuming a clean slate.** A previous session, a
collaborator, or an earlier draft attempt may have already started this exact restructuring —
sometimes under names that don't obviously match (e.g. the original "Color Tokens" collection
simply renamed in-place to "Color Semantic" with nothing else changed, or a fresh collection
holding an outdated iteration of the same plan, or a *different environment's* session — Claude
Code, a prior chat, another collaborator — having advanced the work further than your own visible
context shows). Specifically look for:
- Collections whose **group names** resemble an earlier or partial version of what's being
  proposed (separate single-mode hue groups already unrolled, functional Token groups already
  present, etc.) even if the collection's own name doesn't match the target tier name.
- Collections whose **ID** matches an original collection you inventoried earlier in the
  conversation, even if its name or contents have since changed — this is a strong signal it's
  the same collection mid-transformation, not a fresh one.
- If anything like this turns up, **stop and surface it explicitly** before proposing or building
  anything further — don't guess whether it's stale, intentional, or mid-progress. Ask.

**If the user references a reference/style-model file:**
Inventory it the same way — pass that file's `fileKey` to a `use_figma` read call (each call
targets a file by key, so there's no plugin/file switching to manage) and capture the same things:
- Naming pattern, e.g. `Color/[Group]/[Step]`
- Step range and increment style (e.g. Tailwind `50,100,200,300,...,950` vs. Material
  `0,5,10,...,100`) — and whether that range is actually uniform end-to-end, or has an irregular
  jump near the boundary. Don't assume; read the actual list of steps present.
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

**Repeat this check separately for any group excepted from renumbering** (e.g. a Neutral ramp
kept at `0–100` instead of `50–950`). "Excepted from renumbering" usually means *keep the number
range*, not necessarily *keep the original direction* — these are two separate decisions. Confirm
both explicitly: does the excepted group keep the original range AND the original direction, or
just the range? Conflating these cost a real rework cycle in practice — a Neutral group was built
preserving the source file's own direction, when the actual intent was for it to follow the same
light-to-dark-as-numbers-increase convention as every other group, just on a 0–100 scale instead
of 50–950.

### 1b. Duplicate-hue check

For every primitive group (including ones about to be unrolled from modes), pull 2–3
representative hex values at matching steps and compare across groups. Two groups that resolve
to near-identical color are a flag, not a silent merge and not a silent "keep both" — surface the
specific hex evidence and ask how to resolve (merge into one shared primitive, keep both despite
similarity, or rename one to disambiguate).

### 1c. Granularity check

Count the real, distinct hex values backing each group that needs to hit a target step count
**and target bounds**. Two separate shortfalls are possible: not enough distinct values to fill
every step, *and* not enough range to reach the convention's boundary value (e.g. a group with 18
genuinely distinct hex values may still only span `50–900` if that's all the real ramp covers,
falling short of a `950` boundary used elsewhere). Don't assume a convention's nominal endpoint is
reachable — verify the real source data's count and span before proposing numbers.

If a group has fewer real values than the step count requires, do **not** silently interpolate.
Flag it as a rule conflict against "never invent variables," and get explicit per-group sign-off
before generating any interpolated values. Always check whether more real values for that hue
exist elsewhere in the file before proposing interpolation.

If a group has enough values to fill all needed steps but stops short of the convention's outer
boundary, that's a smaller, separate question — best resolved later, after the initial build, by
relabeling the topmost/bottommost step rather than inventing a new color for it (see Phase 5,
"Renaming vs. rebuilding").

### Human Gate A

Present: the direction-flip recommendation (per group, including any excepted group), any
duplicate-hue findings with hex evidence, and any granularity shortfalls needing an interpolation
exception. Wait for explicit answers to each before drafting Tier 1.

---

## Phase 2 — Tier 1: Primitives proposal

- **Unroll, don't preserve, mode-variance.** Any primitive that currently differs across modes
  (e.g. one "Primary" variable with 5 different hex sets) becomes **N separate single-mode
  groups**, one per mode, named for what that group's hue actually looks like — not for its old
  functional role. Groups that don't vary by mode collapse from N modes down to 1.
- Apply the Gate A duplicate-hue resolution (e.g. a merged group absorbs both sources).
- Apply the confirmed direction and step-numbering scheme to every group uniformly — but verify
  against the **actual** step list present in any existing draft/source rather than assuming a
  textbook scale. An "18-step Tailwind ramp" in the wild may genuinely run `50–900`, not `50–950`;
  don't write a build script around an assumed scheme without checking first. Show the exact
  old→new mapping table per group in the Gate B proposal — don't just describe the renumbering
  verbally.
- Any tier explicitly excluded from renumbering (e.g. a Neutral ramp matching a reference file's
  own convention) keeps its original numbering — call this out explicitly so it isn't missed
  downstream, and confirm separately whether direction is also excepted (see Phase 1a).

### Human Gate B

Present: full group list (final names), step count and hex-per-step source (real vs.
interpolated), and the complete renumbering table. No execution until approved.

---

## Phase 3 — Tier 2: Tokens proposal

- **This is where multi-mode switching lives — never at Tier 1, never skipped at Tier 2.**
- Group by **function**, not hue. Default functional vocabulary unless the user specifies
  otherwise: `Primary`, `Secondary`, `Tertiary`, `Information`, `Success`, `Error` — `Tertiary` is
  optional; only include it if the source file actually has a distinct hue for it.
- **Token groups must carry function names only — zero exceptions, including incidentally.** A
  literal hue name (e.g. `Green`) must not survive into Tier 2 even when it's also a
  reasonable-sounding function-adjacent word; rename it to its true functional role (`Success`).
  Audit every Token group name against this rule explicitly before calling Gate C passed — it's
  an easy thing to carry over by habit from the Tier 1 build.
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
- **Confirm mode count explicitly, even if it seems implied by an earlier answer.** If Tokens has
  N modes for brand/theme switching, Semantic generally needs the same N modes to preserve that
  switching all the way to where screens are built — don't let an earlier, differently-framed
  question (e.g. about a single-mode reference file) cause Tier 3 to silently collapse to one
  mode. If your own draft binding tables already show per-mode variation, trust that signal over
  an earlier answer that may have been a miscommunication, and check before building either way.

### Human Gate D

Present the full binding table, grouped by semantic category, old reference → new reference.
This is the last gate before any writes — wait for explicit go-ahead.

---

## Phase 5 — Execute & Verify (hardened protocol)

Only after all four gates have passed. This phase carries the most operational risk in the whole
skill — a real run of this exact rebuild deleted a tier's only remaining source data and crashed
before rebuilding it, because delete and create were combined in one script. The protocol below
exists specifically because of that. Follow it literally, every time.

**Build tiers strictly in order — Tier 1 fully complete before Tier 2 starts, Tier 2 fully
complete before Tier 3 starts. Never reorder.** Each tier's build script depends on the previous
tier already existing (Tokens alias Primitives by ID; Semantic aliases Tokens by ID) — starting a
tier before its dependency is finished and verified means aliasing against variables that don't
exist yet, or that will be deleted and recreated with different IDs later.

### Core rule: never combine delete and create in the same script execution

If a script deletes existing variables and then creates new ones in the same call, and *anything*
after the delete throws — a missing key, a wrong assumption about step ranges, a typo — the
deletion has already happened and cannot be rolled back by the script. Always split into
separate, isolated tool calls:

1. **Snapshot.** Read-only call. Extract every hex value (or full alias chain) you'll need,
   for every group and mode, and return it in the response. Don't assume you'll be able to
   re-read it after the next step.
2. **Verify the snapshot is complete.** Before touching anything, sanity-check the snapshot
   against the expected group list and step counts from the approved Gate B/C/D proposal. If
   anything expected is missing, stop — don't delete source data while a gap is unexplained.
3. **Delete.** A separate, isolated call. Confirm the count *before* and *after* — the after-count
   must be exactly what's expected (usually `0`) before proceeding.
4. **Build.** A separate, isolated call, using the **hardcoded snapshot values from step 1**
   embedded directly in the script — not a live re-read, since the collection you're rebuilding
   from might be the same one you just emptied, or might have changed. Guard the script with an
   explicit check (e.g. `if (collection.variableIds.length !== 0) throw new Error(...)`) so it
   refuses to run against a non-empty collection and silently create duplicates.
5. **Verify the build.** Re-fetch and confirm exact counts per group, then resolve a few
   representative alias chains end-to-end (see "Verify" below) to confirm correctness, not just
   existence.

### Verify actual step ranges before writing the build script

Don't hardcode an assumed numbering scheme (e.g. "Tailwind ramps go 50–950") into a build script
without first reading the literal list of steps present in the real source data. An "18-step"
group might genuinely span `50–900` with no `950` at all — write the script around what's there,
not around the textbook convention, and reconcile any gap with the user afterward rather than
guessing.

### Renaming vs. rebuilding — prefer renaming whenever only the label is wrong

Figma variable aliases bind by **ID**, not by name. This means:
- Renaming a variable, or the collection it lives in, never breaks any alias that points to it —
  downstream tiers continue resolving correctly with zero changes needed.
- If a post-build correction only requires a different *label* on an already-correct color value
  (e.g. fixing a ramp's direction, or extending a ramp's top label from `900` to `950` when the
  underlying value is already the right one — often a duplicate "pure black" anchor), do a
  **rename**, not a delete-and-recreate. It is dramatically lower-risk and doesn't touch any
  value data at all.
- **Watch for rename collisions.** If a relabeling scheme is a permutation (e.g. flipping a
  ramp's direction swaps most labels pairwise), renaming in a single pass can collide — renaming
  variable A to a name still held by not-yet-renamed variable B throws a "duplicate variable
  name" error, and a script that errors mid-loop may leave the set half-renamed. Always do this
  as **two passes**: first move every affected variable to a unique temporary name (e.g. prefix
  `TEMP_`), then assign the final target names in a second pass. This guarantees no collision
  regardless of the permutation's structure.
- Conversely, when a rename-only fix changes which underlying Tier 1 variable a Tier 2/3 alias
  *should* point to (not just its label) — e.g. a direction flip changes which physical variable
  represents "lightest" — recompute and rebuild those higher-tier aliases explicitly; a label
  change on Tier 1 alone does not automatically fix semantic correctness if the higher tier was
  pointing at the right ID for the wrong reason.

### If a destructive mistake happens anyway

- Stop immediately. Don't run further scripts.
- Tell the person plainly what happened and what's at risk, in the same message — don't soften
  or delay this.
- Ask them to press Undo (`Cmd+Z` / `Ctrl+Z`) in Figma Desktop immediately, before anything else
  happens in the file. Plugin/Plugin-API-driven mutations from a single script execution are
  typically grouped into one undo step, so a prompt single undo is often fully recoverable.
- Don't rely on REST-API-based file version history to recover variable values — variable value
  changes are not reliably captured the same way document-node changes are; it's not a dependable
  recovery path.
- After they confirm the undo, **re-verify live state from scratch** before doing anything else —
  don't assume the undo restored exactly what you expect; check counts and a few hex values.

### Don't trust "I think that's already fixed" without re-checking

If the person says part of a fix is already in place, re-inspect live state before proceeding
anyway. An earlier attempt may have silently failed partway through (e.g. erroring on step 1 of a
multi-step script, leaving every later step un-applied despite the person's impression that it
ran), or a *different session/environment* may have applied a different fix than expected. Cheap
to verify, expensive to assume.

### If the MCP/plugin connection drops mid-task

A connection failure is not data loss — the file itself is unaffected. Ask the person to
reconnect (reopen the plugin / re-establish the MCP connection), then re-verify current state
before resuming. Don't assume anything about what did or didn't complete before the drop.

### Verify

Re-fetch all three collections and assert:

| Assertion | Pass condition |
|---|---|
| Semantic→Primitive direct refs | Zero — every Semantic alias resolves through Tokens |
| Orphaned aliases | Zero — no alias pointing to a deleted/nonexistent variable ID |
| Dead Primitive values | Zero unused, unless intentionally kept for a non-color-binding reason |
| Token ramp completeness | Every Token group includes its full source ramp, not a subset |
| Token group names | 100% function names — no literal hue name anywhere in Tier 2 |
| Duplicate hues | None remaining unless explicitly approved to keep as separate groups |
| Mode-switching | Resolving the same Semantic variable across all modes returns the expected different (or intentionally identical) Primitive group per mode — spot-check at least one brand-switching variable end-to-end, by name, not just by ID existing |

Report failures with variable names and IDs — never silently pass or auto-correct. Surface to
the user for a decision.

---

## Standing rules (apply throughout all phases)

- Primitives are hardcoded; Tokens alias only Primitives; Semantics alias only Tokens.
- **Tier collection naming.** Name the three tier collections with an ordering prefix and no
  type-specific prefix: `1. Primitives`, `2. Tokens`, `3. Semantic`. Drop prefixes like `Color `
  (e.g. `Color Primitives` → `1. Primitives`) so the same numbered tiers can later also host
  Typography, spacing, radius, and other design-system variables — the numbers convey the
  primitives → tokens → semantic order in the variables panel, independent of value type. Renaming
  a collection never breaks existing bindings (variables are referenced by ID, not name), so it's
  safe and reversible. If the target name already exists as an empty stray collection, delete the
  stray first to avoid a duplicate name. Map old → new **by concept, not list position** — a
  reordered "respectively" list (e.g. Semantic listed before Tokens) still maps Semantic→Semantic,
  Tokens→Tokens. Note: this collection-level numbering is distinct from the Tailwind `50–950`
  numbering used for *steps within* a ramp.
- Never invent a new hex value without flagging it as an explicit, scoped exception with
  sign-off — granularity shortfalls are a conversation, not a silent workaround.
- Always check before editing — every phase ends in a proposal, not an action. Phase 5 is the
  only phase that writes anything, and even within Phase 5, destructive steps (delete) are
  isolated from constructive steps (create).
- When a reference file is provided, match its naming convention and numbering *structure*, not
  just its general spirit — confirm exact step values and direction rather than assuming.
- Default numbering convention is Tailwind `50–950` unless told otherwise — but confirm the real
  data actually reaches those bounds before building around them.
- Prefer the lowest-risk operation that achieves the requested fix: rename over rebuild, rebuild
  over reinterpretation, reinterpretation over invention.

---

## Common failure modes

**Assuming direction matches without checking** — a file using `0=darkest→100=lightest` will
produce an inverted, confusing ramp if relabeled directly onto `50=lightest→950=darkest` without
an explicit flip. Always check Phase 1a before assigning any new numbers. And check it again,
separately, for any group excepted from renumbering — "keep the numbers" and "keep the direction"
are independent decisions.

**Mistaking "same family" for "same hue."** Two colors can sit in the same general hue family
(e.g. both blue-ish) without being a true duplicate — only flag genuine near-identical matches at
matching steps, not just thematic similarity.

**Forcing full-ramp parity where the data doesn't support it.** Not every functional group will
have enough real hex values to match other groups' step counts or boundary values — don't force
interpolation silently; that's a Gate A decision, every time. A shortfall in *boundary* (stops at
900 instead of 950) is a smaller, separate issue from a shortfall in *count* — the first can often
be fixed later with a rename, the second cannot.

**Partial-ramp Tokens.** Only including the steps a Semantic variable currently uses at Tier 2
looks efficient but limits future flexibility — always carry the full Primitive ramp forward into
its Token group.

**A literal hue name surviving into Tier 2.** Easy to miss because it doesn't look wrong in
isolation (`Green` reads fine as a name) — but Tier 2 is a function-only namespace. Audit every
Token group name explicitly against the functional vocabulary before considering Gate C done.

**Treating Tier 2 mode-switching as automatic.** Cross-collection mode resolution in Figma
generally requires the consuming context to apply matching modes to both the Tokens and
Primitives collections — preserve the existing alias structure (same target pattern per mode)
rather than assuming hue-switching "just works" without verifying it.

**Combining delete and create in one script.** The single most costly mistake possible in this
skill. A script that deletes a tier's variables and then crashes while rebuilding them leaves
that data gone with no script-level undo. Always isolate delete from create into separate calls,
snapshot first, and verify counts at every step. See Phase 5 in full.

**Renaming in one pass when the relabeling is a permutation.** Causes "duplicate name" errors and
can leave a partially-renamed, inconsistent set behind. Always two-pass: temp names first, final
names second.

**Trusting "that's already fixed" without re-checking live state.** Earlier attempts — including
ones run in a different session or environment — can fail silently partway through, or apply a
different fix than expected. Re-inspect before proceeding, every time.
