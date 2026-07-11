# wright-kit format spec

**Status: stub. Formal JSON Schemas land here as spec v1.**

The kit standardizes a small set of plain-JSON formats. They are what let
the tools compose, and what let an engine that has never heard of these
tools consume the output. This page names them; the normative definitions
(JSON Schema files under `schemas/`, plus a synthetic exemplar game) are
spec v1, in progress.

## The formats

| Format | What it holds | Produced by | Consumed by |
| --- | --- | --- | --- |
| **hexgrid** | Grid calibration: the mapping from (col, row) hex coordinates to pixel positions on a board scan | wargame-map-parser | hexwright, game engines |
| **terrain** | Per-hex base terrain assignments | hexwright (hand-trace), wargame-map-parser (first-pass classification) | game engines |
| **hexsides** | Features on shared hex edges: rivers, escarpments, borders and the like | hexwright | game engines |
| **features** | Typed point features on hexes (cities, forts, objectives) with optional attributes | hexwright | game engines |
| **OOB** | Order of battle: units, factors, formations, reinforcement schedules | musterwright | game engines |

Point-to-point boards (the map family of card-driven games) use a parallel
nodes/edges pair, edited and exported by hexwright.

## Where the current shapes are documented

Until the schemas land, each tool documents its own current formats:

- **hexwright**: the [hexwright README](https://github.com/lerugray/hexwright#readme)
  covers the project file, terrain/hexside/feature exports, and the
  point-to-point nodes/edges formats.
- **wargame-map-parser**: `docs/CONVENTIONS.md` in that repo covers grid
  calibration and the parser-to-hexwright handoff.
- **musterwright**: `docs/DESIGN.md` in that repo covers the OOB spine and
  the per-game profile/adapter model.

Where shipped tools currently diverge on an encoding, spec v1 will declare
one canonical shape and document the legacy shapes as named variants.
