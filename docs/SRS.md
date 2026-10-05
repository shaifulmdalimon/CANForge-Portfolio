# CANForge — Software Requirements Specification (SRS)

| Field | Value |
|---|---|
| **Document status** | Draft — baseline for a ground-up rebuild |
| **Date** | 2026-10-04 |
| **Scope baseline** | CANForge as it exists today: CANForge-local ADR-001…012, CAN Flowline phases P1–P14, UDS Steps 1–3, full closed-loop stack, Nausica plugin integration |
| **Companion document** | `doc/SAD.md` (Software Architecture Document) |

> **Reading note.** This SRS specifies the system **as it is**, captured from the
> current codebase, so that it can be rebuilt from scratch without loss of
> behavior. Requirements are stated in *shall* form. Features that are explicitly
> pending in the current tool (CAN Step 4/5, UDS Step 4) are marked **[FUTURE]**
> and are out of scope for the initial rebuild milestone.

---

## 1. Introduction

### 1.1 Purpose

This document is the software requirements baseline for **CANForge**, a CLI tool
that forges executable CAN test artifacts from `.dbc` database files through a
disciplined, AI-assisted, human-verified pipeline, and a closed-loop UDS
automation engine for live diagnostic services over real CAN hardware:

```
DBC → Markdown (AI-readable spec) → JSON Test Plan (AI-generated) → ASC Log (executable CAN traffic)
```

Its companion purpose is UDS automation:

```
ECU C sources → .uds.yaml (knowledge) → .uds.macro.yaml (macro) → AutomationEngine → CanBackend → ISO-TP → UDS → ECU → .uds.result.json
```

The document is written so that an engineer who has never seen the existing code
can implement a **functionally equivalent system** from it, together with
`doc/SAD.md` (architecture) and the ADR series in `doc/ADR/` (decision rationale).

**Intended audience:** implementation engineers rebuilding or extending the tool,
test leads validating behavior against this baseline, and ASPICE reviewers
tracing requirements to the pipeline's gate and evidence artifacts.

### 1.2 Scope

CANForge shall provide:

1. **Open-loop (CAN) pipeline** — deterministic DBC→Markdown conversion, AI
   test-plan generation (single-shot and interactive chat), deterministic
   JSON→ASC compilation, ASC playback and live-bus recording on TOSUN USB-to-CAN
   hardware, plus a Textual TUI ("Flowline") that builds and rehearses CAN plans
   interactively.
2. **Closed-loop (UDS) pipeline** — a knowledge-extraction step that turns the
   ECU's diagnostic C sources into a schema-validated, citation-checked
   `.uds.yaml`; a Textual TUI (Flowline) that builds UDS macros against real or
   simulated ECUs with a live, artifact-annotated packet trace; and a macro
   execution engine (`automate`) with dry-run / mock / live modes that writes
   machine-readable evidence (`.uds.result.json`).
3. **Test-case units (ADR-012)** — the right-clicked folder *is* the ASPICE
   test case: a `test-case init` command plus a `<unit>.testcase.yaml` anchor
   whose tool-owned keys are re-asserted after every tool gesture, with
   engineer-owned fields the tool never touches.
4. **Host integration** — a Nausica plugin manifest (`plugin.yaml`) exposing all
   pipeline gestures as context-menu items, launched through Nausica's plugin
   engine and `method`/`subprocess` handlers.
5. **Profiles** — pluggable, file-based E2E-protection profiles
   (`profile/e2e/`) and UDS security profiles (`profile/uds/`) that extend the
   tool for a new ECU or OEM **without editing CANForge source**.

**Out of scope for the initial rebuild milestone (marked [FUTURE] where they
appear in later sections):**

- CAN Step 4 "Play + Monitor" (RX verification, timing windows, bus-error
  accounting) — playback exists (Step 3.5) but bus-level *monitoring* is pending.
- CAN Step 5 and UDS Step 4 regression (baseline diffing of result artifacts,
  coverage reporting against a regression baseline).
- ARXML / ODX database parsing (the pipeline lists such files but parses DBC only).
- The TSMaster SDK backend (a registered skeleton exists; its frame path is not
  implemented — `tosun_libtscan` and `mock` are the functional backends).

### 1.3 Definitions, Acronyms, and Abbreviations

