# ADQL on DuckDB with HEALPix: references, rationale, and what changes

Companion to the `adqlduck` README and the `healpix-duckdb` spec. Every decision in those two documents rests on a claim about HEALPix, ADQL/TAP, or DuckDB. This document lists the claims, the evidence, and where to read more. Sources were checked on 2026-10-11; retrieval dates matter because DuckDB and the IVOA document set both move.

## Scope

**For whom.** Colleagues onboarding onto IVOA data access (TAP, ADQL, VOSI) and HEALPix-partitioned catalogs (HATS), and anyone reviewing why `adqlduck` and `healpix-duckdb` are designed the way they are.

**What it covers.** The claims behind: HEALPix level-29 integer indexing and range prefilters; ADQL 2.1 as the target grammar; VOSI/TAPRegExt feature declaration; DuckDB as the engine (Parquet pruning, range joins, WASM, macros, operator traps); the browser limits; and the two-backend split.

**What it does not cover.** Implementation detail (see the READMEs), and performance numbers. The pruning benefit on Gaia DR3 is a prediction until H1 in the `adqlduck` README is run. Where a claim is still unverified it is marked **unverified**.

**Two projects, two layers.** `healpix-duckdb` is a DuckDB extension: primitives (pixels, MOC, exact spherical predicates) usable from plain SQL, with no knowledge of ADQL. `adqlduck` is a compiler: ADQL in, DuckDB SQL and VOSI metadata out, no execution. The compiler runs on stock DuckDB (`plain` backend) and uses the extension when loaded (`healpix` backend). Both compute HEALPix coverage with `cdshealpix`; the compiler does it at compile time for literal regions (constant ranges prune Parquet), the extension at query time for column-valued regions.

**How to read it.** Section 2 is a one-screen primer. Section 3 is the claim-by-claim evidence table. Sections 4 and 5 are the before/after and the worked examples, written for onboarding. Section 7 is the reference list.

---

## 1. Primer: the pieces in one screen

| Concept | What it is | Why it matters here | Ref |
|---|---|---|---|
| **TAP** (Table Access Protocol) | IVOA HTTP protocol for querying tables: `/sync`, `/async` (UWS), `/tables`, `/capabilities`. Current REC is 1.1 (27 Sep 2019). | The surface every VO client (TOPCAT, pyvo, Firefly, Aladin) speaks. | [11] |
| **ADQL** (Astronomical Data Query Language) | SQL92 subset plus sky geometry (`POINT`, `CIRCLE`, `POLYGON`, `CONTAINS`, `DISTANCE`). Current REC is 2.1 (15 Dec 2023). | The language `adqlduck` compiles. 2.1 adds `OFFSET`, `ILIKE`, `LOWER`, 4-argument `DISTANCE`, optional frame argument, CTEs, set operators. | [8] |
| **VOSI / TAPRegExt** | VOSI gives `/tables` and `/capabilities`; TAPRegExt is the XML schema inside `capabilities` that declares languages, output formats, upload methods, limits, and **optional language features**. | A service must advertise exactly the optional ADQL features it supports. Generating this from the compiler's feature registry is what keeps clients and server in agreement. | [12] |
| **DALI xtypes** | Serialization of geometry in VOTable: `point` (double[2]), `circle` (double[3]), `polygon` (double[*]). | How geometry-valued columns (ObsCore `s_region`) and geometry in `SELECT` are returned. | [13] |
| **HEALPix** | Hierarchical Equal Area isoLatitude Pixelisation of the sphere. 12 base cells; each cell splits into 4 per order. Order 29 has 12 × 4²⁹ cells, which fits in a signed 64-bit integer. The NESTED scheme makes a cell's descendants a contiguous integer interval. | A sky region becomes a short list of integer ranges; a catalog sorted by its order-29 index can be pruned with `BETWEEN`. | [1], [2] |
| **MOC** (Multi-Order Coverage map) | IVOA standard (2.0, 2022) for arbitrary sky coverage as a set of HEALPix cells at mixed orders, equivalently as level-29 intervals. | The extension's region type; healpix-alchemy showed the interval-set model works in a relational database. | [7], [2] |
| **HATS** (Hierarchical Adaptive Tiling Scheme) | Parquet catalog layout partitioned by HEALPix `(Norder, Npix)` with a `_healpix_29` spatial index column and a `partition_info.csv` manifest. Used by LSDB, IRSA, MAST. Gaia DR3 is public on AWS Open Data in this layout. | The data layout both tools assume. Partition pruning uses the manifest; row-group pruning uses `_healpix_29`. | [3], [4], [5] |

