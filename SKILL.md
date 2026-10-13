---
name: adqlduck-dev
description: How to build, test, and change the adqlduck ADQL→DuckDB compiler, and which command proves which build phase. Use this before declaring any step or phase done, after editing anything under crates/, python/, tests/ or rules.rs, when adding or changing a translation rule, when a test or snapshot fails, when touching docs/adql-mapping.md, or whenever you are about to invent a cargo/pytest/wasm-pack/duckdb command. Never run tests any other way in this repo.
---

# adqlduck-dev

The spec is `README.md`. This skill is how to work on it without breaking the three things that keep it honest: the oracle tests, the generated mapping doc, and the feature registry.

## Commands

Run in this order. Paste the last ~20 lines of each in your report.

| Command | Run when | Proves |
|---|---|---|
| `make lint` | after any edit | `cargo clippy -D warnings`, `ruff`, `mypy --strict` clean |
| `make test` | after any edit to `crates/`, `python/`, `tests/` | `cargo test` (insta golden SQL) + pytest (execution, sky oracle, VOSI XSD, safety fuzz) |
| `make mapping-doc-check` | after any change to `rules.rs`, `emit/`, `functions.rs`, `features.rs`; always before ending Phase 3+ | committed `docs/adql-mapping.md` equals generator output; every rule has a golden test and an execution test |
| `make conformance` | before ending Phase 2; after any parser change | lyonetia corpus: valid queries parse, invalid queries rejected |
| `make wasm-test` | before ending Phase 6; after any change to the public API or `adqlduck-wasm` | WASM build returns the same golden SQL as native |
| `make test-healpix` | Phase 8 only | `healpix` backend tests. Skips (exit 0, prints SKIPPED) when `ADQLDUCK_HEALPIX_EXT` is unset — that is expected for you |

While iterating, narrow the run: `cargo test -p adqlduck geom::`, `uv run pytest tests/test_sky_oracle.py -x`. Run the full set before reporting.

## Phase → proof

| Phase | Done when |
|---|---|
| 0 | `make lint` and `make test` green on empty crates; CI file present |
| 1 (Gate-0 spike) | `uv run pytest tests/test_sky_oracle.py` green, 10k random cones, zero false negatives. Then give Emmanuel H1 and stop |
| 2 | `make conformance` report: 100% valid parsed, 100% invalid rejected, or each exception in `docs/conformance-exceptions.md` with a reason |
| 3 | `make test` + `make mapping-doc-check` green; doc check wired into CI |
| 4 | golden + oracle tests green for every geometry rule |
| 5 | XSD validation green; test asserts `capabilities` lists exactly `FeatureSet::for_backend(b)` for both backends |
| 6 | `uvx --from . adqlduck --help` works; `make wasm-test` green |
| 7 | pyvo smoke test against `demo-serve` green; give Emmanuel H2–H4 |
| 8 | golden SQL for every `healpix` rule; `make test-healpix` skips cleanly; give Emmanuel H5 |

Do not start the next phase without the proof output in your report.

## Adding or changing a translation rule

1. Add or edit the row in `crates/adqlduck/src/rules.rs`: id, ADQL example, feature, backends.
2. Write the golden test first (`insta`), then the emitter change.
3. Add the execution test in `tests/test_exec.py` (or the oracle test for geometry). It must run the emitted SQL in real DuckDB.
4. If the rule adds or removes an ADQL feature, update `features.rs` for each backend it applies to.
5. `cargo xtask mapping-doc`, then `make mapping-doc-check`. Commit the regenerated doc in the same commit as the code.

`mapping-doc-check` failing means code and doc disagree. Fix by regenerating from code, never by editing the doc.

## Rules

- **Never edit `docs/adql-mapping.md` by hand.** It is generated.
- **Never accept an `insta` snapshot change you did not intend.** `cargo insta review` only after you can say in one sentence why the SQL changed. Put that sentence in the commit message.
- **A sky-oracle failure is a compiler bug.** Do not loosen tolerances, shrink the random sample, or exclude a region to make it pass. The only allowed exclusion is points within 1e-9° of a boundary, and only in backend-parity tests.
- **Feature registry is the truth for `capabilities`.** Never hard-code a feature URI anywhere except `features.rs`.
- **Never vendor lyonetia (GPLv3) or IVOA XSDs.** `corpus/fetch.sh` and the test setup fetch them at pinned refs.
- **Emitted SQL allowlist.** Only catalog relations, mapped functions, prelude macros, and (healpix backend) extension functions from the healpix-duckdb Downstream Contracts. If you need anything else, stop and ask.
- **Half-open vs inclusive ranges.** `SpatialHint.ranges` are inclusive; the extension's are half-open. The conversion lives in one function with an `AIDEV-INVARIANT` comment. Do not convert anywhere else.
- **The core crate does no I/O.** Backend detection, extension loading, and file access belong in `python/` and `adqlduck-wasm`, never in `crates/adqlduck`.
- Use `AIDEV-NOTE:` for DuckDB quirks (e.g. `^` is power, `LOG` is natural log) and `AIDEV-INVARIANT:` for ADQL semantics (inclusive boundaries, ICRS only).

## Emmanuel runs these, not you

Give him the exact command and stop. Do not try them.

```bash
# H1 Gate-0 pruning evidence on Gaia DR3 HATS (S3)
uvx --from git+https://github.com/ejoliet/adqlduck adqlduck catalog from-hats <GAIA_HATS_URL> --name gaia.dr3_source > gaia.yaml
uv run python -m adqlduck.bench --catalog gaia.yaml --cone 56.75,24.12,0.5 --compare-no-prefilter

# H2 taplint
uvx --from git+https://github.com/ejoliet/adqlduck adqlduck demo-serve --catalog gaia.yaml &
stilts taplint tapurl=http://127.0.0.1:8765/tap

# H5 healpix backend (needs the extension built locally)
make -C ../healpix-duckdb debug
ADQLDUCK_HEALPIX_EXT=../healpix-duckdb/build/debug/healpix.duckdb_extension make test-healpix

# publishing
git push origin main
```

Also his: TOPCAT and Firefly checks (H3, H4), PyPI/npm releases, building healpix-duckdb.

## When something fails

1. Re-run only the failing test.
2. Geometry edge case (pole, RA 0/360, radius 0 or ≥ 180, antipodal points)? Read the `AIDEV-INVARIANT` comments in `geom/` first.
3. Native passes, WASM fails? Suspect a dependency doing I/O or threads in the core crate.
4. `make test-healpix` fails with "built specifically for DuckDB version": DuckDB pin mismatch with healpix-duckdb. Report both versions to Emmanuel. Do not change either pin.
5. Error message starts with `healpix:`? The extension raised it. Check whether it is a row-data problem (Q7, off-sphere coordinates) before suspecting the compiler.
