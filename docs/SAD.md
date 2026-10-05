# CANForge — Software Architecture Document (SAD)

| Field | Value |
|---|---|
| **Document status** | Draft — baseline for a ground-up rebuild |
| **Date** | 2026-10-04 |
| **Scope baseline** | CANForge as it exists today: CANForge-local ADR-001…012, CAN Flowline phases P1–P14, UDS Steps 1–3, full closed-loop stack, Nausica plugin integration |
| **Companion document** | `doc/SRS.md` (Software Requirements Specification) |

> **Reading note.** The SRS states *what* the system shall do (requirements,
> contracts, invariants) — this document states *how it is built to do it*:
> the layered architecture, the module-by-module catalog with its public
> interfaces, the key architectural decisions (each mapped to its ADR), and
> the flow/deployment/test views. Together the two documents are the complete
> blueprint for a from-scratch rebuild: an engineer implements a module from
> its §4 catalog entry, its SRS requirement references, and its ADR; the §5
> decisions explain *why* the shape is what it is, so the rebuild makes the
> same trade-offs deliberately rather than by accident.

---

## 1. Introduction

### 1.1 Purpose

This document is the architectural baseline for **CANForge** — the CLI tool
and TUI dashboard suite that forges executable CAN test artifacts from `.dbc`
databases (open-loop pipeline) and runs UDS diagnostic automation against
real or simulated ECUs (closed-loop pipeline). It captures the **current**
architecture in full, so that a ground-up rebuild is an *implementation* of
this document, not a re-design.

The document serves three audiences:

1. **Rebuild engineers** — implement the §4 module catalog component by
   component; every component entry names its SRS requirements and its ADRs
   so nothing is built without a reason on record.
2. **Reviewers (ASPICE / technical)** — verify that the architecture enforces
   the SRS invariants: default-off parity (NFR-01), the gate lattice (§SRS 7),
   the single-canonical-builder rule (NFR-16), and the evidence model
   (NFR-09). The *shape* of the architecture is where those invariants live —
   §5 explains exactly how each structural decision buys one of them.
3. **Extension authors** — §5.12 (extension points) and §9 (deployment) show
   where new ECUs, new hardware families, and new pipeline stages plug in
   without touching the core.

### 1.2 Scope

In scope (architecture as it exists today):

- The CLI (`canforge.py`) and its handler modules (`handler/can`,
  `handler/uds`, incl. `handler/uds/extract`).
- The protocol stack (`isotp.py`, `uds.py`) and its leaf decision modules
  (`dtc.py`, `did_layout.py`, `session_gate.py`, `nrc.py`, `uds_timing.py`).
- The macro engine (`automation.py`) and the security subsystem
  (`security/algorithms.py`, `security/registry.py`, `security/levels.py`).
- The hardware layer (`canforge_io` backend factory + `tosun_libtscan`,
  `mock`, `tosun_tsmaster` skeleton; the legacy `canforge_tosun` ctypes
  bindings and their known decoupling debt).
- The three TUI dashboards (`uds_dashboard.py`, `can_dashboard.py`,
  `asc_dashboard.py`) and their shared cores (`uds_core.py`, `can_core.py`).
- The utilities (`utils/`), the profile directories (`profile/e2e`,
  `profile/uds`), the prompt/templates assets, and the Nausica integration
  surface (`plugin.yaml`, subprocess contract).
- The test architecture (`Test/` suite, golden files, Textual pilot tests,
  parity tests) and the deployment view.

Out of scope: the [FUTURE] pipeline items of SRS §9.1 (CAN Step 4/5, UDS
Step 4, TSMaster frame path, ARXML/ODBX) — they are *architecture-prepared*
and this document marks where each one will attach; the Nausica host
itself (a separate product; here only the integration contract).

### 1.3 References

| # | Document | Role in this document |
|---|---|---|
| [R1] | `doc/SRS.md` | The requirements baseline; component entries and decisions reference its FR/NFR/FR-GATE IDs |
| [R2] | `doc/ADR/ADR_001…ADR_012` + `doc/ADR/Template.md` | CANForge-local decision records; §5 maps each architectural decision to its ADR |
| [R3] | Nausica-root `ArchitectureDecisionRecord/` — ADR-025 (E2E profiles), ADR-027 (ASC dashboard), ADR-030 (knowledge extraction), ADR-036/037 (UDS dashboard + AI chat), ADR-043 (negative testing), ADR-044 (frame editing), ADR-045 (test-case block + notes) | Cross-series decisions implemented in this tool |
| [R4] | `WhatIsCANForge.md` | The pipeline the architecture is organized around (Source → Knowledge → Plan/Unit → Play/Run → Evidence → Regression) |
| [R5] | `CANForge_CAN_upgrade_workflow.md` | The CAN Flowline phase log P1–P14 — the behavioral baseline for the TUI components |
| [R6] | `cli_command_structure.md`, `setupBackend.md` | CLI reference and backend (Ollama) setup |
| [R7] | ISO 15765-2, ISO 14229-1 (incl. Annex C), AUTOSAR E2E, SAE J1850 | The standards the protocol and E2E components conform to |

### 1.4 Definitions and Abbreviations

All terms as defined in SRS §1.3 (single source of truth). Additionally,
architecture-specific terms used in this document:

| Term | Meaning in this document |
|---|---|
| **Component** | A file/module with a defined public interface; the §4 catalog unit. |
| **Leaf module** | A component that imports nothing from the project (stdlib + third-party only); a protocol-fact module that cannot drift from the rest of the system because it *is* the definition (SRS §2.4 / NFR-16). |
| **Canonical builder** | The single function that constructs a given kind of output (request bytes, encoded frame, trace line, gate report, refusal text) so all consumers of that output agree by construction. |
| **Core** | The dashboard's shared logic module (`uds_core.py` / `can_core.py`) — the seam between a TUI and the handlers, and the entry point the TUI *may* call in-process. |
| **Shell** | A TUI's widget/layout code (panes, modals, key handling) — copied between dashboards, never imported (SRS §4.6 rule). |
| **Face** | An optional, default-off feature addition to an artifact or engine (ADR-001…006, ADR-043/044/045) — present or absent without changing legacy behavior (NFR-01). |
| **Gesture** | A user action that produces or updates artifacts (S/C/M/R/E on a dashboard, a CLI save/run) — the unit after which the anchor's tool-owned keys are re-asserted (ADR-012). |
| **Projection** | The `test_sequences` view of the plan document (SRS §3.4 / ADR-008): what M/R/gates read, with the full document as truth. |

### 1.5 Document Structure

- **§2 Architecture Vision** — the pipeline as architecture: artifacts as the
  interface between stages, gates as the trust boundary, truth layers (DBC /
  C sources) vs. proposal layers (AI), and the four execution surfaces over
  shared engines.
- **§3 Architecture Style and Layering** — the layer diagram, the dependency
  rules (leaf discipline, no cross-handler imports, copy-not-import shells),
  and the process model (CLI vs. TUI vs. host subprocess).
- **§4 Component Catalog** — the module-by-module rebuild specification:
  per package, per file, with public interface, responsibilities, SRS
  requirement refs, and ADRs.
- **§5 Key Architectural Decisions** — one section per decision (mapped to
  ADR-001…012 and the cross-series ADRs): context, decision, the SRS
  invariant it buys, and the consequences.
- **§6 Data Model and Flows** — artifact flow diagrams for both pipelines,
  the in-process run event-stream contract, the timing/security/profile/E2E
  resolution cascades, and the anchor lifecycle.
- **§7 Cross-Cutting Concerns** — audit logging, encoding, error/exit-code
  model, AI integration, console degradation, configuration cascade.
- **§8 Deployment View** — the runtime environment, external services,
  hardware, and the host integration topology.
- **§9 Test Architecture** — the suite structure, the parity/golden/pilot
  strategies that encode the NFRs as regressions.
- **§10 Traceability and Known Debt** — ADR matrix, requirement-to-component
  coverage notes, and the registered technical debts (legacy `canforge_tosun`
  import debt, TSMaster skeleton, RX capture v1).

Conventions: `module.function` notation names public interfaces exactly as a
rebuild must expose them; SRS references are `FR-…`/`NFR-…`; ADR references
are qualified by series (`[local]` = `doc/ADR/`, `[Nausica]` = Nausica-root
`ArchitectureDecisionRecord/`).

---

## 2. Architecture Vision

### 2.1 The pipeline IS the architecture

CANForge is organized around one pipeline statement [R4]:

```
Source → Knowledge → Plan/Unit → Play/Run → Evidence → Regression
```

That sentence is not a workflow metaphor — it is the **architectural spine**.
Every stage is a module (or module cluster) with:

