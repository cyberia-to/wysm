# wasm reference

canonical specification of the cyber wasm runtime. this is the source of truth — when code and reference disagree, fix reference first, then propagate to code.

## specifications

| page | scope | status |
|------|-------|--------|
| wasm.md | overview, fork rationale, four enhancements, async contract, sandboxing, trident on-ramp, resource model | canonical |

## upstream

forked from [wasmi-labs/wasmi](https://github.com/wasmi-labs/wasmi) at commit `a8f6d806` (v2.0.0-beta.2, 2026-03-04). upstream is tracked via the `upstream` git remote. merge upstream every release; document divergence in `patches.md` once the first non-trivial patch lands.

## dependencies

wasm is a **leaf** in cyber's runtime dependency graph. it depends on no other cyber crate. it is consumed by:

| consumer | path | how |
|----------|------|-----|
| cyb | `~/cyber/cyb` | embeds wasm as the soul runtime |
| rune | `~/cyber/rune` | calls `~host:%wasm.module.fn.args` |
| trident | `~/cyber/trident` | reads `wasmi_ir` 3.0 via `wasmi-trident` for WASM → LIR lowering |

## reading order

1. wasm.md — full spec
2. (future) patches.md — divergence ledger vs upstream
3. (future) jets.md — jet authoring contract
4. (future) metering.md — finite-wasm cost table

wasm is at Kelvin K140 — the spec is expected to mutate as cyb's soul layer matures. no freeze planned.
