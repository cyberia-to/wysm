---
tags: cyber, cyb, runtime, article
alias: wasm runtime, wasm sandbox, soul runtime
crystal-type: entity
crystal-domain: cyber
---
# wasm

the WASM runtime tier of soft3. a hard fork of [wasmi](https://github.com/wasmi-labs/wasmi) v2.0.0-beta.2, extended with four cyber-native enhancements: jet substitution, finite-style metering, an actor harness, and a trident-lowering on-ramp. the runtime cyb embeds to execute conventional programs (souls) loaded from radio, sandboxed and resource-bounded, with a migration path to provable nox execution.

## position in soft3

| runtime | compiler | backend | what runs | proof contract |
|---------|----------|---------|-----------|----------------|
| [[nox]] | trident | native (Rs, AMX) | proven .nox programs, jets | unconditional |
| [[glia]] | tru | ANE, AMX, Metal | .model inference | conditional on model |
| **wasm** | external toolchains | wasmi v2 + jets | WASM modules, rune host jets | conditional on host |
| wgpu | external toolchains | Metal, Vulkan, WebGPU | GPU compute shaders | conditional on host |

wasm is the seat for conventional programs — anything that compiles to WebAssembly, deployed as a particle, executed as a soul. its proof contract is the weakest of the four runtimes: a host call returns a noun, the noun is recorded as a witness in the calling [[rune]] program, the surrounding pure computation is provable conditional on that witness. wasm itself carries no [[Nox]] trace. this is the price of accepting conventional programs.

the long-term arc is migration: WASM → trident LIR → [[Nox]] (with [[zheng]] proof). wasm is the soft path; nox is the hard one. souls start in wasm and graduate to nox over time as their inner loops get lowered through trident.

## fork rationale

wasmi is consumed as a fork, not a dependency, because cyb does substantive work on the runtime itself:

1. **jet substitution** is a new subsystem. it lives below `wasmi::engine::executor` and must intercept dispatch — not bolt-on after-the-fact. cleanest as an internal crate, not a wrapping layer.
2. **metering** is module-rewrite-time instrumentation (NEAR `finite-wasm` model). it sits between `wasmparser` and `wasmi::Module::new`. wrapping doesn't compose well; folding into the loader does.
3. **harness** ports lunatic's actor/mailbox/capability-token patterns onto wasmi. lunatic itself hard-codes wasmtime. the port lives best as cyber's own code with wasmi-as-engine, not as a fork of lunatic.
4. **trident lowering** reads wasmi's IR (the new `wasmi_ir` 3.0 crate, public types) and emits trident LIR. tightly coupled to wasmi internals — must move with them.

upstream wasmi is small, audited (SRLabs 2023, Runtime Verification 2024), and on a roughly-quarterly release cadence. the fork commits to merging upstream every release. divergence is intentional but bounded: cyber-specific code lives in new crates (`wasmi-jet`, `wasmi-meter`, `wasmi-harness`, `wasmi-trident`); upstream crates are touched only when the dispatch loop needs a jet hook or the loader needs a metering hook. patches are tracked in `specs/patches.md`.

## base — wasmi v2.0.0-beta.2

forked from upstream commit `a8f6d806` (2026-03-04). cyber-base branch tracks this commit; upstream is the `upstream` remote.

what v2 already provides that earlier wasmi did not:

- **new internal IR (`wasmi_ir` 3.0)**, public Rust types — the foldable surface trident lowering reads from
- **tail-call dispatch via Rust `become`** (PR #1655) behind `--features unstable`. resolves issue #1101. stitch-style threaded code, no fork patch needed.
- **`portable-dispatch` / `indirect-dispatch`** feature flags trading throughput for portability
- **rewritten `ResourceLimiter`** (#1817) with cleaner enforcement of `trap_on_grow_failure`
- **`sysv64` calling convention on x86_64** (#1792) for execution handlers — better codegen than default Rust ABI
- edition 2024, MSRV 1.86

the cyb build enables `unstable` by default on tier-1 targets (macOS x86_64/aarch64, Linux x86_64/aarch64, Windows x86_64). `portable-dispatch` is the fallback for Android and embedded.

## four enhancements

### 1. jet substitution

the cyber-unique contribution. at module load, every function body is normalized (strip locals, alpha-rename, canonicalize block structure) and hashed. the hash is looked up in a global jet table. on hit, the function body is replaced with a host trampoline to a native Rust implementation.

| layer | what it is | LOC budget |
|-------|-----------|------------|
| `wasmi-jet/normalize.rs` | canonical form: strip locals, alpha-rename, sort case branches | ~400 |
| `wasmi-jet/registry.rs` | hash → JetFn dispatch table; loader hook | ~300 |
| `wasmi-jet/jets/*` | the jet library — one file per algorithm | ~200 per jet |

target jet library (in order of expected impact):

| jet | source we match | speedup |
|-----|----------------|---------|
| blake3 | hashes/blake3 reference impl | 50–100× (NEON/AVX-512 vs scalar) |
| sha256 | rust-crypto sha2 | 20–50× |
| poseidon2 | hash family used by hemera | 20–80× |
| keccak-256 | tiny-keccak | 30–60× |
| ed25519-verify | ed25519-dalek | 10–30× |
| secp256k1-verify | rust-secp256k1 | 10–30× |
| bls12-381 pair | blst | 5–15× |
| goldilocks field-arith inner loops | nebu | 5–10× |

the design borrows directly from the [[nox]] jet model. jet identity is itself a [[particle]]; jets are content-addressable; upgrading a jet produces a new particle and does not invalidate prior compilations. dispatch is structural (hash match) — no source-language assumptions, jets fire whether the module was compiled from Rust, C, Zig, or AssemblyScript.

equivalence is asserted by the jet author and verified by differential testing: the reference wasm function and the native jet are both executed on a corpus; outputs must match for every input. failures are CI errors, not silent fallback. when in doubt, drop the jet — incorrect jets are a soundness hole, not a perf miss.

### 2. tail-call dispatch — already upstream

shipping in wasmi v2.0.0-beta.2 behind `--features unstable`. the cyb build enables this by default. stitch-shape threaded code via Rust's `become` keyword (RFC 2691). measured by upstream: ~2-3× over the match-loop dispatch on Coremark-class workloads, with regressions on some opcode shapes (cf. Pulley issue wasmtime#9995). the fallback `portable-dispatch` is kept for hosts where `become` cannot be lowered.

no patch required. the work is testing: benchmark cyb's actual soul workloads against both dispatch modes, ship whichever wins per-arch.

### 3. metering — finite-wasm pattern

module-rewrite-time deterministic gas instrumentation, modeled on [near/finite-wasm](https://github.com/near/finite-wasm). lives in a new crate `wasmi-meter` sitting between `wasmparser` and `wasmi::Module::new`.

the rewrite injects:
- per-instruction gas counter decrement (collapsed per basic-block)
- per-instruction stack-depth check (collapsed per function entry)
- a global trap when gas reaches zero

| property | choice | reason |
|----------|--------|--------|
| cost table | per-opcode, fixed at module-instantiation time | deterministic; no host-side state |
| gas representation | wasm `i64` global | engine-agnostic — works the same under wasmi, wasmer, future engines |
| overflow | trap-on-zero | simpler than negative-gas semantics; predictable |
| float ops | non-deterministic, gated separately | floats not allowed in proof-relevant souls |

this decouples metering from the engine forever. swapping wasmi → wasmer-singlepass → trident-lowered nox does not require rewriting the gas contract — the metered module carries its own gas semantics.

### 4. harness — actor model on wasmi

a port of [lunatic](https://github.com/lunatic-solutions/lunatic)'s actor patterns onto wasmi, since lunatic itself is welded to wasmtime and stalled since May 2023. the four patterns worth porting:

| pattern | source | what to take | wasmi-specific work |
|---------|--------|--------------|--------------------|
| capability tokens | `lunatic-process/src/config.rs` + each `*-api` crate | the config structs + permission-gate pattern | trivial; engine-agnostic |
| guest mailbox API | `lunatic-rs/src/mailbox.rs` | the syscall-shape API the guest sees | trivial; engine-agnostic |
| spawn / link / monitor | `lunatic-process/` | the supervision tree shape | rewrite over wasmi `Store`/`Instance`/`Linker`; ~2–4 weeks |
| QUIC distributed nodes | `lunatic-distributed/` | **skip** | [[radio]] already covers this |

target LOC: ~3k for the cyb-side implementation. lunatic's design is sound; only the implementation is wasmtime-coupled.

## async contract — rune `hint`, not async host

wasmi does not support async host functions. wasmtime does (`Func::wrap_async`). this is correct as a default — async-from-the-guest is a leaky abstraction (the soul cannot reason about whether the host is awaiting). cyb sidesteps the gap entirely via [[rune]]'s `hint` primitive:

```
=/  data  ~hint:%get-text.cid
```

the soul yields at `hint`. the host drives the future. when the IPFS read returns, the runtime resumes the soul with the data in the appropriate subject slot. this is the same async model rune already specifies and is the only model where async semantics are visible to the soul, the host, and the proof witness alike.

the gap-class workaround for non-rune wasm modules that call `await` directly is `tokio::runtime::block_on` on a per-soul thread — expensive, bounded, only used for foreign modules that did not author against the hint protocol. document this as an anti-pattern.

## sandboxing model

| layer | what it bounds | mechanism |
|-------|---------------|-----------|
| memory | linear memory size | wasmi `ResourceLimiter` — max pages per Store |
| compute | instructions executed | `wasmi-meter` global gas counter |
| host-call rate | concurrent IPFS/signer/network requests | per-soul semaphores in host shims |
| filesystem | none allowed | no WASI filesystem mount; document deny-by-default |
| network | only via host shims, never WASI sockets | no `wasi_snapshot_preview1::sock_*` imports linked |
| time | rune `~now` slot in subject only | no WASI `wall_clock` — non-determinism leak |

bandwidth is bounded **one layer up**, at the cyb host shims — not at the WASM engine. the existing cyb `ipfs queue` (cyb/docs/backend.md) is the chokepoint; per-soul rate limits live there, not in wasm.

## trident-lowering on-ramp

`wasmi-trident` reads `wasmi_ir` 3.0's public types and emits trident LIR. mirrors trident's existing `mir2nox.rs` (Rust MIR → nox) and `compile/wasm.rs` (nox → wasm). closes the soft3.md migration loop:

```
.tri → trident LIR → nox (existing)
.rs.mir.json → trident LIR → nox (existing)
.wasm → wasmi_ir → trident LIR → nox (this work)
```

phase 0: hand-pick a subset of wasmi-IR ops (i32/i64 arithmetic, locals, calls, no memory). lower to trident LIR. round-trip a few hello-world wasm modules. five tests, modeled on `trident/src/compile/wasm.rs`'s existing five tests.

phase 1: linear memory mapping to nox state slots. control flow lowering (block/loop/br_if to LIR structured control).

phase 2: float and SIMD. these may stay un-lowered indefinitely — if a soul uses floats, it stays in wasm and never graduates to nox.

phase 3: opens — should the lowering pass also do circuit-size-aware optimization, or is that pure trident's job? leans pure trident.

## resource model — what each axis means in cyb

| axis | unit | bound by | when bound |
|------|------|----------|-----------|
| volatile memory | wasm pages (64 KiB) | `ResourceLimiter` | per Store, set at instantiation |
| persistent memory | rune `~mem` subject slot bytes | rune runtime, not wasm | per soul |
| compute | gas units | `wasmi-meter` | per Store; refilled by hint, signer-confirmed top-up, or staking |
| bandwidth | host-call concurrency | host shim semaphore | per soul × per shim |
| host-call count | hint-yields, signer prompts | rune runtime caps | per epoch |

cyb caps are signed by the [[neuron]] running the soul, not by the soul itself. a soul cannot grant itself more gas; it requests, the neuron approves (or auto-approves under policy). this matches the cybermark `~caps` subject slot from rune.md.

## companion repos

| repo | path | role |
|------|------|------|
| wasm | `~/cyber/wasm` | this fork — engine + jets + meter + harness + trident on-ramp |
| nox | `~/cyber/nox` | sibling runtime; jet authors borrow from nox jet patterns |
| rune | `~/cyber/rune` | calls into wasm via `~host:%wasm.module.fn.args` |
| trident | `~/cyber/trident` | consumer of `wasmi-trident` for WASM → LIR lowering |
| cyb | `~/cyber/cyb` | the embedding host — owns the soul lifecycle |
| radio | `~/cyber/radio` | source of soul particles loaded into the engine |

depends on: `wasmi_ir`, `wasmi_core`, `wasmi_collections` (internal). does not depend on any cyber runtime crate — wasm is a leaf in the dependency graph by design.

consumed by: cyb (binary), rune-interp (host call dispatch), trident (via `wasmi-trident` lowering).

## roadmap

estimation per cyber convention: 1 pomodoro = 30 min, 1 session = 6 pomodoros.

| phase | what lands | sessions |
|-------|-----------|----------|
| 0 | fork in place, `unstable` dispatch enabled, baseline benchmarks recorded | 1 |
| 1 | `wasmi-jet` infrastructure (normalize + registry + loader hook) + first 3 jets (blake3, sha256, poseidon2) | 2 |
| 2 | extended jet library (keccak, ed25519, secp256k1, bls, goldilocks) + differential test corpus | 2 |
| 3 | `wasmi-meter` — finite-wasm-style module rewriter, opcode cost table, gas-global injection | 3 |
| 4 | `wasmi-harness` — capability tokens, guest mailbox, supervision tree on wasmi | 4 |
| 5 | rune host bindings ported to wasm runtime (replacing nothing — there's no current implementation) | 2 |
| 6 | cyb integration — soul launcher reads particles from radio, executes via wasm runtime | 3 |
| 7 | `wasmi-trident` phase 0 — i32/i64/locals/calls round-trip through trident LIR | 4 |
| 8 | upstream merge from next wasmi release; verify no regressions | 1 |

total: ~22 sessions for a usable production runtime + first lowering experiment. roughly two months at sustained pace.

## rejected alternatives

reflected here so future contributors do not redebate the choices.

### wasmtime as the embedded engine

rejected. ~250k LOC of fork surface against cyber's ~200k-LOC total runtime stack. Cranelift is the right answer for trusted code authored in-house; it is the wrong answer for adversarial souls loaded from radio. cosmwasm pins wasmer-singlepass, NEAR forked wasmer-singlepass, polkadot is leaving wasm for PolkaVM/RISC-V — *nobody runs Cranelift on adversarial code*. cyb follows the pattern.

### wasmer as the embedded engine

rejected. diverged from the WASI component model. the component model is the composition story for [[cybermark]]-addressable souls. wasmer trades that for WASIX, which cyb does not need.

### WAMR (WebAssembly Micro Runtime)

rejected. best mobile story (iOS/Android NDK clean), but cyb does not target iOS per project policy and Android wasmi works fine. importing ~150k LOC of C to dodge a non-problem is poor trade.

### lunatic as the actor harness

rejected as a dependency; **adopted as a design source**. last release v0.13.2 (May 2023), stalled; code welded to wasmtime via `wasmtime::Store<T>` everywhere. port the four patterns into our own code over wasmi.

### copy-and-patch compilation (Xu & Kjolstad, OOPSLA '21)

deferred. the algorithm works (Stanford wasm prototype hit 39–63% over Liftoff with ~3-5k LOC runtime), but the org-level integration cost (stencil pipeline, Clang-as-library, W^X policy, three-arch debugging) is heavy for a small team. CPython's 30-month CAP-JIT experience is sobering — at parity or slower with the interpreter after sustained effort. revisit only after jets+tail-call land and we have telemetry on what's left slow.

### PolkaVM (wasm → RISC-V)

deferred. PolkaVM is fast and small (~25k LOC, ~0.9× Cranelift), but there is no wasm-to-RISC-V frontend; building it is a year of work. Parity left wasm for RISC-V entirely (pallet-revive). revisit if cyb decides to leave WebAssembly as the input format.

### LuaJIT-style tracing JIT

rejected. tracing JIT on wasm was tried in the V8 era and abandoned. wasm control flow is already structured — traces over-specialize and lose to baseline.

### neural compilation directly on wasm IR

deferred. trident's GNN+Transformer optimizer is sized for register IR. retraining on wasm stack-IR requires lifting to local-SSA first, plus a (input, faster-equivalent) corpus that does not yet exist. the cleaner neural path is wasm → trident LIR → existing trident neural optimizer, which is the on-ramp we're already building. parallel pursuit, not a replacement.

## do not touch zones

| zone | reason |
|------|--------|
| `Cargo.toml` workspace deps | discuss before changing — affects upstream-merge cleanliness |
| `crates/wasmi/`, `crates/wasmi_ir/`, `crates/wasmi_core/`, `crates/wasmi_collections/` source | upstream code. modify only when adding a jet hook or metering hook; document the patch in `specs/patches.md` |
| `LICENSE-APACHE`, `LICENSE-MIT` | upstream license — preserve dual-license attribution |
| `crates/c_api/` | unused by cyb; keep buildable for upstream-merge sake but do not extend |

cyber-specific code lives in **new** crates: `crates/wasmi-jet`, `crates/wasmi-meter`, `crates/wasmi-harness`, `crates/wasmi-trident`. additions to these are free.

## Kelvin discipline

wasm-K140 (working spec, mutations expected). the runtime is not protocol-critical — souls run locally per neuron, not in consensus. no need to freeze the spec. upstream wasmi follows its own semver; we track its releases.

---

discover all [[concepts]]
