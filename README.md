# wright-kit

A wargame construction kit.

Free, open-source tools and shared JSON formats for turning a printed
wargame into a playable digital prototype. Local web apps, plain files on
your disk, MIT licensed. Your maps are yours.

## What this is

You have a hex-and-counter wargame, or a card-driven point-to-point one: a
board, a countersheet, an order of battle, charts. Getting all of that into
a computer usually means retyping everything by hand into whatever shape
one program wants. The kit splits the job into three pillars:

1. **DIGITIZE.** Calibrate a scan of the board and classify its terrain
   with wargame-map-parser, then hand-trace terrain, rivers, roads, and
   point features (or point-to-point nodes and edges) in hexwright.
2. **MUSTER.** Build the order of battle in musterwright: unit rosters,
   combat factors, formations, reinforcement schedules, validated against
   a per-game profile.
3. **PUBLISH.** The art and export layer: print-quality styled maps and
   countersheets rendered from the same data. Honestly labeled: this
   pillar is in design and has not shipped yet.

Every tool reads and writes the same canonical JSON, so the output of one
step is the input of the next, and a game engine (yours, not ours) can
load the result directly. The formats are the real product; the tools are
how you fill them in. See [SPEC.md](SPEC.md).

## The tools

| Tool | What it does |
| --- | --- |
| [hexwright](https://github.com/lerugray/hexwright) | Hex-map editor for digitizing printed boards. Load a calibrated grid over a scan, assign terrain, hexside features, and point features by hand, export canonical JSON. Also edits point-to-point (node/edge) boards for card-driven games. Zero build, vanilla JS. |
| [wargame-map-parser](https://github.com/lerugray/wargame-map-parser) | Python pipeline that calibrates a hex grid from a handful of read-off hex numbers, de-duplicates multi-sheet scans, and classifies terrain by reference hexes instead of brittle color thresholds. Feeds hexwright. |
| [musterwright](https://github.com/lerugray/musterwright) | Order-of-battle editor, the force layer. Rosters, factor sets, pooled strength, formations, reinforcement schedules, live validation, engine-ready export. Renders APP-6 unit counters. Zero build, vanilla JS. |

## Why files, not a service

Everything in the kit runs on your machine and writes plain JSON to your
disk. There is no account, no cloud, no subscription, and no tier where
you pay extra for the right to sell work you made yourself. The tools are
MIT licensed and the formats are documented, so nothing you build is
locked to them. If you stop using the kit tomorrow, your data is still
sitting there in files you can read.

## Status

| Piece | Status |
| --- | --- |
| hexwright | public, v2.1 |
| musterwright | public, v0.1 |
| wargame-map-parser | public |
| format spec ([SPEC.md](SPEC.md) + [JSON Schemas](schemas/v1/)) | v1 |
| publish layer (styled map + countersheet export) | in design |

The kit was extracted from real production, not designed on a whiteboard:
five digitized games so far, including paid digital-prototype work on
published titles.

## Quickstart

Each tool is self-contained; start with whichever end of the pipeline you
need.

**hexwright.** Clone it, then serve its parent directory and open the
editor:

```
cd ..
python3 -m http.server 8000
# open http://localhost:8000/hexwright/
```

On macOS you can instead double-click `Launch Hexwright.command`. See the
[hexwright README](https://github.com/lerugray/hexwright#readme) for the
editor modes and export details.

**musterwright.** Serve the repo root and open `index.html`:

```
python3 -m http.server
```

See the [musterwright README](https://github.com/lerugray/musterwright#readme)
for the profile and adapter model.

**wargame-map-parser.** Python:

```
pip install -r requirements.txt
```

Then follow the worked examples in its
[README](https://github.com/lerugray/wargame-map-parser#readme) and
`examples/` directory.

## License

MIT. See [LICENSE](LICENSE).
