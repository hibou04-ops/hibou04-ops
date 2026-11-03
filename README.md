# Deterministic Verification Gates for AI Agents & LLM Pipelines

> **"Stop auditing stochastic models with more stochastic models."**  
> Zero-API-cost, offline Python gates that catch **hallucinated `file:line` citations**, **train/test split leakage**, **LLM-judge noise illusions**, and **overfitted prompts** before they ship.

[![License: Apache-2.0](https://img.shields.io/badge/Public_Primitives-Apache_2.0-2ea44f?style=flat-square)](https://opensource.org/licenses/Apache-2.0)
[![Verification Ledger](https://img.shields.io/badge/Ecosystem_Ledger-4,306_PASS-0969da?style=flat-square)](#%EF%B8%8F-two-tier-systems-architecture-open-primitives--in-memory-rd-core)
[![Runtime Contract](https://img.shields.io/badge/Contract-Fail--Closed_%2F_Offline-d73a49?style=flat-square)](#%EF%B8%8F-two-tier-systems-architecture-open-primitives--in-memory-rd-core)
[![Korean Docs](https://img.shields.io/badge/Docs-%ED%95%9C%EA%B5%AD%EC%96%B4_README-7057ff?style=flat-square)](./README.ko.md)

---

## ⚡ 10-Second Look: What Happens When an Agent Lies or Overfits?

Modern AI coding agents and prompt optimizers frequently report "green" benchmarks that collapse in production. Instead of trusting stochastic self-reports, enforce deterministic offline gates in your terminal or CI:

```diff
$ antemortem gate --report agent_patch_claim.md
- [FAIL] Hallucinated Citation: src/core/validator.py:142 (file has only 98 lines)
- [FAIL] AST Symbol Mismatch: `enforce_holdout_split` not found in src/core/validator.py:40-75

$ mini-antemortem-cli lint eval_suite.yaml
- [FAIL] Trap Rule #04 (Split Leakage): 14 normalized examples overlap between tune_set and test_set

$ mini-omega-lock probe --judge-runs ab_eval_runs.json
- [BLOCKED] Claimed prompt improvement (+2.1%) is below measured Judge MDE Noise Floor (±3.8%)

$ omega-lock audit --study sweep_trials.json --holdout locked_holdout.json
+ [PASS] Walk-forward generalization survived unseen holdout split (SHA-256 audit receipt issued)
```

---

## 🧭 Start Here: Pick the Gate for Your Failure Mode

Every public package in this ecosystem solves one specific failure mode in AI-assisted software engineering and LLM evaluation (`Apache-2.0`, provider-neutral, CLI + MCP ready):

| Failure Mode in Your Workflow | Install & Run (`Apache-2.0`) | Deterministic Enforcement Mechanism | Interfaces |
| :--- | :--- | :--- | :--- |
| **Coding agent hallucinating `file:line` or symbols** | [**`antemortem-cli`**](https://github.com/hibou04-ops/antemortem-cli) | Parses local files & ASTs offline to machine-check every `file:line` citation in agent outputs | `CLI` · `MCP` · `GitHub Action` |
| **Silent leakage or rigged prompt-eval setups** | [**`mini-antemortem-cli`**](https://github.com/hibou04-ops/mini-antemortem-cli) | Static pre-flight linter across **11 evaluation trap rules** (split overlap, judge bias, homogeneous variants) | `CLI` · `CI Gate` |
| **A/B prompt "wins" that are actually judge noise** | [**`mini-omega-lock`**](https://github.com/hibou04-ops/mini-omega-lock) | Quantifies Minimum Detectable Effect (**MDE**) noise floor of your evaluator before running sweeps | `CLI` · `Python API` |
| **Prompts tuned to ace dev sets that fail in prod** | [**`omegaprompt`**](https://github.com/hibou04-ops/omegaprompt) | Held-out generalization gate on top of `promptfoo` / `DSPy` that blocks overfitted prompt candidates | `CLI` · `MCP` · `CI Gate` |
| **Hyperparameter / sweep overfitting (Optuna, etc.)** | [**`omega-lock`**](https://github.com/hibou04-ops/omega-lock) | Walk-forward & held-out survival gate with tamper-evident SHA-256 verification receipts | `CLI` · `Python API` |
| **Fragile code paths needing counterexample probes** | [**`antemortem-structural-lab`**](https://github.com/hibou04-ops/antemortem-structural-lab) | Experimental structural AST probing, execution feedback, and deterministic counterexample replay | `CLI` · `Python API` |

---

## 🏗️ Two-Tier Systems Architecture (Open Primitives + In-Memory R&D Core)

This workspace is maintained as an **independent, non-commercial personal systems engineering and verification research lab**. It is organized into two architectural tiers:

1. **Tier 1 — Public Modular Primitives (`6 Packages · Apache-2.0`):** Standalone CLI and MCP tools designed for drop-in adoption across any developer workflow or CI pipeline.
2. **Tier 2 — Private Research Harnesses (`2 Harnesses · All Rights Reserved — Non-Commercial Personal R&D`):** Closed-source systems research investigating sub-20ms in-memory pipeline fusion and adversarial mutation testing across the verification stack. *(Internal personal R&D artifacts only; not distributed as commercial software).*

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ TIER 1: PUBLIC MODULAR PRIMITIVES (Apache-2.0 · Standalone CLI / MCP / CI)   │
├────────────────────────┬──────────────────────────┬──────────────────────────┤
│ mini-antemortem-cli    │ antemortem-cli           │ antemortem-structural-lab│
│ (11 Eval Trap Rules)   │ (AST File:Line Grounding)│ (Counterexample Probes)  │
├────────────────────────┼──────────────────────────┼──────────────────────────┤
│ mini-omega-lock        │ omegaprompt              │ omega-lock               │
│ (Judge MDE Noise Floor)│ (Held-Out Prompt Gate)   │ (Walk-Forward Lock)      │
└───────────┬────────────┴─────────────┬────────────┴─────────────┬────────────┘
            │                          │                          │
            ▼        [ Zero-IPC / Zero-Subprocess Fusion ]        ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ TIER 2: PRIVATE RESEARCH HARNESSES (Non-Commercial Personal R&D Only)        │
├──────────────────────────────────────────────────────────────────────────────┤
│ • omega-citadel (v1.1.2) — In-Memory 5-Stage Verification Engine             │
│   ├─ Fuses schema check + AST grounding + MDE filter + holdout lock + SHA256 │
│   └─ Measured Latency: 14.2 ms – 19.3 ms median (63.5x – 79.1x speedup)      │
│                                                                              │
│ • omega-crucible (v1.0.3) — Adversarial Meta-Verification Harness            │
│   ├─ Injects 1,067 synthetic leakage, drift, and citation-hallucination traps│
│   └─ Measured Survival: 1,067 / 1,067 (100.0% kill rate) in 163.77 ms        │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Private R&D Benchmark & Ledger Summary

| Research Harness | Version | Systems Research Scope (Non-Commercial Personal R&D) | Deterministic Benchmark |
| :--- | :--- | :--- | :--- |
| **`omega-citadel`** | `v1.1.2` | **In-Memory 5-Stage Governance Harness:** Eliminates subprocess and disk I/O bottlenecks by fusing prompt/schema contracts, AST citation grounding, MDE noise-floor filtering, held-out isolation, and SHA-256 receipt signing into a single Python runtime. | **14.2 ms – 19.3 ms** median latency across 20-run benchmarks (**63.5x – 79.1x** faster than sequential CLI chaining) |
| **`omega-crucible`** | `v1.0.3` | **Adversarial Meta-Verification Harness:** Audits the decision boundaries of the verification gates against synthetic failure modes (split leakage, schema drift, and hallucinated `file:line` citations). | **1,067 / 1,067 (100.0%)** adversarial mutation kill rate verified in **163.77 ms** |

---

## 🔒 Methodology & Verification Contract

* **Fail-Closed by Default:** Unparsed citations, missing holdout manifests, or effect sizes within the evaluator's MDE noise band return a non-zero exit code automatically.
* **Synchronized Ecosystem Ledger:** **4,306 PASS** deterministic test and mutation assertions verified across the 6 public Apache-2.0 packages and 2 private research harnesses.
* **Scope & License Notice:** All 6 public repositories are open-source under **Apache-2.0**. `omega-citadel` and `omega-crucible` are unreleased **non-commercial personal R&D harnesses** (`All Rights Reserved`).
