# Handoff: adqlduck integration changes for healpix-duckdb

- **From:** Emmanuel (maintainer), 2026-10-11
- **Repo state assumed:** Gate 0 complete, ADR-0001 to ADR-0007 accepted, `hpx_ang2pix` only.
- **Authority:** every README and AGENTS.md edit listed below is **maintainer-authorized**. The usual rule "README is the spec, flag but don't edit" does not apply to these items.
- **Does not change:** Gates 1–2 scope, function signatures, the DuckDB pin, `healpix_strict_input` default.

---

## 1. What changed and why

`adqlduck` (the ADQL → DuckDB compiler) now has two backends:

| Backend | Needs this extension? | Exact predicates |
|---|---|---|
| `plain` (default) | No | SQL macros (haversine, convex polygons only) |
| `healpix` | Yes | `sky_in_cone`, `sky_in_polygon`, `sky_distance`, `moc_*` |

Three consequences for this repo:

1. **adqlduck no longer waits on this extension.** You are off its critical path. Nothing here should be rushed for adqlduck's sake.
2. **`docs/adql-mapping.md` moves to adqlduck.** It is generated there from the compiler's rule table, with one column per backend, and CI fails on drift. A mapping doc written here could not be tested against the compiler, so it would drift. This repo instead owns a **contract test** that pins exactly what adqlduck calls.
3. **One semantics question must be settled before Gate 3:** what `sky_*` predicates do with off-sphere row coordinates (section 4).

---

## 2. Edits to make now (one small commit, before continuing Gate 1)

### README.md

| Location | Change |
|---|---|
| Architecture diagram, line `adqlduck compiler ──► SQL calling healpix functions` | Replace with: `adqlduck compiler (healpix backend) ──► SQL calling healpix functions`. Add a line under the diagram: `adqlduck also has a plain backend that needs no extension; this extension is the optional fast path for exact predicates, MOC-valued regions and cross-match.` |
| Scope line, `the shared primitive under adqlduck …` | Replace `the shared primitive` with `the optional primitive layer` |
| Repository layout, line `adql-mapping.md      # ADQL geometry → SQL (for adqlduck)` | Delete the line |
| SQL macros table, row `adql_circle(ra, dec, r)` / `adql_polygon(ra[], dec[])` | Delete the row. adqlduck emits the real function names; aliases add public surface with no consumer |
| Downstream Contracts, `adqlduck` row | Replace with the row below |
| Downstream Contracts, after the table | Add the ABI note below |
| Gates table, Gate 5 deliverables | Remove `docs/adql-mapping.md` |
| Acceptance Criteria, `docs/adql-mapping.md complete for ...` | Replace with: `test/sql/contract_adqlduck.test passes and covers every function in the adqlduck Downstream Contracts row.` |
| Next Steps 5, `point adqlduck at the mapping doc` | Replace with: `link to adqlduck's generated docs/adql-mapping.md from the README.` |

New Downstream Contracts row:

```markdown
| `adqlduck` (`healpix` backend) | `sky_in_cone`, `sky_in_polygon`, `sky_distance`, `moc_from_cone`, `moc_ranges`, `moc_contains`, `hpx29` | Signatures and the semantics asserted in `test/sql/contract_adqlduck.test`. The ADQL → SQL mapping is generated and owned by adqlduck (`docs/adql-mapping.md` there) |
```

ABI note under the table:

```markdown
> Consumers must run the exact DuckDB version this extension is built for (`C_STRUCT_UNSTABLE`, ADR-0003). A pin bump is a downstream-visible event: record it in `CHANGELOG.md` so adqlduck and sidetap can move their DuckDB / duckdb-wasm pins in lockstep.
```

### AGENTS.md

