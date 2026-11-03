# Deterministic Verification Gates for AI Agents & LLM Pipelines

Python tools for checking local code citations, evaluation configurations, measurement noise, holdout decisions, and reproducible counterexamples. Start with an executable fixture, inspect its output, then apply the documented contract to your own inputs.

[![License: Apache-2.0](https://img.shields.io/badge/Public_Packages-Apache_2.0-2ea44f?style=flat-square)](https://opensource.org/licenses/Apache-2.0)
[![Six public packages](https://img.shields.io/badge/Public_Packages-6-0969da?style=flat-square)](#packages)
[![Research scope](https://img.shields.io/badge/Research-Personal_R%26D-7057ff?style=flat-square)](#architecture)
[한국어](./README.ko.md)

## Run a bundled failure demonstration

Both demos ship in the installed packages. They use authored fixtures and print summaries of real checks; they do not measure model quality or establish generalization. In the previews, `-` and `+` are diff highlighting, not stdout characters.

```bash
python -m pip install mini-antemortem-cli==0.11.3
python -m mini_antemortem_cli.quickstart_demo
```

```diff
- [FAIL] overlapping train/test IDs: gate exit 1
+ [PASS] disjoint train/test IDs: gate exit 0
+ [PASS] checked 11 local traps per fixture
```

The configuration CLI uses `check` and `list-traps`. Its rule mapping includes `OMG008 = train_test_id_overlap`, `OMG003 = variants_homogeneous`, and `OMG007 = no_held_out_slice`.

```bash
python -m pip install antemortem-structural-lab==0.8.2
python -m tomolab.quickstart_demo
```

```diff
- [COUNTEREXAMPLE] point=0 actual=1 expected=0 confirmations=3
+ [PASS] fixed point=0 actual=0 expected=0
+ [PASS] replay matched all 3 failure observations
```

This replay compares a known failing function with an explicit reference and a repaired function. It does not demonstrate autonomous defect discovery or prove correctness.

<a id="packages"></a>
## Choose a package

All six public packages use Apache-2.0. Each repository documents its actual CLI contract and executable quickstart.

| Repository | Install from PyPI | Check the CLI | Scope |
|---|---|---|---|
| [antemortem-cli](https://github.com/hibou04-ops/antemortem-cli) | `pip install antemortem` | `uvx antemortem --help` | Local file-and-line evidence, supported AST checks, and CI decisions |
| [mini-antemortem-cli](https://github.com/hibou04-ops/mini-antemortem-cli) | `pip install mini-antemortem-cli` | `uvx mini-antemortem-cli --help` | Eleven deterministic evaluation-configuration checks, `OMG001–OMG011` |
| [mini-omega-lock](https://github.com/hibou04-ops/mini-omega-lock) | `pip install mini-omega-lock` | `uvx --from mini-omega-lock preflight --help` | Measurement-reliability probes and an empirical noise bound |
| [omegaprompt](https://github.com/hibou04-ops/omegaprompt) | `pip install omegaprompt` | `uvx omegaprompt --help` | Prompt evaluation, calibration, and configuration checks |
| [omega-lock](https://github.com/hibou04-ops/omega-lock) | `pip install omega-lock` | `uvx omega-lock --help` | Holdout auditing, score-degradation checks, and integrity receipts |
| [antemortem-structural-lab](https://github.com/hibou04-ops/antemortem-structural-lab) | `pip install antemortem-structural-lab` | `uvx antemortem-structural-lab --help` | Experimental contract/Oracle-driven counterexamples and replay through `tomolab` |

The repository `antemortem-cli` publishes the distribution `antemortem`. The mini-omega-lock command is `preflight`. Four repositories use source-checkout quickstart helpers; their READMEs include clone steps. The two module demos above are included in their wheels.

For MCP setup, follow each repository's documented extras and entry points. Local verification can run without a model service after installation; evaluation and inference tools can call configured providers.

<a id="architecture"></a>
## Public tools and private personal research

This independent personal software-engineering research workspace has two scopes:

| Scope | Components | Purpose |
|---|---|---|
| Public packages · Apache-2.0 | The six packages above | Separately usable CLI, Python, and selected MCP interfaces |
| Private research harnesses · All Rights Reserved | `omega-citadel` v1.1.2; `omega-crucible` v1.0.3 | In-memory pipeline measurements and synthetic mutation experiments |

### Historical benchmark record

These measurements retain their original versions. They are not new measurements of the current private pins.

| Recorded version | Workload and result | Interpretation |
|---|---|---|
| `omega-citadel` v1.0.1 | 20 alternating paired runs: warm median **14.2 ms**, cold **1,122.6 ms**, **79.1x** ratio | Fixed-payload five-subprocess baseline versus one warm Python process |
| `omega-citadel` v1.1.1 | Warm median **19.3654 ms**, cold **1,229.61405 ms**, **63.50x** ratio | Same stated comparison design; the old display `14.2 ms – 19.3 ms` truncates the later value |
| `omega-crucible` v1.0.0 | Seed-42 corpus: **1,067 / 1,067** synthetic mutations detected in **163.77 ms**, healthy controls passing | Finite authored corpus against the harness's own defenders |
| `omega-crucible` v1.0.3 | Separate clean-wheel record: **1,067 / 1,067** detections in **173.2306 ms** | A distinct receipt; not the source of the 163.77 ms figure |

The latency records use the stated Windows x64 / CPython 3.12.10 environment. They do not compare the public products' actual CLI chains. Process startup and repeated parsing contribute to the ratio. Original private artifacts remain private; these summaries are not an independently reproduced public benchmark.

## Evidence boundaries

- Paths, line ranges, AST matches, and hashes establish only the checks stated by the tool, not the truth of a natural-language claim.
- The empirical noise bound is not a power-based MDE estimate or a calibrated 95% coverage guarantee. Successful preflight does not authorize deployment.
- Holdout results depend on the supplied split, contract, and measurement procedure; integrity receipts do not prove real-world generalization.
- Structural Lab and private research retain `M0_HOLD`; utility and independent OOS remain `NOT_ESTABLISHED`. Passing engineering tests does not change those research statuses.
