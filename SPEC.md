# SPEC.md v1 — wright-kit shared JSON formats

Status: v1 public specification.  
Scope: shared data formats connecting hexwright, wargame-map-parser, musterwright, and downstream game engines.  
Schemas: JSON Schema draft 2020-12, `$id` rooted at `https://github.com/lerugray/wright-kit/schemas/v1/`.

## 1. General conventions

### Coordinate system

Image-space coordinates use the usual raster convention:

- origin `(0, 0)` is the top-left of the full map image;
- `x` increases to the right;
- `y` increases downward;
- image sizes are `[width, height]`.

### Hex numbering scheme

Canonical hex IDs are four decimal digits:

```text
CCRR
```

Where:

- `CC` is the zero-based column, two digits, `00` through `99`;
- `RR` is the zero-based row, two digits, `00` through `99`.

Examples:

- `0000` = column 0, row 0;
- `0604` = column 6, row 4;
- `1006` = column 10, row 6.

The canonical v1 hex ID pattern is `^[0-9]{4}$`.

### Hex grid orientation

The bundled v1 demo defines a flat-top, even-q grid:

- columns are vertical;
- even-numbered columns receive `even_col_y_offset`;
- row pitch is vertical distance between centers in the same column;
- column pitch is horizontal distance between centers in adjacent columns.

The center of hex `(col, row)` is:

```text
x = x_intercept_col0 + col * col_pitch_x
y = y_intercept_row0 + row * row_pitch_y + (col is even ? even_col_y_offset : 0)
```

Adjacency for this even-q layout is:

- same column: `(c, r - 1)`, `(c, r + 1)`;
- if `c` is even: `(c - 1, r)`, `(c - 1, r + 1)`, `(c + 1, r)`, `(c + 1, r + 1)`;
- if `c` is odd: `(c - 1, r - 1)`, `(c - 1, r)`, `(c + 1, r - 1)`, `(c + 1, r)`.

Coordinates outside `0 <= col < n_cols` or `0 <= row < n_rows` are not valid map hexes.

### Edge and pair ordering

Undirected pairs are stored in canonical order:

```text
a < b
```

using lexical comparison of the canonical IDs.

The canonical pair key, when a tool needs one internally, is:

```text
a + "|" + b
```

after ordering the endpoints. Canonical JSON files store pairs as objects:

```json
{ "a": "0603", "b": "0604" }
```

not as string pair keys.

For undirected data, consumers must treat adjacency symmetrically: if an edge connects `a` and `b`, then `a` is adjacent to `b` and `b` is adjacent to `a`.

### Versioning

v1 files are versioned by this specification and schema path. Individual files do not all carry an explicit version field. `hexgrid.json` carries `grid_version: 1` because the calibration format historically did so; other canonical v1 files do not add a required version property.

Backward-incompatible changes must use a new schema path, for example `/schemas/v2/`. New v1 tools should write canonical v1 only.

### Schema policy

Schemas are closed with `additionalProperties: false` except where the format intentionally contains a dictionary, layer map, or user-defined `attrs` object. Game-specific classification vocabularies are strings; this spec defines shape and invariants, not a universal terrain or unit taxonomy.

---

## 2. `project.json`

### Purpose

A hexwright project manifest. It names the project and points to the map data files that make up a hexwright export bundle.

### Producing tool

hexwright.

### Consuming tools

hexwright, import scripts, map parsers, game engines that load a complete map bundle.

### Top-level shape

Object with project metadata and relative file references.

### Fields

- `name`: Human-readable project name.
- `_comment`: Optional explanatory comment.
- `imageFull`: `[width, height]` of the full image or blank lattice canvas.
- `hexgrid`: Relative path to `hexgrid.json`.
- `terrain`: Relative path to `terrain.json`.
- `hexsides`: Relative path to `hexsides.json`.
- `features`: Relative path to `features.json`.
- `names`: Relative path to `names.json`.
- `blankLattice`: Boolean. `true` means the project can render from calibration alone without a scanned source image.

### Invariants

Referenced files are interpreted relative to the manifest location. Engines should fail closed when a required referenced file is absent or invalid.

---

## 3. `hexgrid.json`

### Purpose

Calibration data for converting canonical hex IDs to image-space centers on a flat-top even-q grid.

### Producing tools

hexwright and calibration stages of wargame-map-parser.

### Consuming tools

hexwright, terrain classifiers, feature tracers, game engines, rendering tools.