| Location | Change |
|---|---|
| Workflow Expectations: `... and if ADQL-relevant update docs/adql-mapping.md` | Replace with: `... and if the function is in the adqlduck Downstream Contracts row, add or update its case in test/sql/contract_adqlduck.test.` |
| Deeper Context: `ADQL mapping consumed by adqlduck: docs/adql-mapping.md` | Replace with: `ADQL mapping: generated in the adqlduck repo; our side of the contract is test/sql/contract_adqlduck.test.` |

### ADR-0008 (new): ADQL mapping ownership

Write `docs/adr/0008-adql-mapping-owned-by-adqlduck.md` in the existing ADR format. Content:

- **Context:** README, AGENTS.md, ADR-0004 and ADR-0005 assumed `docs/adql-mapping.md` lives here. adqlduck now has a plain backend that needs no extension, and generates the mapping doc from its rule table.
- **Options:** A) doc here, tested by adqlduck. B) doc generated in adqlduck; contract test here. C) both.
- **Decision:** B. Note that it supersedes the `docs/adql-mapping.md` mentions in ADR-0004 and ADR-0005 Consequences. Do not edit those ADRs.
- **Consequences:** ADR-0004's "state that ours is ADQL 2.1-specified" point and ADR-0005's "`healpix: internal error` is the stable discriminator" point move into the contract test file header and into adqlduck's doc.

### implementation-notes.md

Add one line under a new `## Handoff — 2026-10-11` heading, in the existing format, referencing this file and ADR-0008.

---

## 3. New file: `test/sql/contract_adqlduck.test`

A sqllogictest file that grows with the gates. It asserts only what adqlduck relies on. Changing an existing case is a semver-level contract change: `CHANGELOG.md` entry required.

Header comment (verbatim intent):

```
# Contract with adqlduck (healpix backend). Every case here is something the
# compiler emits or depends on. Do not change an existing expectation without a
# CHANGELOG entry. Semantics are ADQL 2.1 (invariant 5). Errors are always
# InvalidInputException; "healpix: internal error" distinguishes bugs (ADR-0005).
```

Cases to add, by gate:

| Gate | Cases |
|---|---|
| 0 (now) | `hpx_ang2pix` NULL in → NULL out; out-of-range order error message starts with `healpix:` |
| 1 | `hpx29(ra, dec)` returns `BIGINT` and equals `hpx_ang2pix(29, ra, dec)`; NULL in → NULL out |
| 2 | `moc_ranges(moc_from_cone(...))` returns `LIST<STRUCT(lo BIGINT, hi BIGINT)>`, sorted, disjoint, **half-open**, level 29; `moc_contains(moc, hpx29(ra, dec))` true for the cone centre; empty-region behavior |
| 3 | `sky_in_cone` inclusive (`distance <= radius`), tested with points 1e-9° inside and outside the edge, not exactly on it; `sky_in_polygon` on a concave polygon and a polygon containing a pole; self-intersecting polygon → error starting with `healpix:`; `sky_distance` at 0°, 180°, and across RA 0/360; off-sphere row behavior per ADR-0009 |
| 4 | Column names and types of `hats_partitions(root, moc)` output |

Wire it into `make test`. No new Make target.

---

## 4. Decide before Gate 3: ADR-0009, off-sphere row coordinates in `sky_*` predicates

**Problem.** Under strict input (ADR-0007), `sky_in_cone(ra, dec, 10, 20, 1)` over a catalog that has one sentinel row (`dec = -999`) fails the **entire** query. A TAP user filtering by cone gets an error caused by a row they never asked for. adqlduck's plain backend excludes that row instead, and the two backends must return the same rows.

**Key distinction.** ADR-0007 is right for *constructors*: `hpx_ang2pix(12, 10, 95)` has no correct answer, so refusing is correct. A *predicate* does have a correct answer: a point off the sphere is not inside any region on the sphere, so `false` is not fabricated data.

