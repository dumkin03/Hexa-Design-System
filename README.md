# Color Foundation Skill

A Claude skill that restructures a Figma file's color variables into a 3-tier
**Primitives → Tokens → Semantic** architecture, with a human review gate after every tier.

## Skill

- [`build-tiered-color-architecture/SKILL.md`](build-tiered-color-architecture/SKILL.md)

## What it does

Takes a file's existing color variables — however they're currently organized — and
restructures them into three explicit tiers:

- **Primitives** — hardcoded hue ramps, one mode
- **Tokens** — functional groups that alias Primitives (multi-brand/multi-theme mode-switching lives here)
- **Semantic** — component-purpose variables that alias Tokens only, used to build screens

Five human review gates protect against destructive mistakes and silent invention of color values.