### Top-level shape

Object containing grid dimensions, image dimensions, and affine lattice parameters.

### Fields

- `_comment`: Optional explanatory comment.
- `grid_version`: Must be `1` for canonical v1.
- `image_full`: `[width, height]` of the full image/canvas.
- `n_cols`: Number of hex columns.
- `n_rows`: Number of hex rows.
- `x_intercept_col0`: X coordinate for column 0 centers.
- `col_pitch_x`: Horizontal center-to-center distance between adjacent columns.
- `y_intercept_row0`: Y coordinate for row 0 before even-column offset.
- `row_pitch_y`: Vertical center-to-center distance between adjacent rows in a column.
- `even_col_y_offset`: Y offset added to even-numbered columns.

### Invariants

Every valid hex code must decode to a column and row within bounds. Pixel centers are derived, not stored per hex.

---

## 4. `terrain.json`

### Purpose

Per-hex terrain classification.

### Producing tools

hexwright and wargame-map-parser classification pipelines.

### Consuming tools

hexwright, game engines, movement/combat rules modules, map renderers.

### Top-level shape

Object with a `terrain` dictionary.

### Fields

- `_comment`: Optional explanatory comment.
- `terrain`: Object mapping canonical hex IDs to terrain class strings.

### Semantics

Terrain class strings are game-defined. The demo uses values such as `clear`, `woods`, `rough`, `swamp`, `water`, `urban`, and `woods_swamp`; those are examples, not a universal vocabulary.

### Invariants

Each terrain key must be a valid hex ID. A complete game map normally has one terrain value for every in-bounds hex. Sparse terrain files are shape-valid but consumers may reject them if their rules require complete terrain coverage.

---

## 5. `hexsides.json`

### Purpose

Grouped hexside and connection layers: rivers, roads, rails, bridges, and game-specific equivalent edge layers.

### Producing tools

hexwright and promoted parser traces.

### Consuming tools

hexwright, game engines, movement/combat rules modules, renderers.

### Top-level shape

Object whose non-comment properties are named layer arrays. Each layer array contains edge objects.

Example canonical edge:

```json
{ "a": "0603", "b": "0604" }
```

### Fields

- `_comment`: Optional explanatory comment.
- `rivers`: Optional array of river hexside edges.
- `roads`: Optional array of road edges.
- `rails`: Optional array of rail edges.
- `bridges`: Optional array of bridge edges.
- Other layer names: Allowed if they match the layer-name pattern and use the same edge-array shape.

### Semantics

Layer names are game-defined. The canonical v1 shape is grouped by layer name, not by a single typed edge list.

Layers are independent. A single pair may appear in multiple layers; for example, a road, river, and bridge may all refer to the same adjacent hex pair.

### Invariants

For every edge:

- `a` and `b` are valid in-bounds hex IDs;
- `a` and `b` are adjacent under the grid convention;
- `a < b`;
- no duplicate pair appears within the same layer.

---

## 6. `features.json`

### Purpose

Named point features attached to hexes: cities, forts, objectives, ports, capitals, scenario locations, and similar map features.

### Producing tools

hexwright and promoted feature tracing/classification pipelines.

### Consuming tools

hexwright, game engines, scenario setup, victory logic, renderers.

### Top-level shape

Object with a `features` array.

### Feature fields

- `code`: Canonical hex ID where the feature is located.
- `type`: Game-defined feature type string.
- `name`: Human-readable feature name.
- `attrs`: Object containing feature-specific attributes.

### Semantics

`attrs` is the extension point for game-specific data. Examples include victory points, build points, port flags, fort strengths, and objective values.

### Invariants

Feature records have no separate canonical `id` field in v1. For deduplication, tools should treat `(code, type, name)` as the feature identity unless a game adapter defines a stricter rule.

---

## 7. `names.json`

### Purpose

A simple display-name overlay for hexes.

### Producing tool

hexwright.

### Consuming tools

hexwright, game engines, map labels, setup tools.

### Top-level shape

Object with a `names` dictionary.

### Fields

- `names`: Object mapping canonical hex IDs to display strings.

### Semantics

`names.json` is intentionally simpler than `features.json`. It labels hexes whether or not those hexes have typed features. A location may appear in both `features.json` and `names.json`.

### Invariants

Each key must be a valid in-bounds hex ID. Names should be unique when the game requires unambiguous lookup, but the file shape permits repeated display strings.

---

## 8. Forces trio: `oob.json`, `assets.json`, `sp_pools.json`

