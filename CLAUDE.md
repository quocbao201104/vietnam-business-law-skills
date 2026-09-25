# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview & Core Philosophy

This repository develops the **Vietnam Business Law Practitioner Agent Skill** for business and commercial decisions involving Vietnamese law. Its primary artifacts are Markdown reasoning guidance, semantic runtime contracts, and evaluation fixtures. Python scripts support observable runtime trace evaluation (they do not run an LLM).

**Core Design Principle:** *Stable reasoning, live law.*
- Retain durable legal reasoning, framing procedures, and ownership boundaries in the skill.
- Resolve material, volatile law (rates, thresholds, forms, fines, deadlines) from authoritative sources at runtime. Never hardcode volatile figures into durable knowledge unless labeled as dated examples.

## Commonly Used Commands

### Running Tests
- **Run all unit tests:**
  ```powershell
  python -m unittest discover -s scripts
  ```
  or
  ```powershell
  python -m unittest scripts/test_runtime_trace.py
  ```
- **Run a single test method:**
  ```powershell
  python -m unittest scripts.test_runtime_trace.StateValidationTests.test_current_blocker_accepted
  ```

### CLI Utilities & Trace Checking
- **CLI help & verification:**
  ```powershell
  python scripts/check_runtime_trace.py --help
  python scripts/runtime_walker.py --help
  ```
- **Check a runtime JSONL trace against an evaluation oracle:**
  ```powershell
  python scripts/check_runtime_trace.py --repo-root . --oracle evals/composition/rf-oracles-v0.3.json --trace <path-to-trace.jsonl>
  ```
  (Oracles for BL1–BL8 JIT routing: `evals/composition/bl<N>-jit-oracles-v0.1.json`).

- **Observable trace walker commands (`scripts/runtime_walker.py`):**
  - Initialize trace:
    ```powershell
    python scripts/runtime_walker.py init --repo-root . --trace <trace.jsonl> --candidate-sha <sha> --fixture-id <id> --run-id <run_id>
    ```
  - Read candidate file (reads via `git show <candidate_sha>:<path>` to bind trace to exact candidate SHA):
    ```powershell
    python scripts/runtime_walker.py read --trace <trace.jsonl> --path <repo-relative-path>
    ```
  - Emit semantic event:
    ```powershell
    python scripts/runtime_walker.py emit --trace <trace.jsonl> --event-json '<json>'
    ```
  - Trace status:
    ```powershell
    python scripts/runtime_walker.py status --trace <trace.jsonl>
    ```

### Verification Rules
- **Markdown changes:** No compilation or build required. Inspect the full diff, validate referenced relative paths and exact identifiers, and verify consistency with canonical schemas.
- **Python changes:** Run `--help` on modified CLIs and execute regression tests (`python -m unittest discover -s scripts`).

## Architecture & Code Structure

### 1. BL1–BL8 Tracks & Semantic Ownership
Every material legal proposition has **exactly one accountable owner**. Ownership is semantic and not tied to an individual agent or process (one assistant can execute multiple owners sequentially; do not mandate multi-agent orchestration):
- **BL1 (Legal Issue Framing / Regime Routing):** Establishes initial issue framing, identifies candidate regime stacks, and forms route hypotheses. It does **not** decide substantive legal classifications owned by BL2–BL8.
- **BL2 (Corporate / Entity / Authority / Governance):** Entity identity, legal representation, signing authority, corporate approvals, ownership, and control. Does not decide foreign-investor status (delegated to BL8).
- **BL3 (Contracts / Commercial Transactions):** Formation, contract terms, obligations, conditions, performance status, amendments, and waivers.
- **BL4 (Breach / Remedies / Evidence / Disputes):** Breach, excuses/liability, loss mitigation, remedies, deadlines, and dispute postures.
- **BL5 (Tax / Financial Legal Consequences):** Tax characterization, base/timing/rates, statutory tax responsibilities, and invoice evidence.
- **BL6 (Employment / Workplace):** Worker classification, employment contracts, employer obligations, discipline, and separation.
- **BL7 (Regulatory / Compliance / Market Conduct):** Regulatory perimeters, licensing/permits, market conduct, consumer claims, and data privacy.
- **BL8 (Investment / Cross-Border / Trade):** Foreign investment market access, FX / cross-border payments, customs/tariffs, and international treaties.

### 2. Just-In-Time (JIT) Contract Loading
Do not preload all schemas or capability units. Follow the JIT loading matrix in `skills/vietnam-business-law-skills/SKILL.md`:
- `skills/vietnam-business-law-skills/SKILL.md` is the **always-on runtime kernel**.
- `skills/vietnam-business-law-skills/knowledge/INDEX.md` is the **canonical detailed route map**.
- Read only the relevant BL core file and specific capability units required for the task.
- Load companion schemas only when the governed boundary is reached:
  - `schemas/legal-work-state.md`: When revision-sensitive state must survive or reconcile across owner boundaries.
  - `schemas/runtime-composition.md`: Multi-owner composition, late routing, conflict handling, or loop control.
  - `schemas/authority-resolver.md` + `references/search-strategy.md` + `references/source-status.md`: Live-law verification or authority re-resolution.
  - `schemas/specialist-handoff.md`: Before calling a specialist for deep non-core technical depth.
  - `schemas/decision-output.md`: Multi-action readiness assessment or final structured decision brief.
  - `schemas/runtime-trace.md`: Trace recording or evaluation checking.

### 3. State, Dependencies & Invalidation
- **Shared State:** One shared legal state, stable material object IDs, revisions, and owner-scoped mutations. Downstream tracks return contradictions to the owning track rather than rewriting foreign state.
- **Edge Semantics:**
  - `DEPENDS_ON`: Only explicit proposition-level `DEPENDS_ON` edges automatically propagate invalidation/stale state.
  - `SIGNALS` and `FEEDBACK`: Trigger owner review without automatic invalidation cascade.
  - `CONSTRAINS`: Affects action readiness through explicit action links.
- **Conflict Lifecycle:** `ACTIVE` → `RESOLVED` or `TERMINAL_UNRESOLVED`. Unresolved conflicts preclude `READY` or `READY_WITH_CONDITIONS`.

### 4. Authority Resolver vs. Owner Responsibility
- **Authority Resolver:** Discovers sources, confirms instrument identity, verifies currentness/lifecycle, and checks freshness.
- **Accountable Owner:** Decides case applicability using proposition-specific temporal anchors. Crucial rule: `CURRENT_BINDING ≠ APPLICABLE_TO_CASE`.
- **Search vs. Verification:** Search engine snippets/summaries are discovery aids, not legal authority. Verification requires controlling provisions and instrument lifecycle checks.

### 5. Action Readiness Assessment
- Readiness is calculated **per proposed action**, not globally for the matter:
  - `READY`: Positive closure of all prerequisites and fresh authority.
  - `READY_WITH_CONDITIONS`: Material prerequisites satisfied but identifiable non-blocking conditions remain.
  - `DO_NOT_PROCEED`: Requires an independently supported legal blocker (unresolved conflict alone is insufficient).

### 6. Packaging & Manifests
The repository serves as both a standalone skill and a plugin for Claude Code and Codex:
- `skills/vietnam-business-law-skills/` is the single source of truth across all platforms; never duplicate skill files.
- Manifests must stay synchronized in identity and version:
  - `.claude-plugin/plugin.json`
  - `.claude-plugin/marketplace.json`
  - `.agents/plugins/marketplace.json`
  - `.codex-plugin/plugin.json`