| Term | Definition |
|---|---|
| **ASC** | Vector ASCII log format (verbose, TSMaster-compatible) for CAN traffic. |
| **ASC dashboard** | `asc_dashboard.py` — Textual TUI that plays back an ASC file with frame table, trends and optional bus TX. |
| **Artifact** | Any file a pipeline stage produces and the next stage consumes (`.can.md`, `.can.plan.json`, `.asc`, `.uds.yaml`, `.uds.macro.yaml`, `.uds.result.json`, …). |
| **AutomationEngine** | The UDS macro executor (Step 3 runner): validates a macro, opens a backend, dispatches one handler per step type, writes `.uds.result.json`. |
| **Backend** | A `CanBackend` implementation: `tosun_libtscan` (real TOSUN hardware via libTSCAN.dll), `mock` (in-process UDS ECU simulator), `tosun_tsmaster` (skeleton). |
| **CAN Flowline** | `can_dashboard.py` — the CAN twin of UDS Flowline: a red-bordered Textual TUI that builds a `.can.plan.json` from a `.can.md` + its DBC. |
| **Cycle time** | The periodicity of a CAN message (`GenMsgCycleTime`); `cycle_time_ms: 0` in a plan step means a single-shot transmission. |
| **DID** | Diagnostic Identifier (ISO 14229 data identifier, e.g. `0xF190`). |
| **DTC** | Diagnostic Trouble Code. |
| **E2E** | AUTOSAR end-to-end protection (alive counter + CRC) applied to selected CAN signals. |
| **ECU** | Electronic Control Unit — the diagnostic target in UDS mode. |
| **Evidence bundle** | The token-budgeted, citation-annotated extract of the ECU's C sources handed to the knowledge LLM (`.uds.evidence_bundle.md`). |
| **Flowline (UDS)** | `uds_dashboard.py` — Textual TUI that builds a `.uds.macro.yaml` from a `.uds.yaml` knowledge artifact and runs it in-process (mock or live). |
| **Gate** | A mandatory, deterministic quality checkpoint between pipeline stages (Gates 1/2/3 for CAN; Gates A/B/C for UDS knowledge; `validate_macro` for macros). |
| **ISO-TP** | ISO 15765-2 transport protocol (single/first/consecutive frames + flow control) carrying UDS over CAN. |
| **Knowledge artifact** | `.uds.yaml` — the schema-validated, citation-checked machine-readable description of an ECU's UDS implementation. |
| **Macro** | `.uds.macro.yaml` — an ordered list of UDS steps with optional per-step verification faces. |
| **NRC** | Negative Response Code (ISO 14229, `7F <SID> <NRC>`). |
| **Plan** | `.can.plan.json` — the approved JSON description of CAN traffic to generate. |
| **Profile (E2E / UDS)** | A YAML file under `profile/e2e/` or `profile/uds/` parameterizing the tool for one ECU/OEM. |
| **Raw override** | The ADR-044/A8 additive `data` face: a step transmits the engineer's exact bytes instead of the derived ones (marked ⚡ in the UI). |
| **Seed / key** | The security-access value pair (0x27) used to unlock an ECU security level. |
| **Session gate** | ADR-004: the session-precondition state machine that warns (pre-flight) or refuses (opt-in in-engine) steps that require a session/security state not yet established. |
| **Step** | One executable unit of a plan (`test_sequences[]` entry) or macro (`steps[]` entry). |
| **Test-case unit (ADR-012)** | One folder holding one DBC (CAN) or one `.uds.yaml` (UDS); the folder *is* the ASPICE test case, anchored by `<unit>.testcase.yaml`. |
| **Timing cascade** | ADR-003: the four-source resolution order for UDS timing fields (CLI flags → macro block → artifact `transport.isotp` → profile `timeout:` → engine defaults) with per-field source tracking. |
| **TUI** | Terminal User Interface (Textual framework). |
| **TX / RX** | Transmit / receive (from the tester's perspective). |
| **UDS** | Unified Diagnostic Services (ISO 14229). |

**Acronyms:** ALM (Application Lifecycle Management), ASC (see above), CAN
(Controller Area Network), CRC (Cyclic Redundancy Check), DBC (CAN database),
DID/DTC/NRC (see above), FC (Flow Control), GBK/GB2312 (Chinese character
encodings), LLM (Large Language Model), NFR/FR (non-/functional requirement),
OEM (Original Equipment Manufacturer), P2/P2\* (ISO 14229 server timing
parameters), RX/TX (see above), SF/FF/CF (ISO-TP single/first/consecutive
frame), TUI (see above), UDS (see above).

### 1.4 References

| # | Document | Role |
|---|---|---|
| [R1] | `WhatIsCANForge.md` | Pipeline doc of record (CAN Steps 1–5, UDS Steps 1–4, status of each). |
| [R2] | `doc/ADR/ADR_001…ADR_012` (CANForge-local series, `doc/ADR/Template.md` format) | Decision records for the closed-loop feature faces, the CAN Flowline, the Plan Workbench, the unit anchor. |
| [R3] | Nausica-root `ArchitectureDecisionRecord/` — ADR-025 (E2E profiles), ADR-027 (ASC dashboard), ADR-030 (knowledge extraction), ADR-036/037 (UDS dashboard + AI chat), ADR-043 (negative testing), ADR-044 (frame editing), ADR-045 (test-case block + notes) | Cross-series ADRs whose decisions are implemented in this tool. |
| [R4] | `cli_command_structure.md` | CLI reference (subcommands, macro contract, result schema). |
| [R5] | `CANForge_CAN_upgrade_workflow.md` | CAN Flowline phase log P1–P14 (behavior baseline for the TUI). |
| [R6] | `doc/analysis/UDS_Gap_Analysis_v2.md` | UDS capability audit (27 implemented / 0 partial / 0 absent as of 2026-09-17). |
| [R7] | ISO 15765-2 (road vehicles — diagnostic communication over ISO-TP), ISO 14229-1 (road vehicles — UDS, incl. Annex C DTC status bytes), AUTOSAR E2E profiles, SAE J1850 CRC | External standards the protocol stack and E2E computation conform to. |
| [R8] | `json/template_v1.txt` | The CAN plan JSON schema (single source of truth for the plan document shape). |
| [R9] | `setupBackend.md` | Ollama backend setup (model roles, Modelfiles, `num_ctx`). |
| [R10] | `doc/SAD.md` | Companion architecture document. |

### 1.5 Overview

- **Section 2** — overall description: product perspective, users, environment, constraints, assumptions.
- **Section 3** — functional requirements, organized by pipeline (open-loop CAN, closed-loop UDS, the two TUI dashboards, test-case units, utilities), each requirement carrying a stable ID (`FR-…`).
- **Section 4** — interface requirements: the complete CLI contract (arguments, defaults, exit codes), artifact interface, Nausica host interface, AI-backend interface, hardware interface, TUI interface.
- **Section 5** — non-functional requirements (the default-off parity invariant, determinism, encoding robustness, hardware-free operability, …).
- **Section 6** — data requirements: the full artifact catalog with per-format structure.
- **Section 7** — gate requirements (the quality checkpoints and their decision semantics).
- **Section 8** — exit-code conventions (per command).
- **Section 9** — [FUTURE] items and explicit non-goals.

Requirement IDs are stable: a rebuild implementation and its test suite trace to
them; the SAD's component catalog references them per component.

---

## 2. Overall Description

### 2.1 Product Perspective

CANForge is a **standalone Python CLI tool** that additionally exposes a plugin
manifest to the **Nausica** ALM desktop application (PySide2, Windows). Nausica
is the *host surface*: it discovers `Dependency_Tools/CANForge/plugin.yaml`,
builds the right-click context menu, and launches CANForge either as
`subprocess.run([sys.executable, "canforge.py", …])` (pipeline commands) or as a
new console window running a dashboard (`uds_dashboard.py`, `can_dashboard.py`,
`asc_dashboard.py`). CANForge never imports Nausica code and never requires it —
every capability is usable from a plain terminal. Nausica drives it only via
subprocess argv and the per-project `.nausica.yaml` configuration file.

The product's center of gravity is **discipline, not generation**:

- **Every pipeline stage is gated.** AI output is never the source of truth —
  it is a gated proposal. The DBC (CAN) and the ECU's own C sources (UDS) are
  the truth layers; AI artifacts must survive deterministic validation against
  them before anything executable is produced.
- **Every run is evidenced.** Each gesture writes machine-readable evidence
  (`.can.result.json`, `.uds.result.json`, coverage files, traces) and appends
  to `data/audit_log.jsonl`, so an ASPICE review can answer *what ran, with
  which inputs, and what it produced*.
- **Every hardware-free path is a first-class citizen.** `dry-run`, the `mock`
  backend, and offline rehearsal exist so the whole closed loop can be exercised
  without a TOSUN device, a bus, or an ECU.

Within the product, three execution surfaces share one engine each:

| Surface | Entry | Engine it drives |
|---|---|---|
| CLI pipeline | `canforge.py <subcommand>` | the handler modules directly |
| UDS Flowline TUI | `uds_dashboard.py <file.uds.yaml>` | `uds_core.run_mock_inprocess` / `run_live_inprocess` + `AutomationEngine` in-process |
| CAN Flowline TUI | `can_dashboard.py <file.can.md>` | `can_core.run_offline` / `run_live` + `json_to_asc` in-process |
| ASC dashboard TUI | `asc_dashboard.py <file.asc>` | ASC playback (frame table, trends, optional bus TX) |

The TUI surfaces and the CLI produce **byte-identical artifacts** from the same
inputs — the dashboards are not a second, divergent implementation; they are
alternative front doors onto the same handlers (see SAD §5, "single canonical
builders").

### 2.2 User Characteristics

| User | Expertise | Interaction | Primary needs |
|---|---|---|---|
| **Automotive/embedded test engineer (ASPICE)** | Fluent in CAN/UDS, DBC files, ISO 14229; works in Nausica daily | Nausica right-click menu; opens Flowline dashboards in a terminal; verifies AI output by eye at each gate | Gated, evidenced, human-verifiable pipeline; no silent guessing; artifacts that are the ASPICE test-case record (the unit folder + anchor). |
| **Diagnostics engineer** | UDS protocol, ECU C source, seed/key algorithms | Step 1 extraction from source trees; Step 2 macro building in Flowline; Step 3 `automate` on bench hardware | Protocol fidelity (ISO-TP FC, NRC78, session gating, DTC status bits); mock runs that match live semantics; per-NRC security behavior; timing negotiation visibility. |
| **Tool maintainer / extension author** | Reads CANForge source; extends profiles, not code | Writes `profile/e2e/*.yaml` / `profile/uds/*.yaml`; registers algorithms/levels at runtime where a profile is insufficient | A new ECU/OEM must be onboardable without source edits: register algorithm + level + profile; new CAN hardware via `register_backend`. |
| **CI / regression runner [FUTURE-facing]** | Automated execution | Headless CLI; Textual TUI driven by pilot in tests | Deterministic, hardware-free, exit-code-driven runs; artifact schemas stable enough to diff. |

Common user expectations across all personas (each restated as an NFR in §5):
*see the gate that stopped me and why*, *see where a value came from* (timing
cascade sources, evidence citations), *never lose hand-edited content* (the
Plan Workbench document, macro YAML comments, anchor engineer fields), and *a
broken state is visible, never silently repaired or silently discarded*.

### 2.3 Operating Environment

| Aspect | Requirement |
|---|---|
| **Language / runtime** | CPython 3.10+ (project standard; Nausica bundles a 3.10 venv for releases). No C extensions required by the tool itself. |
| **Primary platform** | Windows 11 (TOSUN libTSCAN is a Windows DLL; Nausica host is Windows). Console codepage is frequently **cp1252** — all console output must degrade gracefully (NFR-8). |
| **Secondary platform** | Linux (Ollama servers run on Linux boxes; the knowledge-extraction and mock paths are exercised there). |
| **Python dependencies** | `cantools` (DBC/ASC), `requests` (Ollama HTTP), `ollama`, `pyyaml`, `ruamel.yaml` (comment-preserving anchor round-trips), `tree-sitter` ≥ 0.23 + `tree-sitter-c` (Step 1), `textual` + `rich` (TUIs), `pyperclip` (TUI copy buttons). libTSCAN native libraries ship in `libTSCANDemos/Python/native/` and load via ctypes — **no pip install**. |
| **AI backends** | One or two **Ollama** servers, one per model role: `ai.server`/`ai.model` (chat role) and `ai.knowledge_server`/`ai.knowledge_model` (Step 1 knowledge role; falls back to the chat role when absent). Defaults in `config.yaml`; overridable per role by env (`CAN_BACKEND_IP`/`CAN_MODEL_NAME`, `CAN_KNOWLEDGE_SERVER`/`CAN_KNOWLEDGE_MODEL`) and per invocation by CLI flags. `ai.context_tokens` is sent to Ollama as request-level `options.num_ctx` and drives the evidence-bundle token budget. |
| **Hardware** | TOSUN USB-to-CAN adapter (libTSCAN) for live CAN/UDS; optional CAN-FD; **no hardware is required** for: all deterministic steps, `dry-run`, `mock`, offline rehearsal, and the full test suite. |
| **Host** | Nausica (optional). When present, CANForge reads the per-project `.nausica.yaml` (e.g. `tool_config.canforge.e2e_profile`) via an upward directory search from the right-clicked file; when absent, defaults and CLI flags apply. |
| **Network** | Plain HTTP to the Ollama servers on the LAN; nothing else. No telemetry, no external services. |
| **Filesystem** | The project working tree plus per-unit folders on disk (the unit layout, §6). `data/` holds runtime state (audit log) and is never part of the source drop. |

### 2.4 Design and Implementation Constraints

These constraints are **invariants** — a rebuild that violates one is not
equivalent:

1. **Default-off parity (the project-wide invariant).** Every optional feature
   face added by ADR-001…006 (DID `layout`/`values`/`readback`, DTC
   `verify`/`expect_nrc`/`raw`, `timing:` block, `enforce_session_gates`,
   mock `negative_rules`/`dtc_config`/`timing=`/`tx_fc=`/`security_rules`) and
   every additive plan face (the raw `data` override) is **optional and
   default-off**: with the face absent, behavior shall be *byte-identical* to
   the legacy pipeline. (NFR-1)
2. **DBC-wins (CAN).** The `.dbc` is the single truth layer for the CAN
   pipeline; the `.can.md` is a *display* layer. Any drift between them is a
   **warning, never a failure**, and encoding always uses the DBC. (NFR-3)
3. **Human-in-the-loop at every AI boundary.** AI-generated plans are
   `under_review` until a human verifies and renames/approves them; an AI chat
   step enters the stack only after explicit Accept (or Edit→Accept);
   `validate_macro` refuses to open hardware on an invalid macro (exit 2). (NFR-5)
4. **No silent guessing.** Unknowns shall surface as named errors, warnings, or
   `open_questions` — e.g. DTC subfunctions 0x07/0x0C/0x0D raise
   `DtcParseError` rather than a best-effort decode; a level whose declared
   algorithm is unregistered is a validation error, not a fallback; an
   unresolvable seed/key algorithm yields a named NRC (0x31), never a guessed key.
   (NFR-4)
5. **Python-only, file-based.** No database, no daemon, no build step beyond
   `pip install -r requirements.txt`; all state lives in well-named artifacts
   next to the unit. Native code is limited to loading vendor DLLs via ctypes.
6. **Two-phase markdown.** DBC→Markdown produces *two* files: a low-token
   `description` (fed to the planning LLM — "what to test?") and a full
   bit-level `detail` (for encoding/validation — "how to test?"). Only the
   description is ever fed to the LLM.
7. **Encoding recovery.** DBC files (and C sources) may be GBK/GB2312; the
   loader shall try UTF-8 → GBK → GB2312 → Latin-1 and repair mojibake
   (`fix_mojibake`), because vendor DBCs from Chinese OEMs routinely carry
   corrupted comments. (NFR-7)
8. **Console degradation.** Long-running CLI phases show a Claude-Code-style
   animated status line: Braille on UTF-8/VT-capable consoles, an ASCII
   fallback on cp1252, plain timestamped lines when output is piped. No crash,
   no garbled glyphs, on any of the three. (NFR-8)
9. **Test-case unit identity (ADR-012).** The *folder* is the test case. The
   tool writes only the tool-owned keys of the anchor and **never** touches
   engineer-owned fields (`title`, `status`, `requirement_refs`, text fields,
   `owner`, `execution.defect_refs`); `status` is never tool-transitioned. The
   anchor round-trip must preserve the engineer's comments (ruamel.yaml, not
   `yaml.safe_dump`), and the anchor update **never raises** (missing/unparseable
   → skip, file left intact; no anchor → silent no-op).
10. **Artifact naming follows the unit.** Generated artifacts are named after
    the **unit folder**, not the source file (a re-versioned vendor DBC must not
    rename the unit's artifacts). Canonical layout:
    ```
    <unit>/
    ├── <vendor>.dbc                     # keeps its vendor name
    ├── <unit>.testcase.yaml             # ADR-012 anchor
    ├── DBC_Description/<unit>.can.md            # knowledge (description)
    ├── DBC_Description/<unit>_detail.can.md     # full spec (detail)
    ├── TestPlan/<unit>.can.plan.json            # plan (auto-increments TestPlan_2, … on collision)
    ├── TestPlan/output/<unit>.asc               # compiled output
    └── .can.coverage.yaml / .can.trace.txt / .can.result.json
    ```
    UDS units mirror this: `<unit>.uds.yaml`, `.uds.macro.yaml`, `.uds.result.json`,
    `.uds.coverage.yaml`, `.uds.trace.txt`, `.uds.evidence_bundle.md` next to it.
11. **Standalone operation.** The tool shall run with Nausica absent and no
    `.nausica.yaml` present — every host-facing behavior (profile selection,
    E2E resolution) has a non-host fallback.
12. **AI-offline resilience.** Every deterministic step shall run with no
    network at all; AI steps degrade to a clean, non-zero, *named* failure
    (Step 1: exit 3 "LLM unreachable"), never a hang or a partial artifact
    written as success. (NFR-10)

### 2.5 Assumptions and Dependencies

**Assumptions:**

- The operator supplies DBC files that `cantools` can parse (with the encoding
  cascade of constraint 8 covering the common GBK case), and for Step 1, C
  sources that constitute a UDS implementation (a non-UDS source tree yields
  `generationStatus: rejected`, which is a *correct* outcome, not an error).
- The Ollama servers are reachable on the LAN and run the configured models;
  model quality affects proposal usefulness but never correctness, because all
  AI output passes deterministic gates before it can affect hardware.
- The engineer reviews artifacts at the gates (plan approval, macro Accept,
  result inspection); the tool is designed for *assistive* automation and does
  not claim to certify tests by itself.
- TOSUN hardware, when used, is the `tosun_libtscan` target with the shipped
  native libraries; CAN-FD is available but off by default.

**Dependencies (external):**

| Dependency | Kind | Failure behavior |
|---|---|---|
| Ollama chat server (`ai.server`) | AI backend for plan generation/refine/translate and the dashboard AI chat | Named failure; deterministic steps unaffected; Step 1 → exit 3 |
| Ollama knowledge server (`ai.knowledge_server`) | AI backend for Step 1 (falls back to the chat server when absent) | Same as above |
| TOSUN libTSCAN + adapter | Live CAN I/O | Live mode refuses with a device-scan preflight failure before any frame; mock/offline unaffected |
| `cantools` | DBC parse + signal encode | Hard dependency for the CAN pipeline |
| `tree-sitter` / `tree-sitter-c` | Step 1 pre-pass | Absence → exit 2 (no deterministic pre-pass is possible), with a named message |
| Nausica + plugin engine | Host surface only | Optional; all features reachable from the terminal |
| Python 3.10+ + `requirements.txt` set | Runtime | Preflight at install; the release build locks the set |

**Internal dependency discipline** (reiterated here because §2.4 makes it a
rebuild requirement; full form in SAD §3.2): leaf modules that encode a protocol
fact (`uds_timing`, `session_gate`, `dtc`, `nrc`, `did_layout`, `testcase`,
`e2e_computation`, the `canforge_io` backend layer) import **nothing** from the
project; `handler/can` and `handler/uds` do not import each other; the TUI
shells are **copied, never imported** from one another, so UDS protocol
semantics can never leak into the CAN tool.

---

## 3. Functional Requirements

Requirement-ID map (traceability from later sections and the test suite):

| ID prefix | Area |
|---|---|
| `FR-CAN-*` | Open-loop CAN pipeline: Step 1 dbc→md, Step 2 plan, Step 3 compile, Step 3.5 play/record |
| `FR-UDS-*` | Closed-loop UDS: Step 1 knowledge extraction, one-shot UDS commands, Step 3 `automate`, protocol stack, mock |
| `FR-FUDS-*` | UDS Flowline TUI (`uds_dashboard.py`) |
| `FR-FCAN-*` | CAN Flowline TUI (`can_dashboard.py`) |
| `FR-ASC-*` | ASC playback dashboard (`asc_dashboard.py`) |
| `FR-UNIT-*` | Test-case units and the anchor (ADR-012) |
| `FR-UTIL-*` | Cross-cutting utilities: profiles, audit, config, translation, model care |

### 3.1 Open-loop (CAN) pipeline

#### 3.1.1 Step 1 — DBC to Markdown (`dbc-to-md`)

- **FR-CAN-01.** The tool shall parse a `.dbc` file deterministically and produce
  **two** Markdown artifacts: a low-token **description** ("what to test?" — the
  only one ever fed to the planning LLM) and a full bit-level **detail**
  ("how to test?" — signal tables with bit offsets, encoding, bounds). The same
  input + profile shall always yield byte-identical output (no timestamps, no
  map-order leakage).
- **FR-CAN-02.** Canonical output paths (constraint §2.4.10):
  `<unit>/DBC_Description/<unit>.can.md` and
  `<unit>/DBC_Description/<unit>_detail.can.md`, where `<unit>` is the folder
  the DBC sits in (the unit of test; if the DBC is kept beside its md inside a
  `DBC_Description/` folder, the unit is the parent). Explicit
  `--output-desc` / `--output-detail` override the canonical paths. Output
  directories are created as needed.
- **FR-CAN-03.** The DBC loader shall attempt UTF-8 → GBK → GB2312 → Latin-1 and
  apply mojibake repair to message comments, so vendor DBCs with corrupted
  Chinese comments still produce readable description text.
- **FR-CAN-04.** The description artifact shall start with the line
  `# CAN Test Scope: \`<dbc filename>\`` (the Step-1 signature the host checks)
  and contain, per message sorted by frame ID, a section
  `## <Message> (0x<HEX>)` with: Sender(s), Cycle Time, Description, Category
  (inferred from message name — diagnostic / tracer / reserved / functional) and
  Testability, and a **Signals** bullet list (`- \`sig\`: min–max [choices] —
  description`).
- **FR-CAN-05.** Cycle time shall be read from the DBC message attributes
  (`GenMsgCycleTime`, falling back to `CycleTime`).
- **FR-CAN-06.** **Gate 1 (DBC coverage)** shall run after generation: every
  message and every signal of the DBC must appear in both artifacts; the gate
  returns its error list (empty = PASS). Exit 0 on PASS, exit 1 on gate
  failure.
- **FR-CAN-07.** With `--e2e-profile <name>` active, the description artifact
  shall carry the E2E configuration for that profile (profile name, counter/CRC
  detection mode, byte placement) and the profile's `ai_prompt` instructions
  shall be injected into Step 2/plan prompts so the plan can assign `"auto"`
  value tokens to protected signals.
- **FR-CAN-08.** The detail artifact shall contain the complete per-signal
  table (name, start bit, length, byte order, sign, raw min/max, factor,
  offset, unit, choices, description) sufficient for encoding and for human
  bit-level review without opening the DBC.

#### 3.1.2 Step 2 — Markdown to JSON Test Plan (`plan-from-md`, `refine-plan`)

- **FR-CAN-09.** `plan-from-md --mode single` shall assemble the prompt from
  `prompts/test_plan_prompt.txt` with placeholders `{user_prompt}`,
  `{can_db_md}` (the **description** artifact only) and `{json_template}`
  (`json/template_v1.txt`), call the chat LLM, and extract the JSON plan from
  the response (tolerant of code fences and surrounding prose).
- **FR-CAN-10.** `plan-from-md --mode chat` shall provide an interactive REPL:
  system prompt from `prompts/test_plan_chat_prompt.txt` enforcing that **all
  signals in a message are explicitly assigned** (no omitted signals);
  multi-line input via `"""` delimiters; `save` exports the JSON plan; `exit`
  quits.
- **FR-CAN-11.** When the LLM's JSON is malformed or fails Gate 2, the tool
  shall run a bounded number of **auto-repair rounds** (sending the gate errors
  back to the LLM) before reporting failure; repair rounds are logged.
- **FR-CAN-12.** The generated plan shall be written as an *under review*
  artifact (default `TestPlan/draft_plan.json`; `--output` overrides).
  Approval is a human act (rename/accept) — the tool never self-approves
  (constraint §2.4.3).
- **FR-CAN-13.** **Gate 2 (plan vs DBC)** shall reject unknown messages and
  signals, reject values outside the DBC min/max (enum *numbers* within range
  are accepted), check units where declared, and apply the deterministic
  `complete_missing_signals` repair: because the encoder requires **all**
  signals of a message to be present, missing signals are filled with
  DBC-derived defaults and the repair is reported, never silent.
- **FR-CAN-14.** `"auto"` value tokens shall be permitted only for E2E-protected
  signals when a profile is active; they are computed at **encode time**
  (FR-CAN-19), not at plan time.
- **FR-CAN-15.** `refine-plan --json <plan>` shall refine an existing JSON plan
  via AI chat in `--mode quick` (plan context only) or `--mode full` (full DBC
  context), writing a versioned next-plan path and recording lineage
  (source md / source dbc in `test_metadata`).
- **FR-CAN-16.** The plan document shall conform to `json/template_v1.txt`:
  `generationStatus{status,reason}`, `test_metadata{name,dbc_file,source_md,
  source_dbc,total_duration_ms}`, `pre_conditions[]`, and `test_sequences[]`
  (`step_name`, `start_time_ms`, `end_time_ms`, `message`, `can{id, id_type}`,
  `cycle_time_ms` — `0` = single shot — `signals{...}`). Plan field notes:
  **one message, one transmitter; same-message steps strictly sequential**.
  Unknown top-level keys must survive tool round-trips untouched.

#### 3.1.3 Step 3 — JSON Plan to ASC (`json-to-asc`)

- **FR-CAN-17.** The tool shall encode an approved plan + DBC into a verbose,
  TSMaster-compatible ASC log: each step's signal values are packed via
  `cantools` (all signals of the message required — no DBC-initial-value
  fallback at encode time); `cycle_time_ms > 0` repeats the frame at the cycle
  interval; frames are ordered by `start_time_ms` with a stable sort for tied
  starts; the resulting ASC must self-parse.
- **FR-CAN-18.** **Gate 3** shall (a) re-run Gate 2 over the plan before
  encoding and (b) validate the generated ASC (`validate_asc_file`: frames
  parse, CAN IDs and DLCs consistent with the DBC). Failures are reported with
  a gate report; the tool exits non-zero and does not present a failed file as
  a product.
- **FR-CAN-19.** When a profile is active, E2E fields bearing `"auto"` shall be
  computed at encode time per frame: alive counter (width, start value,
  increment per frame) + CRC (`sa_e_j1850`, `ones_complement_sum`, or
  `ones_complement_sum16` per profile, over the profile's `input_bytes`),
  byte placement taken from the DBC; counter and CRC signals are identified by
  the profile's detection mode (name-suffix regex) or explicit overrides.
- **FR-CAN-20.** The additive raw **`data`** override (default-off,
  constraint §2.4.1) shall, when a step carries `data` (a hex byte list),
  transmit exactly those bytes as the frame payload in place of the encoded
  one; the DLC check (Gate 2) applies; steps without `data` are encoded
  exactly as before (byte-identical parity).
- **FR-CAN-21.** The default output path for a plan at
  `<plan dir>/<unit>.can.plan.json` shall be
  `<plan dir>/output/<unit>.asc` (`asc_path_for`), overridable by `--output`.

#### 3.1.4 Step 3.5 — Playback and Recording (`play-asc`, `record-asc`, `list-devices`)

- **FR-CAN-22.** `play-asc --asc <file>` shall play an ASC log on live CAN
  hardware with `--speed` (time factor), `--repeats` (loop count) and
  `--sync-tx` (synchronous transmit); channel/baudrate/serial/CAN-FD
  parameters come from CLI flags, defaulting to `config.yaml` `hardware:`.
- **FR-CAN-23.** `record-asc --output <file.asc>` shall record live bus
  traffic to ASC with `--duration` (seconds; 0 = until interrupted),
  `--include-tx` (capture TX echoes) and `--append` (append to an existing
  file).
- **FR-CAN-24.** `list-devices` shall scan the backend and report connected
  CAN devices (serial, channel, availability) — used by the host and by the
  dashboards' device-scan preflight.
- **FR-CAN-25 [FUTURE].** Step 4 "Play + Monitor" — RX verification against
  expected frames, per-message timing windows, bus-error/error-frame
  accounting — is pending and out of scope for the initial rebuild.

### 3.2 Closed-loop (UDS) pipeline

#### 3.2.1 Step 1 — Knowledge Extraction (`uds-extract-knowledge`)

- **FR-UDS-01.** The tool shall run a deterministic tree-sitter C pre-pass over
  `--input <dir>`: discover `.c`/`.h` files (skip-list pruning;
  `--include-dirs` replaces the skip list), build the `#define` macro table
  (chained, cycle-safe), collect enums, **switch and if/else-if dispatchers**
  with per-case value text, session-state sites, NRC sites, and function
  signatures.
- **FR-UDS-02.** Files shall be scored for UDS relevance and the top
  `--max-files` (default 24) shall be carried into the evidence bundle.
- **FR-UDS-03.** The **evidence bundle** shall be token-budgeted (budget
  derived from `ai.context_tokens`), with per-file dispatcher caps, >300-line
  span shaping, and contained-span dedup; macros are rewritten in evidence
  lines as `ident → 0x11 /* NAME (file:line) */` (preprocessor lines are never
  touched) so every cited value is traceable to source. The bundle manifest is
  retained for Gates B/C, and `<stem>.evidence_bundle.md` is always written
  next to the output for human review.
- **FR-UDS-04.** A budget overflow shall fail fast **before** any LLM call
  (exit 2, named message) — no partial work, no silently truncated evidence.
- **FR-UDS-05.** LLM normalization shall use the **knowledge model role**
  (`ai.knowledge_server`/`ai.knowledge_model`, falling back to the chat role
  when absent; per-call overrides via `--input-model` /
  `--backend-server-ip`). The system prompt (`prompts/uds_knowledge_system.txt`)
  embeds the schema (`{json_schema}`), mandates `source: file:line` citations
  per entity, enforces ISO 14229 subfunction parity (odd = requestSeed,
  even = sendKey), and instructs `generationStatus: rejected` when the code is
  not a UDS implementation. The user message
  (`prompts/uds_knowledge_user.txt`) renders the bundle: `{source_dir}`,
  `{manifest}`, `{macro_table}`, `{switches}`, `{session_sites}`, `{nrc_sites}`,
  `{function_signatures}`, `{xml_note}`.
- **FR-UDS-06.** **Gate A (schema)** shall accept `schema_version` 1 or 2 and
  verify: required top level (`schema_version`, `transport`, `sessions`,
  `services`, `nrcs`, `open_questions`, `generationStatus`); the canonical
  ISO 14229 SID name table (with the ADR-002 corrections 0x15 =
  `ControlDTCSetting`, 0x19 = `ReadDTCInformation`); per-DID `layout`
  structure (big-endian tile without overlap, span ≤ `size_bytes` — a bad
  layout is a **hard error**, never a warning); the `"19"` DTC service
  (subfunctions ⊆ {01,02,04,05,06,0A,0B}, `dtc_length` 2|3, DTC width, no
  duplicates) and the `"14"` `groups`. Null `seed_sub`/`key_sub` on
  SecurityAccess are review **warnings**, not errors.
- **FR-UDS-07.** **Gate B (citation audit)** shall verify every
  `source: file:line` against the scanned tree. In strip mode (`strip=True`),
  uncited entities are **removed** and the removal noted in
  `open_questions`; a pass-with-strip still writes a draft and exits 0.
- **FR-UDS-08.** **Gate C (switch completeness)** shall compare extracted
  dispatchers against the artifact's subfunction/session/NRC tables; unexplained
  case values feed the repair loop, and surviving omissions land in
  `open_questions` (draft still written, exit 0).
- **FR-UDS-09.** The tool shall run up to **3 LLM repair rounds** (repair
  prompt `prompts/uds_knowledge_repair.txt` with gate errors under `{errors}`)
  with no-progress early exit. `generationStatus: rejected` from the LLM is a
  **hard fail**: exit 1 and the artifact is written as
  `<stem>.uds.rejected.yaml`, **never overwriting** an existing good
  `<stem>.uds.yaml`.
- **FR-UDS-10.** On success the tool writes `<stem>.uds.yaml` with
  `status: draft` (lifecycle draft → reviewed is engineer-owned, §6.8) and
  exits per the code table in §8 (0 pass incl. pass-with-strip; 1 gate failed;
  2 no evidence / budget overflow / no tree-sitter; 3 LLM unreachable).
- **FR-UDS-11.** `--dry-run` shall stop before the LLM call and report scan /
  scoring / bundle statistics; `--show-prompt` shall write a
  `<stem>.prompt.txt` transcript of the final prompt; `--security-profile <name>`
  performs an **advisory** cross-check of artifact security levels against a
  profile (mismatches are notes, not failures).
- **FR-UDS-12.** The `"19"` DTC service slot shall carry `subfunctions`,
  `dtc_length`, and `dtcs[]` with a per-DTC `sessions` allow-list and an
  optional mock-seed `status`; the DTC **status-byte layout is NOT in the
  artifact** — it is fixed by ISO 14229-1 Annex C and decoded by the shared
  DTC interpreter (FR-UDS-17).

#### 3.2.2 One-shot UDS commands and the protocol stack

- **FR-UDS-13.** The `uds` command family shall expose one-shot services over
  live hardware: `session [subfunction]` (default = diagnostic default),
  `read-did <did>`, `write-did <did> --data <hex>`, `tester-present`,
  `seed-key --level <n>` (default 1), `routine start|stop <rid>`; addressing via
  `--request-id` (default 0x7E0) / `--response-id` (default 0x7E3) and hardware
  flags (`--backend`, `--channel`, `--baudrate`, `--serial`, `--fd`,
  `--fd-databaudrate`) defaulting to `config.yaml`.
- **FR-UDS-14.** The ISO-TP transport (ISO 15765-2) shall implement
  single/first/consecutive-frame TX and RX with flow control, and — ADR-006 —
  **honour the ECU's Flow Control on TX**: transmit `blocksize` CFs per FC
  (0 = unlimited) before waiting for the re-FC; pace CFs at the ECU's `stMin`
  (0–100 ms direct; 0xF1–0xFA = 100 µs steps); a non-continue FC raises
  `IsotpFlowControlError` **before a single CF**. Absent FC configuration =
  legacy continuous TX (parity).
- **FR-UDS-15.** The UDS service layer (ISO 14229) shall support 0x10, 0x14,
  0x19, 0x22, 0x2E, 0x27, 0x2F, 0x31, 0x3E plus raw requests, and shall
  auto-retry NRC 0x78 (responsePending): per-pending-frame deadline = the
  **current** P2*, total window = `respond_pending_s` (default 5 s).
- **FR-UDS-16.** 0x50 timing negotiation (ADR-003): a positive 0x50's
  `timingParameterAvailability` byte (0x06) + big-endian P2/P2\* ms pairs shall
  update the session's P2/P2\*; a **new** 0x50 first resets both to the
  cascade baseline; out-of-range values (1 ms … 10 s) are ignored.
- **FR-UDS-17.** DTC semantics (ADR-002) shall live in **one** interpreter
  shared by engine, dashboard and mock: `read_dtc` verifies the `59 <sf>`
  envelope and parses per subfunction (one branch each for 0x01/0x02/0x04/
  0x05/0x06/0x0A/0x0B, checking DTCFormatIdentifier and payload shape;
  0x07 and 0x0C/0x0D raise `DtcParseError` → step failure, never a guess);
  `clear_diagnostic_information` verifies the `54 <group>` echo (a 0x54 for a
  **different** group is malformed, not an ack); `verify` (default **true**)
  re-reads `19 01 FF` after a clear so "cleared (assumed)" is never reported
  (`verify: false` exists for asynchronous-clear ECUs).
- **FR-UDS-18.** Session gating (ADR-004) shall provide: a **pre-flight
  warning pass** (always on, warning-only, `expect_nrc` steps skipped) and an
  opt-in **in-engine gate** (`enforce_session_gates: true`) that refuses a step
  **pre-send** when its session/security preconditions are unmet. One shared
  state machine decides (per-value allow-list → bulk DTC offenders →
  `requires_security`), and the one canonical refusal/warning format is used by
  engine refusal, validation warnings, and the dashboard's ⚠ markers. A
  `uds_session` step moves the session and **clears** unlocked security levels;
  with no 0x10 seen, the assumed session is default (0x01).
- **FR-UDS-19.** Security (ADR-005): seed/key resolution shall go through the
  **single dispatch** (algorithm registry; a level's `algorithm` field, falling
  back to the level name — the builtin convention — an absent name is a named
  error, never a silent fallback); levels resolve through the **level catalog**
  (builtins + profile `security_levels` + runtime registrations, per-field
  merge, runtime > profile > builtin). Per-NRC 0x27 semantics:
  **0x33** invalidKey → one fresh-seed retry by default (`max_key_retries`;
  the fresh seed is the only sane retry), exhausted → fail naming the attempt
  count; **0x34** exceededNumberOfAttempts → fail immediately naming the
  recovery (ECU reset 0x11 or OEM delay), never auto-performing it;
  **0x35** requiredTimeDelayNotExpired → fail immediately (the NRC carries no
  duration — a guessed sleep is a lie); **0x31** → fail naming the level's
  `session:` field; other NRCs keep the generic shape with the canonical name.
  The step result carries `attempts` + final NRC + name.
- **FR-UDS-20.** The canonical NRC table (ISO 14229-1, 2nd-edition numbering)
  shall be the single source of NRC names for engine, mock, dashboard trace and
  gates — including 0x33 `securityAccessDenied`, 0x34
  `exceededNumberOfAttempts`, 0x35 `requiredTimeDelayNotExpired` — with
  per-NRC recovery guidance rendered for 0x33/0x34/0x35.

#### 3.2.3 Step 3 — Macro execution (`automate`)

- **FR-UDS-21.** The macro contract: optional `backend`, `request_id`,
  `response_id`, `hardware`, `continue_on_error`, `timing:`,
  `enforce_session_gates`, `test_case` block (ADR-045) + a **non-empty**
  `steps[]`; each step has a `type` from the 19 known types (`uds_session`,
  `uds_read_did`, `uds_write_did`, `uds_read_dtc`, `uds_clear_dtc`,
  `uds_tester_present`, `uds_ecu_reset`, `uds_request_seed`, `uds_send_key`,
  `uds_unlock_security`, `uds_io_control`, `uds_routine_start`,
  `uds_routine_stop`, `uds_raw`, `send_can`, `wait_message`, `sleep`,
  `start_keepalive`, `stop_keepalive`) with its required params present.
  Optional per-step faces (all default-off): `expect_nrc` (ADR-043 — a
  matching `7F` is the **pass** condition), `raw` (ADR-044 — exact bytes,
  range-checked), `values` + `readback: true` (ADR-001 — field-encoded write,
  read-back verified, never bypassing the session gates), `note` (ADR-045).
- **FR-UDS-22.** `validate_macro` shall pre-flight the macro **before any
  hardware is opened** (unknown step type, missing required param, out-of-range
  timing block, unresolvable security level/algorithm) and exit 2 on failure.
- **FR-UDS-23.** `automate <macro>` shall support `--mode live` (default, real
  hardware), `mock` (in-process UDS simulator, no CAN hardware needed) and
  `dry-run` (validate and print the step sequence only); `--continue-on-error`
  runs past failing steps; `--result-file` overrides the result path (default:
  next to the macro).
- **FR-UDS-24.** Knowledge resolution: when a sibling `<stem>.uds.yaml` exists
  next to the macro it is loaded as the knowledge artifact; an explicit
  `--knowledge <file.uds.yaml>` **wins** over the sibling rule.
- **FR-UDS-25.** Timing resolution shall follow the four-source cascade with
  first-hit-wins **per field** and per-field source tracking recorded in the
  result: CLI flags (`--p2-ms`, `--p2-ext-ms`, `--respond-pending-s`,
  `--stmin-ms`, `--block-size`, `--tx-dl`) → macro `timing:` block (range-
  checked, exit 2) → artifact `transport.isotp` → profile `timeout:` → engine
  defaults (P2 1.0 s / P2\* 5.0 s / total 5.0 s / stMin 20 ms / blocksize 0 /
  tx_dl 8).
- **FR-UDS-26.** The keepalive steps shall start/stop a background
  tester-present loop (period from the step params) for the duration of the
  macro segment; `send_can` transmits a raw frame, `wait_message` waits for a
  CAN ID within a timeout, `sleep` pauses the run, `uds_raw` sends exact
  request bytes for any SID (OEM services included).
- **FR-UDS-27.** The **mock backend** shall be an in-process, ISO-TP-aware UDS
  ECU simulator with deterministic canned responses and optional, all
  default-off faces: `negative_rules` (ADR-043 — session tracked from 0x10;
  0x22/0x2E outside a DID's allow-list rejected with the configured NRC);
  ISO-14229 conformance NRCs (0x12 for 0x11/0x31/0x3E subfunctions outside
  spec, 0x13 for a 0x2E with no data; 0x10 deliberately excluded so OEM custom
  sessions keep passing); a **stateful per-run DID store** (a 0x2E stores its
  bytes per instance; a 0x22 of a stored DID returns them — making
  `readback: true` a real test; NOT cleared by `connect()`); artifact-seeded
  DTC state (status bytes seeded 0x05 or the entry's `status`; 0x19 served
  from it, 0x14 applied to it; unlisted DTC → NRC 0x12, malformed → 0x13,
  session-gated → the rule NRC); 0x10 timing negotiation (`50 <sub> 0x06 <P2 2B>
  <P2* 2B>`; absent = legacy `50 <sub> 00 00`); strict FC as receiver
  (a re-FC pushed after every `blocksize` CFs; absent = legacy `30 00 00 00`);
  rejectable-key security (wrong key → `7F 27 33` + attempt counter,
  exhausted → `7F 27 34`, unresolvable algorithm → `7F 27 31`); ECU frames
  stamped with the request CAN ID (dashboards restamp to the response ID).
- **FR-UDS-28.** `--security-rules <profile>` (mock mode) shall load a UDS
  security profile's rules so the mock enforces seed/key behavior the same way
  the target ECU would.
- **FR-UDS-29.** The run result shall be written as `.uds.result.json`
  (schema v1, §6.10) with identical shape for CLI and dashboard runs:
  `schema_version`, `timestamp`, `mode`, `macro`, `backend`,
  `closedloop_profile`, `request_id`, `response_id`, step totals,
  `all_passed`, `summary_text`, per-step records (`index`, `step_type`,
  `passed`, `message`, optional `note` (ADR-045), data payload), the
  `test_case` block, `timing` **with per-field sources**, and mock counters.
  Exit 0 = all steps passed (or dry-run); 1 = a step failed; 2 = validation
  failure (before hardware).
- **FR-UDS-30 [FUTURE].** Step 4 — regression against a result baseline and
  coverage reporting — is pending and out of scope for the initial rebuild.

### 3.3 UDS Flowline TUI (`uds_dashboard.py`)

- **FR-FUDS-01.** The dashboard shall be a Textual TUI launched as
  `python uds_dashboard.py <file.uds.yaml> [--profile NAME] [--sim-p2-ms 40]
  [--ai-chat]`, titled "Flowline — UDS macro builder", with a blue
  (`#2563eb`) boundary, and shall run the **real** `AutomationEngine`
  in-process — live (real backend) or offline (mock) — never a separate
  simulation of its logic.
- **FR-FUDS-02.** The **knowledge tree** shall render: Sessions / Security /
  Utilities / DIDs (0x22/0x2E merged view) / IO Control (0x2F) / Routines
  (0x31) / DTCs (0x19/0x14) / OEM (other SIDs) — a branch appears only when the
  artifact carries entries; clicking a leaf inserts the corresponding step into
  the stack; OEM leaves prompt for full request bytes and insert `uds_raw`.
- **FR-FUDS-03.** DID leaves: read leaves insert a 0x22 read; the **write leaf
  of a DID with a layout** prompts one value per layout field and inserts
  `values` + `readback: true`; a DID **without** a layout inserts the legacy
  placeholder write (never invented fields).
- **FR-FUDS-04.** DTC branch: one leaf per artifact DTC (click inserts a
  `19 01 FF` mask-all read), 0x02/0x04 leaves when listed, a clear leaf (the
  artifact's "all" group or a prompted 3-byte group) — always `verify` ON.
- **FR-FUDS-05.** Negative-case derivation (ADR-043): session-gated DIDs grow a
  `✗ negative` sub-branch and a `✗ unknown DID 0xFFFF` leaf; cases are derived
  per-DID/per-session plus session-gated DTC cases from `dtcs[].sessions` —
  the same derivation the mock's `negative_rules` consume (one builder).
- **FR-FUDS-06.** Security leaves shall render catalog state
  (`[engine: x]` / `[algorithm 'x' not registered]` / `[no registered
  algorithm]`); the app joins the profile's `security_levels` into the level
  catalog at startup and clears them on unmount.
- **FR-FUDS-07.** The **step stack** pane shall show every transmittable row
  with a dim `→ <hex>` request suffix built by the **single canonical**
  request-bytes builder (row suffix, frame editor, reset/status all agree);
  the live **YAML preview** pane renders the macro document (hex scalars, `raw`
  as a hex-int list) in real time.
- **FR-FUDS-08.** The **frame editor** (ADR-044): the `▣ frame` button (or
  double-click) on any transmittable row opens a modal byte grid + pastable raw
  line + structured fields, two-way synced on the canonical builder; a
  deviating frame saves as a `raw` override (⚡ marker on the row); Save
  re-gates the live stack; reset restores the derived frame.
- **FR-FUDS-09.** **M** shall run the macro on the mock backend in-process
  through a tracing proxy: every ISO-TP frame streamed as events, ECU frames
  restamped to the response CAN ID, simulated P2 delay (`--sim-p2-ms`), then the
  real engine — writing the CLI's `.uds.result.json` schema. When a sibling
  `.uds.yaml` is present, the negative rules, DTC config (ADR-002), resolved
  timing (ADR-003) and profile security rules (ADR-005) are injected.
- **FR-FUDS-10.** **R** shall run on the real backend with a device-scan
  preflight, the same event-stream contract, and a `mode: live` result; the
  live path subclasses the real backend (TX tap + RX queue, no simulation).
- **FR-FUDS-11.** The **packet trace** pane shall annotate frames from the
  artifact: 0x22/0x2E payloads field-by-field from the DID layout; 0x19/0x14
  decoded per subfunction (DTC list with artifact names + Annex C status bits);
  a `7F` matching the running step's `expect_nrc` renders **green**; NRCs named
  from the canonical table when the artifact doesn't cite them; security NRCs
  0x33/0x34/0x35 grow a recovery-hint line; flow-control frames annotated with
  their parameters.
- **FR-FUDS-12.** **S** shall save the macro (`.uds.macro.yaml`) plus
  `.uds.coverage.yaml` (listing `dids_written_readback`, `dtcs_read` /
  `dtcs_cleared`); **E** / Save-trace shall export `.uds.trace.txt`; Copy
  buttons on the trace and YAML panes use one shared content builder.
- **FR-FUDS-13.** The **AI chat strip** (ADR-037; `--ai-chat` or **C** to
  toggle) shall: keep natural language until a message contains `$step`, which
  turns the reply into a step-proposal envelope (`steps` / `insert_after` /
  `rationale`); gate-validate the proposal on a **spliced copy** of the stack
  (provisional gate) before showing it; render proposals as **dashed AI rows**
  with Accept / Edit / Reject — **no AI step enters the stack without
  acceptance**; never invent a DID, session, or security level the document
  lacks; for `uds_write_did` prefer `values` when the document carries a
  layout, else the placeholder `data` form.
- **FR-FUDS-14.** The chat context shall include the knowledge artifact, the
  live macro state (session/security state at each stack position), and the
  **step contract** rendered from the engine's actual step registry
  (drift-checked: if the rendered contract and the engine disagree, that is a
  tool defect).
- **FR-FUDS-15.** After every **S / M / R / E** gesture the dashboard shall
  re-assert the unit anchor's tool-owned keys per FR-UNIT-03/05 (never raises;
  broken anchor warns and stays intact; no anchor = silent no-op).
- **FR-FUDS-16.** Key bindings: **R** live, **M** mock, **S** save macro +
  coverage, **E** export trace, **C** toggle AI chat, **Q** quit; the step
  stack's validation state (`validate_macro` + artifact-aware warnings, ⚠
  markers from the shared gate formats) is always visible before a run.

### 3.4 CAN Flowline TUI (`can_dashboard.py`)

- **FR-FCAN-01.** The dashboard shall be a Textual TUI launched as
  `python can_dashboard.py <file.can.md> [--ai-chat]` with a **red**
  (`#dc2626`) boundary and four panes: **CAN KNOWLEDGE**, **PLAN WORKBENCH**
  (two tabs), **PACKET TRACE**, **AI CHAT**. Its shell is **copied** from the
  UDS dashboard, never imported — UDS protocol semantics must not enter the
  CAN tool.
- **FR-FCAN-02.** Knowledge loading shall combine the `.can.md` **display
  layer** with the DBC **truth layer** (`find_dbc_for_md`); the DBC wins every
  conflict and drifts are **warnings, never failures**; the pane shows a branch
  per message, a leaf per signal; clicking a leaf inserts a `PlanStep` at
  safe-default values.
- **FR-FCAN-03.** Live search (ADR-009): a one-row search box on top of the
  knowledge pane (**F** focuses, Esc clears and refocuses, every keystroke
  re-projects); matching on message name, CAN ID (`1B3` or `0x1B3`), signal
  name, or signal unit; a **message** match keeps ALL its signals; kept/total
  shown in the pane title. The filter is a pure function (no hidden state).
- **FR-FCAN-04.** The **Plan Workbench** (ADR-008) shall maintain the plan
  document as the single source of truth: `self._doc` (always valid JSON — the
  seeded file parsed verbatim, unknown top-level keys included, else the
  knowledge skeleton) with `self.steps` as its `test_sequences` projection;
  every consumer (gates, encoders, M/R runs) reads the projection.
- **FR-FCAN-05.** Sync shall be **asymmetric, one direction at a time**:
  stack-side mutations write the projection back into the document **in place**
  (unknown top-level keys preserved; `total_duration_ms` fill rule) and
  re-render after flushing any pending parse; JSON-side keystrokes debounce
  0.3 s → parse → adopt verbatim when valid, else a red status cell — the
  document and stack stay **untouched** and the tab text is **never rewritten
  mid-edit**.
- **FR-FCAN-06.** Each tab carries Copy + **↺ resync** (ADR-011): pressing
  resync on a tab adopts that tab and discards the **other** side's
  un-adopted edit (the STEP STACK resync mirrors the JSON tab's). A bare
  `auto` E2E **value token** is adopted via decoder-guided repair
  (`quote_bare_auto` — value positions only, idempotent; the repaired text is
  what gets parsed, the tab is never rewritten), with a status note
  `bare auto → "auto"`.
- **FR-FCAN-07.** **Gate 2** (`validate_plan_steps`) on the projection: unknown
  message/signal rejected, values within min/max, units checked, deterministic
  `complete_missing_signals` repair, P8 DLC check for raw `data`; the result
  (errors / warnings / filled steps) drives the status line and Save.
- **FR-FCAN-08.** Row controls: every step row shows the encoded-frame suffix
  (single canonical builder) plus `✎ values` (a modal applying the **gate's
  own coercion** — modal-clean ⇒ Gate-2-clean) and `▣ frame` (double-click too)
  opening a modal frame editor (byte grid + pastable raw line + signal view,
  two-way synced over the cantools funnel); a deviating grid saves as a
  `data` override (⚡ marker); reset clears the override.
- **FR-FCAN-09.** AI chat: `$step` yields proposals as **dashed review rows**
  with Accept / Edit / Reject (✎ Edit opens the step's JSON in a centered
  proposal-edit modal); `$generate` proposes a **full plan** — a gate-clean
  confirm replaces the stack; **S stays the only save path** (the chat never
  writes files).
- **FR-FCAN-10.** **M** (offline rehearsal) shall expand the plan to frames and
  run a **TX-only rehearsal on a virtual timeline** — no ECU, honestly labeled
  `mode: "offline"` — writing `.can.result.json` (steps with frames/span,
  totals, duration).
- **FR-FCAN-11.** **R** (live run) shall transmit the same frames at
  **real-time cadence** (the bus IS the clock — playback speed is ignored),
  after a device-scan preflight; a connect failure is a `fail` **before any
  frame**; per-frame TX errors are counted, not fatal; RX capture is off in
  v1 (honestly labeled; full monitoring is FR-CAN-25 [FUTURE]).
- **FR-FCAN-12.** **C — Compile and Run** (ADR-010): a Gate-2-**dirty** stack
  never compiles; C performs Save's write half, then in-process **Gate 3**
  (`json_to_asc` with the loaded DBC + active E2E profile, stdout captured),
  then launches the ASC playback dashboard **in a new terminal**
  (`CREATE_NEW_CONSOLE`) with `--dbc` from the loaded knowledge; output
  `<plan dir>/output/<stem>.asc`.
- **FR-FCAN-13.** **S** (Save): Gate 2 (repair-then-gate) over the projection →
  writes `.can.plan.json` (the engineer's verbatim `pre_conditions` /
  `test_metadata` / `generationStatus` ride through) + `.can.coverage.yaml`
  next to the md → **always** re-renders the tab (the re-render discards a
  broken tab); save-while-broken saves the **last valid document** + a
  warning.
- **FR-FCAN-14.** **E** / Save-trace exports `.can.trace.txt`; one shared
  trace-line builder serves Copy, Save-trace and E; M/R stream the annotated
  trace to the PACKET TRACE pane.
- **FR-FCAN-15.** The status line shall carry **live coverage** (messages
  covered / uncovered, from the projection).
- **FR-FCAN-16.** Plan path convention: `TestPlan/<unit>.can.plan.json`,
  auto-incrementing to `TestPlan_2/…` on collision; the same path the CLI chain
  consumes (S writes the identical shape).
- **FR-FCAN-17.** After every **S / C / M / R / E** gesture the dashboard shall
  re-assert the unit anchor's tool-owned keys per FR-UNIT-03/05 (same contract
  as the UDS dashboard).
- **FR-FCAN-18.** M and R read the **projection** — a broken PLAN JSON tab can
  never put a frame on the bus.

### 3.5 ASC playback dashboard (`asc_dashboard.py`)

- **FR-ASC-01.** The dashboard shall be a Textual TUI launched as
  `python asc_dashboard.py <file.asc> [--dbc PATH]` presenting an ASC log as a
  frame table (time, CAN ID, DLC, payload; signals decoded when a DBC is
  supplied), a trends pane over selected signals/IDs, and playback controls
  (start/stop, speed, repeat/scrub over the timeline).
- **FR-ASC-02.** An optional bus-TX mode shall transmit the frames on live
  hardware during playback (the compiled-and-run path of FR-FCAN-12 opens this
  dashboard that way).
- **FR-ASC-03.** The dashboard is read-only with respect to the ASC file —
  playback never rewrites the log.

### 3.6 Test-case units and the anchor (ADR-012)

- **FR-UNIT-01.** `test-case init --unit <folder> --kind uds|can --source
  <file> [--e2e-profile NAME]` shall create a test-case unit: **refuse** an
  existing unit (exit 2, `TestcaseExistsError`); bad arguments → exit 2;
  **CAN** kind runs the existing `dbc-to-md` first and copies the DBC into the
  unit (conversion failure → exit 1); **UDS** kind keeps the source stem and
  records `source_from` provenance; then it writes the
  `<unit>.testcase.yaml` anchor.
- **FR-UNIT-02.** The anchor shall carry: `schema_version`, `unit_id`, and —
  **engineer-owned** (seeded empty at init, never written again by the tool):
  `title`, `status` (`draft | under_review | approved`), `requirement_refs`,
  `objective`, `preconditions`, `expected_result`, `pass_fail_criteria`,
  `owner`, `execution.defect_refs` — and **tool-owned**: `environment`
  (`source_dbc` / `source_uds` / `source_from` / `e2e_profile`), `artifacts`
  (unit-folder-relative paths of the kind's artifact set), `execution.date` /
  `execution.mode` / `execution.verdict`. The anchor file is comment-rich so an
  engineer can see the ownership split at a glance.
- **FR-UNIT-03.** `update_tool_owned` shall be the **only** writer of
  tool-owned keys; it shall round-trip via ruamel.yaml so the engineer's
  inline comments and formatting survive; it **never raises** (missing /
  unparseable / OSError → skip, file left intact); with no anchor it is a
  silent no-op.
- **FR-UNIT-04.** Verdict mapping: CAN — `offline` mode → always `pass`;
  `live` mode → `pass` iff the result has no errors; UDS — `pass` iff
  `all_passed`. `blocked` is engineer-only; `status` is never
  tool-transitioned.
- **FR-UNIT-05.** Re-assertion points: every S / C / M / R / E gesture of both
  dashboards, and the CLI save/run paths (`automate` mock/live, plan save,
  compile) — the tool-owned keys are re-written from the fresh result each
  time (never a one-shot stamp).
- **FR-UNIT-06.** Host surface: the two directory right-click items
  "Test Case: UDS" / "Test Case: CAN" shall preflight-refuse an existing
  anchor, browse for a `.uds.yaml` / `.dbc` (cancel = nothing written), run
  `canforge.py test-case init` (CAN adds the resolved `--e2e-profile`), then
  launch `uds_dashboard.py` / `can_dashboard.py` on the fresh artifact with
  `--ai-chat` in a new console (win32).

### 3.7 Cross-cutting utilities

- **FR-UTIL-01.** `translate-md --input-md <md> --output-md <out>` shall
  translate Chinese markdown to English via the AI backend (target language
  from `config.yaml` `translation:`).
- **FR-UTIL-02.** `warm-model` shall warm the configured model on its Ollama
  server (keep-alive) before first use, so the first real call is not slow.
- **FR-UTIL-03.** `list-security-profiles` shall print available UDS security
  profile names, one per line; `inspect-security-profile <name>` shall print
  the profile as JSON to stdout.
- **FR-UTIL-04.** E2E profile resolution (ADR-025): CLI `--e2e-profile` → env
  `CANFORGE_E2E_PROFILE` → the project's `.nausica.yaml`
  (`tool_config.canforge.e2e_profile`, found by upward search with a
  project-root default) → none. Profiles under `profile/e2e/` (e.g.
  `generic_autosar`, `oem_hud`, `none`) carry: CRC algorithm + params, alive
  counter (width, start value), detection mode (name-suffix regex for
  counter/CRC signals, or explicit message/signal overrides), byte placement
  (`from_dbc`), AI-prompt instructions (planning + refine), ASC markers, and
  message include/exclude overrides.
- **FR-UTIL-05.** UDS security profiles under `profile/uds/` shall carry:
  `backend`, `hardware` (channel, baudrate, termination), `timeout` (the
  cascade's profile layer), `can_ids` (tx/rx/func), and `security_levels`
  (per level: `algorithm`, `tx_id`/`rx_id`, `seed_sub`/`key_sub`, `service`,
  `session`). Loading a profile joins its levels into the level catalog
  (FR-UDS-19).
- **FR-UTIL-06.** Every pipeline step shall be appended to
  `data/audit_log.jsonl` (append-only JSONL: timestamp, command, inputs,
  outputs, exit code, duration) via the audit logger — the ASPICE trail that
  an artifact exists because *this* command ran with *these* inputs.
- **FR-UTIL-07.** `config.yaml` shall supply defaults for `ai` (servers,
  models, `context_tokens`), `paths` (output dir, default DBC), `translation`
  (target language) and `hardware` (backend, channel, baudrate, FD, serial,
  playback speed/repeats); every default is overridable by env vars and then
  by CLI flags (flag > env > config).
- **FR-UTIL-08.** Long-running CLI phases shall display the animated status
  line (Braille / ASCII / piped degradation per constraint §2.4.8); phases are
  spinner-labeled (scan, bundle, LLM, gates, repair round n/3, encode, play).

---

## 4. Interface Requirements

### 4.1 CLI interface

The entry point is `canforge.py` (argparse subparsers; every subcommand
delegates to exactly one module function). Three shared argument groups are
mixed in as parser "parents":

| Group | Members (defaults in parentheses) | Applied to |
|---|---|---|
| **AI** | `--input-model` (config `ai.model`), `--backend-server-ip` (config `ai.server`) | `translate-md`, `warm-model`, `plan-from-md`, `refine-plan` (and `uds-extract-knowledge`'s knowledge-role overrides) |
| **E2E** | `--e2e-profile` (None) | `dbc-to-md`, `plan-from-md`, `refine-plan`, `json-to-asc`, `test-case init` |
| **HW** | `--backend` (`tosun_libtscan`; also `tosun_tsmaster`, `mock`), `--channel` (1), `--baudrate` (500 kbps), `--serial` (""), `--fd` (off), `--fd-databaudrate` (2000), `--request-id` (0x7E0), `--response-id` (0x7E3) | `uds …`, `automate` |

**Subcommand contract:**

| Subcommand | Arguments (required marked *) | Behavior (requirement refs) | Exit codes (§8) |
|---|---|---|---|
| `translate-md` | `--input-md` *, `--output-md` * (+AI) | Chinese md → English via AI (FR-UTIL-01) | 0 ok / 1 error |
| `warm-model` | (+AI) | Warm the model on the Ollama server (FR-UTIL-02) | 0 ok / 1 error |
| `dbc-to-md` | `--input` * (.dbc), `--output-desc`, `--output-detail` (+E2E) | Deterministic DBC → 2 markdown; Gate 1 (FR-CAN-01…08) | 0 pass / 1 gate failed |
| `plan-from-md` | `--mode` * single\|chat, `--input-md` *, `--prompt` ("Do not generate the plan yet, just say hello."), `--output` (`TestPlan/draft_plan.json`) (+E2E, AI) | AI plan generation; auto-repair; Gate 2 (FR-CAN-09…16) | 0 ok / 1 gate or AI failure |
| `refine-plan` | `--json` * (existing plan), `--mode` quick\|full (quick) (+E2E, AI) | Chat refine of an existing plan (FR-CAN-15) | 0 ok / 1 failure |
| `json-to-asc` | `--json` *, `--dbc` *, `--output` * (+E2E) | Deterministic encode; Gate 3; E2E at encode time (FR-CAN-17…21) | 0 ok / 1 gate failed |
| `play-asc` | `--asc` * (+HW minus UDS addressing), `--speed` (1.0), `--repeats` (5), `--sync-tx` (off) | Play ASC on live hardware (FR-CAN-22) | 0 ok / 1 failure |
| `record-asc` | `--output` * (+HW minus UDS addressing), `--duration` (0 = until interrupted), `--include-tx` (off), `--append` (off) | Record live bus → ASC (FR-CAN-23) | 0 ok / 1 failure |
| `list-devices` | — | Backend device scan (FR-CAN-24) | 0 ok |
| `list-security-profiles` | — | Profile names, one per line (FR-UTIL-03) | 0 ok |
| `inspect-security-profile` | `profile_name` * (slug without `.yaml`) | Print profile as JSON (FR-UTIL-03) | 0 ok / 1 not found |
| `uds session` | `[subfunction]` (default = diagnostic default) (+HW) | One-shot 0x10 (FR-UDS-13) | 0 ok / 1 error |
| `uds read-did` | `did` * (e.g. `0xF190`) (+HW) | One-shot 0x22 | 0 ok / 1 error |
| `uds write-did` | `did` *, `--data` * (hex string) (+HW) | One-shot 0x2E | 0 ok / 1 error |
| `uds tester-present` | (+HW) | One-shot 0x3E | 0 ok / 1 error |
| `uds seed-key` | `--level` (1) (+HW) | One-shot 0x27 exchange | 0 ok / 1 error |
| `uds routine` | `action` * start\|stop, `rid` * (e.g. `0x0201`) (+HW) | One-shot 0x31 | 0 ok / 1 error |
| `automate` | `macro` * (.uds.macro.yaml / .yaml / .json), `--mode` live\|mock\|dry-run (live), `--continue-on-error` (off), `--closedloop-profile`, `--result-file`, `--p2-ms`, `--p2-ext-ms`, `--respond-pending-s`, `--stmin-ms`, `--block-size`, `--tx-dl`, `--knowledge`, `--security-rules` (+HW) | Step 3 macro run (FR-UDS-21…29) | 0 all passed / dry-run · 1 step failed · 2 validation failed (pre-hardware) |
| `uds-extract-knowledge` | `--input` * (source dir), `--output`, `--max-files` (24), `--include-dirs` (repeatable), `--security-profile` (advisory), `--dry-run` (off), `--show-prompt` (off), `--input-model` (knowledge-role model), `--backend-server-ip` (knowledge-role server) | Step 1 extraction (FR-UDS-01…12) | 0 pass (incl. pass-with-strip) · 1 gate failed · 2 no evidence / budget overflow / no tree-sitter · 3 LLM unreachable |
| `test-case init` | `--unit` * (folder), `--kind` * uds\|can, `--source` * (.dbc / .uds.yaml), `--e2e-profile` | Create the unit + anchor (FR-UNIT-01) | 0 created · 2 bad args or unit exists · 1 CAN conversion failed |

**Environment variables** (override `config.yaml`, overridden by CLI flags):

| Variable | Role |
|---|---|
| `CAN_BACKEND_IP` / `CAN_MODEL_NAME` | Chat-role Ollama server / model |
| `CAN_KNOWLEDGE_SERVER` / `CAN_KNOWLEDGE_MODEL` | Knowledge-role server / model (Step 1) |
| `CANFORGE_E2E_PROFILE` | E2E profile (between CLI flag and `.nausica.yaml`, §4.3) |
| `CANFORGE_CLOSEDLOOP_PROFILE` | UDS security profile (between `--profile` and `.nausica.yaml`, §4.3) |

**Standalone TUI entry points** (separate processes, not `canforge.py`
subcommands — launched by the host in new consoles, or directly):

```
python uds_dashboard.py <file.uds.yaml> [--profile NAME] [--sim-p2-ms 40] [--ai-chat]
python can_dashboard.py <file.can.md> [--ai-chat]
python asc_dashboard.py  <file.asc> [--dbc PATH]
```

Every CLI step appends to `data/audit_log.jsonl` (FR-UTIL-06). Stdout is
human-readable (gate reports, summaries); stdout capture is safe for the
host's progress dialogs. `--e2e-profile` accepts a profile name resolved
against `profile/e2e/`.

### 4.2 Artifact interface

Files are the only persistent medium between stages, the host, and the
regression [FUTURE] steps. Ownership (full formats in §6):

| Artifact | Format | Written by | Consumed by | Location |
|---|---|---|---|---|
| `<unit>.can.md` | Markdown (description) | `dbc-to-md` | Step 2 (LLM), Flowline CAN knowledge pane, host Step-1 signature check | `DBC_Description/` |
| `<unit>_detail.can.md` | Markdown (full spec) | `dbc-to-md` | human review, encoding reference | `DBC_Description/` |
| `<unit>.can.plan.json` | JSON (`template_v1` + optional `data`) | Step 2 save, Flowline **S** | `json-to-asc`, Flowline **M/C/R** | `TestPlan/` (auto-increment on collision) |
| `<unit>.asc` | verbose ASC (TSMaster) | `json-to-asc`, Flowline **C** | `play-asc`, ASC dashboard, TSMaster | `TestPlan/output/` |
| `.can.coverage.yaml` | YAML | Flowline **S** (also CLI save path) | engineer, host, [FUTURE] regression | next to the md |
| `.can.trace.txt` | annotated text | Flowline **M/R/E** | engineer | next to the md |
| `.can.result.json` | JSON | Flowline **M** (offline) / **R** (live); CLI paths | engineer, anchor verdict (FR-UNIT-04), [FUTURE] regression | next to the md |
| `<stem>.uds.yaml` | YAML (schema v2, `status: draft`) | Step 1 | Flowline UDS, `automate` (sibling rule), Gate context | beside the sources' stem |
| `<stem>.uds.rejected.yaml` | YAML | Step 1 hard fail | engineer (rejected knowledge) | sibling of the good artifact, never overwrites it |
| `<stem>.evidence_bundle.md` | Markdown | Step 1 (always) | human review, Gate B/C audit trail | sibling |
| `<stem>.prompt.txt` | text | Step 1 `--show-prompt` | engineer (prompt forensics) | sibling |
| `.uds.macro.yaml` | YAML (hex scalars) | Flowline **S**, engineer hand-edit | `automate`, Flowline **M/R** | beside the `.uds.yaml` |
| `.uds.result.json` | JSON (schema v1) | `automate`, Flowline **M/R** | engineer, anchor verdict, [FUTURE] regression | beside the macro |
| `.uds.coverage.yaml` / `.uds.trace.txt` | YAML / text | Flowline **S** / **E** | engineer | beside the `.uds.yaml` |
| `<unit>.testcase.yaml` | YAML (comment-rich) | `test-case init` (full file); `update_tool_owned` (tool-owned keys only) | engineer, host, [FUTURE] regression | unit root |
| `data/audit_log.jsonl` | append-only JSONL | every pipeline step | ASPICE audit | `data/` (runtime, never committed) |

**Interface rules:** artifacts next to a unit are the *unit's* record; the
tool never rewrites a good `<stem>.uds.yaml` on a failed re-run; unknown
top-level keys in plan/macro documents survive round-trips (FR-CAN-16,
FR-FCAN-05); YAML artifacts the engineer reads use hex scalars for byte
values (lossless round-trip, FR-FUDS-07).

### 4.3 Host (Nausica) interface

The integration is **subprocess + manifest only**: Nausica never imports
CANForge code; `argv` and files are the entire contract.

- **Discovery.** Nausica's plugin engine scans `Dependency_Tools/*/plugin.yaml`
  at startup; the CANForge manifest (`id: CANForge`) is enabled per profile.
- **Manifest shape.** `id`, `name`, `version`, `description`, `menu_items[]`
  where each item has `label`, `type: action`, `condition`
  (`file_extension`, `name_suffix`, `is_file`, `min_selection`,
  `max_selection`) and `handler` (`method` with `handler_args.method_name` +
  `pass_file_path`, or `subprocess` with `script_path` + `args_template`).
  `name_suffix` matching is suffix-exact (`".uds.yaml"` matches `foo.uds.yaml`
  but not `foo.uds.macro.yaml`).
- **Menu items (16).** CAN: *Convert DBC to Markdown* (`.dbc`), *Inspect E2E
  Profile* (`.dbc`), *E2E Profile Settings…* (`.dbc`), *Generate Plan (Chat
  Mode, AI)* (`.can.md`), *Open Flowline (AI)* (`.can.md`), *Compile Plan to
  Trace* (`.can.plan.json`), *Play ASC File in TSMaster* / *Play ASC File
  Directly* / *Play ASC File Directly with Dashboard* / *Record to ASC File
  Directly* (`.asc`), *Refine This Plan (Quick Edit)* / *(Full DBC Context)*
  (`.json`). UDS: *Run Macro…* (`.uds.macro.yaml`), *Chat Steps (AI)*
  (`.uds.yaml`), *List CAN Devices* (directory), *Extract Knowledge (AI)*
  (directory). Units: *Test Case: UDS* / *Test Case: CAN* (directory,
  single selection — FR-UNIT-06).
- **Launch pattern.** `method` handlers execute Nausica `MainWindow` methods
  that build the `canforge.py` argv (or Popen a dashboard with
  `CREATE_NEW_CONSOLE` on win32); `subprocess` handlers run
  `[sys.executable, "canforge.py", …]` directly. Progress is shown in a
  `QProgressDialog` for long CLI steps.
- **Per-project configuration.** The host resolves the project's `.nausica.yaml`
  by **upward directory search** from the right-clicked file (project-root
  default fallback). CANForge consumes:
  - `tool_config.canforge.e2e_profile` — the active E2E profile for *all four*
    CANForge launch points (the three command builders + the Flowline
    launcher), which append `--e2e-profile <name>` when active (ADR-025).
  - the UDS profile layer of the cascade: `--profile` flag →
    `CANFORGE_CLOSEDLOOP_PROFILE` env → `.nausica.yaml` → `config.yaml` →
    auto-match by seed/key pairs (FR-UDS cascade, §5 SAD).
- **Host-absent parity.** With no Nausica and no `.nausica.yaml`, the same
  commands run unchanged with defaults (constraint §2.4.11).

### 4.4 AI backend interface

- **Transport.** Ollama HTTP API via `requests`/`ollama` (LAN only). Three
  call shapes: single-shot completion (`llm_call_single_shot`), streaming chat
  (`llm_call_chat_stream` with an `on_token` callback — the TUI chat strip
  renders tokens live), and plain chat (`llm_call_chat`, returns content +
  token usage). Model keep-alive for `warm-model` and first-use warmup.
- **Model roles.** Two logical roles, each with its own server/model pair:
  *chat* (`ai.server`/`ai.model`) and *knowledge*
  (`ai.knowledge_server`/`ai.knowledge_model`, falling back to the chat role
  when absent). Resolution per call: CLI flags → env vars → `config.yaml`.
- **Context budget.** `ai.context_tokens` is sent as request-level
  `options.num_ctx`; the same value derives the Step-1 evidence-bundle token
  budget (FR-UDS-03), so a smaller server degrades the bundle *before* the
  request, never mid-flight.
- **Prompt catalog.** `prompts/test_plan_prompt.txt` (single-shot plan),
  `prompts/test_plan_chat_prompt.txt` (CLI chat REPL — all-signals-assigned
  rule), `prompts/can_plan_chat_prompt.txt` (Flowline CAN chat: `$step` /
  `$generate`), `prompts/uds_knowledge_system.txt` + `uds_knowledge_user.txt`
  + `uds_knowledge_repair.txt` (Step 1, schema embedded),
  `prompts/uds_step_chat_prompt.txt` (Flowline UDS chat: the step contract,
  session/security state, no-invention rule). E2E profile `ai_prompt`
  instructions are injected into the plan/refine prompts (FR-CAN-07/14).
- **Response discipline.** JSON is extracted tolerantly (code fences,
  surrounding prose); the plan JSON template is part of the prompt
  (`{json_template}`); repair rounds send the *gate's* errors back verbatim.
- **Failure semantics.** Unreachable server: Step 1 exits 3; plan/refine
  report a named error (no partial artifact); TUI chat shows the error in the
  strip — the stack/document are never touched by a failed AI call.

### 4.5 Hardware interface

- **Abstraction.** `CanBackend` (ABC) + `CanFrame` dataclass (CAN ID, payload,
  FD flag) in `hardware/usb_to_can/canforge_io`. Contract:
  `connect(channel, baudrate, fd, data_baudrate, serial)` (`channel` is a
  0-based index), `disconnect()`, `transmit(frame)`, `transmit_batch(frames)`,
  `receive(timeout_ms)`, `register_callback(can_id, queue)` /
  `unregister_callback(can_id)`, `scan() → list[dict]` (device inventory for
  `list-devices` and the dashboard preflights).
- **Factory.** `create_backend(name, **kwargs)` + `register_backend(name,
  factory)` — the extension point: a new hardware family is a new registered
  backend, no engine change (FR-2.4 extensibility). Registered names:
  `tosun_libtscan` (default), `mock`, `tosun_tsmaster`.
- **`tosun_libtscan`.** ctypes binding to the vendor `libTSCAN.dll` (shipped
  in `libTSCANDemos/Python/native/`, loaded at runtime — no pip install);
  `CanFrame` ↔ vendor struct translation (CAN 2.0 + CAN-FD).
- **`mock`.** The in-process UDS ECU simulator (FR-UDS-27) — a backend in the
  same sense as hardware: the engine, ISO-TP and UDS layers are identical
  across `mock` and live, which is what makes mock runs *evidence* about live
  behavior rather than a separate toy.
- **`tosun_tsmaster`.** SKELETON: lazily loaded SDK adapter whose frame
  conversion is unimplemented (raises `NotImplementedError`); kept as the
  registration-shaped placeholder for the TSMaster SDK path [out of scope].
- **Parameters.** Defaults from `config.yaml` `hardware:` (backend,
  `channel_default`, `baudrate_kbps`, `fd_data_baudrate_kbps`, `can_fd`,
  `serial`, `playback_speed`, `playback_repeats`); CLI flags override; the
  macro's `hardware:` block feeds `automate`; `--request-id` /
  `--response-id` carry the UDS addressing (defaults 0x7E0/0x7E3; profiles
  may carry their own `can_ids`).
- **Preflight.** Live runs (one-shot UDS, `automate --mode live`, Flowline
  **R**, `play-asc`) perform a device-scan preflight and fail **before any
  frame** when the device is absent (FR-FCAN-11).

### 4.6 TUI interface

All three TUIs are Textual apps (rich-based panes, modal screens, key
bindings); they are headless-testable (Textual `run_test` pilot) on the
offscreen platform — no display is required for the test suite.

| App | Panes | Modals | Global keys |
|---|---|---|---|
| **UDS Flowline** (`uds_dashboard.py`) | knowledge tree; step stack; live YAML preview; packet trace (bottom); AI chat strip (`C`-toggled, bottom band) | Frame editor (byte grid + raw line + structured fields, two-way); write-values screen (one prompt per layout field); prompt screen (OEM raw bytes); test-case screen | **R** live · **M** mock · **S** save macro+coverage · **E** export trace · **C** chat · **Q** quit |
| **CAN Flowline** (`can_dashboard.py`) | CAN KNOWLEDGE (1-row live search on top); PLAN WORKBENCH (two tabs: PLAN JSON default, STEP STACK); PACKET TRACE; AI CHAT strip | Values screen (gate's own coercion); frame editor (byte grid + raw line + signal view, cantools funnel); proposal-edit screen (step JSON); prompt screen | **r** live · **m** offline · **s** save · **e** export trace · **f** focus search · **c** chat · **q** quit; two-row toolbar (run family / document family, incl. per-tab Copy + ↺ resync) |
| **ASC dashboard** (`asc_dashboard.py`) | frame table (decoded with `--dbc` when given); trends; playback/scrub controls | — | playback keys (start/stop/speed/repeat); bus-TX toggle |

Interface rules shared by all TUIs:

- **Single canonical builders.** Row suffixes, frame editors, reset/status,
  and mock/live request construction all derive from one builder per
  protocol (`request_bytes_for_step` for UDS, `encode_step_frame` for CAN);
  the trace, the Copy buttons, and the saved trace file all derive from one
  line builder. A value the engineer sees is the value that would be sent.
- **Gate before hardware, always.** Runs are gated on the current
  document/stack state (`validate_macro` / Gate 2); a broken state refuses the
  run with the gate report in the status line — it never silently "fixes"
  the document to run (save-while-broken saves the last *valid* document and
  warns; M/R read the projection, FR-FCAN-18).
- **AI rows are proposals, never mutations.** Dashed review rows with Accept /
  Edit / Reject; Accept is the only path into the stack (FR-FUDS-13,
  FR-FCAN-09).
- **Copy buttons** lift pane content (trace, YAML) to the OS clipboard
  (pyperclip) via the shared builders.
- **Console fidelity.** The TUIs assume a modern terminal; the *CLI* paths
  carry the cp1252/Braille/piped degradation duty (FR-UTIL-08).

---

## 5. Non-Functional Requirements

Each NFR carries a **Verification** line — the mechanism the rebuild uses to
prove the property (these are the properties the test suite encodes as
regressions, so a "rebuild" that cannot be verified this way is not
equivalent).

### NFR-01 — Default-off parity (the project-wide invariant)

**Requirement.** Every optional feature face (ADR-001 `layout`/`values`/
`readback`, ADR-002 DTC `verify`/`dtcs`, ADR-003 `timing:`, ADR-004
`enforce_session_gates`, ADR-005 pluggable security, ADR-006 flow-control,
ADR-043 `expect_nrc`, ADR-044 `raw`/`data`, the ADR-045 `test_case` block and
step `note`, and every mock ctor face) shall be **optional and default-off**:
with the face absent from the input, behavior — including every produced byte
— shall be identical to the legacy pipeline that predates the face.

**Why.** The tool runs against real ECUs and real buses in production; an
"additive" feature that changes a legacy run's bytes would silently invalidate
existing baselines and ASC logs.

**Verification.** Parity tests: legacy inputs (no faces) run through the new
code must produce byte-identical artifacts (golden-file diff of ASC, macro
YAML, result JSON modulo timestamps) and identical engine call sequences
(mock counters, ISO-TP frame streams).

### NFR-02 — Determinism

**Requirement.** Deterministic stages shall produce byte-identical output for
identical input: `dbc-to-md` (FR-CAN-01), `json-to-asc` (modulo explicit
timestamp policy), plan document round-trips, macro YAML rendering, and the
offline rehearsal timeline (virtual clock, no wall-clock dependence in `M`).

**Verification.** Run-twice + diff golden files; the offline result
(`.can.result.json` mode `offline`) must be reproducible exactly.

### NFR-03 — DBC-wins (CAN truth layer)

**Requirement.** For the CAN pipeline the `.dbc` is the single truth layer and
the `.can.md` is a display layer: any drift (md says signal X min–max, DBC
says otherwise) is a **warning shown to the engineer, never a failure**, and
encoding/validation always uses the DBC values.

**Verification.** Drift fixtures: md edited against the DBC → warnings
emitted, Gate 2 runs against DBC values, encoding unaffected.

### NFR-04 — No silent guessing

**Requirement.** Unknowns shall surface as named errors, warnings, or
`open_questions` — never as best-effort values. Concretely: DTC subfunctions
0x07/0x0C/0x0D raise `DtcParseError` (step failure), not a partial decode; a
security level whose declared algorithm is unregistered is a validation error
(exit 2), not a fallback to a builtin; an unresolvable algorithm in the mock
yields the named NRC 0x31; a malformed 0x54 for a different DTC group is
malformed, not an ack; a DID with un-evidenced payload shape is emitted
**without** a layout, never with invented fields; a bare `auto` token outside
value positions is left alone, not "repaired".

**Verification.** Each clause has a dedicated test asserting the *named
failure*, not merely "it failed".

### NFR-05 — Human-in-the-loop

**Requirement.** AI output is a gated proposal, never a product: generated
plans stay `under_review` until a human approves; an AI chat step enters the
stack only via explicit Accept (or Edit→Accept); `validate_macro` refuses to
open hardware on an invalid macro (exit 2); the anchor's `status` and
engineer-owned fields are never tool-written (FR-UNIT-02/03).

**Verification.** Contract tests: no code path writes `status:` or engineer
fields; the dashboard's stack can only grow via click-accept or Accept
buttons (pilot-driven test).

### NFR-06 — Hardware-free operability

**Requirement.** The entire closed loop shall be exercisable with no CAN
hardware: `dry-run`, the `mock` backend, offline rehearsal (`M`), and the full
test suite require no device; live paths fail clean (preflight, before any
frame) when hardware is absent.

**Verification.** The suite runs in CI without a TOSUN adapter; a
device-absent live run asserts "fail before any frame" (mock TX tap sees
nothing).

### NFR-07 — Encoding robustness

**Requirement.** DBC files and C sources shall load through the cascade
UTF-8 → GBK → GB2312 → Latin-1 with mojibake repair of comments, so Chinese
OEM vendor files produce readable artifacts; a file that decodes cleanly in
UTF-8 must never be re-decoded through a lossy fallback.

**Verification.** Fixture DBCs in each encoding (incl. a deliberately
mojibake'd comment) → expected repaired text; UTF-8 fixture untouched by the
cascade.

### NFR-08 — Console degradation

**Requirement.** Console output shall degrade gracefully across the three
regimes: Braille spinner on UTF-8/VT-capable consoles, ASCII fallback on
cp1252 (the common Windows case), and plain timestamped lines when output is
piped (no ANSI garbage in logs/redirects); nothing in any regime may raise or
corrupt the actual result output.

**Verification.** Run each long command with the stream redirected and under
a forced cp1252 codepage; assert no escape bytes in the captured output and
clean results.

### NFR-09 — Evidence and traceability

**Requirement.** Every run shall leave a reviewable evidence trail:
`data/audit_log.jsonl` entry per pipeline step; `.uds.yaml` entities carry
`source: file:line` citations (Gate B enforces); `.uds.result.json` carries
per-field timing **sources** and mock counters; coverage files list what was
and was not covered; the anchor's `execution` block records date/mode/verdict
of the last gesture. An ASPICE reviewer must be able to reconstruct *what ran,
with which inputs, and what it produced* from artifacts alone.

**Verification.** Audit-log schema test; result-JSON source-field test (each
timing field names its cascade source); citation audit on the fixture
knowledge tree.

### NFR-10 — AI-offline resilience

**Requirement.** Every deterministic step shall run with no network; AI steps
degrade to clean, named, non-zero failures (Step 1: exit 3; plan/refine/
translate: named error, no partial artifact) — never a hang, and never a
partial artifact written *as success*. The TUI chat strip shows the error and
leaves the stack/document untouched.

**Verification.** Test with the Ollama endpoint pointed at a dead port: each
AI command asserts its exit code/message and that no artifact file appeared or
changed.

### NFR-11 — Extensibility without source edits

**Requirement.** Onboarding a new ECU/OEM shall require **no CANForge source
change**: a new seed/key algorithm is a `register_algorithm()` (+ a profile
entry), a new security level a `register_level()` (+ profile), a new ECU E2E
scheme a `profile/e2e/*.yaml`, a new UDS target a `profile/uds/*.yaml`, and a
new CAN hardware family a `register_backend()`. Source edits remain for new
*step types* and new *pipeline stages* only.

**Verification.** Extension test: register a dummy algorithm + level at
runtime and drive a mock run against them end-to-end (the built-in level set
being the reference case of this mechanism).

### NFR-12 — Portability

**Requirement.** Windows 11 is the primary platform (cp1252 console, libTSCAN
DLL, `CREATE_NEW_CONSOLE` host launches); Linux is supported for the
AI/mock/deterministic paths. Native code is limited to loading vendor DLLs via
ctypes; nothing in the tool requires compilation.

**Verification.** The suite passes on both platforms; the DLL load is
lazy/optional so mock-mode imports never touch a DLL.

### NFR-13 — Performance and token budgets

**Requirement.** The planning LLM sees the **low-token description** artifact
only; the Step-1 evidence bundle is budgeted to `ai.context_tokens` with
per-file caps and dedup, and a budget overflow fails **fast and named before
any LLM call** (exit 2) rather than sending a truncated prompt; CLI phases
show progress (spinner) so a long LLM call is distinguishable from a hang.
(No hard wall-clock targets are specified — LLM latency is an external
property — but no *deterministic* stage may dominate a run.)

**Verification.** Budget-overflow fixture (huge dispatcher) → exit 2 with the
named message and no LLM call (mock transport asserts zero requests).

### NFR-14 — Testability

**Requirement.** The system shall be testable headless: the pytest suite
covers CLI commands (subprocess or in-process), the protocol stack (mock
backend as the test double for the ECU), the TUIs (Textual pilot on the
offscreen platform), and the parity properties of NFR-01/02; golden files pin
byte-identity; the suite requires no hardware and no network (LLM calls are
stubbed or pointed at fixtures).

**Verification.** The suite itself is the verification; the current baseline
is a full-suite pass (hundreds of tests, no skips from missing hardware —
only deliberate environment skips).

### NFR-15 — Security and privacy

**Requirement.** No credentials or bus traffic leave the LAN: the only
network calls are to the operator's own Ollama servers; artifacts contain no
secrets (seed/key *algorithms* are code, and keys are computed at run time,
never stored in artifacts); the audit log records commands and paths, not
payloads that could carry sensitive ECU state beyond what the run itself
produced.

**Verification.** Code-review invariant (no third-party endpoints in the
dependency tree beyond Ollama HTTP); artifact schema review (no key/seed
values in `.uds.yaml`/macro/result fields).

### NFR-16 — Structural discipline (maintainability)

**Requirement.** The rebuild shall preserve the dependency rules that make the
tool auditable: **leaf modules** that encode a protocol fact
(`uds_timing`, `session_gate`, `dtc`, `nrc`, `did_layout`, `testcase`,
`e2e_computation`, `canforge_io`) import nothing from the project;
`handler/can` and `handler/uds` do not import each other; the TUI shells are
copied, never imported, from one another (UDS protocol semantics cannot enter
the CAN tool); and each concern has **one canonical implementation**
(request-bytes builder, encode/expand builders, trace-line builder, NRC table,
security registry, timing cascade, DTC interpreter, gate formats) that all
consumers share.

**Verification.** Import-boundary tests (AST scan: forbidden imports fail the
suite); single-builder assertion — UI suffix and transmitted bytes are
produced by the same function (test compares both outputs).

### NFR-17 — State visibility (no silent states)

**Requirement.** A broken or pending state shall always be visible: the
PLAN JSON tab goes red (document/stack untouched) rather than being rewritten;
save-while-broken saves the last valid document **and warns**; a broken anchor
warns and stays intact rather than being "fixed"; a pending AI proposal is
visibly dashed until accepted; gate warnings (session gates) are shown at
pre-flight even when the in-engine gate is off. The tool never silently
discards, repairs, or transitions state on the engineer's behalf.

**Verification.** Pilot tests: break the tab mid-edit → run M → refused or
runs on last valid projection, tab text byte-identical to what the engineer
typed; anchor-corruption fixture → warning emitted, file untouched.

---

## 6. Data Requirements (artifact formats)

Formats are specified as **structure contracts**: field names, types, and
invariants are normative; prose/comments are not. Where a registry in code is
the single source of truth (step types, NRC names, schema JSON), the artifact
conforms to the registry and the registry is the rebuild target.

### 6.1 `<unit>.can.md` — CAN knowledge (description)

Markdown, UTF-8, deterministic (NFR-02). Structure:

```markdown
# CAN Test Scope: `<vendor>.dbc>`            ← Step-1 signature (host checks this line)

> **Purpose**: Guide AI test planning. Focus on *what* to test.
> **Low-token, high-signal**. Human-editable for intent refinement.

## <Message Name> (0x<HEX>)                   ← one section per message, sorted by frame ID
- Sender(s): <node[, node]>
- Cycle Time: <N> ms                          ← GenMsgCycleTime / CycleTime attribute
- Description: <mojibake-repaired comment>
- Category: functional | diagnostic | tracer | reserved   ← name-keyword inference
- Testability: <inferred note>
- **Signals**
  - `<signal>`: <min>–<max> [<enum choices>] — <description>

## E2E Configuration                          ← ONLY when --e2e-profile is active
<profile name, counter/CRC detection mode, byte placement>
```

Invariants: every DBC message and signal appears (Gate 1); line 1 format is
stable (the host's Step-1 signature check); no timestamps or map-order
leakage.

### 6.2 `<unit>_detail.can.md` — full bit-level spec

Same message-section layout, plus per-signal detail sufficient for bit-level
review and encoding without the DBC: start bit, bit length, byte order
(intel/motorola), sign, raw min/max, factor (scale), offset, unit, choices,
description. Not fed to the LLM (two-phase markdown, constraint §2.4.6).

### 6.3 `.can.plan.json` — the CAN test plan

JSON conforming to `json/template_v1.txt` (the schema of record):

```jsonc
{
  "generationStatus": { "status": "<e.g. generated|under_review|approved>", "reason": "..." },
  "test_metadata": {
    "name": "...", "dbc_file": "<vendor>.dbc",
    "source_md": "<unit>.can.md", "source_dbc": "<vendor>.dbc",
    "total_duration_ms": 0                       // fill rule: max end_time
  },
  "pre_conditions": ["..."],                     // human text, rides through verbatim
  "test_sequences": [
    {
      "step_name": "...",
      "start_time_ms": 0,
      "end_time_ms": 0,
      "message": "<message name exactly as in DBC>",
      "can": { "id": "0x40B", "id_type": "standard" },   // id: hex STRING; id_type: standard|extended
      "cycle_time_ms": 100,                     // 0 = single shot, else repeat at this period
      "signals": { "<signal>": <physical value>, ... }
      // ALL signals of the message present (Gate 2 completes missing ones);
      // values are PHYSICAL (not raw); "auto" only for E2E-protected signals with a profile.
      // "data": [0x..., ...]                    // OPTIONAL additive raw override (NFR-01):
      // when present, these exact bytes are transmitted instead of the encoded payload;
      // omitted (not null) when absent so legacy documents round-trip byte-identically.
    }
  ]
  // Unknown top-level keys must survive tool round-trips untouched (NFR-01).
}
```

Field notes (normative): **one message, one transmitter; same-message steps
strictly sequential**; `can.id` is a hex string; a bare `auto` token is a
display-layer accident repaired decoder-guided (value positions only,
FR-FCAN-06) — it is not a plan format feature.

### 6.4 `.asc` — compiled CAN traffic

Verbose ASCII log (Vector-style), TSMaster-compatible, one line per frame:
timestamp, channel, CAN ID, direction, DLC, payload bytes — plus a header
comment block (carrying the E2E profile marker when a profile was active, per
the profile's `asc_marker`). Cycle-time steps are expanded to their repeated
frames at the cycle interval; frames ordered by start time (stable for ties).
The recorder (`record-asc`) writes the same dialect; `play-asc` and the ASC
dashboard consume it. The file is **read-only output** — no tool rewrites an
existing ASC except `--append` (recorder).

### 6.5 `.can.coverage.yaml`

```yaml
source_md: <unit>.can.md
plan: <unit>.can.plan.json
generated_at: "<ISO timestamp>"
messages_total: <n>
covered_messages: ["..."]
uncovered_messages: ["..."]
```

### 6.6 `.can.trace.txt`

Human-readable annotated trace, one line-builder (shared by Copy, Save-trace
and E): step headers + per-frame lines with the decoded signal view (values
from the DBC) + annotations (raw-override ⚡ markers, cycle repeats). Written
by Flowline **E**; streamed to the PACKET TRACE pane by **M/R**.

### 6.7 `.can.result.json`

```jsonc
// offline rehearsal (M):
{ "mode": "offline",
  "steps": [ { "name": "<step_name>", "frames_tx": <n>, "span_ms": <n> }, ... ],
  "total_frames": <n>, "duration_ms": <n>, "generated_at": "..." }

// live run (R): mode "live" plus transmitted-frame count, per-frame TX error
// count (counted, not fatal), duration, backend name, generated_at;
// a connect failure records the fail BEFORE any frame.
```

### 6.8 `.uds.yaml` — UDS knowledge artifact (schema v2)

YAML; `schema_version: 2` (v1 remains valid — Gate A accepts 1 or 2). The
machine-readable description of one ECU's UDS implementation, produced by
Step 1 and consumed by Flowline/`automate`/mock. Required top level:
`schema_version`, `transport`, `sessions`, `services`, `nrcs`,
`open_questions`, `generationStatus`.

```yamlc
schema_version: 2

transport:
  can:    { /* addressing notes */ }
  isotp:  {                       # timing cascade layer 3 (FR-UDS-25)
    p2_server_ms: 50, p2_extended_ms: 5000, p2_star_ms: 50,
    stmin_ms: 10, block_size: 0, max_pdu_single: 7, max_pdu_multi: 4095 }   # defaults

sessions:                         # one entry per diagnostic session
  - { id: "0x01", name: "default", requires_security: false, source: "file.c:123" }
  # id/name/requires_security/source per entry; source = file:line citation

services:                         # keyed by hex SID string
  "10":  { sessions: [ ... ] }    # session switch table
  "14":  { groups: ["FFF", "C00000", ...] }        # clearable 3-byte DTC groups
  "19":  {                                            # ReadDTCInformation (ADR-002)
            subfunctions: ["01", "02", "04"],        # ⊆ {01,02,04,05,06,0A,0B}
            dtc_length: 3,                            # 2 | 3
            dtcs: [
              { dtc: "123456", name: "...",
                sessions: ["0x03"],                   # allow-list (negative-case join)
                status: "05" }                        # OPTIONAL mock-seed status byte
            ] }
  "22":  { dids: [ <DID> ] }                            # 0x22-readable
  "2E":  { dids: [ <DID> ] }                            # 0x2E-writable
  "27":  { levels: [
              { level: 1, name: "icon", seed_sub: "01", key_sub: "02",
                algorithm: "icon", session: "0x61" } ] }  # algorithm optional → level name
  "31":  { routines: [ { rid: "0x0201", name: "..." } ] }
  "other": { /* OEM SIDs, e.g. "BE" */ }

# <DID> shape:
#   { did: "F190", name: "...", size_bytes: 4,
#     source: "file.c:210",
#     layout: null | {                                  # ADR-001 — EMIT ONLY WHEN EVIDENCED
#       fields: [ { name: "...", offset: 0, length: 2,
#                   encoding: u8|u16|u32|u64|i8|i16|i32|i64|ascii|hex,
#                   bits?: ..., scale?: ..., offset_value?: ...,
#                   min?: ..., max?: ..., unit?: ... } ] } }
#   Big-endian fields tile without overlap; total span ≤ size_bytes.

nrcs: [ { code: "7F", name: "generalReject", raised_by: "file.c:340" }, ... ]
open_questions: [ "..." ]                 # Gate B strips, Gate C omissions, null seed_sub/key_sub
generationStatus: { status: draft, notes: "..." }   # draft → reviewed is ENGINEER-owned
```

Invariants: per-entity `source: file:line` citations (Gate B); a DTC
**status-byte layout is NOT in the artifact** (fixed by ISO Annex C — decoded
by the shared interpreter); the canonical SID name table applies (0x15 =
`ControlDTCSetting`, 0x19 = `ReadDTCInformation`); the `layout` is a hard-error
field (malformed → Gate A fail), never a warning.

### 6.9 Step-1 side artifacts

- `<stem>.evidence_bundle.md` — always written: the token-budgeted,
  citation-annotated evidence (macro-rewritten lines, per-file dispatcher
  sections, session/NRC sites, manifest) — the audit trail of what the LLM saw.
- `<stem>.prompt.txt` — `--show-prompt` transcript of the final prompt.
- `<stem>.uds.rejected.yaml` — hard-fail artifact (exit 1); never overwrites a
  good `<stem>.uds.yaml`.

### 6.10 `.uds.macro.yaml` — the UDS macro

YAML, hex scalars for byte values (lossless round-trip). Document:

```yamlc
backend: mock                    # mock | tosun_libtscan | tosun_tsmaster
request_id: 0x78C
response_id: 0x79C
hardware: { channel: 0, baudrate: 500000 }
continue_on_error: false
# --- optional faces (all default-off, NFR-01) ---
timing:                          # ADR-003 macro layer (range-checked, exit 2)
  { p2_ms: 500, p2_extended_ms: 5000, respond_pending_s: 5.0,
    stmin_ms: 20, block_size: 0, tx_dl: 8 }
enforce_session_gates: false     # ADR-004 opt-in in-engine gate
test_case:                       # ADR-045
  { id: "...", title: "...", preconditions: [...], postconditions: [...] }
steps:                           # NON-EMPTY; order = execution order
  - { type: <step_type>, ... }
```

Step contract — 19 registered types (the engine's `KNOWN_STEP_TYPES` +
`REQUIRED_STEP_PARAMS` registries are the single source of truth; this table
is the documented contract):

| `type` | Required params | Optional faces |
|---|---|---|
| `uds_session` | `session` (hex) | — |
| `uds_read_did` | `did` | assert/expect faces, `expect_nrc`, `note` |
| `uds_write_did` | `did` + `data` (hex) **or** `values` (per-layout-field) | `readback: true` (ADR-001), `expect_nrc`, `note` |
| `uds_read_dtc` | DTC read scope (subfunction) | `expect_nrc`, `note` |
| `uds_clear_dtc` | `group` (3-byte) | `verify` (default **true**, ADR-002), `expect_nrc`, `note` |
| `uds_tester_present` | — | `expect_nrc`, `note` |
| `uds_ecu_reset` | reset subfunction | `expect_nrc`, `note` |
| `uds_request_seed` | `level` | `expect_nrc`, `note` |
| `uds_send_key` | `level` (+ key material) | `expect_nrc`, `note` |
| `uds_unlock_security` | `level` | `max_key_retries`, `expect_nrc`, `note` (ADR-005 per-NRC semantics) |
| `uds_io_control` | `did`, control type | `expect_nrc`, `note` |
| `uds_routine_start` / `uds_routine_stop` | `rid` (+ data) | `expect_nrc`, `note` |
| `uds_raw` | `data` (exact request bytes, hex) | `expect_nrc`, `note` (ADR-044) |
| `send_can` | `can_id`, `data` | — |
| `wait_message` | `can_id`, timeout | — |
| `sleep` | duration | — |
| `start_keepalive` / `stop_keepalive` | keepalive period (start) | — |

Per-step universal faces: `expect_nrc` (ADR-043 — a matching `7F <SID> <NRC>`
is the **pass** condition), `raw` (ADR-044 — transmit exactly these bytes,
range-checked, ⚡ in the UI), `note` (ADR-045 — carried into the result).
Validation (exit 2): unknown type, missing required param, out-of-range
timing, unresolvable level/algorithm.

### 6.11 `.uds.result.json` — run evidence (schema v1)

```jsonc
{ "schema_version": 1,
  "timestamp": "ISO",
  "mode": "live | mock",
  "macro": "<path>",
  "backend": "mock | tosun_libtscan",
  "closedloop_profile": "<name | null>",
  "request_id": "0x78C", "response_id": "0x79C",
  "total_steps": <n>, "passed_steps": <n>, "failed_steps": <n>,
  "all_passed": true,
  "summary_text": "N/M passed — ...",
  "steps": [
    { "index": 0, "step_type": "uds_read_did", "passed": true,
      "message": "...", "note": "..." ,            // note: ADR-045, when present
      "data": { /* step-specific payload: DID values, DTC structure, raw bytes, NRC + name */ } } ],
  "test_case": { /* ADR-045 block echoed from the macro, when present */ },
  "timing": {                                       // ADR-003: per-field SOURCES
    "p2_ms": { "value": 500, "source": "cli | macro | artifact | profile | default" },
    /* ...one entry per timing field... */ },
  "mock": { "uds_requests": <n>, "frames_tx": <n>, "frames_rx": <n> }  // mock mode only
}
```

Identical schema for CLI `automate` and dashboard M/R runs (same builder).
Step `data` for security steps carries `attempts` + final NRC + canonical
name (ADR-005); DTC steps carry the parsed 0x59 structure + raw payload.

### 6.12 `.uds.coverage.yaml` / `.uds.trace.txt`

- **Coverage**: lists what the macro exercises — `dids_written_readback`,
  `dtcs_read`, `dtcs_cleared` (plus the read DID set) — written on **S**.
- **Trace**: the artifact-annotated packet trace (same content as the pane:
  field-decoded 0x22/0x2E, DTC lists with Annex C bits, NRC names, FC
  annotations, expect_nrc-green markers) — written on **E** / Save-trace.

### 6.13 `<unit>.testcase.yaml` — the unit anchor (ADR-012)

Comment-rich YAML; the comments ARE the interface (they document the
ownership split). Structure (as seeded by `test-case init`):

```yamlc
# Test-case unit anchor — CANForge ADR-012.
schema_version: 1
unit_id: "<unit folder name>"
# ── engineer-owned: seeded empty at init, NEVER written by the tool ──
title: ""                        # engineer: one line — what this unit tests
status: draft                    # draft | under_review | approved (never tool-transitioned)
requirement_refs: []             # engineer: ASPICE traceability anchor
objective: ""
preconditions: []
expected_result: ""
pass_fail_criteria: ""
owner: "<seeded at init>"
# ── tool-owned ──
environment:                     # source_dbc / source_uds / source_from / e2e_profile
artifacts:                       # unit-folder-relative paths of the kind's artifact set
execution:
  date: null                     # tool: date of the last run
  mode: null                     # tool: offline | live
  verdict: null                  # tool: pass | fail | blocked  (blocked: engineer-only)
  defect_refs: []                # engineer: links against the defect tracker
```

Ownership rules (normative): `update_tool_owned` is the only writer of
`artifacts` + `execution.date/mode/verdict`; ruamel.yaml round-trip preserves
all comments/formatting; the writer never raises (FR-UNIT-03); `status` and
the engineer fields are byte-untouched across every tool gesture.

### 6.14 `data/audit_log.jsonl`

Append-only JSONL — one record per pipeline step: timestamp, command, inputs
(paths), outputs (paths), exit code, duration. `data/` is runtime state:
never committed, never part of the source drop; the audit trail survives
because records are append-only.

### 6.15 Configuration files

- **`config.yaml`** (tool-local defaults): `ai` (`server`, `model`,
  `knowledge_server`, `knowledge_model`, `context_tokens`), `paths`
  (`output_dir`, `default_dbc`), `translation` (`target_lang`), `hardware`
  (`backend`, `channel_default`, `baudrate_kbps`, `fd_data_baudrate_kbps`,
  `can_fd`, `serial`, `playback_speed`, `playback_repeats`). Precedence:
  CLI flag > env var > config (§4.1).
- **`profile/e2e/*.yaml`** — E2E protection profiles: `profile` block
  (type, CRC `algorithm` + `crc_params.input_bytes`, `alive_counter`
  width/start), `detection` (mode regex — counter suffixes
  `_AliveCounter$`/`_E2E_Cnt$`/`_Cnt$`, CRC suffixes `_Checksum$`/
  `_E2E_Crc$`/`_Crc$` — or explicit overrides), `byte_placement`
  (`from_dbc`), `ai_prompt` (planning + refine instructions), `asc_marker`
  (header comment), `overrides` (include/exclude messages).
- **`profile/uds/*.yaml`** — UDS security profiles: `id`/`name`/`version`,
  `backend`, `hardware` (channel, baudrate, termination), `timeout`
  (the cascade's profile layer: `p2_ms`, `p2_extended_ms`,
  `respond_pending_timeout_s`, `block_size`, `stmin_ms`), `can_ids`
  (`tx`/`rx`/`func`), `security_levels` (per level: `algorithm`, `tx_id`/
  `rx_id`, `seed_sub`/`key_sub`, `service`, `session`).
- **`.nausica.yaml`** (project-local, host-side): the most specific
  configuration layer — `tool_config.canforge.e2e_profile` (CAN, ADR-025) and
  the UDS closed-loop profile selection (cascade layer between env and
  config.yaml). Found by upward directory search from the right-clicked file
  with a project-root default; absent = the cascade simply skips the layer.

---

## 7. Gate Requirements

**Gate philosophy (normative).** A gate is a deterministic, hardware-free
checkpoint between pipeline stages. AI output is a *proposal* that must pass
its gate to become an artifact or reach a bus; gates never call the LLM,
never open hardware, and never guess — they either **pass**, **pass with
warnings**, or **fail with a report**. Every gate is **one canonical
implementation** shared by the CLI and the TUI (NFR-16): the Gate 2 that
`plan-from-md` runs is the Gate 2 that Flowline **S** runs; the macro
validation that `automate` runs is the one the dashboard runs before **M/R**.
Gate output is a structured `(errors, warnings)` pair — **errors block,
warnings never do** — rendered by one shared report builder so the engineer
sees the same text in the CLI, the TUI status line, and (for step-level
gates) the trace.

| Gate | Stage | Blocks |
|---|---|---|
| **G1** DBC coverage | after DBC→md | artifact acceptance (exit 1) |
| **G2** plan vs DBC | plan save / refine / Flowline S, C, M, R | plan acceptance, compilation, runs |
| **G3** JSON→ASC + ASC validation | compile | ASC product |
| **A** schema | Step 1, after each LLM round | `.uds.yaml` acceptance (exit 1) |
| **B** citation audit | Step 1, after Gate A | `.uds.yaml` acceptance (or pass-with-strip) |
| **C** switch completeness | Step 1, after Gate B | repair-loop input (never a hard fail by itself) |
| **V** `validate_macro` | before any hardware | engine start (exit 2) |
| **SG** session gating | pre-flight (warnings) + in-engine (opt-in) | step execution (refusal) |
| **PG** provisional gate | AI proposal, pre-accept | proposal acceptance into the stack |

### 7.1 Gate 1 — DBC coverage (after DBC→Markdown)

- **FR-GATE-01.** Gate 1 shall verify that **every message and every signal**
  of the DBC appears in both generated markdown artifacts (nothing silently
  dropped by the renderer).
- **FR-GATE-02.** Gate 1 runs immediately after generation inside
  `dbc-to-md`; its error list (empty = PASS) drives the exit code: 0 PASS,
  1 failure with the gate report printed. A Gate-1 failure means the
  artifacts are **not** a usable Step-2 input.

### 7.2 Gate 2 — plan vs DBC (the plan's acceptance gate)

- **FR-GATE-03.** Gate 2 shall check, per step: the message exists in the DBC
  and the CAN ID matches it; every referenced signal exists; **all** signals
  of the message are present (the encoder requires them — missing ones are
  filled by the deterministic `complete_missing_signals` repair, and the
  fill is *reported*, never silent); values are within DBC min/max (enum
  numbers in range are accepted) with the gate's own type coercion
  (numeric/enum); units are checked where declared; timing is valid
  (start/end sane, cycle ≥ 0, same-message steps strictly sequential, one
  message one transmitter); raw `data` (when present) passes the P8 DLC
  check against the message.
- **FR-GATE-04.** Gate 2 is invoked on: `plan-from-md` save (with the
  auto-repair loop of FR-CAN-11 feeding its errors back to the LLM),
  `refine-plan` output, and Flowline **S** (repair-then-gate over the
  projection; save-while-broken saves the last valid document + warning),
  **C** (a Gate-2-**dirty** stack never compiles), and is the state that
  **M/R** consult before putting any frame on a timeline or bus
  (FR-FCAN-18). The Values modal applies the **same coercion** as Gate 2 —
  modal-clean ⇒ Gate-2-clean (NFR-16 single builder).
- **FR-GATE-05.** Gate 2 output = `(errors, warnings, filled_steps)`:
  errors block; warnings (e.g. DBC-wins drift notes) never block; the fill
  list is surfaced to the engineer.

### 7.3 Gate 3 — JSON→ASC and ASC validation (the compile gate)

- **FR-GATE-06.** Gate 3 shall (a) re-run Gate 2 over the plan being compiled
  (the plan may have aged since its last save) and (b) after encoding,
  validate the generated ASC: every frame parses, CAN IDs and DLCs are
  consistent with the DBC, and the file round-trips through the ASC parser.
- **FR-GATE-07.** Gate 3 runs in `json-to-asc` (exit 0 ok / 1 gate failed,
  gate report printed) and in-process in Flowline **C** (stdout captured into
  the status/trace; failure stops before the ASC dashboard is launched). The
  DBC for a plan is located by the canonical `find_dbc_for_plan` (plan's
  `test_metadata.source_dbc` first, then the unit layout) — the same
  resolver everywhere.

### 7.4 Gate A — knowledge schema (Step 1)

- **FR-GATE-08.** Gate A shall validate the LLM's `.uds.yaml` against the v2
  schema (accepting v1): required top level; canonical ISO 14229 SID name
  table (0x15 = `ControlDTCSetting`, 0x19 = `ReadDTCInformation`); session
  entries well-formed; per-DID `layout` structure-checked by the **single**
  layout validator (big-endian tile without overlap, span ≤ `size_bytes`,
  known encodings) — a bad layout is a **hard error**, because an
  un-decodable layout would poison every downstream read; the `"19"` service
  (subfunction scope ⊆ v1 set, `dtc_length` 2|3, DTC width matches, no
  duplicates) and the `"14"` `groups` — a DTC table that does not parse is a
  gate error, **not** a warning, because the DTC interpreter would refuse it
  at run time; null `seed_sub`/`key_sub` on SecurityAccess are review
  **warnings**.

### 7.5 Gate B — citation audit (Step 1)

- **FR-GATE-09.** Gate B shall verify every `source: file:line` (and
  `raised_by`) citation against the scanned source tree: the file exists in
  the scan set and the line is within range. In strip mode, entities with
  bad citations are **removed** and the removal noted in `open_questions`;
  the resulting artifact is a **pass-with-strip** (draft written, exit 0).
  The evidence-bundle manifest (not the LLM's memory) is the audit basis.

### 7.6 Gate C — switch completeness (Step 1)

- **FR-GATE-10.** Gate C shall compare the deterministic pre-pass's
  dispatchers (switch **and** if/else-if, with per-case value text) against
  the artifact's subfunction/session/NRC tables; case values the artifact
  cannot explain produce errors that feed the repair loop, and omissions
  **surviving** the repair rounds land in `open_questions` with the draft
  still written (exit 0) — Gate C is never the sole hard fail.

### 7.7 Gate V — macro validation (`validate_macro`)

- **FR-GATE-11.** `validate_macro` shall pre-flight the macro **before any
  hardware is opened** and exit 2 on: unknown step `type`; missing required
  params per the `REQUIRED_STEP_PARAMS` registry; empty/absent `steps`;
  out-of-range `timing:` block fields (each range-checked); a security level
  whose declared algorithm is **unregistered** (named error — NFR-04);
  `raw`/`data` values that fail range checks. The dashboard's
  `validate_steps` = `validate_macro` **plus** artifact-aware warnings
  (session-gate ⚠ markers) and gates the live stack before **M/R** and on
  Save.

### 7.8 Gate SG — session precondition gating (ADR-004)

- **FR-GATE-12.** The pre-flight **warning pass** is always on and
  warning-only: for each step, the static walk (`session_state_at` — a
  `uds_session` moves the session and **clears** unlocked levels; no 0x10 yet
  = default session 0x01 assumed) decides whether the step's preconditions
  (per-value DID allow-list, bulk DTC offenders, `requires_security`) are met
  by the state at that stack position; unmet → warning with the canonical
  message. `expect_nrc` steps are **exempt** (they *test* the refusal).
- **FR-GATE-13.** The opt-in **in-engine gate** (`enforce_session_gates:
  true`) uses the live state (`LiveSessionState`, fed with the **actual**
  acks) and **refuses pre-send** via the single decision point
  (`check_gate`), producing a step failure with the canonical refusal text —
  one format shared with the warnings and the dashboard ⚠ markers. The
  0x11/0x71 reset ack does **not** update the live state yet (staleness errs
  permissive, never stricter). The tester-side gate and the mock's
  ECU-side rules may disagree — the trace shows **both**; that discrepancy
  is information, not a bug.

### 7.9 Gate PG — provisional gate (AI proposals)

- **FR-GATE-14.** An AI chat step proposal shall be gate-validated on a
  **spliced copy** of the current stack (insert at the proposed position,
  run the gate) **before** it is rendered as a review row; a proposal that
  fails its provisional gate is shown as failing (or not accepted), and
  **no AI step enters the stack without explicit acceptance** (NFR-05).
  `$generate` full-plan proposals additionally require the confirm to be
  gate-clean before replacing the stack.

### 7.10 Step-result gates (pass/fail semantics at execution)

- **FR-GATE-15.** Execution pass/fail is decided by step semantics, not by
  "got a response": a `uds_read_dtc` **passes** on a positive `0x59` the DTC
  interpreter accepts (result carries parsed structure + raw payload) and
  **fails** on `DtcParseError` (0x07/0x0C/0x0D — never a guess); a
  `uds_clear_dtc` requires the `0x54 <group>` echo and, with `verify: true`
  (default), a clean post-clear re-read — "cleared (assumed)" is never
  reported; a step with `expect_nrc` **passes** on the matching
  `7F <SID> <NRC>` (and fails on any other response); a `readback: true`
  write passes only if the re-read matches the written `values` (and
  readback never bypasses the session gates).
- **FR-GATE-16.** A `7F` matching the running step's `expect_nrc` renders
  **green** in the trace (a tested negative response is a pass, visually
  distinct from an unexpected negative).

### 7.11 Repair loops and gate reports

- **FR-GATE-17.** LLM-facing repair loops are **bounded and no-progress
  detecting**: Step 1 runs up to 3 rounds (repair prompt carries the gate
  report's errors under `{errors}`); plan auto-repair likewise bounded; a
  round that changes nothing exits early. Gate failures after the budget →
  the hard-fail path (Step 1: exit 1 + `.uds.rejected.yaml`).
- **FR-GATE-18.** One shared **gate report** builder renders every gate's
  `(errors, warnings)` (CLI stdout, TUI status line, repair prompts, trace) —
  the engineer's error text is the gate's error text, verbatim.

### 7.12 Gate summary

| Gate | Trigger | On fail |
|---|---|---|
| G1 | `dbc-to-md` post-gen | exit 1, artifacts not accepted |
| G2 | plan save / refine / Flowline S·C·M·R | no save-as-approved / no compile / no run; repair loop (CLI) or last-valid save + warning (TUI) |
| G3 | `json-to-asc` / Flowline C | exit 1 / no ASC dashboard launch |
| A | Step 1, per LLM round | repair round → exit 1 + rejected artifact |
| B | Step 1, after A | strip + note (pass-with-strip) or repair → exit 1 |
| C | Step 1, after B | repair → `open_questions` (draft kept, exit 0) |
| V | before any hardware (`automate`, Flowline M/R/Save) | exit 2 / refused gesture |
| SG | pre-flight always (warn); in-engine opt-in (refuse) | warning / step failure with canonical refusal |
| PG | AI proposal pre-render | proposal shown failing / not accepted |

---

## 8. Exit-Code Conventions

### 8.1 General rules

- **The exit code is the contract** with the host (Nausica), the shell, and
  [FUTURE] CI: it is decided **before** the process prints its final summary,
  and the summary names the reason. A non-zero exit is **never silent** —
  stdout/stderr carry a human-readable cause (gate report, NRC, missing
  device, dead server).
- **Four-value discipline** across all commands:
  | Code | Meaning |
  |---|---|
  | `0` | Success (or, where noted, a *correct negative* outcome) |
  | `1` | **Runtime failure** — something executed and failed (gate failed, a step failed, encoding error, I/O error, hardware error) |
  | `2` | **Validation / precondition failure** — the input or arguments were invalid **before any side effect**; nothing ran, no hardware opened, no run-artifact written |
  | `3` | **External dependency unavailable** — the required LLM server could not be reached (Step 1 only; other AI steps report code 1 with a named error) |
- **Consistency rules (normative):**
  - (a) A code-2 path **never** opens hardware and **never** writes a
    run-artifact (no `.asc`, no `.uds.result.json` from a run, no anchor
    `execution` update).
  - (b) A code-1 run **does** write its evidence — a failed run's
    `.uds.result.json` / `.can.result.json` **is** the product (failure is
    evidence; the anchor verdict becomes `fail`).
  - (c) A code-3 path writes nothing new (Step 1: no `.uds.yaml`, no
    clobber of a good one).
  - (d) The exit code reflects the **worst** stage reached.
  - (e) Every code is appended to `data/audit_log.jsonl` (FR-UTIL-06).
  - (f) An empty *result* is not an error: `list-devices` with no devices
    exits 0 with an empty list; a plan with zero covered messages exits 0 with
    an empty coverage list.

### 8.2 Per-command contract

| Command | `0` | `1` | `2` | `3` |
|---|---|---|---|---|
| `translate-md` | translation written | AI call or I/O failure | — | — |
| `warm-model` | model warmed | server error | — | — |
| `dbc-to-md` | both artifacts written, **Gate 1 PASS** | DBC parse / I/O failure or Gate 1 failed | — | — |
| `plan-from-md` | single: plan written + Gate 2 PASS · chat: clean session exit (a `save` that fails Gate 2 after the repair budget is a failure) | Gate 2 failed after repair rounds / AI failure / I/O | — | — |
| `refine-plan` | refined plan written + gated | Gate 2 / AI / I/O failure | — | — |
| `json-to-asc` | ASC written + **Gate 3 PASS** | Gate 2/G3 failure, encode error, I/O | — | — |
| `play-asc` | playback completed (repeats done) | connect/device failure, TX error, I/O | — | — |
| `record-asc` | recording completed (`--duration` elapsed or clean interrupt) | connect/device failure, I/O | — | — |
| `list-devices` | scan completed (empty list is OK) | backend/scan error | — | — |
| `list-security-profiles` | names printed | profile dir unreadable | — | — |
| `inspect-security-profile` | JSON printed | profile not found / malformed | — | — |
| `uds <one-shot>` | positive response (or expected behavior) | NRC, timeout, hardware error — one-shot commands have **no** `expect_nrc` face, so any negative response fails | bad arguments | — |
| `automate` | all steps passed · dry-run: validation passed, sequence printed | ≥1 step failed (**result written — (b)**) | `validate_macro` failure (**pre-hardware — (a)**) | — |
| `uds-extract-knowledge` | pass — **incl. pass-with-strip** (draft written) | gates failed after 3 repair rounds → `.uds.rejected.yaml` written, good artifact never clobbered | no evidence / budget overflow / no tree-sitter (**pre-LLM — (a)**) | LLM unreachable (**(c)**) |
| `test-case init` | unit created (anchor written) | CAN-kind `dbc-to-md` conversion failed (runtime, mid-init) | bad arguments **or** unit already exists (`TestcaseExistsError`) | — |

Notes:

- `automate --mode dry-run` exits 0 **only** when validation passes — dry-run
  is "validate and print", not "skip validation".
- `uds-extract-knowledge --dry-run` (scan/bundle statistics, pre-LLM) uses the
  same 0/2/3 discipline: a budget overflow in a dry-run is still a code-2
  finding (it reports what the real run would have hit).
- One-shot `uds` commands map protocol failure to 1 regardless of *which* NRC
  arrived — the NRC and its canonical name are in the output, not in the
  code.

### 8.3 TUI process exit codes

The dashboards are interactive: their **process** exit code is narrow by
design.

- **Clean quit** (Q / app close) → `0`, whether or not the last run passed —
  the run's verdict lives in `.uds.result.json` / `.can.result.json` and the
  anchor, **not** in the process code.
- **Fatal startup error** (missing/unreadable `.uds.yaml`/`.can.md`/`.asc`,
  knowledge load failure, malformed plan seed) → named message + non-zero
  exit; the app never opens in a half-loaded state.
- **In-app gesture failures** (Gate-dirty Save, refused M/R, failed live
  connect, a failing mock run) are **status-line failures with the gate
  report / result**, not process exits — the engineer stays in the session.

### 8.4 Host (Nausica) contract

The host reads only `0` vs non-zero plus captured stdout/stderr: `0` →
success notification (and, for launchable outputs, the "open it?" follow-up);
non-zero → the captured output in a message box. The host never parses exit
codes finer than that — the **fine** semantics (1 vs 2 vs 3) are for the
engineer at the terminal and for [FUTURE] CI.

---

## 9. [FUTURE] Items and Explicit Non-Goals

### 9.1 [FUTURE] pipeline items (pending, out of scope for the initial rebuild)

These are the open ends of the pipeline doc of record [R1]. The rebuild
milestone delivers everything else; each item below is **already
architecture-prepared** (the artifact schemas and extension points it will
consume exist today), and none of them requires revising §1–§8.

| Item | Reference | What it adds | What already exists for it |
|---|---|---|---|
| **CAN Step 4 — Play + Monitor** | FR-CAN-25 | RX verification of the played traffic against the plan's expected frames, per-message timing windows, bus-error / error-frame accounting, bus-off detection; the live run grows an RX capture (v1: TX-only, honestly labeled) | `run_live`'s frame schedule + result shape (`frames_sent`, error counts), the ASC dashboard's RX-free playback, `receive()`/callbacks in the backend contract |
| **CAN Step 5 — Regression** | — | Baseline diffing: a stored `.can.result.json` / coverage baseline vs a fresh run; plan-vs-plan comparison across DBC re-versions | stable, `schema_version`-less but pinned result/coverage formats (§6.7/6.5), unit-folder identity (FR-CAN-02) |
| **UDS Step 4 — Regression + coverage** | FR-UDS-30 | Baseline diffing of `.uds.result.json` across runs (step pass/fail, timing drift via per-field sources), coverage reporting against the knowledge artifact | result schema v1 with per-field timing **sources** and mock counters (§6.11), `.uds.coverage.yaml`, the anchor `execution` block as the last-run pointer |
| **TSMaster SDK backend** | §4.5 | Real frame path for `tosun_tsmaster` (currently a skeleton whose conversion raises `NotImplementedError`) | the registration-shaped placeholder + the full `CanBackend` contract it must satisfy |
| **ARXML / ODBX parsing** | §1.2 | Database ingestion beyond DBC | the two-phase markdown contract (a new parser feeds the same artifacts); the DBC remains the truth layer for the DBC pipeline |

### 9.2 Explicit non-goals (boundary, not just "not yet")

The following are **outside the product's boundary by design** — a rebuild
that adds them has drifted from this SRS:

1. **Not an HIL bench.** No fault injection, no bus error-frame *generation*,
   no stimulus beyond what the plan/macro itself specifies. (Bus-error
   *accounting* is Step 4's monitoring, §9.1.)
2. **Not a DBC editor.** The DBC is a read-only input; no DBC write-back, no
   database authoring. Corrections go to the vendor file.
3. **Not a generic UDS client.** The service surface is the registered
   contract (FR-UDS-13/21); an OEM service not in the contract is driven via
   `uds_raw` (exact bytes), **not** by extending the protocol stack.
4. **AI is never the source of truth** — a standing invariant, not a phase:
   every AI output passes a deterministic gate before it becomes an artifact
   or reaches a bus (§7). No "trust the model" mode exists or is planned.
5. **Engineering judgement is never tool-performed.** `status` transitions
   (draft → under_review → approved), `blocked` verdicts, requirement refs,
   and pass/fail criteria are engineer-owned (§6.13); the tool measures and
   records, it does not certify.
6. **No cloud, no telemetry.** LAN-only network surface (the operator's own
   Ollama servers); no SaaS mode, no phoning home (NFR-15).
7. **No GUI outside the terminal.** The product surface is CLI + Textual TUI;
   the rich desktop surface is the host's (Nausica) responsibility — CANForge
   is a leaf tool the host orchestrates and will not orchestrate others.
8. **No multi-ECU / multi-bus orchestration in one run.** One macro, one
   request/response addressing pair, one backend; coordinated multi-target
   campaigns are a host-level concern.
9. **No database backend.** File-based artifacts next to the unit are the
   persistence model (§6); a regression store is files + git/SVN, not a DB.

### 9.3 Standing invariants for [FUTURE] work

Any [FUTURE] item implemented later must preserve: default-off parity
(NFR-01 — new faces are optional and legacy runs stay byte-identical), the
gate lattice (§7 — new stages get gates), the evidence model (NFR-09 — new
gestures re-assert the anchor and append to the audit log), and the single-
canonical-builder rule (NFR-16).

---

*End of SRS. Companion document: `doc/SAD.md` (architecture, module catalog,
decision rationale mapped to ADR-001…012, deployment and test views).*