The musterwright force format is split into three files:

1. `oob.json` — primary units;
2. `assets.json` — supporting assets;
3. `sp_pools.json` — strength-point pools or replacement pools.

Faction codes are top-level keys. The examples use `ALD` and `VER`; any canonical faction key should be a stable uppercase code.

### 8.1 `oob.json`

#### Purpose

Order of battle spine: primary maneuver units grouped by faction.

#### Producing tool

musterwright.

#### Consuming tools

musterwright, game adapters, scenario setup, game engines, counter renderers.

#### Top-level shape

Object with faction-code keys containing arrays of unit records. Optional note keys use `_note_<FACTION>`.

#### Unit fields

- `id`: Unit identifier, unique within the force set.
- `faction`: Faction code. Must match the containing faction key.
- `sp`: Strength points.
- `dv`: Defense value.
- `mp`: Movement allowance.
- `type`: Game-defined unit type, such as `inf` or `cav`.
- `size`: Game-defined echelon/size code.
- `fresh`: Optional boolean marker for scenario state.

#### Invariants

Unit IDs must be unique across all faction arrays in `oob.json`. Game adapters may additionally require uniqueness across `oob.json` and `assets.json`.

### 8.2 `assets.json`

#### Purpose

Supporting units or assets grouped by faction.

#### Producing tool

musterwright.

#### Consuming tools

musterwright, game adapters, game engines, counter renderers.

#### Top-level shape

Object with faction-code keys containing arrays of asset records.

#### Asset fields

- `id`: Asset identifier.
- `faction`: Faction code. Must match the containing faction key.
- `av`: Attack or asset value.
- `dv`: Defense value.
- `mp`: Movement allowance.
- `type`: Game-defined asset type, such as `art`.
- `size`: Game-defined size code.

#### Invariants

Asset IDs must be unique within `assets.json`; consumers may require uniqueness across the whole force trio.

### 8.3 `sp_pools.json`

#### Purpose

Faction-level pools of unassigned strength points or replacement capacity.

#### Producing tool

musterwright.

#### Consuming tools

musterwright, game adapters, replacement/reinforcement logic, scenario setup.

#### Top-level shape

Object with faction-code keys containing pool records.

#### Pool fields

- `sp_count`: Number of strength points in the pool.
- `av_per_sp`: Attack/asset value represented by each strength point.
- `notes`: Optional explanatory note.

#### Invariants

Pool faction keys should correspond to faction keys used by `oob.json` or `assets.json`.

---

## 9. Point-to-point maps: `nodes.json` and `edges.json`

### Purpose

Canonical v1 shape for non-hex point-to-point maps.

### Producing tools

wargame-map-parser point extraction pipelines, map editors, hand-authored adapters.

### Consuming tools

game engines, route/pathfinding modules, renderers.

### `nodes.json` top-level shape

Object with a `nodes` array.

Node fields:

- `id`: Stable node identifier.
- `name`: Optional display name.
- `x`: Image-space X coordinate.
- `y`: Image-space Y coordinate.
- `attrs`: Optional game-specific attributes.

### `edges.json` top-level shape

Object with an `edges` array.

Edge fields:

- `a`: Source or first endpoint node ID.
- `b`: Target or second endpoint node ID.
- `type`: Optional game-defined edge type.
- `directed`: Optional boolean. Absent or `false` means undirected.
- `attrs`: Optional game-specific attributes.

### Invariants

Node IDs are unique. Every edge endpoint references an existing node ID.

For undirected edges, `a < b` and consumers must treat the connection symmetrically. For directed edges, order is meaningful: `a` is the source and `b` is the target.

---

## 10. Legacy variants

These variants are known historical shapes. New tools write canonical v1 only. Readers may support adapters, but shipped data should not be retrofitted solely to make it look canonical.

### `terrain-variant-a`: bare terrain map

Shape delta:

```json
{
  "0000": "clear",
  "0001": "woods"
}
```

instead of canonical:

```json
{
  "terrain": {
    "0000": "clear",
    "0001": "woods"
  }
}
```

Canonical v1 requires the `terrain` wrapper.

### `terrain-variant-b`: terrain record list

Shape delta:

```json
{
  "hexes": [
    { "code": "0000", "terrain": "clear" }
  ]
}
```

instead of a dictionary keyed by hex ID. This requires consumers to check duplicate `code` values manually. Canonical v1 uses a hex-ID dictionary.