---

## 2. Decisions and the claims behind them

### HEALPix and geometry

| Decision | Claim it rests on | Evidence | Ref |
|---|---|---|---|
| Index everything at order 29, NESTED, as `BIGINT` | 12 × 4²⁹ = 3 × 2⁶⁰ < 2⁶³, so one signed 64-bit integer addresses every order-29 cell; NESTED makes any coarser cell a contiguous interval at order 29 | HEALPix paper (nested quad-tree numbering, Fig. 1); healpix-alchemy stores points as the NESTED index at the maximum level and tiles as `INT8RANGE` | [1], [2] |
| Region = sorted, disjoint, half-open level-29 intervals | This is the healpix-alchemy data model, published and in production in SkyPortal/Fritz | Singer 2022, AJ 163 209 | [2] |
| Prefilter with `_healpix_29 BETWEEN lo AND hi`, exact predicate after | HATS defines `_healpix_29` and provides `healpix_to_spatial_index(order, pixel)` = minimum index inside a pixel; LSDB itself prefilters candidates by HEALPix pixel before computing great-circle separations | HATS guide; SkyKatana description of the LSDB pattern | [3], [4] |
| Prune files before opening them | HATS partitions are named `Norder=k/Dir=d/Npix=n.parquet` and listed in `partition_info.csv`; overlapping cells → file list | HATS directory scheme; IVOA 2024/2025 HATS slides | [3] |
| Coverage from `cdshealpix` (Rust) | CDS maintains it with WebAssembly and Python targets; the sibling MOC library is the core of MOCPy, MOCCli, MOCWasm, and Aladin Lite v3 | GitHub metadata; IVOA MOC page | [6] |
| Exact distance: haversine in the plain backend, Vincenty in the extension | Haversine is well-conditioned for small separations but loses precision near antipodes; the spherical special case of Vincenty is accurate at all separations | Wikipedia, "Great-circle distance", computational formulas | [25] |
| Guard `dec` outside [-90, 90] in the plain macro | Trigonometric functions are periodic, so a sentinel such as -999 maps to a real declination and can match a cone. Found while writing the backend parity rules; no external source, it is arithmetic | — | — |
| Polygons are great-circle edged, inclusive `CONTAINS`, ADQL hemisphere rule | ADQL 2.1 geometry semantics; pgSphere/S2 differ in degenerate cases, and the consumer is an ADQL compiler | ADQL 2.1 REC; healpix-duckdb AGENTS.md invariant 5 | [8], [27] |
| Vertex order and winding | DALI polygons are simple single loops serialized as a double array; winding is being redefined as "inside on the left" (equivalent to CCW) in DALI 1.2 discussions | DALI PR #52; DaliNext page | [13] |

### ADQL and TAP

