# CANForge — SRS & SAD (Ground-Up Rebuild Blueprint)

**CANForge** is a Python CLI + TUI toolchain that forges *executable* CAN test
artifacts for automotive embedded-software validation. It has two pipelines:

- **Open-loop (CAN bus testing)**
  `DBC → Markdown (AI-readable spec) → JSON Test Plan (AI-assisted, human-verified) → ASC log → playback on real hardware`
- **Closed-loop (UDS diagnostics)**
  `ECU C sources → .uds.yaml protocol knowledge (AI-extracted, citation-audited) → .uds.macro.yaml test macro → automation engine → ISO-TP/UDS stack → real CAN hardware (or an in-process mock ECU) → structured evidence`

This repository contains the complete engineering documentation set — the
**Software Requirements Specification (SRS)** and the **Software Architecture
Document (SAD)** — written as a *ground-up rebuild blueprint*: the documents
capture the system exactly as it runs today (12 local ADRs, full protocol
stack, dashboards, host integration) and are written so that a fresh
engineer or team could reconstruct the tool from them alone, without seeing
the original code.

> **Why a rebuild blueprint?** It is the strongest possible test of a
> document set: if the spec loses nothing on a round-trip, the system was
> actually understood — not just described.

---

## What's in this repo

| File | What it is |
|---|---|
| [`docs/SRS.md`](docs/SRS.md) | Software Requirements Specification (~2,100 lines): scope, 120+ functional requirements with stable IDs, interface contracts (CLI, TUIs, host, LLM, hardware), 17 non-functional requirements, complete data schemas for every artifact, the gate lattice, and the exit-code policy |
| [`docs/SAD.md`](docs/SAD.md) | Software Architecture Document (~1,800 lines): architecture vision, 8-layer style with numbered dependency rules, a per-file component catalog (public interfaces + requirement refs), the decision rationale for every ADR, data flows (incl. the in-process event-stream contract), cross-cutting concerns, deployment view, test architecture, and a traceability + known-debt register |

The two documents are cross-referenced throughout: every requirement maps to
a component, every component to its ADR(s) and its tests.

---

## Document map

**SRS** — what must be true

| § | Section |
|---|---|
| 1 | Introduction: purpose, scope, definitions, references |
| 2 | Overall description: product perspective, personas, the 12 design invariants (incl. the canonical artifact layout) |
| 3 | Functional requirements: FR-CAN / FR-UDS / FR-FUDS / FR-FCAN / FR-ASC / FR-UNIT / FR-UTIL |
| 4 | Interface requirements: full CLI subcommand contract, TUI entries, host integration, LLM interface, `CanBackend` hardware contract |
| 5 | Non-functional requirements NFR-01…17, each with a verification clause |
| 6 | Data requirements: the complete contract of all 17 artifacts (markdown, JSON, ASC, YAML knowledge/macro/anchor, result/coverage/trace) |
| 7 | Gate requirements: the gate lattice (G1/G2/G3, A/B/C, V, SG, PG) and its properties |
| 8 | Exit-code policy: the four-value discipline and per-command table |
| 9 | [FUTURE] items, explicit non-goals, standing invariants |

**SAD** — how it is built and why

| § | Section |
|---|---|
| 2 | Architecture vision: the pipeline *is* the architecture; three truth layers (truth / artifact / proposal) |
| 3 | Style & layering: 8 layers, dependency rules D1–D7, process and concurrency models |
| 4 | Component catalog: per package, per file — role, public interface, requirement refs, ADRs |
| 5 | Key architectural decisions: default-off parity, the gate lattice, one-engine-four-surfaces, ADR-001…012, extension points |
| 6 | Data model & flows: pipeline diagrams with gates on the edges, the in-process event-stream contract, value cascades, the unit-anchor lifecycle, AI-proposal flow |
| 7 | Cross-cutting concerns: error/exit model, encoding recovery, audit trail, AI integration, console degradation, secrets policy |
| 8 | Deployment view: the two deployment modes, dependencies, filesystem layout, deployment checklist |
| 9 | Test architecture: 900+ hardware-free tests, parity discipline, coverage mapping, known gaps |
| 10 | Traceability & known debt: the ADR→design→evidence matrix, the 10-item debt register, the closing invariant checklist |

---

## Engineering highlights (what a reviewer should look for)

