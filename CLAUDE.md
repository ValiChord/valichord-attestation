# CLAUDE.md

Guidance for Claude Code (claude.ai/code) in this repository.

**This is valichord-attestation, not ValiChord.** ValiChord (github.com/ValiChord/ValiChord) is a
commit-reveal reproducibility protocol on Holochain. This library is a separate project: an open
format for attesting AI evaluation runs, meant to become one way to bring results into ValiChord,
and not yet connected to it. Never describe this library as ValiChord, or ValiChord as this library.

Moved here from `ValiChord/ValiChord/valichord_attestation` on 2026-10-19 with its full history;
the notes below were written there and paths are now relative to this repository root.

## Build and test


```bash
# (run from the repository root)

# Install (first time)
pip install -e ".[dev]"

# 640 tests, 97% line coverage (1217 statements, 34 missed).
# measured 2026-10-19, CI run 39287013809, identical on Python 3.10 and 3.13.
# CI measures coverage on every push, so
# re-derive the figure from a run rather than quoting this comment.
pytest --cov

# Single test file
pytest tests/test_merkle.py
```

For `InspectAILogAdapter` tests: `pip install -e ".[inspect-ai]"` first.

## Modules (`valichord_attestation/`)

| Module | Purpose |
|---|---|
| `builder.py` | `build_bundle(...)` — assembles a `Bundle` from typed fields |
| `canonical.py` | JCS (RFC 8785) encoding + `hash_bundle()` + `content_hash()` |
| `merkle.py` | construction selection by `format_version`; wrappers for `merkle_root`, `merkle_proof`, `root_from_path`, `verify_faithfulness` |
| `merkle_v1.py` | **frozen** v1 construction — bare pair hashing, odd levels padded by duplication. Kept so v1.x bundles stay verifiable. Do not "fix" its known weaknesses here |
| `merkle_v2.py` | current construction — RFC 6962 §2.1 |
| `challenge.py` | Probabilistic challenge generation (HMAC-SHA256 seed, SHA-256 counter-mode PRNG) |
| `response.py` | `build_response`, `verify_response` — Merkle-path selective disclosure |
| `metrics.py` | Verifier-side metric recomputation (#13): `check_metric` (exact, whole log), `bind_disclosed_samples` + `check_mean_metric` (Hoeffding bound on a challenged subset — "consistent", never "verified") |
| `adapters/` | `AdapterBase` ABC + six adapters: `InspectAILogAdapter` (reads `.eval` files), `InspectEvalsAdapter`, `PiSessionAdapter` (these three exported from the package root), `LmEvalAdapter`, `AiluminateAdapter`, `WandbRunAdapter` |

**Format version: v2** (since 2026-08-18; `build_bundle` writes `"v2"`, package 2.0.0; 2.1.0 adds claim_hash and metrics.py, still format v2). Bundles have a `bundle_hash` (full content) and `content_hash` (excludes `Bundle.meta` provenance block). v1/v1.1/v1.2 bundles remain valid and are **not** rewritten.

**v2 changed the Merkle construction and nothing else** — RFC 6962 §2.1, adopted whole: `0x00`/`0x01` leaf/node domain separation, odd subtrees promoted rather than duplicated, empty and single-leaf cases defined. Every other part of `spec/attestation_format_v1.md` (fields, JCS encoding, both hashes, challenge-response, threat model) still applies. Spec: `spec/attestation_format_v2.md`.

⚠️ **When verifying, pass the bundle's own `format_version`** — never inherit the library default. A root is 64 hex chars under every version, so checking a v1.2 root under v2 returns *does not verify*, indistinguishable from tampering. `construction_for()` raises `UnknownFormatVersion` rather than guessing. This bit three call sites during the v2 release; assume it will bite again.

**Open format backlog: `spec/format-backlog/`** — seven candidate additions raised by outside
implementers, each naming who raised it and when. Not version-named on purpose: `spec/v2-backlog/`
closed when its items shipped and new findings had nowhere to go. ⚠️ `Bundle.meta` is the obvious
home for judge configuration / rubric versions and the **wrong** one — `meta` is excluded from
`content_hash`, so two runs scored by different judges would compare as equivalent.

Conformance vectors: `tests/vectors/merkle_v1_2.json` (+ `_odd_node`) and `merkle_v2.json`. The v1.2 files are **frozen** — they are the evidence old bundles still verify. A new version adds a file, never edits these.


## The bridge to ValiChord

`protocol.py` predicts the `ExternalHash` the ValiChord protocol produces for a submitted bundle.
Since the split no CI sees both repositories: any change to how ValiChord hashes submitted data
must be mirrored here (see the warning in `protocol.py`). The rules any integration with the
protocol must meet are ValiChord's `docs/PROTOCOL_INTEGRATION_BOUNDARY.md`.
