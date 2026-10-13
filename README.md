# adqlduck

> A tested ADQL 2.1 → DuckDB SQL compiler with HEALPix-pruned sky geometry, plus a VOSI `tables` / `capabilities` / `TAP_SCHEMA` generator driven by the same feature registry. One Rust core, shipped to the browser (npm, WASM) and to Python (`uvx`). Runs on stock DuckDB; uses the `healpix` extension when it is loaded.

Type A spec (agent implements from this file). Working name `adqlduck` is free on PyPI, npm, and crates.io (checked 2026-10-08). Companion repo: [ejoliet/healpix-duckdb](https://github.com/ejoliet/healpix-duckdb) (optional backend, Gate 0 done 2026-10-09).

---

## Purpose

**Problem.** sidetap (browser), tapdrop and obstap-local (Python) all run DuckDB behind a TAP surface. Each needs the same hard part: turning ADQL into DuckDB SQL, with sky geometry DuckDB cannot evaluate (core `spatial` is planar), and advertising exactly what it supports in VOSI. Writing it three times means three sets of bugs.

**Solution.** One library that:

1. Parses ADQL 2.1 and rejects invalid queries with ADQL-level error messages.
2. Rewrites geometry into a HEALPix range prefilter (prunes Parquet row groups and HATS partitions) plus an exact spherical predicate in plain SQL.
3. Emits DuckDB SQL through one of two backends:
   - **`plain`** (default): prelude macros + range predicates. Runs identically in DuckDB native and DuckDB-WASM with no extension.
   - **`healpix`**: calls the `healpix` extension (`sky_in_cone`, `sky_in_polygon`, `moc_*`). Adds non-convex polygons, MOC-valued regions, and bucketed cross-match.
4. Generates VOSI `tables`, VOSI `capabilities` (TAPRegExt), `TAP_SCHEMA` DDL, and TAP error documents from one catalog descriptor and one feature registry, so the service never advertises a feature the compiler rejects. The registry is per backend.
5. Generates `docs/adql-mapping.md` (ADQL construct → SQL, per backend) from the compiler's own rule table. CI fails if the committed doc differs. This doc is the product other people read; it cannot drift from the code or from the extension.

**Consumers.**

| Consumer | Binding | Uses |
|---|---|---|
| sidetap (service-worker TAP for Firefly, Aladin Lite) | npm / WASM | `compile`, `vosi*`, HATS partition hints |
| tapdrop, obstap-local (local TAP for pyvo, TOPCAT, Aladin desktop) | Python wheel | same, plus `catalog from-hats` |
| Anyone with Parquet + DuckDB | CLI | `adqlduck compile`, `demo-serve` |

The service-worker reach limit (same-origin, in-tab only) is a consumer concern. This library has no transport.

---

## Relationship to healpix-duckdb

Two layers, one calls the other. Neither depends on the other to be useful.

| | healpix-duckdb | adqlduck |
|---|---|---|
| Kind | DuckDB extension (Rust, C API) | Compiler + metadata generator (Rust core; Python wheel; npm/WASM) |
| Input | SQL function calls | ADQL text + catalog descriptor |
| Output | Values at query time | SQL text + VOSI XML at compile time |
| Knows ADQL / TAP? | No. Usable from plain SQL by anyone | Yes; that is the product |
| Runs without the other? | Yes | Yes, via the `plain` backend |
| Oracles | MOCPy, astropy-healpix, pgSphere/Q3C, lsdb | astropy brute force, lyonetia corpus, taplint |

Division of responsibility:

- **adqlduck owns semantics of the language**: parsing, validation, feature registry, name resolution, `capabilities`, the generated `docs/adql-mapping.md`.
- **healpix-duckdb owns primitives DuckDB lacks**: pixel functions, the MOC type and set operations, exact spherical predicates, `hats_partitions`.
- **HEALPix coverage appears in both, at different moments.** adqlduck computes it at compile time with `cdshealpix` and emits constant `_healpix_29 BETWEEN` ranges, because constants are what DuckDB pushes into Parquet row-group pruning. The extension computes it at query time (`moc_from_cone`) when the region is a column value, not a literal. Same math, different moment.
- **Contract**: pinned on the extension side by `test/sql/contract_adqlduck.test`, on the compiler side by the generated mapping doc and the backend-parity tests.

---

## Prior art (why this is still worth building)

"Nobody has shipped this" is too strong. The defensible pitch is **"ADQL for DuckDB, in the browser and in Python, with conformance tests and generated VOSI."**

| Project | What it does | Gap for us |
|---|---|---|
| [Caltech-IPAC/ADQL](https://github.com/Caltech-IPAC/ADQL) (`adql` on PyPI, v1.0.6) | ADQL → local SQL with HTM/HEALPix index constraints. Pure Python, Oracle and SQLite targets. | Needs precomputed x/y/z + index columns and a C spatial-index module. Token rewriting, not a full AST. No DuckDB, no WASM, no VOSI. |
| [aipescience/queryparser](https://github.com/aipescience/queryparser) (15★, active) | ANTLR-based ADQL → PostgreSQL/MySQL (pg_sphere). | Server-DB targets. No browser path. |
| VOLLT ADQL library (Java) | Reference ADQL 2.1 parser with a supported-feature registry and PgSphere translator. | JVM. Feature-registry idea is worth copying. |
| DuckDB community `astro` extension | 62 functions incl. angular separation and `read_fits` via cfitsio. | Not ADQL. Useful for FITS ingest in tapdrop (see Non-Goals). |
| DuckDB community `geography` (S2, spherical) | True spherical geometry. | Excluded from WASM builds, so not portable. |

> **Warning.** Caltech-IPAC/ADQL is your employer's library. Before going public, check Caltech's outside-work / IP policy for a side project in the same space, and decide whether to cite it as prior art or coordinate with its maintainers (Open Question Q5).

---

## Architecture

```
ADQL text
  └─ lexer ─ parser (ADQL 2.1 BNF) ─▶ AST
       └─ resolver  (Catalog: relations, columns, sky indexes, TAP_UPLOAD) ─▶ resolved AST
            └─ validator (FeatureSet: reject anything not registered) 
                 └─ rewriter
                 │    ├─ geometry: constant-fold literals; CONTAINS/INTERSECTS/DISTANCE
                 │    │            → HEALPix range prefilter + exact predicate
                 │    ├─ TOP/OFFSET → LIMIT/OFFSET; MAXREC → LIMIT maxrec+1
                 │    └─ function + operator mapping
                 └─ emitter ─┬─ backend plain   (prelude macros, no extension)
                             └─ backend healpix (healpix extension functions)
                                  ─▶ CompiledQuery { sql, prelude, columns, spatial_hints, features_used }

Catalog + FeatureSet(backend) ─▶ vosi::tables_xml | vosi::capabilities_xml | tap_schema::sql | vosi::error_votable
rules table ─▶ xtask mapping-doc ─▶ docs/adql-mapping.md (generated; CI diff check)
```

| Component | Responsibility |
|---|---|
| `lexer`, `parser` | ADQL 2.1 grammar only. Hand-written recursive descent. Positions on every node. |
| `catalog` | ADQL name → DuckDB relation, column metadata, per-table sky index (`ra`, `dec`, HEALPix column + order). |
| `features` | Single registry of supported ADQL features, per backend. Drives the validator **and** `capabilities`. |
| `rules` | One table of translation rules: id, ADQL example, feature, backends supported. Drives the mapping doc and the test-coverage check (every rule has a test). |
| `geom` | Spherical math, polygon checks, HEALPix coverage → merged depth-29 ranges (via `cdshealpix`). |
| `functions` | ADQL function/operator → DuckDB mapping table. Each row has an execution test. |
| `emit` | Quoted, canonical DuckDB SQL. `plain` backend: prelude macros (`CREATE OR REPLACE MACRO`). `healpix` backend: extension calls, empty prelude. |
| `vosi` | VOSI tables, TAPRegExt capabilities, TAP_SCHEMA DDL+rows, TAP error VOTable. |
| `adqlduck-wasm` | `wasm-bindgen` API, JSON in/out. |
| `adqlduck-py` + `python/` | PyO3 module, Typer-free stdlib `argparse` CLI, HATS helpers, `demo-serve`. |

> **Rationale.** The compiler never reads data and never builds `FROM read_parquet(...)` itself. The consumer registers views; the catalog only names them. That keeps arbitrary SQL out of the catalog and makes the emitted SQL auditable.

---

## Translation rules (the core contract)

### Geometry

All coordinates ICRS, degrees. A frame argument of `'ICRS'`, `''`, or omitted (2.1 style) is accepted. Anything else → `UnsupportedFeature`.

| ADQL | DuckDB output |
|---|---|
| `1 = CONTAINS(POINT(ra,dec), CIRCLE(a,d,r))` on a sky-indexed table, literal circle | `(<healpix ranges>) AND adql_sep_deg(ra,dec,a,d) <= r` |
| same, circle from another table's columns (cross-match) | `g.dec BETWEEN u.dec - r AND u.dec + r AND adql_sep_deg(...) <= r` (dec band lets DuckDB use a range join) |
| `1 = CONTAINS(POINT(ra,dec), POLYGON(...))`, literal convex polygon | ranges `AND` one unrolled half-space test per edge: `adql_dot(ra,dec,nx,ny,nz) >= 0` (edge normals computed at compile time) |
| `CONTAINS(POINT(...), s_region)` with a polygon-valued column (`DOUBLE[]`, ObsCore) | `adql_in_convex_poly(ra, dec, s_region)` macro (list comprehension), no prefilter |
| `DISTANCE(p1, p2)` and 4-arg `DISTANCE(ra1,dec1,ra2,dec2)` | `adql_sep_deg(...)` |
| `INTERSECTS(circle, point)` / `(point, circle)` | same as CONTAINS |
| `INTERSECTS(circle, circle)` | `adql_sep_deg(c1, c2) <= r1 + r2` |
| `CONTAINS(...)` / `INTERSECTS(...)` used as a value | `CASE WHEN <pred> THEN 1 ELSE 0 END` |
| `COORD1(p)`, `COORD2(p)` | the underlying ra / dec expression |
| `AREA`, `CENTROID` on literals | constant-folded at compile time |
| `POINT/CIRCLE/POLYGON` in SELECT | `DOUBLE[]` in DALI order, with `xtype` point/circle/polygon in `columns` |

Prefilter rules:

- Coverage from `cdshealpix` nested cone/polygon coverage. Start at the depth where cell size ≈ radius/4; coarsen until merged ranges ≤ `max_healpix_ranges` (default 32).
- Ranges are emitted at the table's HEALPix order (HATS: `_healpix_29`, nested): `idx BETWEEN lo AND hi`.
- **Invariant:** prefilter is a superset. The exact predicate decides. Zero false negatives is tested, not assumed.
- Ranges are also returned in `spatial_hints` so consumers can pick HATS partition files before DuckDB opens anything. This is the main byte saving over HTTP.

Prelude macros (emitted once per connection, idempotent):

```sql
-- AIDEV-NOTE: least(1.0, ...) guards asin() against rounding above 1 for antipodal points
-- AIDEV-INVARIANT: dec outside [-90, 90] is off the sphere -> NULL, so the row is excluded.
--   Without the guard, trig periodicity maps a sentinel like dec = -999 onto a real
--   declination and the row can silently match a cone.
CREATE OR REPLACE MACRO adql_sep_deg(ra1, dec1, ra2, dec2) AS
  CASE WHEN dec1 BETWEEN -90 AND 90 AND dec2 BETWEEN -90 AND 90 THEN
    degrees(2 * asin(least(1.0, sqrt(
      pow(sin(radians(dec2 - dec1) / 2), 2)
      + cos(radians(dec1)) * cos(radians(dec2)) * pow(sin(radians(ra2 - ra1) / 2), 2)
    ))))
  END;

CREATE OR REPLACE MACRO adql_dot(ra, dec, nx, ny, nz) AS
  nx * cos(radians(dec)) * cos(radians(ra))
  + ny * cos(radians(dec)) * sin(radians(ra))
  + nz * sin(radians(dec));
```

Polygon checks at compile time: ≥ 3 distinct vertices, no self-intersection, convex. Orientation is normalized (either winding accepted for literals). Non-convex → `UnsupportedGeometry` in the `plain` backend; supported by the `healpix` backend via `sky_in_polygon` (ADQL 2.1 hemisphere rule).

### Backends

The `Geometry` table above is the `plain` backend. The `healpix` backend changes only the exact predicate and adds constructs. Pruning ranges are computed at compile time in **both** backends, because constant `BETWEEN` predicates are what DuckDB pushes into Parquet row-group statistics.

| ADQL | `plain` | `healpix` |
|---|---|---|
| `CONTAINS(POINT, CIRCLE)` literal | ranges + `adql_sep_deg(...) <= r` | ranges + `sky_in_cone(ra, dec, ra0, dec0, r)` |
| `CONTAINS(POINT, POLYGON)` literal | ranges + unrolled `adql_dot` (convex only) | ranges + `sky_in_polygon(ra, dec, [ra...], [dec...])` (concave allowed) |
| `DISTANCE(...)` | `adql_sep_deg` (haversine) | `sky_distance` (Vincenty) |
| Cross-match `CONTAINS(POINT(g), CIRCLE(u, r))` | dec-band range join + exact | join on `g._healpix_29` ∈ `unnest(moc_ranges(moc_from_cone(u.ra, u.dec, r)))` + `sky_in_cone` |
| Region column stored as MOC (`s_moc`) | `UnsupportedFeature` | `moc_contains(s_moc, hpx29(ra, dec))` |
| Region column stored as DALI polygon `DOUBLE[]` | `adql_in_convex_poly` macro | `sky_in_polygon` with the flat array split into ra/dec lists in SQL |

Rules for the `healpix` backend:

- **Selection is explicit.** The compiler does no I/O, so the consumer passes `backend`. Python and JS wrappers provide `detect_backend(con)`, which probes `SELECT hpx29(0.0, 0.0)` and returns `plain` on any error.
- **Exact DuckDB version.** The extension is built with the unstable C API (`C_STRUCT_UNSTABLE`) and loads only on the DuckDB version it was built for (v1.5.6 at Gate 0, see its ADR-0003). sidetap must ship a duckdb-wasm build that embeds the same DuckDB version. A mismatch is a load error, so `detect_backend` falls back to `plain`.
- **Interval convention.** The extension's `moc_ranges` and `hpx_range29` are half-open `[lo, hi)`. `SpatialHint.ranges` here are inclusive `[lo, hi]`. Convert at that one boundary (`hi - 1`), with an `AIDEV-INVARIANT` comment.
- **Boundary precision.** Haversine and Vincenty can disagree on points within ~1e-12° of a circle edge. Backend parity tests exclude points closer than 1e-9° to any boundary; the inclusive-boundary behavior itself is tested per backend against its own oracle.
- **Runtime errors.** Extension errors all arrive as DuckDB `InvalidInputException` (C API limitation, its ADR-0005). `healpix: internal error` → TAP 500; any other `healpix:` message → TAP 400 with the message.
- **Off-sphere rows.** The `plain` backend excludes rows with `dec` outside [-90, 90] (macro returns NULL). With `healpix_strict_input = true` (the extension default, its ADR-0007), the same row fails the whole query. The backends must agree; Open Question Q7.

### Non-spatial

| ADQL | DuckDB | Note |
|---|---|---|
| `SELECT TOP n` | `LIMIT n` | with `OFFSET` (2.1) → `LIMIT n OFFSET m` |
| MAXREC (option) | outer `LIMIT maxrec+1` | consumer detects overflow |
| regular identifiers | canonical catalog name, double-quoted | DuckDB matches case-insensitively even when quoted; resolver rejects catalogs with names differing only by case |
| `LOG(x)` | `ln(x)` | ADQL LOG is natural log |
| `LOG10(x)` | `log10(x)` | |
| `TRUNCATE(x, n)`, `ROUND(x, n)`, `MOD`, `RAND(seed)` | per mapping table | each verified by an execution test against real DuckDB |
| `a ^ b` (2.1 bitwise XOR) | `xor(a, b)` | DuckDB `^` is power. Classic trap |
| `&`, `\|`, `~` | same | |
| `int / int` | `//` or `/` | Q2: default truncating `//`, matching PostgreSQL-backed TAP services |
| `ILIKE`, `LOWER`, `UPPER`, `\|\|` | native | |
| `WITH`, `UNION/INTERSECT/EXCEPT`, `CASE`, `COALESCE`, `CAST` (ADQL types) | native | |
| `TAP_SCHEMA.*` | DuckDB schema `TAP_SCHEMA` | created by `tap_schema::sql` |
| `TAP_UPLOAD.t` | catalog entry under schema `TAP_UPLOAD` | consumer registers the table, compiler resolves it |

---

## Recommended Stack

Stars and push dates pulled from GitHub on 2026-10-08 where the API answered; "—" means not retrieved.

| Layer | Chosen | Signal | Why | Rejected |
|---|---|---|---|---|
| Core language | Rust | — | One core → WASM and Python; `cdshealpix` is native Rust | Python + sqlglot (browser needs Pyodide; no confirmed HEALPix lib there). TypeScript (Python consumers need a port) |
| Parser | Hand-written recursive descent | — | Must **reject** ~80 invalid conformance queries with precise errors; ADQL grammar is small | sqlparser-rs (3462★; accepts far more than ADQL), chumsky (4531★; heavy generics, slow builds), pest |
| HEALPix | `cdshealpix` (cds-astro/cds-healpix-rust) | 33★, pushed 2026-09 | CDS-maintained, built for WASM and Python | healpy (no WASM), astropy-healpix (Python/C) |
| WASM bindings | `wasm-bindgen` + `wasm-pack` | 9179★, active | Standard | — |
| Python bindings | PyO3 + `maturin` | 5829★, active | Mixed Rust/Python project, works with `uv` | cffi |
| Snapshot tests | `insta` | — | Golden SQL per rule | hand-rolled |
| Test oracle | `duckdb` + `astropy` + `hypothesis` (Python) | — | Real execution and independent sky math | `duckdb-rs` bundled (slow C++ build in CI) |
| Conformance corpus | [ivoa/lyonetia](https://github.com/ivoa/lyonetia) | ~495 queries (413 valid, 82 invalid, ADQL 2.1) | IVOA test set, also used to validate the ADQL 2.1 PR | — |

> **Warning.** lyonetia is GPLv3. Fetch it at test time (`corpus/fetch.sh`). Never vendor it into this MIT repo.

---

## Repository Layout

```
adqlduck/
├── Cargo.toml                    # workspace
├── crates/
│   ├── adqlduck/                 # core, no I/O
│   │   └── src/
│   │       ├── lib.rs            # compile(), CompileOptions, CompiledQuery
│   │       ├── lexer.rs, parser.rs, ast.rs
│   │       ├── catalog.rs        # Catalog, Table, Column, SkyIndex (serde JSON/YAML)
│   │       ├── resolve.rs        # name resolution, TAP_SCHEMA/TAP_UPLOAD
│   │       ├── features.rs       # FeatureSet per backend + IVOA feature URIs (single source of truth)
│   │       ├── rules.rs          # translation rule table (drives mapping doc + coverage check)
│   │       ├── functions.rs      # ADQL → DuckDB mapping table
│   │       ├── geom/{mod.rs,sphere.rs,polygon.rs,healpix.rs}
│   │       ├── emit/{mod.rs,plain.rs,healpix.rs}, prelude.sql
│   │       ├── types.rs          # DuckDB type ↔ VOTable datatype/arraysize/xtype
│   │       ├── vosi/{tables.rs,capabilities.rs,tap_schema.rs,error.rs}
│   │       └── error.rs
│   ├── adqlduck-wasm/            # wasm-bindgen API
│   ├── adqlduck-py/              # PyO3 module (_core)
│   └── xtask/                    # `cargo xtask mapping-doc [--check]`
├── python/adqlduck/
│   ├── __init__.py               # re-exports compile, Catalog, vosi_*
│   ├── cli.py                    # argparse entry point `adqlduck`
│   ├── hats.py                   # from_hats(), partition_files(hints, root)
│   └── demo_serve.py             # sync-only TAP for interop testing (Q1)
├── js/package.json               # npm metadata for wasm-pack output
├── tests/                        # pytest: execution, spatial oracle, vosi XSD, conformance
├── corpus/fetch.sh               # pulls lyonetia at a pinned commit
├── docs/adql-mapping.md          # GENERATED — never edit by hand
├── .claude/skills/adqlduck-dev/  # build/test skill for the coding agent
├── examples/catalog.gaia.yaml
├── pyproject.toml                # maturin backend, [project.scripts] adqlduck
├── Makefile
├── LICENSE                       # MIT
└── README.md
```

---

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Rust | stable (pin in `rust-toolchain.toml`) | Needed for `uvx --from git+...` installs too, until wheels are on PyPI |
| uv | latest | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| wasm-pack | latest | `cargo install wasm-pack` |
| Python | 3.11+ | tests and CLI |
| DuckDB / duckdb-wasm | exact version, pinned to healpix-duckdb's pin (v1.5.6 today) | CI runs SQL tests on both. Exact match is required for the `healpix` backend |
| `healpix` extension build | optional | Only for `make test-healpix`. Emmanuel builds it (H5) and points `ADQLDUCK_HEALPIX_EXT` at the file |
| STILTS / TOPCAT | current | Emmanuel's interop checks only |

---

## Quick Start

```bash
# From GitHub (builds the Rust extension locally; needs cargo)
uvx --from git+https://github.com/ejoliet/adqlduck adqlduck --help

# Describe a HATS catalog
uvx --from git+https://github.com/ejoliet/adqlduck adqlduck catalog from-hats \
  <HATS_ROOT_OR_URL> --name gaia.dr3_source > catalog.yaml

# Compile
echo "SELECT TOP 5 source_id FROM gaia.dr3_source
      WHERE 1=CONTAINS(POINT(ra,dec), CIRCLE(56.75, 24.12, 0.5))" \
  | uvx --from git+https://github.com/ejoliet/adqlduck adqlduck compile - --catalog catalog.yaml
```

Browser:

```js
import init, { compile, vosiTables, vosiCapabilities, tapSchemaSql } from "adqlduck";
await init();
const q = compile(adql, catalogJson, { maxrec: 10000 });
await conn.query(q.prelude);
const result = await conn.query(q.sql);
```

---

## Configuration Reference

Library behavior is set by `CompileOptions`. Env vars apply to the CLI only.

| Option / env var | Type | Default | Description |
|---|---|---|---|
| `maxrec` / `ADQLDUCK_MAXREC` | int | none | Outer `LIMIT maxrec+1` |
| `max_healpix_ranges` / `ADQLDUCK_MAX_RANGES` | int | `32` | Cap on OR'ed range predicates |
| `max_polygon_vertices` | int | `256` | Reject larger literals |
| `max_ast_depth` | int | `64` | Guard against pathological queries |
| `int_division` | `truncate` \| `float` | `truncate` | See Q2 |
| `backend` / `ADQLDUCK_BACKEND` | `plain` \| `healpix` | `plain` | Emission target. CLI also accepts `auto` (runs `detect_backend`) |
| `ADQLDUCK_HEALPIX_EXT` | path | none | Test-only: extension file for `make test-healpix` |
| `ADQLDUCK_LOG` | str | `WARNING` | CLI log level, JSON to stderr |

---

## API / Interface Contract

### Catalog descriptor (YAML or JSON)

```yaml
version: 1
tables:
  - adql_name: gaia.dr3_source          # what users write
    relation: gaia.dr3_source           # DuckDB view the consumer registers
    description: Gaia DR3 source catalog
    sky:
      ra: ra
      dec: dec
      healpix: { column: _healpix_29, order: 29, scheme: nested }
      frame: ICRS
    columns:
      - { name: source_id, type: BIGINT, ucd: "meta.id;meta.main", principal: true, indexed: true }
      - { name: ra,  type: DOUBLE, unit: deg, ucd: "pos.eq.ra;meta.main" }
      - { name: dec, type: DOUBLE, unit: deg, ucd: "pos.eq.dec;meta.main" }
      - { name: phot_g_mean_mag, type: FLOAT, unit: mag }
```

### Rust

```rust
pub fn compile(adql: &str, catalog: &Catalog, opts: &CompileOptions) -> Result<CompiledQuery, AdqlError>;

pub struct CompiledQuery {
    pub sql: String,
    pub prelude: &'static str,
    pub columns: Vec<OutputColumn>,      // name, votable datatype/arraysize/xtype, unit/ucd when traceable to a column
    pub spatial_hints: Vec<SpatialHint>, // { relation, order, ranges: Vec<(u64, u64)> } inclusive
    pub features_used: Vec<Feature>,
}

pub fn vosi_tables_xml(catalog: &Catalog) -> String;
pub enum Backend { Plain, Healpix }
impl FeatureSet { pub fn for_backend(b: Backend) -> FeatureSet; }

pub fn vosi_capabilities_xml(catalog: &Catalog, features: &FeatureSet, base_url: &str, limits: &Limits) -> String;
pub fn tap_schema_sql(catalog: &Catalog) -> String;   // CREATE SCHEMA TAP_SCHEMA; tables + rows
pub fn error_votable(err: &AdqlError) -> String;      // TAP error document (QUERY_STATUS=ERROR)
```

### Python

```python
from adqlduck import Catalog, compile, detect_backend, vosi_tables_xml, vosi_capabilities_xml, tap_schema_sql
from adqlduck.hats import from_hats, partition_files

cat = Catalog.from_yaml("catalog.yaml")
backend = detect_backend(con)              # "healpix" if the extension is loaded, else "plain"
q = compile(adql, cat, maxrec=10_000, backend=backend)
con.execute(q.prelude)
rows = con.execute(q.sql).fetchall()
files = partition_files(q.spatial_hints, hats_root)  # Norder/Npix files overlapping the ranges
```

### CLI

```
adqlduck compile <ADQL|-> --catalog FILE [--maxrec N] [--backend plain|healpix] [--json]
adqlduck mapping-doc [--check]               # same as cargo xtask; --check exits 1 on drift
adqlduck catalog from-hats <ROOT|URL> --name SCHEMA.TABLE
adqlduck catalog from-duckdb <DB_FILE>
adqlduck vosi tables --catalog FILE
adqlduck vosi capabilities --catalog FILE --base-url URL
adqlduck tap-schema --catalog FILE
adqlduck conformance [--corpus DIR]          # prints pass/fail per lyonetia file
adqlduck demo-serve --catalog FILE [--host 127.0.0.1] [--port 8765]   # Q1
```

`demo-serve` exposes `/tap/sync`, `/tap/tables`, `/tap/capabilities`, `/tap/availability` only. Sync, VOTable TABLEDATA output, localhost by default. It exists to run taplint, TOPCAT, and pyvo against the compiler. It is not tapdrop.

---

## Usage Examples

Range bounds below are placeholders; real values come from `cdshealpix`.

**1. Cone search**

```sql
-- ADQL
SELECT TOP 10 source_id, ra, dec, phot_g_mean_mag
FROM gaia.dr3_source
WHERE 1 = CONTAINS(POINT('ICRS', ra, dec), CIRCLE('ICRS', 56.75, 24.12, 0.5))
ORDER BY phot_g_mean_mag
```

```sql
-- DuckDB
SELECT t0."source_id", t0."ra", t0."dec", t0."phot_g_mean_mag"
FROM "gaia"."dr3_source" AS t0
WHERE (t0."_healpix_29" BETWEEN <lo1> AND <hi1> OR t0."_healpix_29" BETWEEN <lo2> AND <hi2>)
  AND adql_sep_deg(t0."ra", t0."dec", 56.75, 24.12) <= 0.5
ORDER BY t0."phot_g_mean_mag"
LIMIT 10
```

With `--backend healpix` only the last predicate changes: `AND sky_in_cone(t0."ra", t0."dec", 56.75, 24.12, 0.5)`.

**2. Convex polygon**

```sql
SELECT source_id FROM gaia.dr3_source
WHERE 1 = CONTAINS(POINT(ra, dec), POLYGON(10, 10, 12, 10, 12, 12, 10, 12))
```

→ range prefilter `AND adql_dot(ra, dec, nx1, ny1, nz1) >= 0 AND ... (4 edges)`.

**3. Cross-match with an upload**

```sql
SELECT u.id, g.source_id, DISTANCE(POINT(u.ra, u.dec), POINT(g.ra, g.dec)) AS sep
FROM TAP_UPLOAD.mylist AS u
JOIN gaia.dr3_source AS g
  ON 1 = CONTAINS(POINT(g.ra, g.dec), CIRCLE(u.ra, u.dec, 1.0/3600))
```

```sql
SELECT t0."id", t1."source_id", adql_sep_deg(t0."ra", t0."dec", t1."ra", t1."dec") AS "sep"
FROM "TAP_UPLOAD"."mylist" AS t0
JOIN "gaia"."dr3_source" AS t1
  ON t1."dec" BETWEEN t0."dec" - 0.000277777777777778 AND t0."dec" + 0.000277777777777778
 AND adql_sep_deg(t1."ra", t1."dec", t0."ra", t0."dec") <= 0.000277777777777778
```

---

## Type mapping (VOSI and result metadata)

| DuckDB | VOTable datatype | arraysize | xtype |
|---|---|---|---|
| `BOOLEAN` | boolean | | |
| `TINYINT`, `SMALLINT` | short | | |
| `UTINYINT` | unsignedByte | | |
| `INTEGER`, `USMALLINT` | int | | |
| `BIGINT`, `UINTEGER` | long | | |
| `UBIGINT` | long | | warn: overflow possible |
| `FLOAT` / `DOUBLE` | float / double | | |
| `VARCHAR` | char | `*` | |
| `TIMESTAMP`, `DATE` | char | `*` | timestamp |
| `DOUBLE[]` (geometry) | double | `*` | point / circle / polygon |
| `BLOB` | unsignedByte | `*` | |

Unmapped types → `UnsupportedType` at catalog load, not at query time.

---

## Error Handling

| Error | When | TAP mapping |
|---|---|---|
| `ParseError { line, col, expected, found }` | Not valid ADQL 2.1 | error VOTable, HTTP 400 |
| `UnknownTable`, `UnknownColumn`, `AmbiguousColumn` | Resolution fails | 400 |
| `UnsupportedFeature { feature, hint }` | Valid ADQL, not registered (IN_UNIT, BOX, REGION, non-ICRS) | 400 |
| `UnsupportedGeometry` | Non-convex or self-intersecting polygon, unsupported INTERSECTS pairing | 400 |
| `TypeError` | e.g. CONTAINS on non-geometry | 400 |
| `LimitExceeded` | AST depth, vertex count | 400 |
| `CatalogError` | Bad descriptor, unmapped type, case-only name clash | raised at load |

| `healpix:` runtime message (healpix backend) | Raised by DuckDB at execution | `healpix: internal error` → 500; other `healpix:` → 400. Helper: `classify_runtime_error(msg)` |

No retries: compilation is pure. Rust enum → Python exception subclasses of `AdqlError` → JS `{ code, message, line, col }`.

**Safety guarantee (tested):** emitted SQL references only catalog relations, mapped functions, and prelude macros. No table functions, no `ATTACH`, no `COPY`, no `SET`. User identifiers are resolved and re-quoted; literals are re-escaped, never copied.

---

## Testing

```bash
make test        # cargo test + pytest
make conformance # fetch lyonetia, run full corpus
make wasm-test   # wasm-pack test --node
make mapping-doc-check   # regenerate docs/adql-mapping.md, fail on diff
make test-healpix        # healpix backend; skips unless ADQLDUCK_HEALPIX_EXT is set
```

| Suite | Covers |
|---|---|
| `cargo test` (insta) | Golden SQL for every translation rule; parser error positions |
| `tests/test_exec.py` | Each mapping-table row executed in real DuckDB; result matches expected |
| `tests/test_sky_oracle.py` | Hypothesis: random cones/polygons over 100k synthetic points incl. poles, RA 0/360 wrap, and off-sphere sentinel rows (`dec = -999`, NaN) that must never match; compiled result == astropy brute force. Zero false negatives, zero sentinel matches |
| `tests/test_vosi.py` | `tables` / `capabilities` validate against IVOA XSDs; `TAP_SCHEMA` queryable |
| `tests/test_conformance.py` | lyonetia: valid parse, invalid rejected, per-file report |
| `tests/test_safety.py` | Fuzzed ADQL never yields SQL outside the allowlist |
| `make wasm-test` | JS API smoke; same golden SQL as native |
| `make mapping-doc-check` | Doc equals generator output; every rule in `rules.rs` has an example, a golden test, and an execution test |
| `tests/test_backend_parity.py` | Same ADQL, both backends, same row set on the oracle dataset (points within 1e-9° of a boundary excluded) |

---

## Non-Goals (v1)

- FITS and ASDF readers. FITS ingest belongs in tapdrop/obstap-local; evaluate the community `astro` extension (`read_fits`, cfitsio) there first. ASDF stays out.
- Transport: no service worker, no async/UWS, no auth. `demo-serve` is sync-only test tooling.
- VOTable serialization of results (except the error document). Consumers own it.
- Non-ICRS frames, `IN_UNIT`, `BOX`, `REGION`, `COORDSYS`, MOC-valued geometry.
- In the `plain` backend: non-convex polygons, MOC-valued regions, HEALPix prefilter for region-valued columns. The `healpix` backend covers them.
- Requiring any extension. `healpix` is optional; `spatial` and `geography` are never used.
- Building or vendoring the `healpix` extension in this repo's CI. It is consumed as a pinned artifact.
- Cost-based planning, query rewriting for performance beyond the prefilter.

---

## Open Questions

- [ ] **Q1** — Ship `demo-serve` here, or keep all serving in tapdrop? Default: ship sync-only here; it makes the HN demo one command and gives taplint a target.
- [ ] **Q2** — Integer division: truncate (PostgreSQL behavior, what existing ADQL in the wild assumes) or DuckDB float? Default: truncate. Requires integer typing from the catalog.
- [ ] **Q3** — Feature identifiers in `capabilities`: legacy `tapregext#features-adqlgeo` vs newer ADQL-record ids (ivoa-std/ADQL issue #71). Default: emit what a current DaCHS service emits; verify with taplint.
- [x] **Q4** — Non-convex polygons. Resolved: `plain` backend stays convex-only; `healpix` backend supports them via `sky_in_polygon`.
- [ ] **Q5** — Relationship to Caltech-IPAC/ADQL and Caltech outside-work policy. Default: cite as prior art in README; check policy before making the repo public.
- [ ] **Q6** — Name: `adqlduck` (free everywhere) or something without "duck"?
- [ ] **Q7** — Off-sphere catalog rows under the `healpix` backend (see Backends). Needs a healpix-duckdb ADR (proposed ADR-0009) before its Gate 3. Preferred outcome: strict validation applies to region arguments only; row coordinates off the sphere evaluate to `false` in `sky_*` predicates. Owner: healpix-duckdb, via `HANDOFF-adqlduck.md`.
- [ ] **Q8** — Should the CLI default become `auto` once the extension is on the community endpoint? Default: keep `plain` until healpix-duckdb Gate 5.

---

## Agent Build Instructions

> Implement end-to-end from this README. Read `.claude/skills/adqlduck-dev/SKILL.md` first. Do not run browser, TOPCAT, STILTS, GitHub pushes/PRs, or large-data benchmarks. Print the exact commands for Emmanuel instead (see Human Checks).

### Build Order

| Phase | Deliverable | Done when |
|---|---|---|
| 0 | Workspace, maturin `pyproject.toml`, Makefile, CI (cargo, clippy, pytest, wasm-pack build) | CI green on empty crates |
| 1 | **Gate-0 spike:** minimal SELECT/WHERE parser, catalog, `CONTAINS(POINT, CIRCLE)` → ranges + `adql_sep_deg` | `test_sky_oracle.py` passes 10k random cones, zero false negatives; prints Human Check H1 |
| 2 | Full ADQL 2.1 parser | lyonetia: 100% valid parse, 100% invalid rejected, or each exception listed in `docs/conformance-exceptions.md` with reason |
| 3 | Resolver, non-spatial rewrite, function mapping, MAXREC; `rules.rs`; `cargo xtask mapping-doc` | `test_exec.py` green; `make mapping-doc-check` green and wired into CI |
| 4 | Remaining geometry: polygons, DISTANCE 2/4-arg, INTERSECTS, AREA/CENTROID/COORD1/2, geometry in SELECT, dec-band cross-match | golden + oracle tests green |
| 5 | `features`, VOSI tables/capabilities, TAP_SCHEMA, error VOTable | XSD validation green; capabilities lists exactly `FeatureSet` |
| 6 | PyO3 + CLI + `hats.py`; wasm-bindgen + npm package | `uvx --from .` CLI works; `wasm-test` green |
| 7 | `demo-serve` (if Q1 yes) | pyvo smoke test in pytest against localhost passes; prints H2–H4 |
| 8 | `healpix` backend: `emit/healpix.rs`, `FeatureSet::for_backend`, `detect_backend`, `classify_runtime_error`, healpix column in the mapping doc | Golden SQL for every rule with `healpix` support; `make test-healpix` skips cleanly without the extension; prints H5. Start only after healpix-duckdb Gate 3 (`sky_*`) is merged |

### Constraints

- Core crate: no I/O, no `unsafe`, no panics on user input (`clippy::unwrap_used` denied).
- `cargo clippy -D warnings`, `ruff`, `mypy --strict` on `python/`.
- Python packaging must run straight from GitHub: `uvx --from git+https://github.com/ejoliet/adqlduck adqlduck`.
- Do not vendor lyonetia (GPLv3) or IVOA XSDs; fetch at pinned commits/URLs in test setup.
- `AIDEV-NOTE` / `AIDEV-TODO` comments for non-obvious decisions (spherical edge cases, DuckDB quirks). No comments restating code.
- No secrets, account IDs, or internal URLs anywhere. Catalog examples use public data only.
- Never edit `docs/adql-mapping.md` or `insta` snapshots by hand. Change `rules.rs` or the emitter, regenerate, and explain the diff in the commit message.
- Use `.claude/skills/adqlduck-dev` for every build and test command.

### Human Checks (Emmanuel runs)

```bash
# H1 — pruning on real HATS (Gate-0 evidence): bytes and time, prefilter on vs off
uvx --from git+https://github.com/ejoliet/adqlduck adqlduck catalog from-hats <GAIA_HATS_URL> \
  --name gaia.dr3_source > gaia.yaml
uv run python -m adqlduck.bench --catalog gaia.yaml --cone 56.75,24.12,0.5 --compare-no-prefilter

# H2 — taplint
uvx --from git+https://github.com/ejoliet/adqlduck adqlduck demo-serve --catalog gaia.yaml &
stilts taplint tapurl=http://127.0.0.1:8765/tap

# H3 — TOPCAT: File > Table Access Protocol, URL http://127.0.0.1:8765/tap, run example 1

# H4 — Firefly via sidetap: load the npm build in sidetap, run examples 1–3

# H5 — healpix backend (after healpix-duckdb Gate 3): build the extension, run parity
# (local build is unsigned; the test harness opens DuckDB with allow_unsigned_extensions=true)
git -C ../healpix-duckdb pull && make -C ../healpix-duckdb debug
ADQLDUCK_HEALPIX_EXT=../healpix-duckdb/build/debug/healpix.duckdb_extension make test-healpix
```

`adqlduck.bench` is a small script the agent writes in Phase 6; it reports bytes read via DuckDB profiling output.

### Acceptance Criteria

- [ ] Oracle tests: zero false negatives across all random geometries, including poles and RA wrap
- [ ] lyonetia results documented; no undocumented failures
- [ ] Every row in the function mapping table has an execution test
- [ ] `capabilities` advertises exactly the registered features (test compares both)
- [ ] VOSI XML validates against IVOA XSDs
- [ ] Safety fuzz test passes
- [ ] Same golden SQL from native and WASM builds
- [ ] `docs/adql-mapping.md` is generated and `make mapping-doc-check` runs in CI
- [ ] Backend parity: identical row sets on the oracle dataset (H5, Emmanuel)
- [ ] H1 shows fewer bytes read with prefilter on (Emmanuel)
- [ ] H2 taplint: no errors in ADQL/VOSI sections (Emmanuel)
- [ ] Open Questions resolved or moved to v1.1

---

## Next Steps

1. Answer Q1–Q3 and Q5 (Q4, Q6 can default).
2. Agent runs Phases 0–1. Emmanuel runs H1. Stop here if pruning does not pay off on Gaia HATS.
3. Phases 2–5.
4. Phase 6–7, then H2–H4.
5. Phase 8 once healpix-duckdb Gate 3 lands; then H5.
6. Switch sidetap, tapdrop, obstap-local to depend on `adqlduck`; delete their local ADQL code and update their READMEs.
7. Publish wheels (maturin-action) and npm package; then write the HN post around `docs/adql-mapping.md`, examples 1 and 3, and the conformance table.