- **A gated artifact factory.** AI output enters the system only as a
  *proposal*; every transition between artifacts stands behind a **pure,
  deterministic gate** (the gate lattice, SRS §7). No AI-generated step or
  frame ever reaches the bus without a gate *and* a human gesture.
- **Three truth layers.** `DBC / C sources` (truth) → `artifacts` (validated,
  human-owned) → `AI proposals` (quarantined). The DBC wins every drift; the
  scanner reads C code, never the yaml.
- **Default-off parity (NFR-01).** A project-wide invariant: every optional
  face (DID layouts, DTC services, dynamic timing, session gates, pluggable
  security, ISO-TP flow control, negative testing, frame overrides) is
  additive and absent-by-default — absence means **byte-identical legacy
  behavior**, pinned by golden tests. No historical artifact ever needed
  migration.
- **One engine, four surfaces.** A single automation engine and a single
  frame-expansion core drive the CLI and three Textual TUIs (blue UDS
  "Flowline", red CAN Flowline, ASC playback dashboard) through an
  in-process **event-stream contract** — the UIs render; they never
  re-implement engine logic.
- **A real protocol stack, hardware-abstracted.** ISO 15765-2 (ISO-TP,
  incl. ECU flow-control handling) and ISO 14229 (UDS services, NRC-0x78
  pending handling, 0x50 timing negotiation) over a `CanBackend` ABC —
  vendor translation lives in one adapter, and an **in-process mock ECU**
  (canned responses, ISO-TP-aware, session/DTC/timing/FC/security stateful)
  makes the *entire* closed loop testable without hardware.
- **Pluggable security with canonical semantics.** A single dispatch
  registry + level catalog (builtins + profile + runtime) and one canonical
  ISO 14229 NRC table; per-NRC 0x27 recovery semantics (fresh-seed retry vs
  fail-with-recovery) implemented once, rendered everywhere.
- **ASPICE-grade traceability.** The test-case *folder* is the test case:
  a comment-preserving `.testcase.yaml` anchor with a strict
  engineer-owned / tool-owned key split, re-asserted after every
  save/compile/run gesture; append-only audit log; per-field
  source-of-value records so "where did this run's value come from?" is
  answerable from the evidence alone.
- **Disciplined CLI contract.** A four-value exit-code policy (0 / 1 / 2 / 3)
  with the rule that validation failures (2) write nothing, runtime failures
  (1) still write their evidence, and the LLM-unreachable case (3) writes
  nothing.
- **Documented, not hidden.** 12 local ADRs + the host-level series, a
  10-item known-debt register (incl. one grandfathered dependency, pinned
  by a test), and explicit non-debts.

---

## The pipelines

```
OPEN-LOOP (CAN)
 .dbc ──G1──▶ description.md ──AI + G2──▶ plan.json ──G3──▶ .asc ──▶ playback
              (low-token spec)   (human review)   (encode)        (TSMaster / live)

CLOSED-LOOP (UDS)
 ECU C sources ──tree-sitter + LLM + Gates A/B/C──▶ .uds.yaml
                                                        │
                              dashboard ──▶ .uds.macro.yaml ──Gate V──▶
                                                        │
        AutomationEngine ──▶ CanBackend (real hardware | mock ECU)
                                                        │
                                     .uds.result.json (regression evidence)
```

---

## Tech stack

Python 3 · cantools (DBC) · Textual/Rich (TUIs) · PyYAML / ruamel.yaml
(comment-preserving anchor writer) · tree-sitter (+ C grammar) · Ollama
(local LLMs, two model roles) · ctypes over a vendor CAN library (TOSUN
libTSCAN) · pytest (900+ tests, hardware-free)

---

## Suggested reading path (15 minutes)

1. **SAD §2** — the vision and the three truth layers (2 min)
2. **SAD §3** — the layer diagram and dependency rules D1–D7 (3 min)
3. **SAD §5.1–5.3** — the three meta-decisions that shape everything (4 min)
4. **SRS §7** — the gate lattice; then **SRS §8** — the exit-code policy (3 min)
5. **SAD §10** — the traceability matrix and the debt register (3 min)

---

## Status

The documents describe a running system in its current feature state
(UDS Steps 1–3 and the CAN dashboard complete; CAN Step 4/5 and UDS Step 4
are explicitly marked **[FUTURE]** in the SRS). The implementation lives in
a separate private codebase; this repository is the documentation deliverable.

## License

See [LICENSE](LICENSE) — MIT.