### `hexside-variant-a`: typed edge list

Shape delta:

```json
{
  "hexsides": [
    { "type": "river", "a": "0003", "b": "0004" }
  ]
}
```

instead of canonical grouped layer arrays:

```json
{
  "rivers": [
    { "a": "0003", "b": "0004" }
  ]
}
```

Canonical v1 groups edges by layer name.

### `hexside-variant-b`: pair-key maps

Shape delta:

```json
{
  "rivers": {
    "0003|0004": true
  }
}
```

instead of canonical edge objects. Canonical v1 does not encode endpoints inside object keys.

### `hexside-variant-c`: origin-side encoding

Shape delta:

```json
{
  "rivers": [
    { "code": "0003", "side": "S" }
  ]
}
```

or equivalent per-hex side flags. This depends on an orientation-specific side-to-neighbor expansion. Canonical v1 stores the resolved adjacent hex pair directly as `{ "a": "...", "b": "..." }`.

---

# JSON Schemas

## `project.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/lerugray/wright-kit/schemas/v1/project.schema.json",
  "title": "wright-kit project.json v1",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "name",
    "imageFull",
    "hexgrid",
    "terrain",
    "hexsides",
    "features",
    "names",
    "blankLattice"
  ],
  "properties": {
    "name": { "type": "string", "minLength": 1 },
    "_comment": { "type": "string" },
    "imageFull": {
      "type": "array",
      "minItems": 2,
      "maxItems": 2,
      "items": { "type": "integer", "minimum": 1 }
    },
    "hexgrid": { "type": "string", "minLength": 1 },
    "terrain": { "type": "string", "minLength": 1 },
    "hexsides": { "type": "string", "minLength": 1 },
    "features": { "type": "string", "minLength": 1 },
    "names": { "type": "string", "minLength": 1 },
    "blankLattice": { "type": "boolean" }
  }
}
```

## `hexgrid.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/lerugray/wright-kit/schemas/v1/hexgrid.schema.json",
  "title": "wright-kit hexgrid.json v1",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "grid_version",
    "image_full",
    "n_cols",
    "n_rows",
    "x_intercept_col0",
    "col_pitch_x",
    "y_intercept_row0",
    "row_pitch_y",
    "even_col_y_offset"
  ],
  "properties": {
    "_comment": { "type": "string" },
    "grid_version": { "const": 1 },
    "image_full": {
      "type": "array",
      "minItems": 2,
      "maxItems": 2,
      "items": { "type": "integer", "minimum": 1 }
    },
    "n_cols": { "type": "integer", "minimum": 1, "maximum": 100 },
    "n_rows": { "type": "integer", "minimum": 1, "maximum": 100 },
    "x_intercept_col0": { "type": "number" },
    "col_pitch_x": { "type": "number", "exclusiveMinimum": 0 },
    "y_intercept_row0": { "type": "number" },
    "row_pitch_y": { "type": "number", "exclusiveMinimum": 0 },
    "even_col_y_offset": { "type": "number" }
  }
}
```

## `terrain.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/lerugray/wright-kit/schemas/v1/terrain.schema.json",
  "title": "wright-kit terrain.json v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["terrain"],
  "properties": {
    "_comment": { "type": "string" },
    "terrain": {
      "type": "object",
      "additionalProperties": false,
      "patternProperties": {
        "^[0-9]{4}$": { "type": "string", "minLength": 1 }
      }
    }
  }
}
```

## `hexsides.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/lerugray/wright-kit/schemas/v1/hexsides.schema.json",
  "title": "wright-kit hexsides.json v1",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "_comment": { "type": "string" }
  },
  "patternProperties": {
    "^[A-Za-z][A-Za-z0-9_]*$": {
      "type": "array",
      "items": { "$ref": "#/$defs/edge" }
    }
  },
  "$defs": {
    "hexCode": {
      "type": "string",
      "pattern": "^[0-9]{4}$"
    },
    "edge": {
      "type": "object",
      "additionalProperties": false,
      "required": ["a", "b"],
      "properties": {
        "a": { "$ref": "#/$defs/hexCode" },
        "b": { "$ref": "#/$defs/hexCode" }
      }
    }
  }
}
```

## `features.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/lerugray/wright-kit/schemas/v1/features.schema.json",
  "title": "wright-kit features.json v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["features"],
  "properties": {
    "_comment": { "type": "string" },
    "features": {
      "type": "array",
      "items": { "$ref": "#/$defs/feature" }
    }
  },
  "$defs": {
    "hexCode": {
      "type": "string",
      "pattern": "^[0-9]{4}$"
    },
    "jsonValue": {
      "oneOf": [
        { "type": "null" },
        { "type": "boolean" },
        { "type": "number" },
        { "type": "string" },
        {
          "type": "array",
          "items": { "$ref": "#/$defs/jsonValue" }
        },
        {
          "type": "object",
          "additionalProperties": { "$ref": "#/$defs/jsonValue" }
        }
      ]
    },
    "feature": {
      "type": "object",
      "additionalProperties": false,
      "required": ["code", "type", "name", "attrs"],
      "properties": {
        "code": { "$ref": "#/$defs/hexCode" },
        "type": { "type": "string", "minLength": 1 },
        "name": { "type": "string", "minLength": 1 },
        "attrs": {
          "type": "object",
          "additionalProperties": { "$ref": "#/$defs/jsonValue" }
        }
      }
    }
  }
}
```

## `names.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/lerugray/wright-kit/schemas/v1/names.schema.json",
  "title": "wright-kit names.json v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["names"],
  "properties": {
    "names": {
      "type": "object",
      "additionalProperties": false,
      "patternProperties": {
        "^[0-9]{4}$": { "type": "string", "minLength": 1 }
      }
    }
  }
}
```

## `oob.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/lerugray/wright-kit/schemas/v1/oob.schema.json",
  "title": "wright-kit oob.json v1",
  "type": "object",
  "additionalProperties": false,
  "patternProperties": {
    "^[A-Z][A-Z0-9_]*$": {
      "type": "array",
      "items": { "$ref": "#/$defs/unit" }
    },
    "^_note_[A-Z][A-Z0-9_]*$": {
      "type": "string"
    }
  },
  "$defs": {
    "unit": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "faction", "sp", "dv", "mp", "type", "size"],
      "properties": {
        "id": { "type": "string", "minLength": 1 },
        "faction": { "type": "string", "minLength": 1 },
        "sp": { "type": "integer", "minimum": 0 },
        "dv": { "type": "integer", "minimum": 0 },
        "mp": { "type": "integer", "minimum": 0 },
        "type": { "type": "string", "minLength": 1 },
        "size": { "type": "string", "minLength": 1 },
        "fresh": { "type": "boolean" }
      }
    }
  }
}
```

## `assets.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/lerugray/wright-kit/schemas/v1/assets.schema.json",
  "title": "wright-kit assets.json v1",
  "type": "object",
  "additionalProperties": false,
  "patternProperties": {
    "^[A-Z][A-Z0-9_]*$": {
      "type": "array",
      "items": { "$ref": "#/$defs/asset" }
    }
  },
  "$defs": {
    "asset": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "faction", "av", "dv", "mp", "type", "size"],
      "properties": {
        "id": { "type": "string", "minLength": 1 },
        "faction": { "type": "string", "minLength": 1 },
        "av": { "type": "integer", "minimum": 0 },
        "dv": { "type": "integer", "minimum": 0 },
        "mp": { "type": "integer", "minimum": 0 },
        "type": { "type": "string", "minLength": 1 },
        "size": { "type": "string", "minLength": 1 }
      }
    }
  }
}
```

## `sp_pools.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/lerugray/wright-kit/schemas/v1/sp_pools.schema.json",
  "title": "wright-kit sp_pools.json v1",
  "type": "object",
  "additionalProperties": false,
  "patternProperties": {
    "^[A-Z][A-Z0-9_]*$": {
      "type": "object",
      "additionalProperties": false,
      "required": ["sp_count", "av_per_sp"],
      "properties": {
        "sp_count": { "type": "integer", "minimum": 0 },
        "av_per_sp": { "type": "number", "minimum": 0 },
        "notes": { "type": "string" }
      }
    }
  }
}
```

## `nodes.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/lerugray/wright-kit/schemas/v1/nodes.schema.json",
  "title": "wright-kit point-to-point nodes.json v1",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "nodes"
  ],
  "properties": {
    "_comment": {
      "type": "string"
    },
    "nodes": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/node"
      }
    },
    "meta": {
      "type": "object",
      "description": "Optional tool/provenance metadata; not interpreted by consumers."
    }
  },
  "$defs": {
    "jsonValue": {
      "oneOf": [
        {
          "type": "null"
        },
        {
          "type": "boolean"
        },
        {
          "type": "number"
        },
        {
          "type": "string"
        },
        {
          "type": "array",
          "items": {
            "$ref": "#/$defs/jsonValue"
          }
        },
        {
          "type": "object",
          "additionalProperties": {
            "$ref": "#/$defs/jsonValue"
          }
        }
      ]
    },
    "node": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "id",
        "x",
        "y"
      ],
      "properties": {
        "id": {
          "type": "string",
          "minLength": 1
        },
        "name": {
          "type": "string",
          "minLength": 1
        },
        "x": {
          "type": "number"
        },
        "y": {
          "type": "number"
        },
        "attrs": {
          "type": "object",
          "additionalProperties": {
            "$ref": "#/$defs/jsonValue"
          }
        }
      }
    }
  }
}
```

## `edges.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/lerugray/wright-kit/schemas/v1/edges.schema.json",
  "title": "wright-kit point-to-point edges.json v1",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "edges"
  ],
  "properties": {
    "_comment": {
      "type": "string"
    },
    "edges": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/edge"
      }
    },
    "meta": {
      "type": "object",
      "description": "Optional tool/provenance metadata; not interpreted by consumers."
    }
  },
  "$defs": {
    "jsonValue": {
      "oneOf": [
        {
          "type": "null"
        },
        {
          "type": "boolean"
        },
        {
          "type": "number"
        },
        {
          "type": "string"
        },
        {
          "type": "array",
          "items": {
            "$ref": "#/$defs/jsonValue"
          }
        },
        {
          "type": "object",
          "additionalProperties": {
            "$ref": "#/$defs/jsonValue"
          }
        }
      ]
    },
    "edge": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "a",
        "b"
      ],
      "properties": {
        "a": {
          "type": "string",
          "minLength": 1
        },
        "b": {
          "type": "string",
          "minLength": 1
        },
        "type": {
          "type": "string",
          "minLength": 1
        },
        "directed": {
          "type": "boolean"
        },
        "attrs": {
          "type": "object",
          "additionalProperties": {
            "$ref": "#/$defs/jsonValue"
          }
        }
      }
    }
  }
}
```

---

# Reconciliation notes

- The exemplars define canonical v1 shape wherever they are present; prose conventions are interpreted through those files.
- `hexgrid.json` uses `grid_version`, while the other canonical v1 files have no required version field; schemas preserve that asymmetry.
- `project.json` uses `imageFull`; `hexgrid.json` uses `image_full`; schemas preserve the exemplar casing.
- Canonical `terrain.json` is the wrapped `{ "terrain": { ... } }` form; bare terrain maps are legacy only.
- Canonical `hexsides.json` is the grouped layer bundle; typed edge lists, pair-key maps, and origin-side encodings are legacy only.
- Hexside layer names are open by pattern so games can define additional layers, but every layer must use canonical `{ "a", "b" }` edge objects.
- `features.attrs` is required because every canonical demo feature has it; features with no extra data should use an empty object.
- `names.json` has no `_comment` in the exemplar, so the schema keeps it closed to only `names`.
- JSON Schema cannot enforce in-bounds hex references, pair adjacency, pair ordering, duplicate pair keys, or cross-file completeness; those are semantic validation invariants.
- JSON Schema cannot enforce that a unit’s `faction` field equals its containing faction key; that is a musterwright/game-adapter validation invariant.
- JSON Schema cannot enforce global unit ID uniqueness across faction arrays or across the forces trio; tools must validate it separately.
- `oob.json` and `assets.json` intentionally have separate record shapes: OOB units use `sp`; assets use `av`.
- Faction keys are constrained to uppercase code-style identifiers to match the attached musterwright exemplars.
- `sp_pools.av_per_sp` is numeric rather than integer because the exemplar uses `5.5`.
- The point-to-point `nodes.json` and `edges.json` schemas have no attached exemplar; they define the narrow v1 interoperable minimum for that pair.
- The four-digit hex code convention limits canonical v1 schemas to columns and rows `00` through `99`; larger maps need a future convention or schema version.

---

*Validation note (2026-07-10): every schema in this document was machine-validated
(JSON Schema draft 2020-12) against the kit's real exemplar files — hexwright's
bundled demo and musterwright's example project — plus hexwright's point-to-point
test fixtures. The `nodes`/`edges` schemas were corrected after validation to
allow the optional top-level `meta` object the real fixtures carry (exemplar
wins). Machine-usable copies live in `schemas/v1/`.*