- a **deterministic** core (no stage's correctness depends on an LLM or a bus),
- an **artifact** it emits (the only thing the next stage may read),
- a **gate** at its exit (the trust boundary — SRS §7), and
- an **evidence** record of having run (audit log line, result artifact,
  anchor update).

Because the stages are coupled **only through files**, the same stage can be
reached from the CLI, from a TUI, or (for the [FUTURE] regression steps) from
CI — and a stage can be rebuilt without disturbing its neighbors. The
architecture's job is to make the pipeline *honest*: the artifacts on disk are
exactly what would have been sent to the bus, and the evidence on disk proves
it.

### 2.2 The three layers of truth (and where AI sits)

```
        TRUTH LAYER        the DBC (CAN) / the ECU's C sources (UDS)
                           — read-only inputs; they win every conflict
                                │  deterministic parse/scan
                                ▼
        ARTIFACT LAYER     .can.md · .can.plan.json · .asc
                           .uds.yaml · .uds.macro.yaml · .asc · results
                           — validated, gated, engineer-reviewable files
                                ▲  proposals that must pass a gate
        PROPOSAL LAYER       the LLM's raw output (plan JSON, .uds.yaml
                           draft, chat step proposals) — never trusted
                           verbatim; gated, repaired or rejected
```

Two consequences follow for the whole system:

1. **AI output never bypasses a gate.** The LLM is a *fast but fallible
   producer* of the proposal layer; the deterministic gates (G1/G2/G3,
   A/B/C, V, PG — SRS §7) are the only road from proposal to artifact. This
   is why "the AI was wrong" is always recoverable: the artifact either
   passed the gate (the gate is testable) or it never existed.
2. **Truth beats artifact beats proposal.** A `.can.md` that disagrees with
   its DBC produces a *warning*, not a failure (NFR-03); a plan that
   disagrees with the DBC *fails Gate 2*; a `.uds.yaml` citation that points
   at nothing is stripped by Gate B. The direction of authority is fixed and
   the code encodes it in exactly the places where conflicts can arise.

### 2.3 The two pipelines at a glance

**Open-loop (CAN)** — `DBC → md → plan → ASC → (play)`, human-approved at
every arrow, DBC as truth:

```
 <vendor>.dbc ──G1──▶ <unit>.can.md / _detail.can.md ──(AI, G2)──▶ <unit>.can.plan.json
                                                                      │  deterministic
                                       <unit>.asc ◀──G3── json_to_asc │
                                          │
                                   play-asc / ASC dashboard / TSMaster
        Flowline CAN (can_dashboard.py): same three stages in one TUI —
        knowledge pane (DBC truth) · two-tab workbench (G2) · M/C/R · coverage/trace
```

**Closed-loop (UDS)** — `C sources → knowledge → macro → engine → ECU →
evidence`, citations as the truth-link, the mock as a first-class backend:

```
 ECU .c/.h ──scan/bundle──▶ (AI) ──A/B/C──▶ <stem>.uds.yaml (draft)
                                        │
        .uds.macro.yaml ◀──Flowline UDS──┤  (V + session gates + AI proposals gated by PG)
                        │
        AutomationEngine ──▶ CanBackend ──▶ ECU (live)   or   MockCanBackend (offline)
                        │
                        ▼
            .uds.result.json (schema v1) + coverage + trace ──▶ <unit>.testcase.yaml (anchor)
```

The two pipelines **share discipline, not code**: they share the gate
philosophy, the anchor (ADR-012), the audit trail, the profile mechanism, and
the AI-call layer — but `handler/can` and `handler/uds` never import each
other, and the TUI shells are copied, not imported (§3.2). A UDS concept
(leak) into the CAN tool is an architectural defect, not a shortcut.

### 2.4 One engine, four surfaces

The engines are the scarce, carefully-built resources; the surfaces are cheap
front doors:

| Surface | Entry | Process model | Drives |
|---|---|---|---|
| CLI pipeline | `canforge.py <sub>` | subprocess (host or shell) | the handler functions directly |
| UDS Flowline | `uds_dashboard.py <file.uds.yaml>` | own console (host Popen or shell) | `uds_core.run_mock_inprocess` / `run_live_inprocess` → the **real** `AutomationEngine`, in-process, through a tracing proxy |
| CAN Flowline | `can_dashboard.py <file.can.md>` | own console | `can_core.run_offline` / `run_live` + in-process `json_to_asc` (Gate 3) |
| ASC dashboard | `asc_dashboard.py <file.asc>` | own console | ASC playback (+ optional bus TX) |

The invariant: **a surface never re-implements engine logic.** The dashboard
asks the core, the core asks the same handler the CLI asks; the only thing a
surface adds is presentation (panes, modals, the event stream, the
gates-before-gestures wiring). This is what makes "dashboard S and CLI
`json-to-asc` produce byte-identical artifacts" a property of the
architecture rather than a discipline (SRS §4.6, NFR-16).

### 2.5 Architecture forces → quality attributes

| Force (design driver) | How the architecture answers it |
|---|---|
| **Legacy runs must never change** (production baselines, ASC logs) | Faces are additive and default-off; parity is structural (absent face ⇒ the legacy code path is literally the one taken) and *tested* byte-for-byte (NFR-01, §9) |
| **ASPICE evidence** | Evidence is a *stage output*, not an option: results, coverage, traces, audit log, anchor verdict — each gesture has exactly one writer, so the trail is complete by construction (NFR-09) |
| **No hardware in CI / on a laptop** | `mock` is a `CanBackend`, `dry-run` a mode, `M` a virtual timeline — the closed loop's offline paths use the *same* engine and stack as live (NFR-06) |
| **New ECU / OEM without source edits** | The registry/level catalog, the backend factory, and the profile directories are the three named extension points (§5.12) |
| **One person must be able to audit a step's behavior** | Leaf modules isolate protocol facts; one canonical builder per output kind; gates have one report format (§3.2, §5) |
| **Windows-first reality (cp1252, DLLs, consoles)** | ctypes-only native boundary, console degradation as a cross-cutting utility, subprocess isolation so a crashed TUI console can never take down the host (SRS §2.4.8/12) |
| **AI is external and unreliable** | The AI layer is a thin, swappable transport with per-role servers; every consumer of AI output sits behind a gate; the pipeline's deterministic core has zero AI dependency (NFR-10) |

### 2.6 The vision in one paragraph

CANForge is a **gated artifact factory with a hardware-optional closed
loop**: deterministic stages turn trusted sources (DBC, C code) into
reviewable artifacts, AI accelerates but never authorizes, one engine serves
every surface, every run leaves evidence on disk, and every extension —
new ECU, new hardware, new face — attaches through a named seam without
touching the legacy bytes.

---

## 3. Architecture Style and Layering

### 3.1 Style

CANForge is a **layered pipeline architecture**: a thin entry layer (CLI +
TUI shells) over shared cores, over handler modules, over protocol leaves,
over the hardware abstraction, over leaf utilities — with **files as the
inter-stage interface** and **processes as the fault-isolation boundary**
(host ↔ tool, tool ↔ TUI console). There is no server, no database, no
shared memory: the only "middleware" is the filesystem and (in-process)
queues inside the backends. The style is deliberately boring — every
advanced property (parity, evidence, hardware-free testability) is bought by
structure, not by framework.

### 3.2 Layer diagram and dependency rules

```
┌──────────────────────────────────────────────────────────────────────────┐
│ HOST — Nausica (OUT of process: discovers plugin.yaml, builds the        │
│ context menu, launches subprocesses with argv, reads exit codes)         │
└──────────────────────────────▲───────────────────────────────────────────┘
                               │ argv / Popen (CREATE_NEW_CONSOLE on win32)
┌──────────────────────────────┴───────────────────────────────────────────┐
│ L1  ENTRY                                                                │
│   canforge.py — CLI composition root (argparse subparsers, one sub-      │
│     command → one handler function; audit-log + exit-code policy)        │
│   uds_dashboard.py · can_dashboard.py · asc_dashboard.py — TUI SHELLS    │
│     (Textual widgets, panes, modals, key handling; COPY-PASTED between   │
│     dashboards, never imported across them)                              │
├──────────────────────────────────────────────────────────────────────────┤
│ L2  DASHBOARD CORES (the in-process seam between a TUI and the handlers) │
│   uds_core.py — knowledge/profile loading, Step model, macro render/     │
│     write, request_bytes_for_step, validate_steps, negative/DTC/security │
│     derivation, run_mock_inprocess / run_live_inprocess                  │
│   can_core.py — load_can_knowledge (md display + DBC truth), PlanStep,   │
│     Gate 2 (validate_plan_steps), encode_step_frame, expand_plan_to_     │
│     frames, run_offline / run_live, coverage, path/anchor helpers        │
├──────────────────────────────────────────────────────────────────────────┤
│ L3  HANDLERS                                                             │
│   handler/can/: dbc_to_markdown · ai_reasoner · plan_validator ·         │
│                 json_to_asc · asc_player · recorder                      │
│   handler/uds/: automation (AutomationEngine, validate_macro) ·          │
│                 security/{algorithms, registry, levels}                  │
│   handler/uds/extract/: c_scanner · evidence · knowledge_schema ·        │
│                 knowledge_validator (Gates A/B/C) · extract_knowledge    │
├──────────────────────────────────────────────────────────────────────────┤
│ L4  PROTOCOL                                                             │
│   isotp.py (ISO 15765-2, ADR-006 FC) · uds.py (ISO 14229, NRC78, 0x50)   │
│   — depend ONLY on the CanBackend ABC (no vendor, no project state)      │
│   LEAVES (import nothing from the project): nrc.py · session_gate.py ·   │
│   uds_timing.py · dtc.py · did_layout.py                                 │
├──────────────────────────────────────────────────────────────────────────┤
│ L5  HARDWARE (hardware/usb_to_can/)                                      │
│   canforge_io/: CanBackend ABC + CanFrame · create_backend()/            │
│     register_backend() · tosun_libtscan (ctypes → libTSCAN.dll) ·        │
│     mock (in-process UDS ECU simulator) · tosun_tsmaster (skeleton)      │
│   canforge_tosun/ (legacy ctypes bindings: DLL structs, ASC parsing,     │
│     device context manager — partially superseded, see §10 debt)         │
├──────────────────────────────────────────────────────────────────────────┤
│ L6  UTILITIES (stdlib + third-party only)                                │
│   llm_handler · audit_logger · terminal_spinner · e2e_computation ·      │
│   e2e_profile_loader · uds_profile_loader · testcase (stdlib+ruamel)     │
├──────────────────────────────────────────────────────────────────────────┤
│ L7  DATA                                                                 │
│   config.yaml · profile/e2e/*.yaml · profile/uds/*.yaml · prompts/*.txt  │
│   json/template_v1.txt · data/ (runtime: audit_log.jsonl — never         │
│   committed) · per-unit artifact folders (SRS §2.4.10)                   │
└──────────────────────────────────────────────────────────────────────────┘
```

**Dependency rules (normative — enforced by AST-boundary tests, NFR-16):**

| # | Rule | Buys |
|---|---|---|
| D1 | **Leaf modules import nothing from the project.** Leaves: everything in L4 marked leaf, `canforge_io` (L5), all of L6. A leaf that imports the project is a leaf no more. | Protocol facts are auditable standalone; `session_gate`/`uds_timing`/`dtc`/`nrc`/`testcase` can be unit-tested and reused (they are already shared across engine, CLI, mock and dashboard by *being* leaves) |
| D2 | `handler/can` ⇏ `handler/uds` (neither imports the other) | The CAN tool can never acquire UDS semantics; the pipelines share only L6 utilities and L7 data |
| D3 | TUI shells import their **own** core only; shells never import another shell or another core | The red CAN dashboard cannot leak blue UDS behavior; the copy-not-import shells stay independently evolvable (they deliberately duplicate ~widgets, never logic) |
| D4 | `canforge_io` imports nothing from L3/L4 (hardware sits **below** protocol); the factory + ABC is the seam | A new hardware family touches one directory; the mock proves the protocol stack without a bus |
| D5 | `canforge.py` may import anything (it is the **composition root** — the one place allowed to wire layers together) | Wiring is visible in one file; no hidden coupling |
| D6 | L6 utilities import stdlib + third-party only (they are leaves even though cross-cutting) | `testcase.py`, `audit_logger.py`, `e2e_computation.py` stay host-free and test-free-of-context |
| D7 | L2 cores import L3/L4/L5/L6 — never L1 | The core stays TUI-free (no widget imports in `uds_core`/`can_core`); the same core could back a non-Textual front door |

**Registered exception (debt, §10):** `handler/uds/isotp.py` and `uds.py`
import exception classes from the legacy `canforge_tosun/exceptions.py`
module. Target state: the exceptions live in a leaf owned by `handler/uds`
(or `canforge_io`), and the legacy module re-exports them for back-compat.
Until then the import is grandfathered and pinned by a test — it must not
spread.

### 3.3 Process model

```
 Nausica (Qt, Windows)
  ├─ canforge.py <sub> …            (subprocess.run, QProgressDialog, exit code)
  ├─ uds_dashboard.py / can_dashboard.py / asc_dashboard.py
  │    (subprocess.Popen, CREATE_NEW_CONSOLE on win32 — a crashed console
  │     can never take the host down; the engineer watches the run live)
  └─ .nausica.yaml / config.json    (file reads; upward search)
```

- **CLI process** is synchronous and single-threaded; long phases (LLM,
  encode, playback) run inline behind the spinner. `automate --mode mock`
  and the TUI in-process runs both construct the **real** backend objects —
  `mock` is simply a `CanBackend` that lives in the same process.
- **TUI processes** run the engine **in-process** (via their core), not via
  a child `canforge.py`: the per-frame trace, the live YAML preview and the
  gate-before-gesture wiring need frame-level visibility that a subprocess
  boundary would destroy. The in-process run executes **off the UI thread**
  (Textual worker) and communicates with the app only through the event
  stream contract (§6.2) — the UI thread never blocks on a bus or an LLM.
  The one deliberate subprocess from a TUI is the **C** gesture launching
  `asc_dashboard.py` in a new console (SRS FR-FCAN-12).
- **Host** launches and supervises only: it reads exit codes and captured
  output (SRS §8.4); it never imports, threads into, or signals tool
  processes beyond console lifetime.

### 3.4 Concurrency model

| Where | Model |
|---|---|
| CLI | Single-threaded; the spinner is the only "background" activity. Deterministic stages never spawn threads. |
| `AutomationEngine` keep-alive | One background thread for the `start_keepalive`…`stop_keepalive` segment (periodic 0x3E), joinable; the step dispatch itself is sequential — a macro is a *script*, not a scheduler. |
| Live backends | RX arrives on the backend's receive path (callback queue / dispatcher thread in the vendor binding); the ISO-TP layer consumes from it with timeouts. TX is synchronous call/return. |
| TUI | Textual event loop on the UI thread; in-process runs on a worker; the two meet only at the event stream (§6.2) and at clipboard/file writes. No shared mutable state crosses the boundary. |
| AI | `llm_call_chat_stream` delivers tokens via callback (TUI strip) or collects the full text (CLI single-shot); the LLM is never called from the engine's step path — only from the chat/plan/extraction layers. |

### 3.5 Configuration flow (principle)

Every configurable value has a **fixed resolution order**, documented at its
use site, and — where the value affects a run — a **recorded source** in the
evidence (the timing cascade is the canonical example: CLI > macro >
artifact > profile > default, per-field, and `.uds.result.json` says which
layer won — SRS FR-UDS-25). The layers, from most to least specific:
CLI flag → env var → per-project `.nausica.yaml` (host-side) → profile
YAML → `config.yaml` → engine default. The principle exists so that "where
did this value come from?" always has a one-line answer (NFR-09).

---

## 4. Component Catalog

**How to read this section.** Each entry names: the component's *role*, its
*public interface* (the symbols a rebuild must expose — internal helpers are
free as long as behavior and tests hold), and its *refs* (SRS requirements
+ ADRs). The interface is the contract; the current implementation's internal
decomposition is a reference, not a mandate. Dependency rules D1–D7 (§3.2)
apply to every entry and are stated per layer where they bind.

### 4.1 L1 — Entry components

#### 4.1.1 `canforge.py` — CLI composition root

- **Role.** The one file allowed to wire all layers (D5). Parses argv
  (argparse subparsers, SRS §4.1), loads `config.yaml` defaults, applies env
  overrides, dispatches each subcommand to exactly one handler function,
  runs the audit log (FR-UTIL-06) and the exit-code policy (SRS §8).
- **Public interface.**
  - `main()` — subcommand table per SRS §4.1 (21 subcommands).
  - `_run_automate_command(args)` — the Step-3 wiring: `validate_macro`
    **first** (exit 2, pre-hardware) → knowledge resolution (sibling
    `.uds.yaml` rule, explicit `--knowledge` wins, FR-UDS-24) → `timing`
    cascade via `resolve_timing` (CLI flags beat the macro block) → backend
    construction (`mock` gets the profile security rules injected when
    `--security-rules` is set) → engine run → result write → anchor update.
  - `_write_macro_result(…)` — the **single result builder** for
    `.uds.result.json` (schema v1, SRS §6.11): the dashboard's in-process
    runs produce the identical schema through the same shape (per-field
    timing sources, mock counters, step notes, `test_case` block).
  - `_load_automate_knowledge(args)` — the sibling/explicit knowledge rule.
  - `_run_testcase_init(args)` — wraps `utils.testcase.init_testcase`
    (CAN kind: `dbc-to-md` first, conversion failure → exit 1).
  - `_cli_anchor_update(unit_dir, result, kind)` — re-asserts the anchor's
    tool-owned keys after CLI save/run gestures; **never raises**
    (FR-UNIT-03/05).
- **Refs.** SRS §4.1, §8; FR-UDS-21…29; FR-UNIT-01/05; NFR-09.

#### 4.1.2 `uds_dashboard.py` — UDS Flowline shell (blue, `#2563eb`)

- **Role.** The UDS Step-2 TUI (SRS §3.3): Textual app titled
  "Flowline — UDS macro builder"; four zones (knowledge tree, step stack +
  live YAML preview, packet trace, AI chat strip) driving `uds_core`
  in-process.
- **Public interface.**
  - `main()` — CLI: `<file.uds.yaml> [--profile NAME] [--sim-p2-ms 40]
    [--ai-chat]` (SRS §4.1).
  - `FlowlineApp` (Textual `App`) — key bindings R/M/S/E/C/Q; `_anchor_update`
    after every S/M/R/E gesture (FR-FUDS-15).
  - `UdsFrameDecoder` — the trace's field-level decoder: 0x22/0x2E payloads
    via the DID layout, 0x19/0x14 via the DTC interpreter + Annex C, NRC
    naming from `nrc.py`, FC parameter annotation, `expect_nrc`-green
    marking, security-NRC recovery hints (FR-FUDS-11).
  - Screens/modals: `FrameEditorScreen` (ADR-044: byte grid + pastable raw
    line + structured fields, two-way synced on
    `uds_core.request_bytes_for_step`; deviation → `raw` override ⚡; Save
    re-gates), `WriteValuesScreen` (one prompt per layout field →
    `values` + `readback: true`), `PromptScreen` (OEM raw bytes →
    `uds_raw`), `TestCaseScreen` (the anchor, read-only view), `ChatTextArea`
    (the `$step` strip, token streaming, dashed proposal rows with
    Accept/Edit/Reject).
  - Shared widgets: `Gutter`, `NoFlashButton` (shell primitives, copied to
    the CAN dashboard — D3).
- **Refs.** FR-FUDS-01…16; ADR-036/037/043/044/012 [Nausica/local].

#### 4.1.3 `can_dashboard.py` — CAN Flowline shell (red, `#dc2626`)

- **Role.** The CAN Step-2 TUI (SRS §3.4): four panes (CAN KNOWLEDGE with
  live search, two-tab PLAN WORKBENCH, PACKET TRACE, AI CHAT) driving
  `can_core` in-process. Shell **copied** from the UDS dashboard (D3) —
  UDS semantics must not enter.
- **Public interface.**
  - `main()` — CLI: `<file.can.md> [--ai-chat]`.
  - `CanFlowlineApp` (Textual `App`) — keys q/r/m/s/e/f/c; two-row toolbar
    (run family / document family); `_anchor_update` after S/C/M/R/E
    (FR-FCAN-17).
  - `CanFrameDecoder` — trace decoder (signal view from the DBC, raw
    override ⚡, cycle-repeat annotation).
  - Screens/modals: `ValuesScreen` (applies `can_core.coerce_signal_value`
    — the gate's own coercion, FR-FCAN-08), `FrameEditorScreen` (P8: byte
    grid + raw line + signal view over the cantools funnel; deviation →
    `data` override), `ProposalEditScreen` (AI proposal step JSON, centered
    modal), `PromptScreen`, `SearchInput` (ADR-009: F/Esc, per-keystroke
    re-projection through `filter_knowledge`).
  - Plan Workbench tabs: PLAN JSON (default) + STEP STACK, each with Copy +
    ↺ resync (ADR-008/011); 0.3 s parse debounce; red status cell on
    invalid (document/stack untouched).
- **Refs.** FR-FCAN-01…18; ADR-007…011 [local].

#### 4.1.4 `asc_dashboard.py` — ASC playback dashboard

- **Role.** Plays an ASC log (frame table + trends + playback/scrub
  controls; optional bus TX; `--dbc` for signal decoding). Read-only w.r.t.
  the log (SRS FR-ASC-03). Launched standalone or by the C gesture.
- **Public interface.** `main()` — CLI: `<file.asc> [--dbc PATH]`;
  `ASCPlaybackApp` (Textual `App`).
- **Refs.** FR-ASC-01…03; ADR-027 [Nausica].

### 4.2 L2 — Dashboard cores (the in-process seam)

#### 4.2.1 `uds_core.py` — UDS shared core

The single seam between the UDS TUI and the handlers (D7: no widget
imports). Everything the UI shows and everything the UI runs flows through
here, so the UI and the CLI cannot disagree.

- **Knowledge & accessors.** `load_knowledge(path)`; `get_sessions`,
  `get_dids`, `get_writable_dids`, `get_io_dids`, `get_routines`,
  `get_dtc_service`, `get_dtc_entries`, `get_clear_groups`,
  `get_other_services`; `get_did_view` (merged 0x22/0x2E view);
  `get_did_layout` (0x22-then-0x2E, **first non-empty wins** — the ADR-001
  resolution).
- **Derivations (one builder per concern).** `get_negative_cases` (ADR-043:
  per-DID/per-session cases + unknown-DID 0xFFFF + session-gated DTC cases
  from `dtcs[].sessions`); `mock_negative_rules` (the same derivation shaped
  for the mock); `mock_dtc_config` (status bytes seeded 0x05 unless pinned;
  the NRC cited from the **artifact**, never the macro's `expect_nrc`);
  `mock_security_rules` (profile → mock rejectable-key model);
  `get_security` (levels resolved against `LEVEL_CATALOG`, ADR-005).
- **Path helpers.** `macro_path_for`, `coverage_path_for`, `trace_path_for`,
  `result_path_for` — the canonical "next to the `.uds.yaml`" derivations
  (SRS §2.4.10).
- **Step model & canonical builder.** `Step` (row model; ⚡ raw-override
  labels as `raw <hex> (was <type label>)`);
  `request_bytes_for_step(step)` — the **single canonical builder** of every
  step's request bytes (row suffix, frame editor, reset/status, mock/live
  transmission all derive from it — NFR-16).
- **Macro document.** `build_macro_doc`, `render_macro_yaml` (hex-int dump —
  lossless round-trip), `write_macro`, `validate_steps` (`validate_macro` +
  artifact-aware warnings: the ⚠ set).
- **Warnings & gating.** `knowledge_warnings`; `session_gate_violations`
  (drives the pre-flight ⚠ markers through the shared formats).
- **AI chat (ADR-037).** `render_step_contract` (rendered from the engine's
  actual step registry — **drift-checked** against `KNOWN_STEP_TYPES` /
  `REQUIRED_STEP_PARAMS`), `state_at_position` / `render_macro_state`
  (session/security state at each stack position), `extract_proposal` (the
  `steps`/`insert_after`/`rationale` envelope), `provisional_gate` (Gate PG —
  splice + gate before the row is shown, FR-GATE-14).
- **Profiles.** `load_profile(args_profile, …)` — the cascade: `--profile`
  → env `CANFORGE_CLOSEDLOOP_PROFILE` → `.nausica.yaml` → `config.yaml` →
  **auto-match by seed/key pairs** (SRS §4.3, FR-UDS cascade).
- **In-process runs (the event-stream producers).** `run_mock_inprocess(…)`
  (tracing proxy around the mock: every ISO-TP frame → events, ECU frames
  restamped to the response CAN ID, simulated P2 delay; then the **real**
  `AutomationEngine`; writes the CLI result schema; sibling `.uds.yaml`
  injects negative rules + DTC config + timing + security rules) and
  `run_live_inprocess(…)` (same stream; `mode: live`; subclasses the real
  backend — TX tap + RX queue, no simulation). Both emit the contract of
  §6.2: `("pkt", t_ms, "TX"|"RX", can_id, payload_hex)` · `("step", idx,
  stype)` · `("info", text)` · `("ok", result)` · `("fail", text)`.
- **Refs.** FR-FUDS-02…14; FR-UDS-18…29; ADR-001/002/003/004/005/036/037/
  043/044/045.

#### 4.2.2 `can_core.py` — CAN shared core

- **Knowledge.** `find_dbc_for_md(md_path)` (canonical DBC location for a
  md); `load_can_knowledge(md, dbc)` — the **DBC-wins** load: md display
  layer + DBC truth layer, drifts → warnings list, never failures
  (NFR-03); `filter_knowledge(kd, query)` (ADR-009, pure: message name /
  CAN ID `1B3`|`0x1B3` / signal name / unit; a message match keeps ALL its
  signals).
- **Plan model & document truth (ADR-008/011).** `PlanStep` (schema
  `template_v1` + the additive `data: Optional[list]` raw override —
  `to_dict` **omits** `data` while `None`, so pre-override plans round-trip
  byte-identically); `steps_to_plan_doc` / `plan_doc_to_steps` (the
  projection pair); `parse_plan_doc_text` (PLAN JSON tab shape check on the
  **repaired** text — deeper meaning is Gate 2's job); `quote_bare_auto`
  (ADR-011: decoder-guided repair of the bare `auto` **value token**, value
  positions only, idempotent); `render_plan_json`; `coverage_from_steps`.
- **Paths.** `plan_path_for` (`TestPlan/` + auto-increment
  `TestPlan_2/…` on collision), `unit_dir_for` (grandparent when the parent
  is `DBC_Description`, else parent — the ADR-012 unit derivation),
  `anchor_artifacts_for` / `anchor_artifacts_for_plan` (the five tool-owned
  artifact paths, unit-relative), `asc_path_for` (`<plan dir>/output/<stem>.asc`).
- **Gate 2 (the same gate the CLI save runs).** `validate_plan_steps`
  (errors/warnings/filled_steps: unknown message/signal rejected, values
  within min/max, units, deterministic `complete_missing_signals` repair, P8
  DLC check for raw `data`); `safe_default_signal_value` (knowledge-pane
  click inserts); `coerce_signal_value` (**the gate's own coercion** — the
  Values modal uses it, so modal-clean ⇒ Gate-2-clean).
- **Canonical builders.** `encode_step_frame` (the one "what bytes" builder;
  `use_raw` switch consults `data` first); `expand_plan_to_frames` (the exact
  expansion M/R and the CLI share — raw `data` consulted first).
- **Runs.** `run_offline` (the **TX-only rehearsal scheduler** on a virtual
  timeline — no ECU, honestly labeled `mode: "offline"`) and `run_live`
  (same frames at real-time cadence — the bus IS the clock, speed ignored;
  connect failure = `fail` before any frame; per-frame TX errors
  counted-not-fatal; RX capture off in v1); `hardware_config` (config.yaml
  `hardware:` → backend kwargs).
- **AI chat (CAN).** `load_can_chat_template`, `render_step_contract_can`,
  `render_macro_state_can`, `extract_json_can`, `extract_proposal_can`,
  `gate_proposal_can` (Gate PG for CAN proposals).
- **Refs.** FR-FCAN-02…18; FR-CAN-13/17/20; ADR-007…011/012 [local].

### 4.3 L3 — Handlers

#### 4.3.1 `handler/can/` (D2: never imports `handler/uds`)

| File | Role | Public interface |
|---|---|---|
| `dbc_to_markdown.py` | Step 1: deterministic DBC → two markdown artifacts (SRS §6.1/6.2) | `load_dbc_with_encoding_fix(path)` (UTF-8→GBK→GB2312→Latin-1 + `fix_mojibake`); `_unit_base_name(dbc_path)` (the unit-folder naming, §2.4.10 — kept in sync with the host's command builder because the host drives CANForge as a subprocess); `default_output_paths(dbc_path)`; `dbc_to_markdown(dbc, out_desc=None, out_detail=None, e2e_profile=None)` → Gate-1 error list (empty = PASS) |
| `ai_reasoner.py` | Step 2: AI plan generation/refine (proposal layer) | `generate_test_plan_from_md(md_path, prompt, output, mode, …)` (single + chat entry; prompt assembly, `"""` multi-line REPL, `save`/`exit`); `refine_plan(json_path, mode=quick\|full, …)`; `_extract_json(text)` (tolerant fence/prose stripping); `_run_plan_gate` (Gate 2 invocation); `_auto_repair_rounds` (bounded, FR-GATE-17); `_record_lineage` / `_next_version_path` (source md/dbc + versioned saves) |
| `plan_validator.py` | **The gates of the CAN pipeline** (one home for G1/G2/G3 validation + the shared report) | `validate_dbc_coverage` (G1); `validate_plan_against_dbc` (G2 core); `validate_step_timing`; `complete_missing_signals` (the deterministic fill, FR-GATE-03); `validate_asc_file` (G3b: frames parse, IDs/DLCs DBC-consistent); `find_dbc_for_plan` (the canonical DBC locator for a plan); `print_gate_report` (the **one shared report builder**, FR-GATE-18); `_coerce_numeric` / `_coerce_enum` (the coercion Gate 2 and the modal share) |
| `json_to_asc.py` | Step 3: deterministic plan+DBC → ASC (Gate 3 owner) | `json_to_asc(json_path, dbc_path, output, e2e_profile=None)` — re-runs G2, encodes via cantools (all signals required), cycle expansion + stable tied-starts sort, E2E at encode time (see below), raw `data` transmitted verbatim (additive, guarded), writes + self-validates the ASC; E2E internals: `_find_e2e_signals` (profile detection), `_compute_crc` (via `utils.e2e_computation`), `_apply_e2e_to_payload` (counter + CRC bytes per frame) |
| `asc_player.py` | Step 3.5: ASC playback on hardware | `play_asc_file(asc, channel, baudrate, …, speed, repeats, sync_tx)`; `list_devices(…)` (backend `scan()`) |
| `recorder.py` | Step 3.5: live bus → ASC | `ASCWriter` (the verbose-ASC dialect writer shared with `json_to_asc`'s format expectations); `record_to_asc(output, …, duration, include_tx, append)` |

**Refs.** FR-CAN-01…24; FR-GATE-01…07; ADR-025 [Nausica] (E2E face).

#### 4.3.2 `handler/uds/` — engine and security (D2; D1 for the security leaves)

| File | Role | Public interface |
|---|---|---|
| `automation.py` | **Step 3 runner** — the macro executor both dashboards and the CLI drive | `KNOWN_STEP_TYPES` (the 19-type registry — the single source of truth for step types); `REQUIRED_STEP_PARAMS` (per-type required params — the single source of truth for the macro contract); `RAW_CAPABLE_STEP_TYPES`; `StepResult` (passed, message, `attempts`, final NRC + canonical name, data); `AutomationEngine` (ctor: `tc`, `request_id`, `response_id`, `continue_on_error`, `knowledge`, `timing`) with `_dispatch` + one `_step_*` per type, `_session_gate_refusal` (ADR-004 in-engine refusal via the shared `check_gate`), `_step_raw_override` (ADR-044: transmit the exact bytes), keep-alive loop (background thread, §3.4); `RunSummary` (`all_passed` — the verdict source for the anchor); `_check_timing_block` (range-checked, exit-2 material); `validate_macro` (Gate V — **the exit-2 pre-flight before any hardware**, FR-GATE-11) |
| `security/algorithms.py` | Seed/key derivation for the built-in WHUD ECU (icon, oem_level1-4, vendor2) — the seed data the registry auto-populates | `compute_key` (thin wrapper over `registry.get_algorithm` — **one dispatch point**, ADR-005); `ALGORITHM_MAP`; `SECURITY_LEVELS` (the builtins); `NRC_MESSAGES` (deprecated alias re-exporting `nrc.NRC_NAMES` — kept for back-compat, new code uses `nrc.py`) |
| `security/registry.py` | **The single dispatch point** for algorithms | `register_algorithm(name, fn)`; `get_algorithm(name)` (absent name → named error, never silent fallback — NFR-04); auto-populated from `algorithms.ALGORITHM_MAP`; runtime registrations reachable from the engine (NFR-11) |
| `security/levels.py` | The extensible **level catalog** | `LEVEL_CATALOG` (builtins + profile `security_levels` + runtime, per-field merge, runtime > profile > builtin); `get_level` (the single resolver); `set_profile_levels(dict\|None)` (the profile layer; `None` clears — the dashboard joins in `__init__`, clears on unmount, FR-FUDS-06); `register_level` / `unregister_level` (runtime extension, NFR-11) |

**Refs.** FR-UDS-18…29; FR-GATE-11/12/13; ADR-004/005 [local].

#### 4.3.3 `handler/uds/extract/` — Step 1 (knowledge extraction)

| File | Role | Public interface |
|---|---|---|
| `c_scanner.py` | The deterministic tree-sitter C pre-pass | `read_with_encoding_cascade` (SRS NFR-07 for C sources); `parse_c_file` (macros, enums, **switch and if/else-if** dispatchers with per-case value text — `SwitchInfo.kind` ∈ `switch`\|`if-chain` —, session sites, NRC sites, function signatures); `build_macro_table` (chained, cycle-safe) / `resolve_macro_value`; `score_file` (UDS relevance) / `scan_directory` (skip-list pruning, `include_dirs` replaces it, top-N `used`, default 24); `TreeSitterUnavailable` (→ exit 2); dataclasses `MacroDef`, `EnumDef`, `SwitchInfo`, `FunctionInfo`, `CFileAnalysis`, `ScanResult` |
| `evidence.py` | The token-budgeted evidence bundle | `BudgetOverflow` (fail-fast → exit 2, NFR-13); `prompt_char_budget` / `estimate_tokens` (budget from `ai.context_tokens`); `rewrite_macros_in_line` (`ident → 0x11 /* NAME (file:line) */`; preprocessor lines never touched); `build_bundle` (per-file dispatcher cap, >300-line shaping, contained-span dedup, manifest for Gates B/C); `render_user_message` (the `{…}`-placeholder user message); `render_manifest_text` (the `<stem>.evidence_bundle.md` renderer) |
| `knowledge_schema.py` | The `.uds.yaml` **v2 schema** (SRS §6.8) | `SCHEMA_VERSION`; `REQUIRED_TOP_LEVEL`; `ISO14229_SID_NAMES` (canonical table — 0x15/0x19 corrected per ADR-002); `SCHEMA_JSON` + `schema_json_text()` (embedded in the LLM prompt — **one schema, prompt and gate agree**); `_DID_SHAPE` (the layout spec: big-endian tile, span ≤ `size_bytes`, EMIT-ONLY-WHEN-EVIDENCED); `_DTC_SHAPE` (the `"19"` slot); `SCHEMA_DEFAULTS` (the isotp defaults) |
| `knowledge_validator.py` | **Gates A/B/C** | `validate_knowledge(doc)` (Gate A — v1/v2, structure, layout via `did_layout.validate_layout`, `"19"`/`"14"` checks, null seed/key = warning); `audit_citations(doc, manifest, strip=True)` (Gate B — citation audit + removal mode); `audit_switch_completeness(doc, scan_result)` (Gate C — dispatcher coverage); `print_uds_gate_report` (the shared report for the Step-1 gates, FR-GATE-18) |
| `extract_knowledge.py` | The Step-1 orchestrator | `run_uds_extract_knowledge(args) → (exit_code, summary, uds_stats)` — scan → bundle (budget fail-fast) → LLM (knowledge role) → Gates A→B→C → **up to 3 repair rounds** (no-progress early exit; `{errors}` from the gate report) → write `.uds.yaml` (`status: draft`) or `.uds.rejected.yaml` (never clobbers a good artifact); `--dry-run` / `--show-prompt` / `--security-profile` (advisory cross-check) |

**Refs.** FR-UDS-01…12; FR-GATE-08…10, 17, 18; ADR-030 [Nausica].

### 4.4 L4 — Protocol (ISO-TP / UDS) and the decision leaves

The protocol stack depends **only on the `CanBackend` ABC** — all vendor
translation happens in L5 adapters, and the stack itself is what the mock
exercises in CI. The leaves beside it encode protocol facts and import
nothing (D1).

| File | Layer | Role | Public interface |
|---|---|---|---|
| `isotp.py` | protocol | ISO 15765-2 transport (ADR-006) | `stmin_ms(byte)` (FC stMin decode: 0-100 ms; 0xF1-0xFA = 100 µs steps); `IsoTpSession` (ctor takes `tx_dl`, `p2_timeout`, `fc_blocksize`, `fc_stmin_ms` from the resolved `UdsTiming`): `send_and_wait` / `send_raw`, the ADR-006 TX path (`blocksize` CFs per FC before the re-FC; CF pacing at the ECU's `stMin`; non-continue FC → `IsotpFlowControlError` **before a single CF**; blocksize-0/stMin-0 = legacy continuous TX), `send_flow_control`, the RX side (`_receive`, `_wait_frame`, `_wait_for_fc`) |
| `uds.py` | protocol | ISO 14229 service layer over `IsoTpSession` | `UdsSession`: `send` (NRC-0x78 auto-retry — per-pending-frame deadline = **current** P2\*, total window = `respond_pending_s`); `_apply_50_timing` (ADR-003: 0x50 `timingParameterAvailability` byte + big-endian P2/P2\* pairs; new 0x50 resets to the cascade baseline first; 1 ms…10 s range check); `_assemble_response`; service methods `session_*`, `read_data_by_identifier`, `write_data_by_identifier`, `read_dtc` (verifies the `59 <sf>` envelope via `dtc.py`), `clear_diagnostic_information` (verifies the `54 <group>` echo — ADR-002), `tester_present`, `ecu_reset`, `request_seed`, `send_key`, `routine_control_start/stop`, `io_control_by_identifier`, `send_raw_uds` |
| `nrc.py` | **leaf** | The canonical NRC table (ADR-005) | `NRC_NAMES` (2nd-edition numbering — 0x33 `securityAccessDenied`, 0x34 `exceededNumberOfAttempts`, 0x35 `requiredTimeDelayNotExpired`; the ONE table engine/mock/trace/gates name from); `SECURITY_NRC_RECOVERY` (per-NRC guidance for 0x33/0x34/0x35 — one builder renders engine refusal and the trace hint) |
| `session_gate.py` | **leaf** | ADR-004: the shared session-precondition state machine | `session_state_at(steps, index)` (static walk — `uds_session` moves the session and **clears** unlocked levels; `known: False` = no 0x10 yet, default 0x01 assumed); `LiveSessionState` (fed with **actual** acks; the 0x11/0x71 ack does not update it yet — staleness errs permissive); `gate_table_from(knowledge)`; `check_gate(...)` (**the one decision point**: per-value allow-list → bulk DTC offenders → `requires_security` → `GateViolation` or None); `format_refusal` / `format_warning` (ONE source for engine refusal, validation warnings, dashboard ⚠ markers) |
| `uds_timing.py` | **leaf** | ADR-003: the four-source timing cascade | `UdsTiming` (per-field value + **source** tracking; `source()`, `macro_doc()`, `as_result()` — the result's per-field source records); `resolve_timing(cli_flags, macro_block, artifact, profile)` (first-hit-wins per field: CLI > macro > artifact `transport.isotp` > profile `timeout:` > engine defaults P2 1.0 s / P2\* 5.0 s / total 5.0 s / stMin 20 ms / blocksize 0 / tx_dl 8); `describe_timing` (the human-readable cascade report) |
| `dtc.py` | **leaf** | ADR-002: the single interpreter of DTC service semantics | `decode_status_byte` (ISO 14229-1 **Annex C**); `parse_dtc_record`; `parse_059(subfunction, payload, dtc_length)` — one branch per v1 subfunction (0x01/0x02/0x04/0x05/0x06/0x0A/0x0B) checking DTCFormatIdentifier + payload shape; 0x07 and 0x0C/0x0D → `DtcParseError` (the engine turns it into a step failure, never a guess — NFR-04) |
| `did_layout.py` | **leaf** | ADR-001: the single DID payload codec | `decode_value(layout, raw_bytes)` / `encode_value(layout, values)` (the one encode/decode pair — trace decode, `values` writes, readback comparison all use it); `validate_layout` (Gate A's layout hard-error check: big-endian tile without overlap, span ≤ `size_bytes`) |

**Refs.** FR-UDS-14…20; FR-GATE-12/13/15; ADR-001…006 [local].

### 4.5 L5 — Hardware (`hardware/usb_to_can/`)

#### 4.5.1 `canforge_io/` — the backend abstraction (D4: imports nothing from L3/L4)

| File | Role | Public interface |
|---|---|---|
| `__init__.py` | The factory (the hardware extension point) | `register_backend(name, factory)`; `create_backend(name, **kwargs)` (unknown name → named error) |
| `backend.py` | The contract every backend implements | `CanFrame` (dataclass: CAN ID, payload, FD flag); `CanBackend` (ABC): `connect(channel, baudrate, fd, data_baudrate, serial)` (`channel` 0-based) · `disconnect()` · `transmit(frame)` · `transmit_batch(frames)` · `receive(timeout_ms)` · `register_callback(can_id, queue)` / `unregister_callback(can_id)` · `scan() → list[dict]` |
| `tosun_libtscan.py` | The production backend | `TosunLibtscanDevice` — ctypes binding to `libTSCAN.dll` (shipped in `libTSCANDemos/Python/native/`, loaded at runtime): `CanFrame` ↔ vendor struct (CAN 2.0 + FD), RX callback → queue, `scan()` for `list-devices`/preflights |
| `mock_backend.py` | The in-process UDS ECU simulator (FR-UDS-27 — every face default-off) | `MockCanBackend` (ctor faces: `negative_rules=`, `dtc_config=`, `timing=(p2_ms, p2_extended_ms)`, `security_rules=`, `tx_fc=(blocksize, stmin)`); ISO-TP reassembly (`_on_single_frame` / `_on_first_frame` / `_on_consecutive_frame`; strict-FC-as-receiver when `tx_fc` set — re-FC after every `blocksize` CFs, else legacy `30 00 00 00`); `_canned_response` (deterministic responses; ISO-14229 conformance NRCs 0x12/0x13; 0x10 deliberately excluded); `_security_send_key` (rejectable-key: wrong key → `7F 27 33` + counter, exhausted → `7F 27 34`, unresolvable algorithm → `7F 27 31`); `_dtc_read_response` / `_dtc_clear_response` (artifact-seeded DTC state: seeded 0x05 or entry `status`; unlisted → 0x12; malformed → 0x13; session-gated → rule NRC); the **stateful per-run DID store** (0x2E stores per instance, 0x22 of a stored DID returns it; NOT cleared by `connect()` — `readback: true` is a real test); ECU frames stamped with the request CAN ID (dashboard restamps); counters `uds_requests`/`frames_tx`/`frames_rx` |
| `tosun_tsmaster.py` | SKELETON (registered, unimplemented frame path) | `TSMasterBackend` — lazy SDK load; frame conversion raises `NotImplementedError` [out of scope, SRS §9.1] |

#### 4.5.2 `canforge_tosun/` — legacy ctypes bindings (known debt, §10)

| File | Role |
|---|---|
| `device.py` | `TosunDevice` — context manager over the DLL: scan/connect/transmit/transmit_batch (sync + timeout), pass filters, RX dispatcher thread |
| `openloop.py` | `ASCFrame`, `parse_asc_line` / `parse_asc`, `ASCPlayer` — the legacy ASC playback path (predecessor of `handler/can/asc_player.py`) |
| `types.py` / `_native.py` | ctypes struct definitions and raw DLL bindings |
| `exceptions.py` | Exception classes — **the most-widely-imported submodule**: `handler/uds/isotp.py` and `uds.py` import from here (the grandfathered debt of §3.2; target: a project-leaf exception module with re-exports here) |
| `closedloop.py` | **Dead** — superseded by `handler/uds/isotp.py` + `uds.py`; zero imports; removal is scheduled, not urgent |

### 4.6 L6 — Utilities (D6: stdlib + third-party only)

| File | Role | Public interface |
|---|---|---|
| `llm_handler.py` | **All Ollama communication** (SRS §4.4) | `keep_model_alive` (warm-model); `translate_text` / `translate_markdown_file` (translate-md); `llm_call_single_shot` (plan single-shot); `llm_call_chat_stream` (`on_token` callback — the TUI strips); `llm_call_chat` (content + token usage). Server/model per call from the resolved role (CLI → env → config, SRS §4.4); `context_tokens` sent as request-level `options.num_ctx` |
| `audit_logger.py` | The ASPICE trail (FR-UTIL-06) | `AuditLogger.log_step(...)` — append-only JSONL to `data/audit_log.jsonl` (timestamp, command, inputs, outputs, exit code, duration) |
| `terminal_spinner.py` | The console-degradation spinner (NFR-08) | `TerminalSpinner` — Braille on UTF-8/VT, ASCII on cp1252, plain timestamped lines when piped |
| `e2e_computation.py` | E2E CRC math (SRS §6.15) | `crc_sae_j1850`; `crc_ones_complement_sum`; `crc_ones_complement_sum16`; `get_crc_function(name)` (the profile's `algorithm` → function dispatch) |
| `e2e_profile_loader.py` | E2E profile layer (ADR-025) | `load_profile(name)` (from `profile/e2e/`); `list_profiles()`; `resolve_profile(cli_name, ...)` (CLI flag > env `CANFORGE_E2E_PROFILE` > `.nausica.yaml` > None) |
| `uds_profile_loader.py` | UDS security profile layer | `load_profile(name)` (from `profile/uds/`); `list_profiles()`; `resolve_profile(cli_name, ...)` (the CLI side of the FR-UDS cascade) |
| `testcase.py` | **ADR-012: the unit anchor lifecycle** (leaf — stdlib + ruamel.yaml only) | `TestcaseExistsError`; `unit_name_for(dir)` / `testcase_path_for(dir)` / `find_testcase(dir)`; `init_testcase(unit_dir, kind, environment, artifacts, owner="")` (writes the comment-rich anchor, **refuses** an existing unit); `update_tool_owned(anchor, artifacts=, execution=)` — the **only** writer of tool-owned keys, ruamel round-trip (engineer comments survive), **never raises** (missing/unparseable/OSError → skip, file intact); `update_for_anchor` (find + update; no anchor → silent no-op); `verdict_from_can_result` (offline → always pass; live → pass iff no errors); `verdict_from_uds_result` (all_passed) |

### 4.7 L7 — Data & asset components

- `config.yaml` — defaults for AI servers/models/`context_tokens`, paths,
  translation, hardware (SRS §6.15).
- `profile/e2e/*.yaml` — E2E profiles (`generic_autosar`, `oem_hud`,
  `none`); `profile/uds/*.yaml` — UDS security profiles (`oem_whud`).
  Shapes: SRS §6.15.
- `prompts/*.txt` — the prompt catalog (SRS §4.4): `test_plan_prompt.txt`,
  `test_plan_chat_prompt.txt`, `can_plan_chat_prompt.txt`,
  `uds_knowledge_system.txt` / `uds_knowledge_user.txt` /
  `uds_knowledge_repair.txt`, `uds_step_chat_prompt.txt`.
- `json/template_v1.txt` — the CAN plan schema of record (SRS §6.3).
- `plugin.yaml` — the host manifest (SRS §4.3): 16 menu items, `method` /
  `subprocess` handlers; the method names resolve to host-side (Nausica)
  `MainWindow` methods — CANForge's contract is the manifest + argv, not the
  handler bodies.
- `data/` — runtime only (`audit_log.jsonl`); never committed.
- `tests/` — sample DBCs + generated artifacts (fixture sources for the
  suite, §9).

### 4.8 Component → SRS coverage note

Every SRS functional requirement maps to at least one component above
(FR-CAN-* → 4.3.1; FR-UDS-* → 4.3.2/4.3.3/4.4/4.5.1; FR-FUDS-* → 4.1.2+4.2.1;
FR-FCAN-* → 4.1.3+4.2.2; FR-ASC-* → 4.1.4; FR-UNIT-* → 4.6 `testcase.py` +
4.1.1/4.1.2/4.1.3 gesture wiring; FR-UTIL-* → 4.6). The full matrix is kept
in §10.

---

## 5. Key Architectural Decisions

This section is the *why* behind §4. It covers the two meta-decisions that
shape the whole system (§5.1, §5.2), the engine/surface topology (§5.3),
then each local ADR (ADR-001…012, CANForge series) in the order it was made.
Full normative text lives in `doc/ADR/ADR_NNN_*.md`; the entries here are the
rebuild-grade summary: problem → decision → consequences, plus which Nausica
series ADRs each local decision composes.

### 5.1 Meta-decision: default-off parity (NFR-01)

**Problem.** The pipeline matured by adding faces (layouts, DTC, timing,
gates, security, FC, `expect_nrc`, `raw`) to a system that already ran
production macros. Any face that changed default behavior would have silently
altered the frames existing plans/macros put on the bus.

**Decision.** Every face is *additive and absent-by-default*; absence means
**byte-identical legacy behavior** — a hard invariant, not a goal:

- A face is absent when its trigger is missing: no `layout` on a DID, no
  `"19"`/`"14"` service in the artifact, no `timing:` block, no
  `enforce_session_gates`, no `expect_nrc`/`raw`/`values`/`readback` on a
  step, no mock `negative_rules`/`dtc_config`/`timing=`/`tx_fc=`/
  `security_rules`.
- Implementation discipline: seams are `None`-checked and the legacy path is
  the *original* code path, not a special case of the new one (`PlanStep.to_dict`
  omits `data` while `None` so pre-override plans round-trip byte-identically;
  the mock without faces emits exactly the legacy canned responses, incl.
  `50 <sub> 00 00` and FC `30 00 00 00`).
- Verification: parity is pinned by golden tests — the same input must
  produce the same artifact/frames/result before and after a face exists
  (§9, `test_parity_*`).

**Consequences.** Rebuild order is constrained: the legacy core must exist
and be golden-pinned *before* any face is rebuilt; each face ships with its
own parity check. The cost is the `None`-checks everywhere; the benefit is
that no historical artifact ever needs migration.

### 5.2 Meta-decision: three truth layers + a gate lattice

**Problem.** AI-generated test content and deterministic artifacts must mix
without either being able to corrupt the other.

**Decision.** Three layers, strictly ordered:

```
TRUTH      DBC (CAN) / ECU C sources (UDS)   — read-only, machine-parsed
ARTIFACT   .can.md, .can.plan.json, .uds.yaml, .uds.macro.yaml — human-owned,
           machine-validated, versioned
PROPOSAL   AI text/plans/steps                — enters only via a gate + human
```

- The truth layer never parses an artifact to learn anything (the DBC wins
  every drift; the scanner reads C, not the yaml).
- The artifact layer is validated by **gates** — pure functions of
  (artifact, truth) that return a structured error list; a gate never writes
  and never calls the LLM. Gate set: G1/G2/G3 (CAN), A/B/C (Step 1), V
  (macro pre-flight), SG (session gating), PG (provisional AI gate) —
  SRS §7.
- The proposal layer is quarantined: AI output is parsed out of prose
  (tolerant extraction), gate-validated, and shown for human acceptance —
  in the TUIs an AI step is a *dashed review row* (Accept/Edit/Reject), and
  PG gates the proposal on a spliced stack copy before it is even rendered
  (ADR-037). **No AI step or frame reaches the bus without a gate + a human
  gesture** — the single sentence the whole gate lattice exists to make
  true.

**Consequences.** One shared gate-report builder (FR-GATE-18) and one
exit-code policy (SRS §8) follow directly: gates are uniform enough to
compose (e.g. C gesture = Save write half → Gate 3 in-process) and to be
trusted with pre-hardware refusal (Gate V, exit 2, no side effects).

### 5.3 Meta-decision: one engine, four surfaces

**Problem.** The same execution is needed from the CLI (scripts, CI, Nausica
host) and from three TUIs (live trace, in-place editing). Two engines would
diverge; a subprocess-per-gesture TUI would lose the trace stream.

**Decision.**

- **One execution core per pipeline.** `AutomationEngine` (UDS) and
  `expand_plan_to_frames` + the offline/live schedulers (CAN) are the only
  producers of frames and results. The CLI and the TUIs are *surfaces* over
  them.
- **In-process, through a core seam.** The TUIs run engines in-process via
  `uds_core` / `can_core`, which wrap the engines behind a small event
  stream (§6.2: `pkt`/`step`/`info`/`ok`/`fail`). The seam is the contract:
  cores import no widget code (D7), so the TUI can be rebuilt without
  touching the engines.
- **Shells are copied, not imported** (D3): the CAN dashboard shell was
  copied from the UDS one and re-themed (red boundary) so the two TUIs can
  diverge in UX without an import edge pulling UDS semantics into CAN.
- **Subprocess is reserved for true topology changes** — the C gesture
  launching the ASC playback dashboard in a **new console**
  (`CREATE_NEW_CONSOLE`), and the Nausica host invoking CANForge as a
  subprocess (the host's only interface is `argv` + exit code + captured
  output, SRS §4.3).

**Consequences.** Engine changes are tested once (handler unit tests + Textual
pilot tests for surfaces); the TUI can't drift from the CLI because both
write the *same* result schema from the *same* builders (NFR-16).

### 5.4 ADR-001 [local] — DID payload layout + write read-back

**Problem.** A DID is a byte blob; "write a meaningful value" and "decode
what the ECU returned" were undefined for any DID whose payload shape wasn't
documented.

**Decision.**
- Optional per-DID `layout` in `.uds.yaml` (v2 schema, SRS §6.8): big-endian
  field tiles `{name, offset, length, encoding}` with `bits` / `scale` /
  `offset_value` / `min` / `max` / `unit`; Gate A structure-checks it via
  `did_layout.validate_layout` (no overlap, span ≤ `size_bytes`).
- `did_layout.py` is the **single decode/encode pair** — `values`-write
  encoding, readback comparison, and trace field-decoding all call it.
- The layout is **EMIT-ONLY-WHEN-EVIDENCED**: the Step-1 pipeline emits a
  layout only when the C sources evidence the payload shape; a DID without
  evidence gets no layout, never invented fields.
- `readback: true` re-reads after a write and compares through the *same*
  codec; it never bypasses the session gates (a gated readback fails like any
  gated read).

**Consequences (default-off).** DIDs without a layout keep the legacy
placeholder-write path; the WRITE leaf of a laid-out DID prompts one value
per field (dashboard) and inserts `values` + `readback: true`.

**Refs.** SRS §6.8 (layout shape), FR-UDS-06/19, FR-GATE-08.

### 5.5 ADR-002 [local] — DTC read/clear services

**Problem.** 0x19/0x14 have multiple subfunctions with different payload
shapes, and "clear succeeded" needs proof, not an ack.

**Decision.**
- `dtc.py` is the **single interpreter** of DTC service semantics, shared by
  engine, dashboard and mock: `decode_status_byte` (ISO 14229-1 **Annex C**)
  and `parse_059(subfunction, payload, dtc_length)` with one branch per v1
  subfunction (0x01/0x02/0x04/0x05/0x06/0x0A/0x0B), each checking the
  DTCFormatIdentifier and payload shape. 0x07 and 0x0C/0x0D are deferred and
  raise `DtcParseError` — the engine turns that into a step failure, **never
  a guess** (NFR-04).
- The status byte layout is *not* in the artifact: it's fixed by Annex C, so
  the artifact only carries the `dtcs` table (ids, names, optional
  `sessions` allow-list, optional mock-seed `status`) plus the `"19"`
  subfunction scope + `dtc_length` and the `"14"` 3-byte groups.
- `uds_read_dtc` passes only on a positive 0x59 the parser accepts (result
  carries parsed structure + raw payload); `uds_clear_dtc` verifies the
  `54 <group>` echo (a 0x54 echoing a *different* group is malformed, not an
  ack) and `verify: true` (default) re-reads `19 01 FF` — never
  "cleared (assumed)".
- `dtcs[].sessions` join the ADR-043 negative-case derivation (a DTC read
  outside its allowed session is a derived ✗ case).

**Consequences (default-off).** No DTC services in the artifact ⇒ no DTC
steps, no mock DTC state — legacy.

**Refs.** SRS §6.8 (`"19"`/`"14"` slots), FR-UDS-10/15/16.

### 5.6 ADR-003 [local] — dynamic UDS timing

**Problem.** P2/P2*/stMin/blocksize were constants; real ECUs negotiate them
(0x50) and need them from several possible sources — with no record of which
value actually ran.

**Decision.**
- One `UdsTiming` (leaf `uds_timing.py`) resolved by a **four-source
  cascade**, first-hit-wins **per field**: CLI flags > macro `timing:`
  block > artifact `transport.isotp` > profile `timeout:` > engine defaults
  (P2 1.0 s / P2* 5.0 s / total 5.0 s / stMin 20 ms / blocksize 0 / tx_dl 8).
  Every field carries its **source**, and the result record says which layer
  won (NFR-09).
- A 0x50 with `timingParameterAvailability` set updates the session's
  P2/P2* from the big-endian ms pairs (range-checked 1 ms…10 s; out of range
  ignored); a new 0x50 first resets both to the cascade baseline.
- The NRC-0x78 poll uses the **current** P2* per pending frame with
  `respond_pending_s` as the total window.
- `tx_dl`/`p2_timeout`/`fc_blocksize`/`fc_stmin_ms` flow into the `IsoTpSession`
  ctor, so ISO-TP and UDS layers share one resolved value.

**Consequences (default-off).** No source set ⇒ exactly the legacy
constants; the mock negotiates `50 <sub> 00 00` unless `timing=` is
injected.

**Refs.** SRS §6.8 (`transport.isotp`), FR-UDS-20/25.

### 5.7 ADR-004 [local] — session precondition gating

**Problem.** Steps fail late and opaquely when the wrong session is active
(a 0x22 gated to the extended session, issued from the default one).

**Decision.**
- `session_gate.py` (leaf) holds the shared state machine + allow-list
  check. `session_state_at` is a static walk (a `uds_session` moves the
  session and **clears** the unlocked levels; before any 0x10 the state is
  `known: False` and the default session 0x01 is assumed); `LiveSessionState`
  is the live variant fed with **actual** acks (the 0x11/0x71 reset ack
  deliberately does not update it — staleness errs permissive, never
  stricter).
- `check_gate(...)` is the **one decision point**: per-value DID
  allow-list → bulk DTC offenders → `requires_security` → `GateViolation` or
  None. Refusal/warning message formats live here, so the engine refusal,
  `validate_macro` warnings and the dashboard ⚠ markers render from ONE
  source.
- Two faces, both default-off in strength: the **pre-flight warning**
  (always on, warning-only — `expect_nrc` steps exempt, they are *supposed*
  to be refused) and the opt-in **in-engine gate**
  (`enforce_session_gates: true`) which refuses pre-send. The artifact is
  resolved at run time (explicit `--knowledge` wins over the sibling
  `.uds.yaml`).
- The tester-side gate and the mock's ECU-side rules **can disagree** — the
  trace shows both; that discrepancy is information, not a bug.

**Refs.** SRS §6.8 (`dids[].sessions`, `requires_security`), FR-GATE-12/13.

### 5.8 ADR-005 [local] — pluggable security + security-NRC specifics

**Problem.** Seed/key algorithms and level semantics were hardcoded to one
OEM, and 0x27 failures gave no per-NRC guidance (0x33, 0x34 and 0x35 mean
three different things: retry with a fresh seed, reset required, delay
running).

**Decision.**
- `security/registry.py` is the **single dispatch point**: engine and
  dashboard resolve a level's `algorithm` through `get_algorithm` (falling
  back to the level name — the builtin convention; an absent name is a named
  error, never a silent fallback). New algorithms = `register_algorithm()`,
  no source edit.
- `security/levels.py` holds the **level catalog**: builtins
  (`SECURITY_LEVELS` — the built-in WHUD ECU: icon / oem_level1-4 / vendor2
  seeds) + the profile's `security_levels` + runtime `register_level()`
  registrations, per-field merge (runtime > profile > builtin); `get_level`
  is the single resolver; `set_profile_levels(None)` clears the layer.
- **Per-NRC 0x27 semantics** (the engine, one builder): 0x33
  `securityAccessDenied` → one fresh-seed retry by default
  (`max_key_retries`, optional step param — the fresh seed is the only sane
  retry); exhausted → fail naming the attempt count. 0x34
  `exceededNumberOfAttempts` → fail immediately naming the recovery (ECU
  reset 0x11 or OEM delay), **never performing it** (no auto-reset). 0x35
  `requiredTimeDelayNotExpired` → fail immediately (the NRC carries no
  duration — a guessed sleep is a lie). 0x31 → fail naming the level's
  `session:` field. Other NRCs keep the generic shape with the canonical
  name.
- `nrc.py` is the ONE canonical NRC table (`NRC_NAMES`, 2nd-edition
  numbering) + `SECURITY_NRC_RECOVERY` — engine refusal, the mock's
  7F-27-33/34/31 sequence, and the dashboard trace hint all render from it.
- `StepResult` carries `attempts` + final NRC + canonical name.

**Consequences (default-off).** No profile + no runtime registrations ⇒
exactly the static builtin dict. A level whose declared algorithm is
unregistered is an **exit-2 validation error** (Gate V), not a run-time
surprise. The mock's rejectable-key model only activates with
`security_rules=`.

**Refs.** SRS §6.8 (security), §6.15 (profile shape), FR-UDS-21/26.

### 5.9 ADR-006 [local] — ISO-TP TX flow control

**Problem.** The TX side ignored the ECU's Flow Control — it blasted
consecutive frames, which real ECUs can drop.

**Decision.**
- The TX path **honours the ECU's FC**: transmit `blocksize` CFs per FC
  (0 = unlimited) before waiting for the re-FC; CFs are paced at the ECU's
  `stMin` (0-100 ms; 0xF1-0xFA decode to 100 µs steps via `stmin_ms`); a
  non-continue FC raises `IsotpFlowControlError` **before a single CF**.
- The strictness is injected: `fc_blocksize`/`fc_stmin_ms` come from the
  resolved `UdsTiming` cascade (ADR-003), so FC strictness rides the same
  cascade and is recorded the same way.
- The mock implements the *receiving* side of the same contract
  (`tx_fc=(blocksize, stmin)` pushes a re-FC after every `blocksize` CFs),
  so the TX path is exercised in CI against a strict ECU.

**Consequences (default-off).** Absent ⇒ blocksize-0/stMin-0 = the legacy
continuous TX; absent mock `tx_fc` ⇒ legacy `30 00 00 00` FC.

**Refs.** SRS §6.8 (`transport.isotp`), FR-UDS-18, NFR-01.

### 5.10 ADR-007…011 [local] — the CAN Flowline family

The five decisions that built the CAN dashboard (the UDS twin is ADR-036
[Nausica]). They are summarized together because they form one UX
architecture, but each ships and parity-checks independently.

- **ADR-007 — the Flowline CAN dashboard.** A Textual TUI (RED boundary
  `#dc2626`, shell copied from — never imported from — the UDS dashboard,
  D3) that builds a `.can.plan.json` from a `.can.md` + its DBC. Knowledge
  tree (branch per message, leaf per signal; click inserts a safe-default
  step), step stack, two-tab workbench, AI chat, M (offline rehearsal) / R
  (live run, device-scan preflight), S (save). The DBC is the truth layer:
  the md is a display layer, drifts are warnings, never failures (NFR-03).
- **ADR-008 — the two-tab PLAN WORKBENCH.** The plan document is the single
  source of truth (`self._doc`, always valid JSON — the seeded file parsed
  verbatim, unknown top-level keys included, else the knowledge skeleton)
  and the step stack is its `test_sequences` **projection**. Asymmetric
  sync, one direction at a time: stack mutations write the projection back
  in place (unknown keys preserved, `total_duration_ms` fill rule) and
  re-render after flushing any pending parse; JSON-side keystrokes debounce
  0.3 s → shape-check → adopt verbatim when valid, red status cell when
  invalid (**document and stack untouched** — the tab is never rewritten
  mid-edit). S = Gate 2 over the projection → writes the engineer's
  verbatim document + coverage → always re-renders the tab (a save-while-
  broken saves the last valid document + warning). M/R read the projection,
  so a broken tab can never put a frame on the bus.
- **ADR-009 — live knowledge filter.** A 1-row search on the knowledge pane
  (F focuses, Esc clears + refocuses, every keystroke re-projects through
  the pure `filter_knowledge`): message name / CAN ID (`1B3`|`0x1B3`) /
  signal name / signal unit; a message match keeps ALL its signals.
- **ADR-010 — Compile and Run (C).** C = Save's write half (Gate-2-dirty
  never compiles) → in-process Gate 3 (`json_to_asc` with the loaded DBC +
  E2E profile, stdout captured) → the ASC playback dashboard in a **NEW
  terminal** (`CREATE_NEW_CONSOLE`, `--dbc` from the loaded knowledge);
  output `<TestPlan>/output/<base>.asc`. The standalone Export button was
  retired — one path, not two.
- **ADR-011 — two-way resync + bare-`auto` repair.** Each tab carries
  `↺ resync` (each direction adopts the tab it is pressed on and discards
  the other side's un-adopted edit; the STEP STACK tab got its own mirror
  of the JSON tab's); `quote_bare_auto` is the decoder-guided repair of the
  bare `auto` **value token** (value positions only, idempotent — the E2E
  "auto" the AI loves to emit unquoted). `parse_plan_doc_text` parses the
  REPAIRED text; the tab text itself is still never rewritten mid-edit.

**Refs.** FR-FCAN-01…18; ADR-025 [Nausica] for the E2E face C compiles with.

### 5.11 ADR-012 [local] — the test-case unit anchor

**Problem.** ASPICE traceability needs the *folder* to be the test case —
not one file among many — but the tool writes artifacts and the engineer
writes meaning, and neither may clobber the other.

**Decision.**
- The right-clicked **folder IS the test case** (host surface: the two
  directory menu items "Test Case: UDS" / "Test Case: CAN" — Nausica owns
  the browse + init trigger + dashboard launch). The unit's anchor
  `<unit>.testcase.yaml` is comment-rich YAML with a strict ownership split:
  - *Engineer-owned* (tool never touches): `title`, `status` (never
    tool-transitioned — a `blocked` verdict is engineer-only),
    `requirement_refs`, free-text fields, `owner`.
  - *Tool-owned* (one writer): `environment`, `artifacts` (the five
    canonical artifact paths), `execution.date/mode/verdict`.
- `utils/testcase.py` (leaf) is the whole lifecycle: `init_testcase`
  **refuses** an existing unit (`TestcaseExistsError` → host preflight
  refusal, exit 2 on the CLI); `update_tool_owned` is the **only** writer
  of tool-owned keys, ruamel round-trip so the engineer's inline comments
  survive, and **never raises** (missing/unparseable/OSError → skip, file
  left intact — a broken anchor warns and stays intact, a missing anchor is
  a silent no-op).
- **Re-assertion cadence:** after every S / C / M / R / E gesture in both
  dashboards (via each core's `_anchor_update`) and after the CLI save/run
  paths. Verdicts come from one mapper each:
  `verdict_from_can_result` (offline → always pass; live → pass iff no
  errors) and `verdict_from_uds_result` (all_passed).
- Unit layout is canonical (SRS §2.4.10): the md sits in
  `<unit>/DBC_Description/`, the plan in `<unit>/TestPlan/`, the anchor at
  the unit root; `unit_dir_for` and the host's command builders share the
  `DBC_Description` convention.

**Consequences.** The anchor is the bridge to ASPICE evidence without the
tool ever owning the engineer's words; the never-raises writer means a
corrupt anchor can never wedge a save gesture.

**Refs.** SRS §2.4.10, §6.12, FR-UNIT-01…06.

### 5.12 Extension points

Where the system is deliberately open — a rebuild must preserve these seams
exactly (NFR-11):

| Extension | What changes | What does NOT change |
|---|---|---|
| **New CAN backend** (vendor device) | A module implementing the `CanBackend` ABC (SRS §4.5) + `register_backend(name, factory)` | Engines, ISO-TP/UDS, dashboards, the mock; `create_backend` is the only lookup |
| **New security level / algorithm** (new OEM ECU) | `register_algorithm()` + `register_level()` at runtime, or a `security_levels` entry in a profile YAML | `automation.py`, `algorithms.py`, the dashboards; `get_algorithm` / `get_level` stay the single dispatch/resolution points (ADR-005) |
| **New E2E profile** | A YAML in `profile/e2e/` (+ a function in `e2e_computation.py` only for a new CRC algorithm, via `get_crc_function`) | All pipeline code — the E2E face is profile-driven (ADR-025 [Nausica]) |
| **New UDS security profile** | A YAML in `profile/uds/` (CAN IDs, seed/key entries, `timeout:`, `security_levels`) | The cascade (`resolve_profile`), the engine |
| **New macro step type** | `KNOWN_STEP_TYPES` + `REQUIRED_STEP_PARAMS` entry + one `_step_*` method in the engine | The AI chat prompt: `render_step_contract` renders from the registry and is **drift-checked** against it, so the proposal surface updates with the registry; Gate V picks the new type up from the same tables |
| **New DID layout / DTC entry / session** | The `.uds.yaml` (artifact) — engineer edit or Step-1 re-extraction | Engine, gates (Gate A re-validates), dashboards (tree re-projects) |
| **New Ollama model role** | `config.yaml` key + env var + a `llm_handler` call site | The LLM seam itself (one module, SRS §4.4) |
| **New host menu item** | An entry in `plugin.yaml` (+ a host-side method) | CANForge — the host contract is manifest + argv + exit code (SRS §4.3) |
| **New gate** | A pure (artifact, truth) → error-list function + rendering through the shared report builder (FR-GATE-18) | The exit-code policy (SRS §8) and the gate lattice's properties (pure, no LLM, no writes — §5.2) |
| **Future CAN Step 4 (Play+Monitor) & Step 5 (regression); UDS Step 4** | Consumers of the existing evidence (`.can.trace.txt`, `.uds.result.json`, coverage artifacts, audit log) | The evidence schemas — they were shaped to be regression inputs (SRS §9; §10) |

**The one anti-extension:** the TUI shells. A fourth surface should reuse a
core in-process (the pattern of §5.3), not fork another shell; shells are
copied only when the UX must diverge.

---

## 6. Data Model and Flows

SRS §6 fixes *what* each artifact contains; this section fixes *how data
moves* — which component produces it, which gates stand on the edges, and
which builders keep the copies in sync. Schemas are referenced, not
repeated.

### 6.1 End-to-end flows

**CAN open-loop** (each arrow is a gate; a gate failure stops the flow at
the source side, never mid-edge):

```
 .dbc ──G1──▶ DBC_Description/<unit>.can.md (+ <unit>_detail.can.md)
                 │  (AI planning reads the description; the DBC stays truth)
                 ▼
        CAN dashboard / plan-from-md ──(proposal, PG)──▶
                 │
                 ▼  G2 (validate_plan_steps: unknown msg/sig, bounds,
                    units, complete_missing_signals repair, P8 DLC)
   TestPlan/<unit>.can.plan.json  (+ <unit>.can.coverage.yaml)
                 │
                 ▼  G3 (json_to_asc: re-gates, encodes, E2E at encode time,
                    writes, self-validates the ASC)
   TestPlan/output/<unit>.asc ──▶ playback (TSMaster / asc_dashboard / live)
```

**UDS closed-loop** (Steps 1–3; Step 4 is [FUTURE], SRS §9):

```
 ECU C sources ──scan (tree-sitter)──▶ evidence bundle (token-budgeted)
        │                                   │  BudgetOverflow ⇒ exit 2, fail-fast
        ▼                                   ▼
   .uds.yaml (status: draft) ◀──Gate A→B→C── LLM (knowledge role), ≤3 repair
   (.uds.rejected.yaml on hard fail — never clobbers a good artifact)
                 │
                 ▼  (Step 2: dashboard builds the macro from the artifact)
        .uds.macro.yaml ──V (validate_macro, exit 2 pre-hardware)──▶
                 │  (SG: pre-flight warnings always; enforce opt-in)
                 ▼
        AutomationEngine ──▶ CanBackend (tosun_libtscan | mock)
                 │
                 ▼
        .uds.result.json (evidence: per-step, per-field timing sources,
                          mock counters, verdict → anchor execution block)
```

Every edge is owned by exactly one component (§4): no component writes an
artifact another owns, and the only cross-component writes are the
tool-owned anchor keys (6.4).

### 6.2 The in-process run event-stream contract

The seam between the engines and the TUIs is a generator of tagged tuples —
the ONLY channel by which an in-process run reaches the trace pane. Both
cores produce it; the shells only render it.

| Event | Shape | Emitted when |
|---|---|---|
| `("pkt", t_ms, "TX"\|"RX", can_id, payload_hex)` | one ISO-TP frame | every frame the run transmits or receives — mock runs through the tracing proxy (ECU frames **restamped** to the response CAN ID so the pane shows the wire picture; simulated P2 delay, `--sim-p2-ms`), live runs through the backend subclass (TX tap + RX queue, no simulation) |
| `("step", idx, stype)` | one macro/plan step | the engine starts step `idx` (lets the pane group frames under steps and mark the running one) |
| `("info", text)` | free text | progress / warnings (e.g. NRC-78 pending, gate warnings, connection) |
| `("ok", result)` | the result dict | the run finished; the consumer writes the SAME result schema the CLI writes (`uds_core` mock/live runs and `can_core` offline/live runs — SRS §6.10/6.11) |
| `("fail", text)` | free text | the run aborted (connect failure in live mode is a `fail` **before any frame**) |

Consequences of the contract:

- The shells contain **zero** engine logic — a rebuilt TUI is a renderer
  over this stream (D7, §5.3).
- `("ok", result)` is data, not control: the shell writes the result file
  and re-asserts the anchor (§6.4) from it, exactly as the CLI does.
- The offline CAN run is a `("pkt",…)` stream too — frames on a virtual
  timeline — so the same pane renders rehearsal and live runs; the mode is
  labeled honestly (`mode: "offline"`, never faked).

### 6.3 Value cascades (one pattern, four instances)

All configurable values resolve by the same cascade discipline
(§3.5) and record their source (NFR-09). The four instances:

| Value | Cascade (first hit wins) | Source recorded where |
|---|---|---|
| **UDS timing** (P2, P2\*, respond-pending total, stMin, blocksize, tx_dl) — per FIELD (ADR-003) | CLI flags → macro `timing:` block → artifact `transport.isotp` → profile `timeout:` → engine defaults | `.uds.result.json` per-field `source` (`UdsTiming.source()` / `as_result()`); human form via `describe_timing` |
| **Security level / algorithm** (ADR-005) | per FIELD of a level: runtime `register_level` → profile `security_levels` → builtin `SECURITY_LEVELS`; algorithm via `get_algorithm` (level-name fallback, absent = named error) | engine refusal / result text names the resolution; the mock cites the profile |
| **E2E profile** (ADR-025 [Nausica]) | CLI `--e2e-profile` → env `CANFORGE_E2E_PROFILE` → `.nausica.yaml` (`tool_config.canforge.e2e_profile`, upward search) → none | the md/plan/ASC carry the applied profile's effects (counter + CRC signals); the host resolves the same way for all four launch points |
| **UDS profile (closedloop)** | CLI `--closedloop-profile` → env `CANFORGE_CLOSEDLOOP_PROFILE` → `.nausica.yaml` → `config.yaml` → auto-match by seed/key pairs (`uds_core.load_profile`) | the macro run's profile name in the result; `list-security-profiles` / `inspect-security-profile` expose the catalog |

Plus the **AI role** resolution (SRS §4.4): per call, server + model from
CLI → env (`CAN_BACKEND_IP`/`CAN_MODEL_NAME` for chat,
`CAN_KNOWLEDGE_SERVER`/`CAN_KNOWLEDGE_MODEL` for knowledge, the knowledge
role falling back to the chat role) → `config.yaml` (`ai.server`,
`ai.model`, `ai.knowledge_server`, `ai.model`); `ai.context_tokens` is sent
as request-level `options.num_ctx` and drives the evidence budget (6.5).

**Timing is unique in being live-updatable:** the cascade sets the
baseline; a 0x50 ack then updates P2/P2* within the running session
(ADR-003, §5.6) — and the NRC-0x78 poll reads the *current* P2* per
pending frame. No other cascade mutates mid-run.

### 6.4 The unit anchor lifecycle (ADR-012)

```
                 ┌─────────────── engineer owns ───────────────┐
                 │ title · status · requirement_refs · text · owner │
                 └─────────────────────────────────────────────┘
        <unit>.testcase.yaml      tool-owned: environment · artifacts
                 │                  · execution{date, mode, verdict}
                 ▼
  "Test Case: UDS/CAN" (host) ──refuse if anchor exists (exit 2)
        │ browse a .uds.yaml / .dbc (cancel = nothing written)
        ▼
  canforge.py test-case init ──▶ anchor created (comment-rich, ruamel)
        │                         (CAN: source DBC copied + dbc-to-md run;
        ▼                          UDS: stem kept + source_from provenance)
  dashboard on the fresh artifact (--ai-chat, new console on win32)
        │
        │   after EVERY S / C / M / R / E gesture:
        ▼
  core._anchor_update(result) ──▶ update_tool_owned(anchor, artifacts,
        │                           execution{date, mode, verdict})
        │                           ruamel round-trip (engineer comments
        │                           survive) · NEVER raises (broken anchor
        ▼                           warns + stays intact; none = no-op)
  verdict mappers (one per pipeline):
      CAN: offline → always pass · live → pass iff no errors
      UDS: all_passed
```

Invariants: the tool never writes engineer-owned keys (`status` in
particular — no tool transition ever); `update_tool_owned` is the only
writer of tool-owned keys; the anchor is *re-asserted* (idempotent), never
incremented.

### 6.5 The Step-1 flow (deterministic spine, LLM in the middle)

```
 .c/.h ──c_scanner──▶ ScanResult (macros·enums·switch/if-chains·
        │              session sites·NRC sites·signatures; top-N used=24)
        ▼
 evidence.build_bundle ──▶ Bundle (macro-rewritten lines
        │                  `ident → 0x11 /* NAME (file:line) */`,
        │                  per-file dispatcher cap, >300-line shaping,
        │                  contained-span dedup, manifest)
        │                  budget = f(ai.context_tokens) — overflow:
        │                  BudgetOverflow ⇒ exit 2 BEFORE any LLM call
        ▼
 LLM (knowledge role) ──▶ candidate .uds.yaml
        │
        ▼   Gate A (schema v1|v2, layout, "19"/"14") →
        │   Gate B (citation audit; strip mode removes uncited) →
        │   Gate C (switch completeness)
        │   errors ──▶ repair round (≤3, no-progress early exit;
        │              {errors} from the shared gate report)
        ▼
 write  .uds.yaml (status: draft)     exit 0 (pass incl. pass-with-strip:
        .uds.rejected.yaml (hard)      Gate-B-only failures stripped +
                                       noted in open_questions; surviving
                                       Gate-C omissions → open_questions)
```

Side artifacts: `<stem>.evidence_bundle.md` (the manifest render),
`<stem>.prompt.txt` (`--show-prompt`). `--dry-run` stops before the write;
`--security-profile` is an **advisory** cross-check (never a gate).

### 6.6 The plan-document flow (ADR-008/011 sync invariants)

The dashboard keeps `self._doc` (always valid JSON — unknown top-level keys
included) and `self.steps` (its `test_sequences` projection). Sync is
**asymmetric, one direction at a time**:

```
 STACK tab ──mutate──▶ projection written IN PLACE into _doc
                       (unknown keys preserved · total_duration_ms fill)
                       ──▶ re-render AFTER flushing any pending parse
 JSON tab  ──keystroke──▶ debounce 0.3 s ──▶ parse_plan_doc_text
                       (on the quote_bare_auto-REPAIRED text)
                       ├── valid  → adopt verbatim into _doc → re-project steps
                       └── invalid → red status cell; _doc AND steps UNTOUCHED
                                     (the tab is never rewritten mid-edit)

 ↺ resync (on either tab): adopt THE tab it is pressed on,
   discard the other side's un-adopted edit
 S: Gate 2 over the projection → write _doc (the engineer's verbatim
    pre_conditions / test_metadata / generationStatus ride through)
    + coverage → ALWAYS re-render the tab (discards a broken tab;
    save-while-broken = last valid document + warning)
 M/R: read the projection — a broken tab can never put a frame on the bus
```

### 6.7 The AI-proposal flow (one shape, two pipelines)

```
 user text in chat strip ──▶ LLM (chat role, streamed tokens)
   │  $step / $generate prefix (CAN) · $step (UDS) turns the reply into
   │  a JSON proposal envelope
   ▼
 extract_proposal / extract_proposal_can (tolerant JSON-out-of-prose)
   │  envelope: steps · insert_after · rationale
   ▼
 splice onto a COPY of the current stack ──▶ provisional_gate (Gate PG)
   │  (UDS: session/security state of the spliced copy via
   │   state_at_position; CAN: validate_plan_steps over the spliced plan)
   ▼
 dashed AI review row(s) (✎ Edit → centered modal on the step JSON)
   │  Accept → the step enters the real stack (→ projection write, 6.6)
   │  Edit   → accepted as edited      Reject → dropped, nothing written
   ▼
 (a $generate full plan replaces the stack only on a gate-clean confirm)
```

The contract the LLM sees is **rendered from the engine, not hand-written**:
`render_step_contract` (UDS) / `render_step_contract_can` (CAN) generate
from `KNOWN_STEP_TYPES` + `REQUIRED_STEP_PARAMS` and are drift-checked
against them — the prompt and the gate cannot diverge. The prompt carries
the artifact (never the truth layer) + the live state at each stack
position, so a proposal can only reference what the document contains.

### 6.8 Evidence flow (what a run leaves behind)

| Evidence | Producer | Consumer (today / future) |
|---|---|---|
| `.uds.result.json` (schema v1) | `canforge.py` `_write_macro_result` / `uds_core.run_*_inprocess` — one shape | dashboards (status + anchor verdict) / Step-4 regression [FUTURE] |
| `.uds.coverage.yaml` | dashboard S (DIDs written+read back, DTCs read/cleared) | engineer review / Step 4 |
| `.uds.trace.txt` / `.can.trace.txt` | one shared `_trace_lines` builder per pipeline (Copy / Save-trace / E use the same) | engineer review / regression diffing |
| `.can.plan.json` + `.can.coverage.yaml` | S gesture (Gate 2) or C (its write half) | the CLI chain (`json-to-asc`), Step 4/5 [FUTURE] |
| `<unit>.testcase.yaml` `execution` block | `update_tool_owned` re-assertion (6.4) | ASPICE review / traceability export |
| `data/audit_log.jsonl` | `AuditLogger` on every CLI step | the ASPICE trail (FR-UTIL-06); append-only, never pruned |

Lineage rule: every artifact records its inputs (plan carries source
md/dbc lineage + versioned saves via `_record_lineage` / `_next_version_path`;
the anchor lists the tool-owned artifact paths; the result carries the
profile + timing sources that shaped the run). A reviewer can walk from any
frame back to the plan, the plan back to the md/DBC (or C sources), without
guessing.

---

## 7. Cross-Cutting Concerns

Concerns that cut across every component. Each is a rule the rebuild must
keep, not a per-module feature.

### 7.1 Error handling and the exit-code model

Architecture-side of the SRS §8 policy (the table of *which* code for
*which* command lives there):

- **Named errors, no silent fallbacks (NFR-04).** Every failure mode has a
  name: `TestcaseExistsError`, `TreeSitterUnavailable`, `BudgetOverflow`,
  `IsotpFlowControlError`, `DtcParseError`, `GateViolation`, unknown
  backend/algorithm = named lookup error. A wrong value is *rejected and
  said*, never substituted — an unresolvable algorithm is a named error
  (ADR-005), a deferred DTC subfunction raises rather than guessing
  (ADR-002), a malformed 0x54 group echo is "malformed", not "ack"
  (ADR-002).
- **Fail-fast before expensive side effects.** Gate V runs before any
  hardware opens (exit 2); the evidence budget overflows before any LLM
  call (exit 2); live-run connect failure is `fail` before any frame.
  Code-2 paths write nothing (SRS §8 rule 1); code-1 paths still write
  their evidence (a failed run is evidence).
- **Two exception philosophies, one boundary.** *Gestures* that mutate
  engineer-visible state are never-raising and degrade loudly instead
  (the anchor writer: broken file → warn + intact; missing → no-op).
  *Pipeline* paths raise named errors that the composition root maps to
  exit codes. The boundary is the CLI: it is the only place exceptions
  become process codes, and the TUIs are the only place they become
  status-cell text.
- **One gate-report shape (FR-GATE-18).** Gate A–C, G1–G3, V and SG all
  return/emit through the shared report builder, so a reviewer (human or
  repair-round LLM) reads one format everywhere — the repair loop's
  `{errors}` placeholder and the terminal report are the same rendering.

### 7.2 Encoding and text recovery

Three distinct text worlds, one rule each:

| World | Cascade | Recovery |
|---|---|---|
| `.dbc` (often GBK/GB2312 from Chinese OEM toolchains) | UTF-8 → GBK → GB2312 → Latin-1 | `fix_mojibake()` repairs double-encoded comment text after the cascade picks the codec |
| C sources (Step 1) | the scanner's `read_with_encoding_cascade` (same family) | preprocessor lines are **never rewritten** during macro expansion — the evidence bundle keeps `#` lines verbatim |
| Terminal (Windows cp1252 consoles are the norm here) | console codepage detection | the spinner degrades Braille → ASCII → plain timestamped lines when piped (NFR-08); TUI output assumes a UTF-8 VT terminal and the TUIs are the only component that require one |

Rule: **no component ever corrupts text it does not own** — a DBC that
fails the whole cascade still loads as Latin-1 with a warning (parsing
proceeds; mojibake is repaired where detectable), and markdown artifacts
are always written UTF-8.

### 7.3 Audit and traceability (the ASPICE trail)

- `data/audit_log.jsonl` (append-only, never pruned, never committed):
  every CLI step logs timestamp, command, inputs, outputs, exit code,
  duration via `AuditLogger` (FR-UTIL-06). In-process dashboard gestures
  are covered by the same trail *indirectly* — they write the same
  artifacts and re-assert the same anchor the CLI would.
- **Source-of-value is first-class data** (NFR-09): the timing cascade
  records per-field sources into the result; the result names the profile;
  the plan carries md/dbc lineage; the anchor lists the tool-owned paths.
  "Where did this run's value come from?" is answerable from the evidence
  alone, offline.
- The audit trail + result schema + coverage artifacts are the inputs the
  [FUTURE] regression steps (CAN Step 5, UDS Step 4) will consume — their
  shape was fixed for that purpose (SRS §9).

### 7.4 AI integration (one seam, four disciplines)

- **One module.** All Ollama traffic goes through `llm_handler`
  (SRS §4.4): single-shot, chat (content + token usage), streamed chat
  (`on_token` — the TUI strips), keep-alive, translation. Servers/models
  resolve per role (chat vs knowledge, §6.3); the knowledge role falls
  back to the chat role, so a single-box deployment works unchanged.
- **Tolerant out, strict in.** LLM output is extracted tolerantly
  (fence/prose stripping, `extract_json*`, proposal envelopes) and then
  held to the gate standard — extraction leniency and validation strictness
  are deliberately separated.
- **Contracts rendered, not written** (6.7): step contracts and schemas
  reach the prompt *generated from code* (`render_step_contract*`,
  `schema_json_text()`) and are drift-checked — the prompt cannot promise
  what the engine doesn't do, and the gate cannot reject what the prompt
  forbade.
- **Budget is a hard limit** (NFR-13): `ai.context_tokens` sizes both the
  request `num_ctx` and the evidence bundle budget; overflow is a fail-fast
  exit 2, not a truncated prompt.
- **AI is never in a gate** (§5.2): no gate calls the LLM; the LLM calls
  through gates. The repair loop is the only place AI sees gate errors,
  and only through the shared report format.

### 7.5 Console, host, and process model

- **Three process topologies** (§3.3, restated as a concern): the host
  runs CANForge as a subprocess and reads *only* exit code + captured
  output — so every CLI path must be console-safe (no TUI, no interactive
  prompt on the piped path; the spinner goes plain when piped); the CLI
  runs engines in its own process; the TUIs run engines in-process and
  spawn exactly one child (the C gesture's ASC dashboard,
  `CREATE_NEW_CONSOLE` — a new console because the TUI owns its own).
- **No shared state across processes.** Everything that must survive a
  process boundary is a file on the canonical layout (SRS §2.4.10) —
  artifacts, anchor, result, config. There are no sockets, no locks, no
  daemon: mosquitto-style side processes belong to the Nausica host, not
  to CANForge.
- **Interactive REPL is the one TTY dependency of the CLI** (chat mode's
  `"""` multi-line input, Step 2 chat); it is a mode the user invokes,
  never something a pipe or the host hits.

### 7.6 Configuration, profiles, and secrets

- The cascade of §3.5/§6.3 is the *only* configuration mechanism; there is
  no second config surface (no registry keys, no hidden files beyond
  `.nausica.yaml` at the host boundary).
- Profiles are plain YAML under `profile/` (committed, reviewable); the
  `profile/` tree + `config.yaml` + `plugin.yaml` are the entire deployed
  configuration (SRS §6.15).
- **No secrets in the tool.** Ollama is reached by IP on a trusted LAN —
  no API keys, tokens or certificates anywhere in the config surface; a
  rebuild that introduces secrets must change this section.
- The audit log (`data/`) is the only write outside the user's project
  tree, and it is excluded from version control.

### 7.7 Hardware-optional testability

- The `mock` backend is a **first-class citizen**, not a test stub: it is
  registered through the same factory as production backends, it
  implements the same `CanBackend` ABC, and the dashboards expose it as a
  user-facing mode (M) — so the entire closed loop (ISO-TP, UDS, gates,
  security, DTC, timing, FC) is exercised end-to-end **without hardware**
  in CI.
- The **leaf discipline** (D1/D6) is what makes the unit-test surface
  small: `uds_timing`, `session_gate`, `dtc`, `nrc`, `did_layout`,
  `testcase`, `e2e_computation`, `llm_handler`, `canforge_io` import
  nothing from the project, so each is testable in isolation; the handler
  split (D2) means a CAN test can never drag in a UDS mock.
- The **event stream** (6.2) is the seam that makes the TUIs testable:
  Textual pilot tests drive the shells against `run_*_inprocess` with the
  mock and assert on rendered state, with no engine change (SRS NFR-15, §9).

---

## 8. Deployment View

### 8.1 Deployment modes

CANForge deploys in exactly two modes, and the difference is only the
interpreter:

| Mode | Interpreter | How it is launched | Where it lives |
|---|---|---|---|
| **Standalone** (development / direct use) | a dedicated `venv` with `requirements.txt` installed | `python canforge.py …`, `python uds_dashboard.py …`, `python can_dashboard.py …`, `python asc_dashboard.py …` | the `Dependency_Tools/CANForge/` working copy |
| **Host-integrated** (Nausica) | **`sys.executable` of the running Nausica** — the host invokes every CLI as a subprocess of its own interpreter | Nausica menu items (`plugin.yaml`), Popen with `CREATE_NEW_CONSOLE` for the dashboard/Flowline launches | the `Dependency_Tools/CANForge/` directory **inside the Nausica tree** (the host resolves `script_path` relative to itself) |

The host-integrated mode carries one hard constraint: **every Python
dependency CANForge imports must be importable in the Nausica
interpreter** — the subprocess inherits `sys.executable`, not a venv. In
the shipped Nausica bundle that means the bundle's locked environment
(`build_requirements.txt`) must carry CANForge's third-party imports
(cantools, ruamel.yaml, textual, …); the standalone venv is for
development, not for the shipped product. The one deployment fact a
rebuild must preserve: CANForge ships **no executable of its own** — it
is script + assets, and its "installer" is the Nausica bundle.

### 8.2 Runtime dependencies

| Dependency | Kind | Required for | Notes |
|---|---|---|---|
| Python 3 (the host interpreter, or the venv) | runtime | everything | same interpreter rule as §8.1 |
| `cantools` | library | CAN pipeline (DBC parse, encode), Gate G2 | the DBC truth layer's parser |
| `textual` (+ `rich`) | library | the three TUIs | CLI and engines run without it |
| `ruamel.yaml` | library | the anchor lifecycle (round-trip comments), profile loading | stdlib `yaml` used elsewhere (config) — the anchor writer specifically needs the comment-preserving one |
| `tree-sitter` + `tree-sitter-c` | library | Step 1 scanner | absence degrades to a named `TreeSitterUnavailable` → exit 2, never a crash (SRS §8) |
| `requests` | library | Ollama HTTP | no other network I/O |
| `ollama` | library | convenience client | all traffic still funnels through `llm_handler` |
| Ollama server(s) | external service | every AI face | one box per role (`ai.server` chat; `ai.knowledge_server` knowledge, falling back to the chat box); reachable by LAN IP, no auth; warm with `keep-model-alive` before first use |
| `libTSCAN.dll` (+ `libTSCANDemos/Python/native/`) | native lib | the `tosun_libtscan` backend only | shipped inside the tool tree, loaded at runtime via ctypes; absent ⇒ only `mock` (and the TSMaster skeleton) are usable backends — every other feature is unaffected |
| TOSUN USB-CAN hardware | device | live runs (R), playback (play-asc), recording | `scan()`-discovered; live runs preflight with a device scan and fail before any frame without one |

### 8.3 Filesystem layout (deployed)

```
CANForge/                      ← read-only ASSETS (committed):
  canforge.py · uds_dashboard.py · can_dashboard.py · asc_dashboard.py
  uds_core.py · can_core.py
  handler/ hardware/ utils/
  config.yaml                  ← defaults (installed values, user may edit)
  profile/e2e/*.yaml · profile/uds/*.yaml
  prompts/*.txt · json/template_v1.txt
  plugin.yaml                  ← the host manifest
  libTSCANDemos/…              ← the native DLL
  tests/dbc-to-md/ · tests/plan_from_md/   ← committed fixtures
  doc/ADR/ · doc/SRS.md · doc/SAD.md       ← this blueprint
  data/audit_log.jsonl         ← RUNTIME (the only tool-tree write; never committed)

<project>/                     ← user data (everything else lands here):
  <unit>/
    <unit>.testcase.yaml       ← anchor (engineer + tool owned)
    DBC_Description/<unit>.can.md (+ _detail)
    TestPlan/<unit>.can.plan.json (+ .can.coverage.yaml)
    TestPlan/output/<unit>.asc
    <unit>.uds.yaml · <unit>.uds.macro.yaml · .uds.result.json
    …(coverage/trace side artifacts, §6.8)
```

Rule: **the tool tree is read-only at runtime** — every user-facing write
goes into the project tree on the canonical layout (SRS §2.4.10), plus the
one audit file. There are no temp files, no caches, no state files outside
`data/`.

### 8.4 External services and network

- **One service, one protocol, no auth:** Ollama over HTTP on a trusted
  LAN, addressed by IP in `config.yaml` / env (§6.3). No tokens, no
  certificates, no ports opened by the tool itself (§7.6).
- **No daemons, no listeners.** mosquitto/ffmpeg side processes belong to
  the Nausica host's other tools, not to CANForge (§7.5).
- **The only outbound connections** are (a) Ollama and (b) the USB/ctypes
  path to the CAN hardware — which is a device handle, not a network
  connection.

### 8.5 Environment (deployment knobs)

| Variable | Sets | Resolves in |
|---|---|---|
| `CAN_BACKEND_IP` / `CAN_MODEL_NAME` | chat-role Ollama server/model | CLI → env → `config.yaml` (§6.3) |
| `CAN_KNOWLEDGE_SERVER` / `CAN_KNOWLEDGE_MODEL` | knowledge role (Step 1) | same cascade, with chat-role fallback |
| `CANFORGE_E2E_PROFILE` | the E2E profile (ADR-025) | CLI flag > env > `.nausica.yaml` > none |
| `CANFORGE_CLOSEDLOOP_PROFILE` | the UDS security profile | the closedloop cascade (§6.3) |
| `QT_QPA_PLATFORM=offscreen` (test-only) | headless Qt for host-side regression | tests, not deployment |

The host-side per-project overrides (`tool_config.canforge.e2e_profile` in
`.nausica.yaml`, found by upward search) are the **project** deployment
layer — they travel with the project, not the tool.

### 8.6 Platform assumptions

- **Windows-first, by construction**: `CREATE_NEW_CONSOLE` gestures,
  `os.startfile`-style host behaviors, cp1252 console degradation (NFR-08),
  the DLL/ctypes hardware path. The Python core is portable, but the
  deployment contract is Windows (SRS §2.3).
- **A real terminal is required for the TUIs** (UTF-8 VT); the CLI is
  pipe-safe (§7.5).
- **SVN working copy** for the source tree (release tooling assumes it);
  `data/` and build output are the standing exclusions.
- Versioning: CANForge carries no independent release — it ships inside
  the Nausica bundle (the host's `version.py` / changelog / `release.ps1`
  SOP), and its `plugin.yaml` `version:` is the manifest's own field.

### 8.7 Deployment checklist (a rebuild is "deployed" when)

1. `pip install -r requirements.txt` (standalone) — or the host bundle's
   lock carries the same imports (§8.1).
2. `config.yaml` points at a reachable Ollama box; `keep-model-alive`
   succeeds for both roles.
3. A sample DBC round-trips: `dbc-to-md` → `plan-from-md --mode single`
   → `json-to-asc` — all with exit 0 and gate reports PASS.
4. The mock loop runs hardware-free: `automate --mode mock` on a sample
   macro → `.uds.result.json` written, all_passed.
5. With a TOSUN device connected, `list-devices` names it.
6. In Nausica: the 16 menu items appear (plugin status healthy), and the
   "Test Case: UDS/CAN" folder flow creates an anchor + launches the
   dashboard in a new console.
7. `data/audit_log.jsonl` has entries; the project tree holds only the
   canonical-layout artifacts.

---

## 9. Test Architecture

### 9.1 The suite and its fixtures

All tests live in **`Test/`** (pytest; `conftest.py` patches `sys.path` to
the project root so imports work exactly as `canforge.py` does at runtime —
no package install, no `PYTHONPATH`). As of this writing the suite is in
the high hundreds of passing tests (last full run: 927 passed, 2 skipped),
hardware-free: live-mode tests run the real engine against the mock
backend, never against a device.

| Fixture source | What it is | Used by |
|---|---|---|
| `tests/dbc-to-md/*.dbc` — real DBCs shipped in the repo (`ehv_hud.dbc`, `T1E_Uzbek.dbc` — the latter a GBK/mojibake case) | the CAN **truth layer** for integration tests | every CAN test; encoding cascade, gates, encode, offline/live runs |
| `tests/plan_from_md/` | example approved plans (single-shot + chat transcripts) | the AI-extraction and refine tests |
| `prompts/` + `json/template_v1.txt` | the real prompt/template assets (imported by path, not copied) | `ai_reasoner` tests — the suite tests against the **shipped** assets |
| `Test/goldens/` | committed golden outputs (`no_raw_golden.can.plan.json`, `no_raw_golden.asc`) | the parity tests (9.4) — a no-raw plan must byte-produce the golden ASC |
| `Test/testPlan.md` | the original coverage plan (per-module priority/risk table) | the design record for what "covered" means here |
| the mock backend (§7.7) | the stand-in ECU + hardware | every UDS engine/security/DTC/timing/FC test and both `*_live_run` suites |

### 9.2 Test layers (bottom-up, mirroring the architecture layers)

1. **Leaf units** (D1 modules — no project imports, no mocks needed):
   `test_uds_timing` (cascade math + sources), `test_uds_session_gate`
   (state walk + `check_gate` decisions), `test_uds_dtc` (Annex C + 0x59
   parse, incl. the deferred-subfunction errors), `test_security_algorithms`
   (seed/key + the registry/level catalog), plus the audit logger and
   `llm_handler` (mocked `requests`/`ollama.Client` — the only layer that
   mocks the network).
2. **Handler integration** (real DBCs, no LLM where avoidable):
   `test_dbc_to_markdown` (structure + the encoding cascade +
   `fix_mojibake` on crafted byte sequences), `test_plan_validator`
   (G1/G2/G3 incl. the deterministic fill), `test_json_to_asc`
   (**byte-level ASC verification** — the critical encoding path),
   `test_isotp_fc` / `test_isotp_tx_fc` (ADR-006 over a scripted peer),
   `test_ai_reasoner` (`_extract_json` against noisy LLM-output variants,
   prompt assembly, gated save).
3. **Engine over the mock** (closed loop, hardware-free):
   `test_uds_negative` (ADR-043 parity: engine rules vs mock rules),
   `test_uds_frame_edit` (ADR-044 raw-override transmission),
   `test_uds_did_layout` (ADR-001 encode/readback), `test_uds_dtc` engine
   cases, `test_uds_security` (ADR-005 per-NRC 0x27 semantics against the
   rejectable-key mock), `test_uds_timing` run cases (cascade in the result
   record), `test_uds_live_run` / `test_can_live_run` (the live-mode
   engine path: connect-failure-before-frame, counted TX errors, real-time
   cadence on the mock).
4. **TUI pilot tests** (Textual, offscreen): the `*_ui` / `*_editor` /
   `*_frame_editor` / `test_uds_dashboard_drain` / `test_can_dashboard_ui`
   families drive the real shells against `run_*_inprocess` (mock) and
   assert on rendered state — the event stream (6.2) is the seam that makes
   this possible without engine changes.
5. **Unit-anchor tests**: `test_uds_testcase` / `test_can_testcase`
   (ADR-012: init refusal, the never-raises writer, engineer-comment
   survival, the **anchor-corruption fixture** → warning emitted, file
   untouched, verdict mappers).
6. **Host-side regressions** (Nausica root `Test/`, not CANForge's):
   `test_can_flowline_launch.py` (all four launch points resolve the E2E
   profile identically — ADR-025), `test_can_testcase_launch.py` (the
   "Test Case: UDS/CAN" directory flow: preflight refusal → init →
   dashboard Popen — ADR-012), and `test_structured_artifact.py`'s
   ADR-042 branch (the `name_suffix_not` exclusion must not cross-gate
   CANForge's `.can.md` files).

### 9.3 Coverage mapping (component → suites)

| Component (§4) | Primary suites |
|---|---|
| `canforge.py` CLI (exit codes, dispatch) | exercised through the handler suites + the host-side launch regressions |
| `handler/can/*` | `test_dbc_to_markdown`, `test_plan_validator`, `test_json_to_asc`, `test_ai_reasoner`, `test_can_offline_parity`, `test_can_e2e_autoload`, `test_can_raw_frame` |
| `handler/uds/*` engine + security | `test_uds_negative`, `test_uds_frame_edit`, `test_uds_did_layout`, `test_uds_dtc`, `test_uds_security`, `test_uds_timing`, `test_uds_session_gate`, `test_security_algorithms`, `test_uds_live_run` |
| `handler/uds/extract/*` | **gap** — see 9.5 |
| `isotp.py` / `uds.py` | `test_isotp_fc`, `test_isotp_tx_fc` (+ the engine suites that run them against the mock) |
| `canforge_io/*` (mock) | every engine suite (it is the test double); `test_uds_dashboard_drain` |
| `uds_core.py` / `can_core.py` | the `*_ui` families + `test_can_core` (document truth, resync, bare-`auto`, Gate 2) |
| the three shells | `test_uds_dtc_ui`, `test_uds_timing_ui`, `test_uds_session_gate_ui`, `test_uds_security_ui`, `test_uds_did_layout_ui`, `test_uds_frame_editor`, `test_isotp_tx_fc_ui`, `test_can_dashboard_ui` |
| `utils/testcase.py` | `test_uds_testcase`, `test_can_testcase` |
| `utils/*` (llm, audit) | `test_llm_handler`, `test_audit_logger` |

### 9.4 Parity discipline (NFR-01's verification arm)

- **Golden pinning:** `Test/goldens/` holds the committed outputs of the
  legacy paths. `test_can_offline_parity` (and the golden ASC) assert that
  the same no-raw plan byte-produces the same ASC — a face that changes
  default output fails the suite.
- **Per-face parity:** each ADR-001…006 / ADR-043 / ADR-044 face ships
  with the pair "absent ⇒ legacy output" and "present ⇒ new behavior"
  (e.g. mock without `negative_rules` emits the legacy canned responses;
  `PlanStep` round-trip without `data` is byte-identical).
- **Binary correctness first:** the encode path is verified byte-by-byte
  against expected values — the test plan's own words: "silently wrong CAN
  frames" is the risk the whole suite exists to kill.
- **The mock is the parity oracle for ECU behavior:** the tester-side gate
  and the mock's ECU-side rules are asserted *against each other* (where
  they can disagree, the test asserts the discrepancy is reported, not
  hidden — §5.7).
- **Contract drift checks:** the AI step contract is asserted to match
  `KNOWN_STEP_TYPES` / `REQUIRED_STEP_PARAMS` (rendered-from-code, 6.7);
  the schema prompt is asserted to match the gate's accepted shape
  (one schema, prompt and gate agree).

### 9.5 Known gaps (rebuild must not widen them)

- **Step-1 extraction has no dedicated suite yet** (`c_scanner`,
  `evidence`, `knowledge_validator`, `extract_knowledge` are exercised
  indirectly at best). The rebuild should add one: fixture C sources
  (committed) + stubbed LLM responses → scanner determinism, budget
  overflow (exit 2), Gate A/B/C decision tables, the repair loop
  (≤3, no-progress exit), and the rejected-artifact path. This is the
  largest uncovered deterministic surface in the tool.
- **The TSMaster skeleton** is registered but its frame path raises
  `NotImplementedError` — untestable by design until implemented [out of
  scope, SRS §9.1].
- **RX capture** in the CAN live run is off in v1 (SRS §9) — the trace's
  RX side for CAN live runs is therefore unexercised.
- **Real-hardware behavior** (TOSUN timing quirkiness, FC edge cases from
  real ECUs) is only covered by the mock's modeling of it — the 2 skipped
  tests mark the device-dependent paths.

### 9.6 The rebuild gate

A component is **done** when: its suites pass, its parity check (9.4)
passes, and no gate's report format changed without the shared-builder
assertions still holding. A face is **shippable** only with its
absent-⇒-legacy test in the same commit. The suite runs hardware-free on
CI (offscreen where Textual is involved) — it must stay that way.

---

## 10. Traceability and Known Debt

### 10.1 ADR → design → evidence matrix (local series)

| ADR [local] | Design home (§4) | SRS refs | Test home (§9) |
|---|---|---|---|
| ADR-001 DID layout + readback | `did_layout.py`, `uds_core.get_did_layout`, `automation` values/readback, mock DID store | §6.8, FR-UDS-06/19 | `test_uds_did_layout(_ui)`, `test_uds_frame_editor` |
| ADR-002 DTC services | `dtc.py`, `uds.py` 0x19/0x14, `automation` read/clear, `mock_dtc_config`, artifact `"19"`/`"14"` | §6.8, FR-UDS-10/15/16 | `test_uds_dtc(_ui)` |
| ADR-003 dynamic timing | `uds_timing.py`, 0x50 parse, NRC-78 poll, `IsoTpSession` ctor, `mock timing=` | §6.8, FR-UDS-20/25 | `test_uds_timing(_ui)`, `test_isotp_fc` |
| ADR-004 session gating | `session_gate.py`, pre-flight + `enforce_session_gates`, mock ECU-side rules | §6.8, FR-GATE-12/13 | `test_uds_session_gate(_ui)` |
| ADR-005 pluggable security | `security/` (registry/levels/algorithms), per-NRC 0x27, `nrc.py`, mock rejectable-key | §6.8/§6.15, FR-UDS-21/26 | `test_security_algorithms`, `test_uds_security(_ui)` |
| ADR-006 ISO-TP TX flow control | `isotp.py` TX path, `stmin_ms`, mock `tx_fc` receiver | FR-UDS-18, NFR-01 | `test_isotp_tx_fc(_ui)` |
| ADR-007 Flowline CAN dashboard | `can_dashboard.py` shell, `can_core.py` | §3.4, FR-FCAN-01…18 | `test_can_dashboard_ui`, `test_can_core` |
| ADR-008 two-tab Plan Workbench | `can_core` document/projection pair, shell tabs | FR-FCAN-05…09 | `test_can_core`, `test_can_dashboard_ui` |
| ADR-009 live knowledge filter | `can_core.filter_knowledge`, `SearchInput` | FR-FCAN-03 | `test_can_core`, `test_can_dashboard_ui` |
| ADR-010 Compile and Run (C) | C gesture (shell) → Gate 3 in-process → `asc_dashboard` Popen | FR-FCAN-16, FR-ASC-01 | `test_can_dashboard_ui`, host launch regression |
| ADR-011 resync + bare-`auto` | `quote_bare_auto`, `parse_plan_doc_text`, tab toolbars | FR-FCAN-09 | `test_can_core` (idempotence, value-positions-only) |
| ADR-012 test-case unit anchor | `utils/testcase.py`, `_anchor_update` in both cores, CLI init, host directory items | §2.4.10, §6.12, FR-UNIT-01…06 | `test_uds_testcase`, `test_can_testcase`, host `test_can_testcase_launch` |

**Cross-series (Nausica root `ArchitectureDecisionRecord/` — full text there):**
ADR-025 (E2E profiles — the `--e2e-profile` face on all three CAN commands
+ Flowline, resolved identically at all four launch points), ADR-027 (ASC
playback dashboard), ADR-030 (UDS Step-1 knowledge extraction), ADR-036
(UDS Step-2 Flowline dashboard), ADR-037 (the `$step` AI chat strip),
ADR-043 (negative response testing — `expect_nrc` + derived ✗ cases),
ADR-044 (step-stack frame editing — `raw` overrides), plus the remaining
UDS closed-loop entries of the 030/031/036/037/043–045 range (see those
documents). Commit messages for the local series use the qualifier
`[CANForge ADR_NNN]`.

### 10.2 Requirement → design coverage

The SRS §3/§4/§7 requirement sets map to §4 components through §4.8 (FR-CAN
→ 4.3.1; FR-UDS → 4.3.2/4.3.3/4.4/4.5.1; FR-FUDS → 4.1.2+4.2.1; FR-FCAN →
4.1.3+4.2.2; FR-ASC → 4.1.4; FR-UNIT → 4.6 + gesture wiring; FR-UTIL → 4.6;
FR-GATE → the gate owners named in §4.3/§4.4). NFR-01…17 are carried by:
the parity discipline (NFR-01, §5.1/§9.4), the leaf rules (NFR-02/16,
§3.2), truth-layer separation (NFR-03, §5.2), named errors (NFR-04, §7.1),
the gate lattice (NFR-05…07, §5.2/SRS §7), console degradation (NFR-08,
§7.5), recorded sources (NFR-09, §6.3), canonical layout (NFR-10,
SRS §2.4.10), extension points (NFR-11, §5.12), evidence shape (NFR-12,
§6.8), hard budgets (NFR-13, §7.4), the result schema (NFR-14, §6.8),
TUI-testability (NFR-15, §7.7), and the canonical builders (NFR-16, §4.2).
No SRS requirement is unowned.

### 10.3 Known debt register

| # | Debt | State | Impact / scheduled action |
|---|---|---|---|
| D1 | **`canforge_tosun/exceptions.py` imported by the L4 protocol stack** (`isotp.py`/`uds.py`) — a forward dependency from the modern stack into the legacy bindings, grandfathered and pinned by the dependency test (§3.2) | registered | Target: a project-leaf exception module (`handler/uds/` or `utils/`) that `canforge_tosun` re-exports; then the legacy dir is importable-by-no-one-who-matters |
| D2 | **`canforge_tosun/closedloop.py`** — dead code, superseded by `isotp.py` + `uds.py`; zero imports | registered | Removal scheduled, not urgent; the parity tests reference its *absence*, not its behavior |
| D3 | **`tosun_tsmaster.py` skeleton** — backend registered, frame path `NotImplementedError` | out of scope (SRS §9.1) | Implement only when a TSMaster-SDK deployment is required; the factory seam (§5.12) means no other file changes |
| D4 | **CAN live-run RX capture off (v1)** — the live trace's RX side is unexercised | [FUTURE] (SRS §9) | The backend ABC already has `receive()` + callbacks; enabling it is a scheduler change, not an architecture change |
| D5 | **Step-1 extraction untested in CI** (no dedicated suite — §9.5) | gap | The largest uncovered deterministic surface; the rebuild adds fixture C sources + stubbed LLM (scanner, budget, Gates A/B/C, repair loop, rejected path) |
| D6 | **Two copied TUI shells** (UDS/CAN) — deliberate duplication (D3 of §3.2), so shared-widget fixes must be applied twice | accepted cost | Revisit only if a third shell is needed — at which point the copy rule is the wrong rule |
| D7 | **Deferred DTC subfunctions** (0x07, 0x0C/0x0D raise `DtcParseError`) | by design (ADR-002) | Extend `dtc.py`'s one-branch-per-subfunction table when an ECU requires them; the step fails cleanly until then |
| D8 | **0x11/0x71 reset ack does not update `LiveSessionState`** | by design (ADR-004) | Staleness errs permissive, never stricter; tighten only with an ECU that misbehaves the other way |
| D9 | **GBK mojibake repair is best-effort** (`fix_mojibake`) | accepted | The cascade never corrupts (§7.2); undecodable comment text survives as mojibake with a warning, not a failure |
| D10 | **`NRC_MESSAGES` deprecated alias** in `security/algorithms.py` | back-compat shim | New code names NRCs from `nrc.NRC_NAMES`; the alias stays until the last importer is gone |

**Explicit non-debts** (look like debt, are decisions): the shell
duplication (D6 above — accepted, with an exit strategy), the mock's
"deliberately excluded 0x10" (OEM custom sessions, e.g. 0x61, must keep
passing), the permissive reset-ack (D8), and the deferred subfunctions
(D7) — each has an ADR row and a test asserting the current behavior.

### 10.4 Invariants a rebuild must keep (closing checklist)

1. Absent face ⇒ byte-identical legacy output (NFR-01) — golden-pinned.
2. Truth never parses artifact; artifact validated by pure gates; AI only
   as proposal (§5.2).
3. One engine per pipeline; surfaces render, never re-implement (§5.3).
4. Every frame's bytes come from the canonical builder for its pipeline
   (NFR-16); every NRC named from one table; every gate reported through
   one builder.
5. Exit-code policy intact: code 2 writes nothing, code 1 writes evidence,
   code 3 (Step 1 only) writes nothing (SRS §8).
6. The anchor's ownership split and never-raises writer (ADR-012).
7. The event-stream contract (6.2) — the TUI seam.
8. The cascade discipline with recorded sources (6.3).
9. Hardware-free testability (mock as first-class citizen; suite stays
   offscreen/CI-runnable).
10. The dependency rules D1–D7 with their single registered exception
    (§3.2), pinned by the dependency test.

*End of SAD. Companion document: `doc/SRS.md` (requirements, interfaces,
data schemas, gates, exit codes). Together they are the complete
rebuild blueprint: the SRS says what must be true, the SAD says how it is
built and why.*