| Decision | Claim it rests on | Evidence | Ref |
|---|---|---|---|
| Target ADQL 2.1, not 2.0 | 2.1 is the current REC (15 Dec 2023) and already deployed (ESA PSA, ESA registry TAP declare `ADQL#v2.1`) | IVOA ADQL page; live `capabilities` documents | [8], [14] |
| Hand-written parser that can reject invalid queries | The IVOA conformance corpus (lyonetia) contains ~495 queries, ~82 marked `valid="false"`; a permissive general SQL parser cannot fail them | lyonetia repo (`src/adql/ivoa/*.xml`), used by the ADQL 2.1 RFC validation | [9], [8] |
| Fetch lyonetia at test time, never vendor | Its LICENSE is GPLv3; `adqlduck` is MIT | lyonetia LICENSE | [9] |
| One feature registry drives validator and `capabilities` | TAPRegExt requires optional language features to be declared; the ADQL 2.1 RFC states supported optional features must appear in the service capabilities | TAPRegExt 1.0; ADQL21RFC comments | [12], [8] |
| Keep feature URIs in one file and verify with taplint | The identifiers are inconsistent between TAPRegExt (`#features-adqlgeo`, `#features-udf`) and the ADQL record (`adql#features-…`); RegTAP reviewers hit the same confusion (`features-adql-conditional` vs `features-adql-type`) | ivoa-std/ADQL issue #71; RegTAP 1.2 RFC | [10] |
| `TAP_UPLOAD` resolved from the catalog, not parsed specially | TAP 1.1 defines upload table naming; the compiler only needs the schema name | TAP 1.1 RFC notes ("guidance on allowed table names for UPLOAD") | [11] |
| Validate the running service with taplint | taplint runs a battery of test queries against TAP, UWS, VOSI, TAPRegExt, DALI and ADQL, and is the standard validator the IVOA process expects | Taylor, taplint paper and usage docs | [15] |
| Position the project against prior art, not as "first" | Caltech-IPAC `adql` translates ADQL to Oracle/SQLite with HTM/HPX index constraints; AIP `queryparser` targets PostgreSQL/MySQL; VOLLT is the Java reference used to validate ADQL 2.1 | Repos; ADQL21RFC | [16], [17], [8] |

### DuckDB

| Decision | Claim it rests on | Evidence | Ref |
|---|---|---|---|
| Emit constant `BETWEEN` on `_healpix_29` for pruning | DuckDB pushes filters into the Parquet reader and skips row groups using per-row-group min/max (zonemaps); sorted data gives tight ranges | DuckDB Parquet docs and performance guide | [20] |
| Sort catalogs by `_healpix_29` (HATS already does) | Row-group pruning only helps when min/max ranges are tight and non-overlapping | DuckDB Parquet tips, "Sorting rows to improve row group pruning" | [20] |
| Cross-match as a declination-band join plus exact check | DuckDB has dedicated range-join operators (piecewise merge join, IEJoin) for inequality predicates, 20-30× faster than nested loops in their benchmark | DuckDB blog, "Range Joins in DuckDB" | [21] |
| Plain backend uses SQL macros, no extension | `CREATE OR REPLACE MACRO` defines scalar functions in SQL, idempotently, in any DuckDB build | DuckDB `CREATE MACRO` docs | [24] |
| Map ADQL `^` (XOR) to `xor()` | In DuckDB `^` is exponentiation (alias of `**`); bitwise XOR is the function `xor(x, y)` | DuckDB numeric operators | [19] |
| Decide integer division explicitly (Q2) | DuckDB `/` on two integers is float division; `//` is integer division; PostgreSQL truncates, and most deployed TAP services are PostgreSQL-backed | DuckDB numeric operators; PostgreSQL compatibility page | [19] |
| Resolver rejects catalogs with names differing only by case | DuckDB identifiers are case-insensitive throughout, case-preserving on output, even when quoted | DuckDB PostgreSQL compatibility page | [19] |
| Do not use core `spatial` for sky geometry | `ST_Distance` and `ST_Distance_GEOS` are documented as planar; only `ST_Distance_Spheroid` is ellipsoidal, and there are no spherical containment predicates | DuckDB spatial function list | [23] |
| Do not depend on the `geography` (S2) community extension | Its `description.yml` sets `excluded_platforms: "wasm_mvp;wasm_eh;wasm_threads"`, so it never runs in the browser | duckdb/community-extensions repository | [23] |
| FITS ingest is out of scope here; evaluate `astro` in tapdrop | The community `astro` extension (v1.4.1, C++) ships `read_fits` via cfitsio and does not exclude WASM in its descriptor. **Unverified** whether it works in DuckDB-WASM | duckdb/community-extensions repository | [23], [28] |
| Pin DuckDB and duckdb-wasm to the extension's exact version | A Rust C-API extension is built with the unstable C API and loads only on the DuckDB version it was built for; verified by loading a v1.5.6 build in a v1.5.5 CLI | healpix-duckdb ADR-0003 | [27] |
| Classify extension errors by message prefix | The C extension API has one error channel, so every error surfaces as `InvalidInputException`; `healpix: internal error` is the only discriminator | healpix-duckdb ADR-0005 | [27] |
| Two backends, extension optional | The Rust template builds all three WASM targets, but runtime behavior in the browser is unproven until its Gate 5; adqlduck must not block on it | healpix-duckdb ADR-0002 | [27] |

