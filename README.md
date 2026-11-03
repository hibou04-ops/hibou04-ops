# Deterministic Verification Gates for AI Agents & LLM Pipelines

> **"Stop auditing stochastic models with more stochastic models."**<br>
> Offline, zero-API-cost checks for **hallucinated `file:line` citations**, **train/test split leakage**, **LLM-judge noise illusions**, and **prompt/sweep overfitting** before they ship.

![Public Primitives](https://img.shields.io/badge/Public_Primitives-6_Packages_(Apache_2.0)-2ea44f?style=flat-square)
![Verification Ledger](https://img.shields.io/badge/Historical_Ecosystem_Ledger-4,306_PASS-0969da?style=flat-square)
![Runtime Contract](https://img.shields.io/badge/Contract-Fail--Closed_%2F_Offline-d73a49?style=flat-square)
[![Korean Docs](https://img.shields.io/badge/Docs-%ED%95%9C%EA%B5%AD%EC%96%B4_README-7057ff?style=flat-square)](./README.ko.md)

---

## ⚡ 10-Second Look: What Happens When an Agent Lies or Overfits?

A green self-report can hide a broken citation, a contaminated split, or a gain smaller than measurement noise. Put a deterministic check between the claim and your CI decision:

The commands below were exercised with the reproducible inputs in the expandable setup. **Colored lines summarize actual fixture results; they are not verbatim stdout.** `-` / `+` are diff highlighting. Install once; the checks shown then run offline without API keys.

```diff
$ python -m pip install antemortem==0.12.3 mini-antemortem-cli==0.11.3 mini-omega-lock==0.8.5 omegaprompt==2.2.5 omega-lock==0.3.12 antemortem-structural-lab==0.8.2

# 1. Catch a coding agent citing a line beyond EOF, then check its claimed AST symbol:
$ antemortem metrics agent_patch_claim.json --repo . --fail-over 0
- [FAIL] Fabricated Citation: src/core/validator.py:142 (file has only 98 lines; exit 4)
$ python -c "from pathlib import Path; from antemortem.citations import verify_ast_symbol_grounding as verify; ok=verify(Path('src/core/validator.py'),40,75,'enforce_holdout_split'); print('AST symbol grounded:',ok); raise SystemExit(0 if ok else 1)"
- [FAIL] AST Symbol Mismatch: enforce_holdout_split is absent from lines 40–75 (exit 1)

# 2. Catch silent train/test leakage before evaluation:
$ mini-antemortem-cli check --target-provider openai --judge-provider anthropic --train tune_set.jsonl --test test_set.jsonl --rubric rubric.json --variants variants.json --validation-mode disjoint --fail-on-severity high
- [FAIL] Split Leakage: 14 normalized test inputs duplicate train content despite disjoint IDs (exit 1)

# 3. Block a claimed gain below the measured empirical noise ceiling, using stored deltas:
$ python -c "import json; from mini_omega_lock import compute_empirical_noise_ceiling as ceiling; noise=ceiling(json.load(open('judge_deltas.json'))); gain=0.021; print(f'gain={gain:.3f}; empirical_noise_ceiling={noise:.3f}'); raise SystemExit(1 if gain <= noise else 0)"
- [BLOCKED] Claimed gain 0.021 <= empirical noise ceiling 0.038 (exit 1)

# 4. Require the supplied holdout scores to survive the configured gate:
$ omega-lock gate --train train_scores.json --holdout holdout_scores.json
+ [PASS] Supplied holdout fixture passed the gate (exit 0)
```

**The contract behind the preview:** `antemortem-cli` is the repository; `antemortem` is its distribution and executable. AST symbol grounding is a separate Python helper. The mini-omega-lock offline command uses its Python API because `preflight` requires an inference endpoint. Its empirical noise ceiling (`compute_minimum_detectable_effect` is a compatibility alias) is not a power-based MDE or a calibrated coverage guarantee. The explicit `gain <= noise` comparison is the example's rejection policy. The final PASS applies to the supplied score arrays; that command does not issue a receipt or establish independent OOS generalization.

<details>
<summary><strong>Reproduce these failures locally — exact fixture setup</strong></summary>

Save the following as `make_showcase_fixture.py` in an empty working directory and run `python make_showcase_fixture.py`, then `cd showcase-fixture`. Run the commands above one at a time, without copying `$` or colored summary lines. Nonzero exits are expected for the deliberately failing cases. Provider names in the static linter are configuration metadata; this check makes no provider requests.

```python
import json
from pathlib import Path

root = Path("showcase-fixture")
root.mkdir(exist_ok=False)
(root / "src/core").mkdir(parents=True)
(root / "src/core/validator.py").write_text("value = 1\n" * 98, encoding="utf-8")
data = {
    "agent_patch_claim.json": {
        "classifications": [{"id": "t1", "label": "GHOST", "citation": "src/core/validator.py:142"}],
        "decision": "SAFE_TO_PROCEED",
    },
    "rubric.json": {"dimensions": [{"name": "correctness", "description": "Correct answer", "weight": 1.0, "scale": [1, 5]}]},
    "variants.json": {"system_prompts": ["Return a concise factual answer.", "Explain all reasoning before responding."]},
    "judge_deltas.json": [0.038, -0.038, 0.038, -0.038],
    "train_scores.json": [0.6, 0.7, 0.8, 0.9],
    "holdout_scores.json": [0.59, 0.69, 0.79, 0.89],
}
for name, value in data.items():
    (root / name).write_text(json.dumps(value), encoding="utf-8")
for name, prefix in [("tune_set.jsonl", "train"), ("test_set.jsonl", "test")]:
    rows = [{"id": f"{prefix}-{i}", "input": f"Question {i}" if prefix == "train" else f"  Question   {i}  ", "reference": str(i)} for i in range(14)]
    (root / name).write_text("\n".join(json.dumps(row) for row in rows) + "\n", encoding="utf-8")
```

</details>

---

## 🧭 Start Here: Pick the Gate for Your Failure Mode

Don't guess which package to install—match the failure in your AI workflow. These **six pinned releases are on PyPI** and use **Apache-2.0**; installation names and versions below match the artifact ledger.

| If your AI workflow suffers from... | Install & Run (`PyPI · Apache-2.0`) | What it checks deterministically | Interfaces |
| :--- | :--- | :--- | :--- |
| **Coding agent hallucinating `file:line` or symbols** | [**`antemortem-cli`**](https://github.com/hibou04-ops/antemortem-cli) · `pip install antemortem==0.12.3` | Physical line-range citation verification against local sources; separate Python AST symbol helper | `CLI` · `Python API` · `MCP` · `GitHub Action` |
| **Silent leakage or rigged prompt-eval setups** | [**`mini-antemortem-cli`**](https://github.com/hibou04-ops/mini-antemortem-cli) · `pip install mini-antemortem-cli==0.11.3` | Preflight static linter with **11 evaluation trap rules**, including split overlap, bias, and homogeneous variants | `CLI` · `CI Gate` |
| **A/B prompt "wins" that may be judge noise** | [**`mini-omega-lock`**](https://github.com/hibou04-ops/mini-omega-lock) · `pip install mini-omega-lock==0.8.5` | Empirical noise ceiling from measured deltas; endpoint-based measurement preflight | `CLI: preflight` · `Python API` |
| **Prompts that fit development sets but fail held-out checks** | [**`omegaprompt`**](https://github.com/hibou04-ops/omegaprompt) · `pip install omegaprompt==2.2.5` | Prompt evaluation, calibration, configuration and held-out split checks; documented `promptfoo` / `DSPy` workflows | `CLI` · `MCP` · `CI Gate` |
| **Hyperparameter / sweep overfitting (Optuna, etc.)** | [**`omega-lock`**](https://github.com/hibou04-ops/omega-lock) · `pip install omega-lock==0.3.12` | Walk-forward / holdout checks, score-degradation guards, and separate SHA-256 integrity-receipt workflows | `CLI` · `Python API` |
| **Fragile code paths needing counterexample probes** | [**`antemortem-structural-lab`**](https://github.com/hibou04-ops/antemortem-structural-lab) · `pip install antemortem-structural-lab==0.8.2` | Experimental structural AST probing, contract/Oracle-driven execution feedback, and deterministic counterexample replay | `CLI` · `Python API` |

Local verification can run without a model service after installation. Evaluation and inference paths may use configured providers. For MCP, install the repository's documented extras and use its supported entry point.

---

## 🏗️ Two-Tier Systems Architecture (Open Primitives + In-Memory R&D Core)

This workspace is an **independent personal systems engineering & verification research lab** structured into two tiers:

* **Tier 1 — Public Modular Primitives (`6 Packages · Apache-2.0 · PyPI Live`):** Standalone CLI, Python and selected MCP tools for developer workflows and CI pipelines.
* **Tier 2 — Private Research Harnesses (`2 Harnesses · All Rights Reserved — Personal R&D`):** Closed-source systems research on in-memory pipeline fusion (`omega-citadel`) and adversarial boundary mutation testing (`omega-crucible`).

```text
┌────────────────────────────────────────────────────────────────────────────────────┐
│ TIER 1: PUBLIC MODULAR PRIMITIVES (Apache-2.0 · PyPI · CLI / API / MCP / CI)       │
├───────────────────────────┬───────────────────────────┬────────────────────────────┤
│ mini-antemortem-cli       │ antemortem-cli            │ antemortem-structural-lab  │
│ 11 Eval Trap Rules        │ File:Line + AST helper    │ Counterexample Probes      │
├───────────────────────────┼───────────────────────────┼────────────────────────────┤
│ mini-omega-lock           │ omegaprompt               │ omega-lock                 │
│ Empirical Noise Ceiling   │ Prompt Evaluation Gates   │ Walk-Forward / Holdout     │
└───────────────────────────┴───────────────────────────┴────────────────────────────┘
             │                            │                            │
             ▼       [ Zero-IPC / Zero-Subprocess Research Fusion ]      ▼
┌────────────────────────────────────────────────────────────────────────────────────┐
│ TIER 2: PRIVATE RESEARCH HARNESSES (Personal R&D · All Rights Reserved)            │
├────────────────────────────────────────────────────────────────────────────────────┤
│ • omega-citadel (v1.1.2 pin) — In-Memory 5-Stage Governance Engine                 │
│   ├─ Schema + AST grounding + noise filter + holdout lock + SHA-256 receipts       │
│   └─ Historical medians: 14.2 / 19.3654 ms; 79.1x / 63.50x (v1.0.1 / v1.1.1)       │
│                                                                                    │
│ • omega-crucible (v1.0.3 pin) — Adversarial Meta-Verification Harness              │
│   ├─ 1,067 synthetic leakage, drift, and citation-hallucination mutations          │
│   └─ 1,067 / 1,067 detected: 163.77 ms (v1.0.0); 173.2306 ms (v1.0.3)              │
└────────────────────────────────────────────────────────────────────────────────────┘
```

| Private R&D Harness | Artifact Pin | Systems Research Scope (Personal R&D) | Measured Deterministic Benchmark |
| :--- | :--- | :--- | :--- |
| **`omega-citadel`** | `v1.1.2` | **In-Memory 5-Stage Governance Engine:** Combines prompt/schema contracts, AST grounding, noise filtering, holdout locking, and SHA-256 integrity receipts in one Python runtime. | Historical 20-pair records: **14.2 ms / 79.1x** at **v1.0.1**; **19.3654 ms / 63.50x** at **v1.1.1**. The earlier display **14.2 ms – 19.3 ms** truncates the latter median. |
| **`omega-crucible`** | `v1.0.3` | **Adversarial Meta-Verification Harness:** Injects synthetic split leakage, schema drift, and hallucinated `file:line` citations into the harness's defenders. | Historical **v1.0.0:** **1,067 / 1,067 (100.0%)** synthetic mutation detections in **163.77 ms**. Separate **v1.0.3 clean-wheel** record: **1,067 / 1,067** in **173.2306 ms**. |

**Benchmark contract:** Windows x64 / CPython 3.12.10; fixed-payload research harnesses. The latency ratio compares five cold subprocess stages with one warm in-memory process, including process-start and parsing overhead. It is not a measurement of the six public packages' actual CLI chain. Historical measurements retain their original versions; the finite authored mutation corpus is not evidence of universal robustness. Private source artifacts remain private.

**Research state:** Structural Lab and the private research retain **`M0_HOLD`**; utility and independent OOS remain **`NOT_ESTABLISHED`**. Engineering PASS counts, line/AST checks and integrity hashes do not establish natural-language truth or real-world generalization.

---

## 🔐 Cryptographic Artifact Attestation & SHA-256 Ledger

`omega-citadel` and `omega-crucible` remain private **`All Rights Reserved` personal R&D artifacts**. The recorded wheels, source archives and compiled-bytecode `locked-runtime.zip` bundles below are pinned for authorized integrity checks. The public section pins both PyPI files for each of the six releases. **17 artifact hashes** were checked against actual bytes; public hashes also match current PyPI metadata (**2026-11-03**).

Copy the following block verbatim into `SHA256SUMS.txt` beside the corresponding files:

```text
# Tier 2: Private personal R&D release artifacts
79e59e88f69d0f976ff48be42a0a0417c20f2aac95a41d9a075c2d3825a0b8fb  omega_citadel-1.1.2-py3-none-any.whl
fbf374e0ea9f8888bc00c64bf57fb13062ee57908df68150c7a2ccddc4ac45bc  omega_citadel-1.1.2.tar.gz
2604f342a2d375a4fd643cd510853524a534587a750d127edf8431f52849174f  omega-citadel-1.1.2-locked-runtime.zip
9b159fe454f591a0a5459a7c48a30570ebe4b2a2f7d7c1ca52090a0b9345bc82  omega_crucible-1.0.3-py3-none-any.whl
1f5282884bd2316e5cfa4a8733ed417d395c5ea71c60fc20ab70fec735b7cc34  omega-crucible-1.0.3-locked-runtime.zip

# Tier 1: Six public PyPI releases (wheel + source archive)
7eedb82f97091256f53b3856486f4338b2231bc1bda3e770411de8177a039801  antemortem-0.12.3-py3-none-any.whl
81238122c30c0008144554140ad57acee706cebfb4dca9622c9d362dcdacd7bd  antemortem-0.12.3.tar.gz
7990a0e7ba76748657242bc1dcc81d4290b8e3a4c45b2e7da2665bcccd0ead84  mini_antemortem_cli-0.11.3-py3-none-any.whl
51878281a803e26e87cff7985e316bd2b58ea7e3f2de625e70c94f2cd0c8c3e5  mini_antemortem_cli-0.11.3.tar.gz
c3145d703948deaf21f2c9305cddca7211af22b0fb2810cfe2a1c25bf44914e9  mini_omega_lock-0.8.5-py3-none-any.whl
3b22d7ce3261793f488e1bf46970a7734e7895ee78d1756a740ce1deb1155eef  mini_omega_lock-0.8.5.tar.gz
127eb50b1a7b100a932c2e893dc8af10ffb2a4bdd18bd028a19ca5222815d43b  omegaprompt-2.2.5-py3-none-any.whl
c85dbf97cbe12761ec3a88022ba7c257d059a4fddf1e9bb82677e11dfbc81e92  omegaprompt-2.2.5.tar.gz
6dd698c24954d6fd5a0292f77569b8ecd8b0dbfe7f8cfdc553d22a4f5c54c180  omega_lock-0.3.12-py3-none-any.whl
b5c1957749a93ee697f52a7401e26311ce5410874c55f6a252018d79395295de  omega_lock-0.3.12.tar.gz
8f7fa2662f5adc97b8670031caa1fb332911571d910857145d6bb5613ade001e  antemortem_structural_lab-0.8.2-py3-none-any.whl
1053aef3e97fecfb22151995c4e1800602e1c6b730fef1431ade39861615c1ff  antemortem_structural_lab-0.8.2.tar.gz
```

* **Verify artifact integrity locally:** Authorized auditors with all listed files run:

  ```bash
  sha256sum -c SHA256SUMS.txt
  ```

  For public artifacts only, copy the Tier 1 lines into a separate checksum file. Checksums establish byte identity against this ledger; they are not digital signatures or evidence of artifact safety.
* **Public download sources:** [antemortem 0.12.3](https://pypi.org/project/antemortem/0.12.3/#files) · [mini-antemortem-cli 0.11.3](https://pypi.org/project/mini-antemortem-cli/0.11.3/#files) · [mini-omega-lock 0.8.5](https://pypi.org/project/mini-omega-lock/0.8.5/#files) · [omegaprompt 2.2.5](https://pypi.org/project/omegaprompt/2.2.5/#files) · [omega-lock 0.3.12](https://pypi.org/project/omega-lock/0.3.12/#files) · [antemortem-structural-lab 0.8.2](https://pypi.org/project/antemortem-structural-lab/0.8.2/#files).
* **Ecosystem Verification Ledger:** **4,306 PASS** is the historical cross-run ledger (**2,160 + 1,561 + 230 + 355**) spanning public-package checks, Structural Lab, and private-harness runs. It is not a fresh synchronized test total for the eight artifact pins on this page. The fail-closed examples above preserve explicit rejection exits.
* **한국어 문서 (Korean Documentation):** [README.ko.md](./README.ko.md)
