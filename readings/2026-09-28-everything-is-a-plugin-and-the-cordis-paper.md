# Everything-is-a-plugin, and the Cordis paper

Session date: 2026-09-14 (written up 2026-09-28)

Tree these findings were verified against: local `main` = `13029913cc`, based on upstream `b2e3b2a012` (`dsh 0.1.5-alpha.2`, 2026-09-09). Every `file:line` citation below was read in that tree; upstream has moved ~3800 commits since, so re-check a line number before quoting it elsewhere.

Two questions drove the session: how the repository actually delivers its "everything is a plugin" claim, and what the paper behind Cordis says.

## Table of contents

- [1. How "everything is a plugin" is achieved](#1-how-everything-is-a-plugin-is-achieved)
- [2. The paper](#2-the-paper)
- [3. How the two connect](#3-how-the-two-connect)
- [4. Loose ends](#4-loose-ends)
- [5. Provenance](#5-provenance)

## 1. How "everything is a plugin" is achieved

Five mechanisms carry the claim, in rough order of how much weight they bear.

**Cordis is vendored, not depended on.** `vendor/cordis` pins `cordis@4.0.0-rc.7` rescoped to `@deepseek-ai/cordis`, declared a peerDependency by every harness package (`vendor/README.md`). The framework layer is owned, auditable, and patchable — `vendor/README.md` logs 18 local modifications, most of them lifecycle hardening in `cordis/src/fiber.ts` (reentrant disposal gaps, effect creation rejected while `UNLOADING`, contained teardown-notification failures).

**There is no privileged core to patch.** 267 workspace packages across 50 groups. A context is a repository of services; a plugin claims a stable key (`ctx.tools`, `ctx.llm`, `ctx.sessions`) and other plugins find it *by key* rather than importing an implementation. The agent loop itself is an ordinary `Service` subclass mounted like anything else (`packages/core/agent-loop/src/index.ts:1`), as are the model adapter, tool registry, and session log. Replacing any of them is a configuration edit, not a patch.

**Registrations are revertible effects.** Every contribution goes through `ctx.effect()` or `ctx.on()` and hands back its disposer. `ToolRuntime.register()` documents and returns "the exact disposer that unregisters the tool" (`packages/core/tools/src/index.ts:1027`); `restrict()` and `guard()` follow the same shape. This is what makes mount-beside-the-others safe — unloading a plugin unwinds its registrations LIFO instead of leaving residue.

**Dependencies are declared, not sequenced.** Plugins name required services via `inject`, and activation is driven by service availability. The base bundle's patch file states it outright: "Row order carries no load semantics" (`packages/bundle/base/cordis.patch.yml:14`). No boot script orders the ~80 rows.

**Composition is data, behind one enforced entry point.** A running `dsh` is an empty root plus ordered patch layers: each bundle in the profile's listed order, then the profile's `cordis.patch.yml`, then the home-level one, then any `--patch` overlay. A patch targets a row by id and replaces its whole config. `dsh --profile web --dump-config` prints the resulting tree, and any row it prints can be replaced.

The enforcement is the part that keeps this honest. Every supported Node application starts at `dsh --profile <name>`, and `scripts/verify-application-entrypoints.ts` classifies every package bin, executable source, and root demo, rejecting any Node application path that bypasses `dsh` (`docs/architecture.md`, "Application launch"). Without that gate the architecture would erode the first time someone shipped a bespoke `main()`.

Two multipliers sit on top.

*Capability seams* — the Service Definition / Service Provider / Consumer triple — mean one provider swap moves the whole product. Filesystem and subprocess providers share an execution world, so pointing them at a remote sandbox carries Bash, PTY, and LSP along with no provider forks. A seam is complete or it is not a seam; adding a capability means designing all three roles.

*Scoped registration* (`packages/core/scope/README.md`) gives each agent its own registration world through `agent.ctx`. The registration context determines both visibility and ownership, "preventing a contribution from being visible in one scope but torn down with another". That is what lets one session run a different capability set from its neighbour via presets.

The reflexive case is `packages/extensions/`: `dsh-tool-cordis` exposes the live Cordis runtime to the model, which can inspect services and create, run, update, or remove dynamic packages in process. Definitions live in memory only and vanish on restart — it does not write repository files or touch `cordis.yml`. This is the harness editing its own plugin graph while running.

## 2. The paper

**[A Programming Paradigm for Spatiotemporal Composability](https://arxiv.org/abs/2608.25512)** — Yifan Shi (Peking University / DeepSeek-AI), Wei Zhang (Peking University), Tianyi Cui (DeepSeek-AI). 92 pages, predominantly formal.

### Problem

Composition is traditionally static — function calls, imports, and inheritance resolved at compile time. Dynamic composition is increasingly demanded and formally underdeveloped. The paper splits it into two orthogonal dimensions:

- **Temporal composability** — on removal, a component's modifications to the shared environment must be completely and safely reversed. Statically this collapses to lexical scoping (RAII, bracket patterns).
- **Spatial composability** — components must declare, discover, and resolve dependencies on one another verifiably. Statically this collapses to module import resolution.

### Motivation

VSCode is the worked example, with marketplace data from 2026-06-09. The extension host cannot unload an individual extension's code, and 87 of the top 100 extensions by installs contain executable code, so removing them requires a host restart. The `deactivate` hook fires only at host termination and separates disposal from creation in `activate`, violating locality of concern. Spatially, only 7 of the top 100 declare `extensionDependencies` on non-built-ins, and inter-extension interop runs through `getExtension(...).exports`, typed `any`.

Self-evolving agent harnesses raise the stakes: without temporal composability each self-modification forces a restart that discards process-local state, and "even worse, a faulty self-modification can disable the very process needed to recover".

The usual workaround is priced explicitly. Operating systems give temporal composability at process granularity and container orchestrators give spatial composability at service granularity; restarts discard caches, connections, and partial computations and cost seconds to minutes, forcing redundant replicas, while containers cannot express dependencies between components sharing an address space and add network overhead to what could be local calls.

### Approach

Effect systems formalize how computations *modify* their environment; coeffect systems formalize how they *depend* on it. Both are compile-time analyses over lexically fixed scopes. The paper lifts each to a runtime mechanism.

1. **Revertible effects** (§3.1) — every context transformation carries an explicit inverse that the runtime holds. `track` is proved a monoid homomorphism and `recover` preserves composition, so inverses compose and apply LIFO. Yields *local* temporal composability.
2. **Reactive coeffects** (§3.2) — a coeffect context is a dependent map from keys to typed values; a component declares a coeffect specification, and every context change is classified against it as activating, deactivating, or neutral, driving activation and deactivation. Extended with isolation (realms) and interception. Yields *local* spatial composability.
3. **The context paradigm** (§3.3) — effect context and coeffect context are unified into one context type and every effect and coeffect is mediated through it. The mediation induces an observational equivalence up to which distinct components' effects interleave without disturbing one another; Theorem 45 gives independence for operations at distinct keys.
4. **A calculus of dynamic composition** (§4) — a component is a triple of declarations, provisions, and an effect iterator; a fiber is one instantiation carrying parent, table, admin flag, lifecycle state, accumulator, and committed view. Orchestration rules (`O-Insert`, `O-Retire`, `O-Remove`) and lifecycle rules (`L-Begin`, `L-Iter`, `L-Finish`, `L-Divert`, `L-Leave`, `L-Unload`), plus confinement.
5. **Cordis** (§5) — the implementation, in three tiers.

### Metatheory

The theorems carry both guarantees from one component to a system of interleaved ones.

| Result | Statement |
|---|---|
| Preservation (Thm 64) | Registry well-formedness is invariant under every rule. |
| Temporal composability (Thm 68, Cor 69) | Applying a fiber's accumulator leaves every fiber's table where the intervening steps alone would have left it from the episode's start. |
| Ordering (Thm 70) | A provider withdraws a binding only after every dependent that resolved it has deactivated. |
| Resolution coherence (Thm 71) | A multi-step transition runs against one resolution, or diverts out of it. |
| Progress (Thm 73) | No deadlock, plus termination `S(n) <= (K+3)(V(n)+1)`, assuming acyclic precedence and finitely many names. |
| Confluence (Thm 80) | The quiescent state is a function of the final configuration alone, whatever order the loader took to get there. |

Two limits the paper states plainly, worth carrying forward:

- Recovery holds only up to the observational equivalence. "A monotone allocator is not rewound, a heap's layout free does not restore, and a message already sent stays sent."
- The *witness* — that a supplied inverse actually inverts its effect — "is an obligation on the component author rather than a property the runtime verifies." Same for the commutativity witness on a coeffect key.

### Implementation

The core library makes `ctx.effect` the sole primitive through which the context is mutated: coeffect provision, component instantiation, and every other context-mutating operation reduce to it, so anything done through the context is tracked and reverted on unload. Coeffect operations act on three symbol-keyed slots, `@@store`, `@@isolate`, `@@intercept`.

The component loader adds a declarative layer. An entry records `id`, `url`, `isolate`, `intercept`, `config`, `disabled` — exactly what an instantiation needs — and reconciliation is incremental rather than teardown-and-rebuild, justified directly by Thm 80 (final state depends only on final configuration) and Thm 73 (the system does quiesce). `group` and `include` are ordinary components, so nested trees stay inside the calculus.

HMR needs no developer-annotated acceptance boundaries, unlike Webpack or Vite, precisely because a fiber already bounds all of its component's effects: disposing the old fiber recovers everything, and a new fiber from the reloaded module reinstalls it.

Table 2 is the theory-to-implementation map, and it reads as the dsh vocabulary:

| Theory | Implementation |
|---|---|
| context type | `ctx` |
| `effect(e)` | `ctx.effect(callback)` |
| declarations `d` | `fiber.inject` |
| accumulator / recover | `fiber.dispose` |
| committed view | `fiber.committed` |
| `isolate(k, r)` | `ctx.isolate(key, realm)` |

### Evaluation

The case study is **Koishi**, not this repository: an open-source chatbot framework on Cordis, four years old, 4000+ community-contributed plugins, whose web console is a second independent Cordis application composing browser primitives rather than server ones. The paper draws two claims from it — expressiveness (the primitives carry a complete production system, the host supplying only domain vocabulary) and generality (no assumed domain or runtime).

Threats to validity are stated without hedging: one ecosystem, one host language, observational rather than a controlled comparison. "What the case study establishes is thus an existence-and-adoption result rather than a quantitative one."

Koishi runs Cordis v3. The paper presents v4, "which refines the effect and coeffect semantics and redesigns the loader" — v4 is what this repository vendors.

## 3. How the two connect

The paper's conclusion names self-evolving agent harnesses as the compelling direction for *future* validation: a setting where an AI agent generates and replaces its own harness components continuously with little human oversight would test the temporal guarantee of complete recovery under rapid replacement and the spatial guarantee of dependency coordination under frequent topological change.

DeepSeek Harness is an attempt at that validation. It is not evaluated in the paper.

Read in that light, several repository rules stop looking like style preferences and start looking like the theory's author obligations made mechanical:

- "Registrations are effects" is the witness obligation, pushed into a convention because the runtime cannot check it.
- The single `dsh --profile` entry point with a gate is what keeps the calculus total over the running system — an application tree outside it would be effects the runtime does not hold inverses for.
- Capability seams are the coeffect keys: a Service Definition is the key, a Provider installs the binding, a Consumer declares it.

## 4. Loose ends

`CLAUDE.md`'s repository-layout table lists a `self-modification/` group, but that group is now `packages/extensions/` (`packages/README.md:63`). Doc drift, not a code problem. Not fixed in this session.

Worth following up another time:

- §6.3 "Access Control and Sandboxing" and §6.2 "Service Multiplexing" were not read; both look directly relevant to the harness's sandbox and preset design.
- The paper's §4.4 "Extensions" covers failure handling (`FAILED` fibers carrying an error outcome), which maps to boot diagnostics in `dsh-app-boot`.

## 5. Provenance

Paper retrieved as PDF from `https://arxiv.org/pdf/2608.25512` and read with `pdftotext -layout`; quotations above are verbatim from that text. Repository claims were read from the working tree at the commit named at the top, not from memory.
