# agent collaboration

principles for working with AI coding agents on the cyber wasm runtime. read this and the foundational documents to have full context:

- [[cyber/engineering]] — pipeline contracts, dual-stream optimization, verification dimensions
- [[cyber/quality]] — 12 review passes, severity tiers, audit protocol
- [[cyber/projects]] — repo layout, namespace conventions, git workflow
- [[cyber/documentation]] — Diataxis framework, reference vs docs, spec before code

## auditor mindset

the project is supervised by an engineer with 30 years of experience. deception does not work. never fake results, never substitute "looks correct" for "is correct". one time correctly is cheaper than five times beautifully.

## what this repo is

a hard fork of [wasmi](https://github.com/wasmi-labs/wasmi) v2.0.0-beta.2 (commit `a8f6d806`), extended with four cyber-native enhancements: jet substitution, finite-wasm metering, an actor harness, and a trident-lowering on-ramp. the runtime cyb embeds to execute conventional WebAssembly programs (souls) loaded from radio.

see `specs/wasm.md` for the full design.

## companion repos

| repo | path | role |
|------|------|------|
| nox | `~/cyber/nox` | sibling runtime; jet authors borrow nox jet patterns |
| rune | `~/cyber/rune` | calls into wasm via `~host:%wasm.module.fn.args` |
| trident | `~/cyber/trident` | reads `wasmi_ir` for WASM → LIR lowering |
| cyb | `~/cyber/cyb` | embeds wasm as soul runtime |
| radio | `~/cyber/radio` | source of soul particles loaded into the engine |
| cyber | `~/cyber/cyber` | knowledge graph (parent subgraph) |

## upstream tracking

origin is `wasmi-labs/wasmi` as the `upstream` remote. publish cyber-specific commits on the `main` branch only. merge upstream releases as their own commits; never rebase to hide divergence.

```
git fetch upstream
git merge upstream/main
```

document non-trivial divergence in `specs/patches.md`.

## do not touch zones

- `Cargo.toml` workspace dependency versions — discuss before changing
- `crates/wasmi/`, `crates/wasmi_ir/`, `crates/wasmi_core/`, `crates/wasmi_collections/` — upstream code. modify only when adding a jet hook or metering hook; document the patch in `specs/patches.md`
- `LICENSE-APACHE`, `LICENSE-MIT` — preserve dual-license attribution
- `specs/` — canonical spec, change there first then propagate to code

cyber-specific code lives in **new** crates: `crates/wasmi-jet`, `crates/wasmi-meter`, `crates/wasmi-harness`, `crates/wasmi-trident`. additions to these are free.

## git workflow

- atomic commits — one logical change per commit
- conventional prefixes: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`
- never amend upstream-derived commits — keep the upstream lineage clean for merge tooling
- commit on completion; do not wait for the user to say "commit"

## build matrix

| target | dispatch | unstable rust |
|--------|----------|---------------|
| macOS aarch64 | tail-call (`become`) | yes |
| macOS x86_64 | tail-call (`become`) | yes |
| Linux aarch64 | tail-call (`become`) | yes |
| Linux x86_64 | tail-call (`become`) | yes |
| Windows x86_64 | tail-call (`become`) | yes |
| Android (arm64) | portable-dispatch | no |
| embedded / no_std | portable-dispatch | no |

```
cargo build --no-default-features --features unstable     # tier-1
cargo build --no-default-features --features portable-dispatch  # tier-2
```

## shell: nushell

use `nu -c '...'` or `nu script.nu` for scripting. reserve bash for git.

## testing

- unit tests in the crate they test (`#[cfg(test)]`)
- integration tests for new crates go in `crates/<crate>/tests/`
- **differential tests for jets** — every jet is tested against its reference wasm implementation over a corpus. mismatches are CI errors, not silent fallback
- benchmark against upstream wasmi on every PR — regressions in dispatch throughput are bugs