### Browser

| Decision | Claim it rests on | Evidence | Ref |
|---|---|---|---|
| In-browser TAP (sidetap) serves only in-page clients | Service workers are bound to an origin and control only pages matching their scope; a desktop client (TOPCAT, pyvo) cannot reach them | Chromium service worker design doc | [26] |
| Same engine as a `uvx` local TAP for everything else | Follows from the above | — | — |
| Read remote Parquet by HTTP range requests | DuckDB's `httpfs` reads only the byte ranges a query needs; DuckDB-Wasm does the same through its virtual filesystem (with a documented caveat for the built-in httpfs path) and is subject to the bucket's CORS policy | DuckDB httpfs and Wasm docs | [22] |
| Gaia DR3 as the demo catalog | STScI publishes Gaia DR3 in HATS format on AWS Open Data (`s3://stpubdata/gaia`), attribution required | Registry of Open Data on AWS | [5] |

---

## 3. Before and after

Each row is a task a colleague actually has. "Before" is today's path; "after" is the path once `adqlduck` (and, where marked, `healpix-duckdb`) exists. Nothing in "after" is shipped yet.

| Task | Before | After |
|---|---|---|
| Learn ADQL geometry | Read the spec, then experiment against a remote TAP service whose SQL you cannot see. Errors come back as that service's database errors. | `adqlduck compile "…" --catalog gaia.yaml` prints the DuckDB SQL. `docs/adql-mapping.md` is a generated table of every ADQL construct and its SQL. Errors are ADQL-level, with line and column. |
| Query a HATS catalog on S3 with ADQL | Not possible. HATS is read with LSDB (Python/Dask) or plain SQL over Parquet. ADQL needs a server with PostgreSQL + pgSphere/Q3C, Oracle + spatial index, or a Java stack. | `uvx --from git+… adqlduck demo-serve --catalog gaia.yaml`, then point TOPCAT or pyvo at `http://127.0.0.1:8765/tap`. No database. |
| Run a cone search without a full scan | Server-side index, or LSDB's pixel prefilter in Python. | The compiler emits `_healpix_29 BETWEEN …` ranges; DuckDB skips row groups, and consumers skip HATS files from the same ranges. |
| Build a browser tool (Firefly, Aladin Lite) over a static bucket | Needs a hosted TAP endpoint somewhere. | sidetap: the same compiler in WASM behind a service worker. Zero server. Limited to in-page clients, by design. |
| Write a TAP service and get `capabilities` right | Hand-write TAPRegExt XML; discover mismatches when taplint or a client fails. | `adqlduck vosi capabilities` is generated from the feature registry the validator uses. The service cannot advertise something the compiler rejects. |
| Cross-match an uploaded list against a catalog | Server-side only; most file-based tools have no `TAP_UPLOAD`. | Plain backend: declination-band range join plus exact check. With `healpix-duckdb`: per-row MOC ranges joined on `_healpix_29`. |
| Non-convex footprints, MOC-valued regions | Depends on the server. | `healpix-duckdb` backend (`sky_in_polygon`, `moc_contains`); plain backend says so explicitly with `UnsupportedFeature`. |
| Check your implementation is right | Trust the vendor. | Oracle tests against astropy (compiler) and against MOCPy, astropy-healpix, pgSphere/Q3C, lsdb (extension); lyonetia conformance; taplint. |

