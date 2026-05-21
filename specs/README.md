# wysm reference

canonical specification of the cyber wysm runtime. this is the source of truth — when code and reference disagree, fix reference first, then propagate to code.

`wysm` is cyber's name for its wasm runtime. `wasm`/`WASM` is the WebAssembly format. `wasmi` is the upstream project this fork tracks.

## specifications

| page | scope | status |
|------|-------|--------|
| wysm.md | overview, fork rationale, four enhancements, async contract, sandboxing, execution profiles, trident on-ramp, resource model, prior art, borrowed patterns, rejected alternatives | canonical |

## upstream

forked from [wasmi-labs/wasmi](https://github.com/wasmi-labs/wasmi) at commit `a8f6d806` (v2.0.0-beta.2, 2026-03-04). upstream is tracked via the `upstream` git remote. merge upstream every release; document divergence in `patches.md` once the first non-trivial patch lands.

## dependencies

wysm is a **leaf** in cyber's runtime dependency graph. it depends on no other cyber crate. it is consumed by:

| consumer | path | how |
|----------|------|-----|
| cyb | `~/cyber/cyb` | embeds wysm as the soul runtime |
| rune | `~/cyber/rune` | calls `~host:%wasm.module.fn.args` |
| trident | `~/cyber/trident` | reads `wasmi_ir` 3.0 via `wysm-trident` for WASM → LIR lowering |

## reading order

1. wysm.md — full spec
2. (future) patches.md — divergence ledger vs upstream
3. (future) jets.md — jet authoring contract
4. (future) metering.md — finite-wasm cost table

wysm is at Kelvin K140 — the spec is expected to mutate as cyb's soul layer matures. no freeze planned.