| Option | Behavior | Verdict to evaluate |
|---|---|---|
| A | Status quo: strict applies to every argument; query fails | Simple; makes the extension unusable as a TAP backend over real catalogs with sentinels |
| B | Region arguments (`ra0, dec0, radius`, polygon vertices) stay strict; row coordinates with `dec ∉ [-90, 90]` or NaN → `false` | **Recommended.** Matches the plain backend; keeps loud errors for malformed queries |
| C | Off-sphere row → NULL | Excluded from WHERE like `false`, but `NOT sky_in_cone(...)` also excludes it, which surprises users |

Requirements whichever option wins:

- RA outside [0, 360) on a **row** must be treated consistently with Dec. Recommendation: normalize RA for rows (sound, per ADR-0007's own analysis), apply the Dec rule above.
- `sky_distance` with an off-sphere argument: NULL is the natural answer under B. Decide explicitly.
- `healpix_strict_input` still governs constructors (`hpx_*`, `moc_from_*`). Under B it no longer affects `sky_*` row arguments. Document that in the README Configuration Reference.
- ADR-0006 check: `sky_in_cone(ra, dec, <literals>)` has two column arguments, so it is not dictionary-eligible. Re-confirm for each new predicate signature.
- Add the chosen behavior to `contract_adqlduck.test` (Gate 3 row).

This changes the README error contract for `sky_*`, so it is a maintainer decision. Write the ADR as **Proposed** with the options table and stop; Emmanuel accepts it.

---

## 5. Things adqlduck does on its side (for your awareness, no action)

- Pruning ranges are computed at compile time with `cdshealpix` and emitted as constant `_healpix_29 BETWEEN` predicates in both backends. The extension is used for exact predicates, MOC-valued regions, and cross-match bucketing (`unnest(moc_ranges(moc_from_cone(u.ra, u.dec, r)))` joined on `_healpix_29`).
- adqlduck converts your half-open `[lo, hi)` to its inclusive `[lo, hi]` at one boundary.
- adqlduck's backend-parity tests compare haversine (plain) and Vincenty (`sky_distance`) results, excluding points within 1e-9° of an edge. No precision change is requested here.
- adqlduck does not build this extension in CI. Emmanuel builds it locally (`make debug`) and runs adqlduck's `make test-healpix` against `build/debug/healpix.duckdb_extension`.
- No new functions are requested. DALI flat polygon arrays are split into ra/dec lists by adqlduck in SQL.

---

## 6. Already-flagged README corrections (Emmanuel: tick to authorize)

These came from your own Gate 0 notes. Apply only the ticked ones, in the same commit as section 2.

- [ ] Prerequisites: DuckDB `1.4.x` → `1.5.6` (ADR-0003)
- [ ] Prerequisites: Rust `1.80+` → `1.85+` (ADR-0003)
- [ ] Error handling: `Rust panic` row → `InvalidInputException` with message `healpix: internal error (...)`; never aborts (ADR-0005). Same fix in the AGENTS.md Conventions line about `InternalException`
- [ ] Quick Start: note that `duckdb -unsigned` must be exactly v1.5.6; link `CONTRIBUTING.md`

---

## 7. Order of work

1. Section 2 edits + ADR-0008 + implementation-notes line. Run `make test`. One commit.
2. Create `contract_adqlduck.test` with the Gate 0 cases (only `hpx_ang2pix` exists). Run `make test`.
3. Continue Gate 1 as planned. Add contract cases at the end of each gate (section 3 table).
4. Before any Gate 3 code: ADR-0009 as **Proposed**, then stop for Emmanuel.

Do not push. Give Emmanuel the commands:

```bash
git log --oneline -5
git push origin main
gh run watch
```

---

## Prompt for the agent

```
Read HANDOFF-adqlduck.md, then AGENTS.md. Apply section 2 and create
test/sql/contract_adqlduck.test with the Gate 0 cases from section 3.
Section 2 edits to README.md and AGENTS.md are maintainer-authorized.
Apply section 6 items only if ticked. Use the duckdb-ext-test skill;
paste the tail of `make test`. Then continue Gate 1. Before writing any
Gate 3 code, write ADR-0009 (section 4) as Proposed and stop.
Do not push.
```