The honest "after" caveats: pruning gains are unmeasured until H1; the browser path depends on CORS and on the duckdb-wasm version matching the extension's pin; and the in-browser endpoint is not a public TAP service.

---

## 4. Showcase: three queries, what happens, what to learn

Range values are placeholders; the real ones come from `cdshealpix` at compile time.

### 4.1 Cone search (the Pleiades)

```sql
SELECT TOP 10 source_id, ra, dec, phot_g_mean_mag
FROM gaia.dr3_source
WHERE 1 = CONTAINS(POINT('ICRS', ra, dec), CIRCLE('ICRS', 56.75, 24.12, 0.5))
ORDER BY phot_g_mean_mag
```

Compiles (plain backend) to:

```sql
SELECT t0."source_id", t0."ra", t0."dec", t0."phot_g_mean_mag"
FROM "gaia"."dr3_source" AS t0
WHERE (t0."_healpix_29" BETWEEN <lo1> AND <hi1> OR t0."_healpix_29" BETWEEN <lo2> AND <hi2>)
  AND adql_sep_deg(t0."ra", t0."dec", 56.75, 24.12) <= 0.5
ORDER BY t0."phot_g_mean_mag"
LIMIT 10
```

What to learn: `TOP` is `LIMIT`; `CONTAINS(...) = 1` is a boolean in disguise; the `BETWEEN` block is the HEALPix coverage of the circle and is a superset (it never drops a true match); the macro is the exact test. With the `healpix` backend the last predicate becomes `sky_in_cone(t0."ra", t0."dec", 56.75, 24.12, 0.5)`.

### 4.2 Polygon

```sql
SELECT source_id FROM gaia.dr3_source
WHERE 1 = CONTAINS(POINT(ra, dec), POLYGON(10, 10, 12, 10, 12, 12, 10, 12))
```

Plain backend: the same range prefilter, then one half-space test per edge, `adql_dot(ra, dec, nx, ny, nz) >= 0`, where the edge normals were computed at compile time. Convex only. Non-convex polygons compile only with `healpix-duckdb` (`sky_in_polygon`).

What to learn: ADQL 2.1 polygons have great-circle edges, not straight lines in RA/Dec; a quad that looks like a rectangle on a plot is not one on the sphere near the pole.

### 4.3 Cross-match with an upload

```sql
SELECT u.id, g.source_id, DISTANCE(POINT(u.ra, u.dec), POINT(g.ra, g.dec)) AS sep
FROM TAP_UPLOAD.mylist AS u
JOIN gaia.dr3_source AS g
  ON 1 = CONTAINS(POINT(g.ra, g.dec), CIRCLE(u.ra, u.dec, 1.0/3600))
```

Plain backend:

```sql
SELECT t0."id", t1."source_id", adql_sep_deg(t0."ra", t0."dec", t1."ra", t1."dec") AS "sep"
FROM "TAP_UPLOAD"."mylist" AS t0
JOIN "gaia"."dr3_source" AS t1
  ON t1."dec" BETWEEN t0."dec" - 0.000277777777777778 AND t0."dec" + 0.000277777777777778
 AND adql_sep_deg(t1."ra", t1."dec", t0."ra", t0."dec") <= 0.000277777777777778
```

What to learn: a cone centred on a column, not a literal, has no compile-time coverage; the declination band is a cheap conservative filter that DuckDB turns into a range join. This is also where ADQL `1.0/3600` vs `1/3600` matters (Q2: integer division).

---

## 5. Still unverified

| Claim | Status | How it gets settled |
|---|---|---|
| Pruning reduces bytes read on Gaia DR3 HATS over HTTP | Prediction | H1 benchmark in the `adqlduck` README |
| The `healpix` extension runs correctly in DuckDB-WASM | Builds for all three WASM targets; runtime untested | healpix-duckdb Gate 5 (`make test-wasm`) |
| Community `astro` extension's `read_fits` works in WASM | Not excluded by its descriptor; not tested | tapdrop FITS ingest spike |
| Which TAPRegExt/ADQL feature URIs clients actually expect | Known to be inconsistent | taplint against `demo-serve`; compare with a DaCHS `capabilities` |
| pyacid `XMATCH` prior art | Not found in the community registry; nature unknown | healpix-duckdb ADR-0004 open item |

