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

NEAR (`finite-wasm`) and DFINITY (IC embedder static instrumentation) independently arrived at the same design — module-rewrite, engine-agnostic, deterministic across nodes. the convergence is the design's validation. see "prior art and borrowed patterns" for the full cross-reference.

### 4. harness — actor model on wasmi

a port of [lunatic](https://github.com/lunatic-solutions/lunatic)'s actor patterns onto wasmi, since lunatic itself is welded to wasmtime and stalled since May 2023. the four patterns worth porting:

| pattern | source | what to take | wasmi-specific work |
|---------|--------|--------------|--------------------|
| capability tokens | `lunatic-process/src/config.rs` + each `*-api` crate | the config structs + permission-gate pattern | trivial; engine-agnostic |
| guest mailbox API | `lunatic-rs/src/mailbox.rs` | the syscall-shape API the guest sees | trivial; engine-agnostic |
| spawn / link / monitor | `lunatic-process/` | the supervision tree shape | rewrite over wasmi `Store`/`Instance`/`Linker`; ~2–4 weeks |
| QUIC distributed nodes | `lunatic-distributed/` | **skip** | [[radio]] already covers this |

target LOC: ~3k for the cyb-side implementation. lunatic's design is sound; only the implementation is wasmtime-coupled.

the supervision tree is built on a `PausedExecution` primitive borrowed from DFINITY's IC embedder: a soul that has paused — by fuel exhaustion, hint yield, or signer prompt — is a first-class state in the supervisor's model, distinct from "running" or "dead". the supervisor can snapshot, replay, or hot-reload paused souls without losing in-flight work. it is the wasm-level primitive that completes [[rune]]'s solid-state-interpreter model — the kernel function `(subject, event) -> (new-subject, effects)` was already replayable; `PausedExecution` makes it physical. ~20 lines of trait definition, re-implemented independently of IC-1.0-licensed code.

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

## prior art and borrowed patterns

three production WASM-on-chain systems run deterministic adversarial WASM at scale. each independently arrived at variants of the same architecture. cyb borrows specific patterns from each.

### production parallels

| project | engine | metering | size of embedder | what we copy |
|---------|--------|----------|------------------|--------------|
| **Stellar Soroban** | wasmi 0.31 fork (Apache-2.0) | engine fuel + 80+ host-fn cost types | ~25k LOC | deterministic config, fuel refill, three-tier storage, x-macro host registry |
| **NEAR Protocol** | wasmer-2 singlepass fork (~40k LOC) | `finite-wasm` module rewrite | ~8k LOC | metering pattern (already adopted as enhancement #3) |
| **DFINITY IC** | wasmtime + ~19k embedder (IC-1.0) | static bytecode injection at install time | ~19k LOC | `PausedExecution`, snapshot-at-await, NaN canonicalization |

common ground across all three: nobody runs Cranelift on adversarial code. all use singlepass JIT or interpreter. all enforce determinism (floats banned or NaN-canonicalized, threads off, sorted iteration). all use module-rewrite or fuel for metering, never after-the-fact accounting. cyb's wasmi v2 + finite-wasm-style metering + Soroban deterministic profile is the convergent design these three arrived at variants of.

### borrowed: Soroban deterministic wasmi config

the most battle-tested deterministic wasmi profile in production. cyb copies it verbatim for proof-relevant souls. source: `stellar/rs-soroban-env/soroban-env-host/src/budget/wasmi_helper.rs` (Apache-2.0).

| feature | setting |
|---------|---------|
| floats | `floats(false)` + reject any FP opcode at parse |
| bulk-memory | ON |
| mutable-global | ON |
| sign-extension | ON |
| SIMD, threads, multi-value, reference-types, tail-call, extended-const, multi-memory, shared memory, component-model | OFF |
| start functions | rejected |

this is the "deterministic" profile, used for souls whose execution trace is a proof witness. cyb additionally ships a "permissive" profile that re-enables SIMD and floats for UI/render souls that have no proof contract; the choice is per-soul, declared in particle frontmatter.

**vendoring note**: the helper config is small (~30 LOC) and stable. literal-copy from `stellar/rs-soroban-env/soroban-env-host/src/budget/wasmi_helper.rs` into `crates/wasmi-meter/src/profile.rs`, preserve Apache-2.0 attribution in the file header, sync against upstream every Soroban release (they update it for new wasm proposals as the determinism story evolves).

### execution profiles — deterministic vs permissive

cyb is a soul runtime, not a generic wasm host. souls choose one of two profiles in `.cyb` frontmatter:

```toml
profile = "deterministic"   # proof-relevant: trace is a witness
profile = "permissive"      # UI/render/inference: result accepted on faith
```

| dimension | deterministic | permissive |
|-----------|---------------|------------|
| wasm features | Soroban config (above) | all valid wasmi v2 features |
| floats | banned at parse | allowed |
| SIMD | OFF | ON |
| trust contract | trace is a witness; trident-lowerable to nox eventually | rune host-call witness; result accepted on faith |
| who picks | semcon authors, proof-relevant souls | UI souls, render, inference, general compute |
| default | no | **yes** |
| typical workload | hash, signature, deterministic state transition | three.js-style 3D, audio DSP, ML pre/post-processing |

both profiles **deny WASI imports**. souls must be linked against `cyb-sdk` (the cyb host binding set, mirroring `soroban-sdk` / `ic-cdk` / `cosmwasm-std`). a WASI shim layer rewrites common WASI imports (`fd_write` → console log, `clock_time_get` → `~now`, filesystem ops → deny) at module load — same pattern as DFINITY's `ic-wasi-polyfill` (community-maintained). this is the second, larger gate on "running random wasm": almost no off-the-shelf `.wasm` will load without either being recompiled against `cyb-sdk` or running through the WASI shim.

the honest characterization: **cyb runs souls, not random wasm**. a soul is a wasm module authored for cyb's host API in a chosen profile. every production wasm embed (Soroban, Spin, IC, wasmCloud) does the same; cyb just makes the boundary explicit.

### borrowed: Soroban fuel refill pattern

host pushes fuel into the VM before each call (`add_fuel_to_vm`) and pulls remainder back on return (`return_fuel_to_host`). budget is held by the host, not the VM; the VM is a transient fuel-burner. Soroban needed this because wasmi 0.31 made `FuelCosts` private — they patched it back open in their fork. wasmi v2 keeps these accessors public via `--features unstable`. integrates with `wasmi-meter` for the metering side.

### borrowed: Soroban three-tier storage

souls have three storage classes, not one:

| tier | lifetime | restore | use case |
|------|----------|---------|----------|
| persistent | TTL'd; archives on expiry, restorable via signed restore | yes | durable soul identity, balances, model weights |
| temporary | TTL'd; vanishes irreversibly on expiry | no | sessions, oracle reads, signer cache |
| instance | shares contract code TTL; loaded every invoke | n/a | small per-soul hot config map |

source: Stellar CAP-0066 (Protocol 23, "Whisk"). maps cleanly onto [[bbg]] dimensions: persistent → bbg, temporary → in-memory, instance → particle frontmatter. [[rune]]'s `~mem` slot becomes persistent by default; souls opt into temporary or instance tiers via `~caps`.

### borrowed: DFINITY snapshot-at-await

instead of stackful coroutines (wasmi has no good story for these), commit state at the await point, spin up a fresh instance for the callback, only roll back post-await state on failure. dovetails with rune's `hint` primitive — hint is already a snapshot point in the semantics. mechanism:

1. soul calls a host import that needs to await (e.g. `host.ipfs_read`)
2. host enqueues the work, registers a callback (fn pointer + env i32) in the soul's continuation table
3. soul's current state is committed; the wasm instance is torn down
4. when the work completes, a fresh instance is spun up from the snapshot and the callback is dispatched
5. if the callback traps, only post-await state rolls back to the snapshot; pre-await effects stay committed

every `hint` is an interleaving point — same security footgun as IC ("reentrancy on await"). document this in the soul authoring guide as a load-bearing concern, not a footnote.

### borrowed: DFINITY NaN canonicalization

one-line engine config for cross-host determinism. cyb's deterministic profile rejects FP entirely (via Soroban config), so this is subsumed. for the permissive profile (UI souls with floats), enable NaN canonicalization explicitly at the engine level. relevant only because some hosts may emit different NaN bit patterns from the same operation — proof-relevant code must agree bit-exactly.

### borrowed: Soroban x-macro host-fn registry

`call_macro_with_all_host_functions` is Soroban's single source of truth for ~190 host functions: declare once, get zero-cost dispatch + version gating + ABI documentation. cyb's host bindings layer (the existing cyb scripting API from `cyb/docs/scripting.md`) adopts the same pattern for the rune ↔ wasm shim. one macro invocation replaces hand-maintained dispatch tables in three places (engine glue, guest SDK, version manifest).

### deferred: Soroban module cache (CAP-0066)

Stellar's Protocol 23 ("Whisk", September 2025) introduced a module cache: contracts are parsed and translated **once per transaction**, then reused across every invocation in the same tx. for cyb the analogous unit is "once per neuron session" or "once per soul lifetime". translation is the dominant cost for short host calls. defer until benchmarks show translation as the bottleneck — likely after jets + tail-call land. wasmi v2's lazy compilation already amortizes some of this; the explicit cross-invocation cache is a v2-of-cyb optimization.

### key files to study (with line counts)

| file | LOC | what to look at |
|------|-----|-----------------|
| `stellar/rs-soroban-env/soroban-env-host/src/budget/wasmi_helper.rs` | ~30 | the deterministic profile — copy verbatim |
| `stellar/rs-soroban-env/soroban-env-host/src/vm.rs` | ~600 | `add_fuel_to_vm` / `return_fuel_to_host`; Vm wrapper pattern |
| `stellar/rs-soroban-env/soroban-env-host/src/vm/dispatch.rs` | varies | x-macro host-fn registry expansion |
| `dfinity/ic/rs/embedders/src/wasm_executor.rs` | ~860 | `PausedExecution` trait, lines 120–166 |
| `dfinity/ic/rs/embedders/src/wasm_utils/instrumentation.rs` | ~1,690 | static bytecode metering injection — readable in an afternoon |
| `near/finite-wasm` (separate repo) | ~5,000 | module-rewrite metering as standalone Rust crate |
| `lunatic-solutions/lunatic/crates/lunatic-process/src/config.rs` | ~1,000 | capability tokens pattern |

read these before writing the corresponding cyb-side code. each is the production reference for one pattern. cyb's implementations will look structurally similar — that is the design intent, not a coincidence.

### bonus findings worth knowing about

| finding | source | relevance to cyb |
|---------|--------|------------------|
| Stellar Audit Bank — $3M across 40+ audits via Certora, Veridise, RV, OtterSec, CoinFabrik, QuarksLab, Coinspect | Stellar Foundation | when cyb's soul runtime needs adversarial audit, this is the template — pick 2–3 firms, scope tightly to the new crates (jet/meter/harness), don't re-audit upstream wasmi |
| Soroban's hybrid 2D budget (CPU instructions × memory bytes) | `soroban-env-host/src/budget.rs` | single-axis gas is a simplification — two-axis budget matches resource reality better. consider for cyb when souls start hitting memory walls rather than compute walls |
| DFINITY's `wasi2ic` + `ic-wasi-polyfill` (community, wasm-forge org) | `github.com/wasm-forge` | the WASI-shim pattern cyb needs for "run random .wasm" support. study the import-rewrite approach |
| Certora's Sunbeam Prover (Soroban-specific formal verification) | Certora | formal verification of *contracts*, not the runtime. relevant when cyb wants formal proofs over semcons; less relevant for the wasm runtime itself |
| Soroban's `RestoreFootprintOp` for archived persistent state | Stellar CAP-0046-08 | the "explicit restore step" pattern for cold storage. maps onto bbg cold-particle retrieval cleanly |

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

### CosmWasm (cosmwasm-vm) as the embedded engine

rejected for the cyb client runtime, retained as conceptual prior art for the [[semcons|semcon]] ABI. cosmwasm-vm is ~15k LOC and audited, but transitively pulls in wasmer-singlepass (~150k LOC), reintroducing the size problem wasmer was rejected for. the `execute`/`query`/`instantiate`/`migrate`/`sudo` contract shape is wrong for cyb souls, which are async UI callbacks — not synchronous request/response. the cw-cyber lineage points back to the [[go-cyber]] legacy chain, not toward [[soft3]]. **borrow the entry-point vocabulary** for semcons (`instantiate`, `execute`, `query`, `migrate`, `sudo`); do not embed the engine.

### DFINITY IC embedder (`dfinity/ic/rs/embedders/`)

rejected as a code dependency, **adopted as a design source** (see "prior art and borrowed patterns"). the IC-1.0 license grants only extend to code "running directly on the Internet Computer platform"; cyb is outside scope. additionally, the embedder is welded to wasmtime (~250k LOC), the same size problem we rejected at the engine layer. four patterns borrowed independently: `PausedExecution`, snapshot-at-await, NaN canonicalization, and static bytecode metering — the last of which validates our `finite-wasm` choice, since DFINITY and NEAR independently arrived at the same metering design.

### soroban-wasmi fork (stellar/wasmi)

deferred. Stellar's fork is 7 patches on top of upstream wasmi v0.31.1 (stack-machine era) — crate rename, custom-section exposure, `FuelCosts` accessors, an `InstanceCache` bugfix, version bump. cyb runs on v2.0.0-beta.2 (register-machine, ~2-4× faster, lazy compile, Stellar-funded Runtime Verification audit). Soroban stays on 0.31 only because of cost-model recalibration risk; cyb has no such anchor. revisit if specific Soroban patches prove useful — both custom-section exposure and `FuelCosts` accessors are already public in wasmi v2 via `--features unstable`.

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