---

## 6. Glossary

| Term | Meaning |
|---|---|
| Order (HEALPix) | Refinement level; `nside = 2^order`; order 29 is the finest used here |
| NESTED | HEALPix numbering where a parent's four children are consecutive indices |
| Zonemap | Per-row-group min/max statistics DuckDB uses to skip Parquet data |
| UWS | Universal Worker Service; the async job model behind TAP `/async` |
| xtype | VOTable extended type tag (`point`, `circle`, `polygon`, `timestamp`) |
| C_STRUCT_UNSTABLE | DuckDB extension ABI mode that ties a build to one DuckDB version |

---

## 7. References

Primary sources first within each group. All URLs retrieved 2026-10-11.

**HEALPix, MOC, HATS**

1. Górski, K. M., et al. 2005, "HEALPix: A Framework for High-Resolution Discretization and Fast Analysis of Data Distributed on the Sphere", ApJ 622, 759. https://dx.doi.org/10.1086/427976 — ADS: https://ui.adsabs.harvard.edu/abs/2005ApJ...622..759G
2. Singer, L. P., et al. 2022, "HEALPix Alchemy: Fast All-Sky Geometry and Image Arithmetic in a Relational Database for Multimessenger Astronomy Brokers", AJ 163, 209. Project wiki: https://github.com/skyportal/healpix-alchemy/wiki — abstract: https://dcc-llo.ligo.org/LIGO-P2100433/public
3. HATS documentation: guide https://hats.readthedocs.io/en/latest/guide/hats-guide.html — directory scheme https://hats.readthedocs.io/en/stable/guide/directory_scheme.html — IVOA note slides (June 2025) https://wiki.ivoa.net/internal/IVOA/InterOpJune2025Apps/HATS_IVOA_noteMaryland_2025.pdf — IRSA Parquet/HATS catalogs https://irsa.ipac.caltech.edu/docs/parquet_catalogs/
4. "Using LSDB to enable large-scale catalog distribution, cross-matching, and analytics", arXiv:2501.02103 (listing: https://huggingface.co/papers/2501.02103) — LSDB prefilter pattern described at https://skykatana.readthedocs.io/en/latest/hatsbasic/
5. Gaia DR3 (HATS) on the Registry of Open Data on AWS, managed by STScI: https://registry.opendata.aws/gaia-dr3
6. cdshealpix (Rust): https://github.com/cds-astro/cds-healpix-rust — MOC Lib Rust: https://github.com/cds-astro/cds-moc-rust — IVOA MOC tooling page (MOC Lib Rust as the core of MOCPy, MOCCli, MOCWasm, Aladin Lite v3): https://wiki.ivoa.net/twiki/bin/view/IVOA/MocInfo
7. IVOA MOC 2.0 Recommendation (corrected 2022-07-27 document): https://ivoa.net/documents/MOC/20220727/index.html — erratum note: https://wiki.ivoa.net/twiki/bin/view/IVOA/MOC-2_0-Err-1

**ADQL, TAP, VOSI**

8. IVOA ADQL page (current REC 2.1, 15 Dec 2023): https://wiki.ivoa.net/twiki/bin/view/IVOA/ADQL — source: https://github.com/ivoa-std/ADQL — ADQL 2.1 RFC (validation with VOLLT, feature declaration): https://wiki.ivoa.net/twiki/bin/view/IVOA/ADQL21RFC?rev=27 — plain-language tour of 2.1 features: https://blog.g-vo.org/speak-out-on-adql-2-1.html
9. lyonetia, IVOA ADQL grammar validation corpus (GPLv3): https://github.com/ivoa/lyonetia
10. Feature identifier inconsistency: https://github.com/ivoa-std/ADQL/issues/71 — RegTAP 1.2 RFC comment on `features-adql-conditional`: https://wiki.ivoa.net/twiki/bin/view/IVOA/RegTAP12RFC?rev=13
11. IVOA TAP page (TAP 1.1 REC, 27 Sep 2019): https://wiki.ivoa.net/twiki/bin/view/IVOA/TableAccess — TAP 1.1 RFC: https://wiki.ivoa.net/twiki/bin/view/IVOA/TAP11RFC — TAP 1.0 text, VOSI section (arXiv): https://arxiv.org/pdf/1110.0497
12. TAPRegExt 1.0 Recommendation (REC-TAPRegExt-1.0-20120827): https://arxiv.org/abs/1402.4742v1 — worked example in vo-models docs: https://vo-models.readthedocs.io/latest/pages/protocols/vosi.html
13. DALI xtypes: point/circle/polygon arraysize and ordering, DALI source PR #52: https://github.com/ivoa-std/DALI/pull/52/files — winding and multipolygon discussion: https://wiki.ivoa.net/twiki/bin/view/IVOA/DaliNext
14. Live `capabilities` declaring ADQL 2.1 (ESA PSA): https://psa.esa.int/psa-tap/tap/capabilities
15. Taylor, M. B., "Taplint, the TAP Service Validator" (ADASS XXXI): https://arxiv.org/pdf/2112.08003 — usage: https://www.star.bris.ac.uk/mbt/stilts/sun256/taplint-usage.html — examples: https://www.star.bris.ac.uk/mbt/stilts/sun256/secB.27.2.html

**Prior art**

16. Caltech-IPAC ADQL to local SQL translator (`adql` on PyPI, v1.0.6): https://github.com/Caltech-IPAC/ADQL
17. AIP queryparser (ANTLR; ADQL to PostgreSQL/MySQL): https://github.com/aipescience/queryparser
18. VOLLT ADQL library (Java reference implementation used in the ADQL 2.1 RFC): see [8], ADQL21RFC page

**DuckDB**

19. Numeric operators (`^` is exponent, `//` integer division, `xor()`): https://www.duckdb.org/docs/preview/sql/functions/numeric.html — PostgreSQL compatibility (integer division, case-insensitive identifiers): https://duckdb.org/docs/sql/postgresql_compatibility
20. Parquet filter pushdown: https://duckdb.org/docs/current/data/parquet/overview — row groups and sorting for pruning: https://www.duckdb.org/docs/current/data/parquet/tips — zonemaps: https://duckdb.org/docs/current/guides/performance/indexing.html
21. "Range Joins in DuckDB" (IEJoin, piecewise merge join): https://duckdb.org/2022/05/27/iejoin.html
22. httpfs partial reads over HTTP: https://www.duckdb.org/docs/preview/core_extensions/httpfs/https.html — DuckDB-Wasm overview: https://www.duckdb.org/docs/current/clients/wasm/overview — Wasm CORS and range-request notes: https://duckdb.org/docs/current/clients/wasm/troubleshoot
23. Spatial function list (planar `ST_Distance`): https://duckdb.org/docs/current/core_extensions/spatial/functions — Community extensions: https://duckdb.org/community_extensions/ — repository with per-extension `description.yml` (`astro`, `geography` platform exclusions): https://github.com/duckdb/community-extensions — extension development and WASM builds: https://duckdb.org/community_extensions/development
24. `CREATE MACRO`: https://duckdb.org/docs/current/sql/statements/create_macro
25. Great-circle distance, computational formulas (haversine conditioning, Vincenty special case): https://en.wikipedia.org/wiki/Great-circle_distance

**Browser and project repositories**

26. Chromium service worker design doc (origin-bound scope, controlled pages): https://chromium.googlesource.com/chromium/src/+/7bb3c7285ff8daf89411231f11d8c5c3b8fe42c2/content/browser/service_worker/
27. healpix-duckdb repository (ADR-0002 WASM build, ADR-0003 version pin and ABI, ADR-0005 error class, ADR-0007 strict input): https://github.com/ejoliet/healpix-duckdb
28. Community `astro` extension source: https://github.com/synapticore-io/astro-duck — `geography` (S2) extension source: https://github.com/paleolimbot/duckdb-geography
