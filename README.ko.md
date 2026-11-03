# AI 에이전트 및 LLM 파이프라인을 위한 결정론적 검증 게이트

> **"확률적(Stochastic) 모델의 거짓말을 또 다른 확률적 모델로 감사하지 마십시오."**  
> 외부 API 비용 없이 오프라인에서 **환각 `file:line` 인용**, **학습/평가 데이터 누수(Leakage)**, **LLM Judge 노이즈 착시**, **프롬프트 및 파라미터 과적합**을 배포 전에 차단하는 결정론적 Python 검증 스택입니다.

[![License: Apache-2.0](https://img.shields.io/badge/Public_Primitives-Apache_2.0-2ea44f?style=flat-square)](https://opensource.org/licenses/Apache-2.0)
[![Verification Ledger](https://img.shields.io/badge/Ecosystem_Ledger-4,306_PASS-0969da?style=flat-square)](#%EF%B8%8F-2%EA%B3%84%EC%B8%B5-%EC%8B%9C%EC%8A%A4%ED%85%9C-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98-%EA%B3%B5%EA%B0%9C-%ED%94%84%EB%A6%AC%EB%AF%B8%ED%8B%B0%EB%B8%8C--%EC%9D%B8%EB%A9%94%EB%AA%A8%EB%A6%AC-rd-%EC%BD%94%EC%96%B4)
[![Runtime Contract](https://img.shields.io/badge/Contract-Fail--Closed_%2F_Offline-d73a49?style=flat-square)](#%EF%B8%8F-2%EA%B3%84%EC%B8%B5-%EC%8B%9C%EC%8A%A4%ED%85%9C-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98-%EA%B3%B5%EA%B0%9C-%ED%94%84%EB%A6%AC%EB%AF%B8%ED%8B%B0%EB%B8%8C--%EC%9D%B8%EB%A9%94%EB%AA%A8%EB%A6%AC-rd-%EC%BD%94%EC%96%B4)
[![English Docs](https://img.shields.io/badge/Docs-English_README-7057ff?style=flat-square)](./README.md)

---

## ⚡ 10초 미리보기: 에이전트가 거짓 인용을 하거나 과적합될 때 무엇을 잡는가?

AI 코딩 에이전트나 프롬프트 최적화 도구가 보고하는 '통과(Green)' 점수는 실제 운영 환경에서 무너지는 경우가 많습니다. 에이전트의 자기보고를 믿는 대신 터미널과 CI에서 결정론적 오프라인 게이트를 실행하십시오:

```diff
$ antemortem gate --report agent_patch_claim.md
- [FAIL] Hallucinated Citation: src/core/validator.py:142 (실제 파일 길이는 98줄에 불과함)
- [FAIL] AST Symbol Mismatch: src/core/validator.py:40-75 구간 AST에서 `enforce_holdout_split` 심볼 미발견

$ mini-antemortem-cli lint eval_suite.yaml
- [FAIL] Trap Rule #04 (Split Leakage): 튜닝셋과 평가셋 간 정규화 샘플 14건 중복(누수) 적발

$ mini-omega-lock probe --judge-runs ab_eval_runs.json
- [BLOCKED] 주장된 프롬프트 성능 향상(+2.1%)이 Judge 자체 MDE 노이즈 플로어(±3.8%) 미만이므로 기각

$ omega-lock audit --study sweep_trials.json --holdout locked_holdout.json
+ [PASS] 미공개 홀드아웃 분할에서 워크포워드 일반화 생존 검증 완료 (SHA-256 무결성 영수증 발행)
```

---

## 🧭 시작 가이드: 겪고 있는 실패 유형에 맞는 게이트 선택

본 생태계의 6개 공개 패키지(`Apache-2.0`)는 AI 기반 소프트웨어 엔지니어링 및 LLM 평가 과정에서 발생하는 핵심 실패 모드를 각각 결정론적으로 차단합니다:

| 워크플로우에서 겪고 있는 문제 | 설치 및 실행 패키지 (`Apache-2.0`) | 결정론적 검증 메커니즘 | 지원 인터페이스 |
| :--- | :--- | :--- | :--- |
| **코딩 에이전트가 없는 `file:line`이나 함수를 거짓 인용함** | [**`antemortem-cli`**](https://github.com/hibou04-ops/antemortem-cli) | 로컬 파일 및 Python AST를 오프라인 대조하여 에이전트 보고서의 모든 코드 인용을 기계적으로 검증 | `CLI` · `MCP` · `GitHub Action` |
| **프롬프트 평가셋에 데이터 누수나 편향 함정이 숨어 있음** | [**`mini-antemortem-cli`**](https://github.com/hibou04-ops/mini-antemortem-cli) | 데이터 분할 누수, 판정 편향, 동질적 변이 등 **11가지 평가 함정 규칙**을 사전 정적 분석 | `CLI` · `CI Gate` |
| **A/B 프롬프트 점수 상승이 실제 개선인지 노이즈인지 불분명함** | [**`mini-omega-lock`**](https://github.com/hibou04-ops/mini-omega-lock) | 실험 전 평가자(Judge)의 최소 탐지 효과(**MDE**) 노이즈 플로어를 산출하여 통계적 착시 차단 | `CLI` · `Python API` |
| **튜닝셋에서는 고득점인데 실제 운영 입력에서 프롬프트가 깨짐** | [**`omegaprompt`**](https://github.com/hibou04-ops/omegaprompt) | `promptfoo` / `DSPy` 상단에서 미공개 홀드아웃 재검증을 강제하여 과적합 프롬프트 배포 차단 | `CLI` · `MCP` · `CI Gate` |
| **하이퍼파라미터/스윕 최적화(Optuna 등) 결과가 과적합됨** | [**`omega-lock`**](https://github.com/hibou04-ops/omega-lock) | 워크포워드 및 홀드아웃 생존 게이트를 적용하고 위변조 방지 SHA-256 감사 영수증 발행 | `CLI` · `Python API` |
| **취약한 코드 경로에 대해 재현 가능한 반례 탐색이 필요함** | [**`antemortem-structural-lab`**](https://github.com/hibou04-ops/antemortem-structural-lab) | 실험적 AST 구조 프로빙, 실행 피드백 수집 및 결정론적 반례(Counterexample) 리플레이 제공 | `CLI` · `Python API` |

---

## 🏗️ 2계층 시스템 아키텍처 (공개 프리미티브 + 인메모리 R&D 코어)

본 워크스페이스는 **독립적인 비영리 개인 시스템 엔지니어링 및 검증 방법론 연구실(Non-Commercial Personal R&D Lab)**로 운영되며, 2계층 아키텍처로 구성됩니다:

1. **Tier 1 — 공개 모듈형 프리미티브 (`6종 패키지 · Apache-2.0`):** 누구나 로컬 개발 환경, MCP 에이전트, CI 파이프라인에 즉시 도입할 수 있는 독립형 검증 도구입니다.
2. **Tier 2 — 비공개 연구 하네스 (`2종 하네스 · All Rights Reserved — 비영리 개인 연구 전용`):** 5단계 검증 파이프라인을 서브프로세스 오버헤드 없이 단일 인메모리 런타임으로 결속했을 때의 극한 지연시간 단축 및 적대적 변이 방어력을 연구하는 비공개 아키텍처 코어입니다. *(상업적 배포나 외부 서비스 목적이 아닌 순수 개인 연구 아티팩트입니다).*

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
│   ├─ 프롬프트/스키마 계약 + AST 근거 대조 + MDE 필터 + 홀드아웃 잠금 + SHA256│
│   └─ 실측 지연시간: 중앙값 14.2 ms – 19.3 ms (순차 CLI 대비 63.5x – 79.1x 가속)│
│                                                                              │
│ • omega-crucible (v1.0.3) — Adversarial Meta-Verification Harness            │
│   ├─ 데이터 누수, 스키마 이탈, 환각 인용 등 1,067건의 합성 적대적 변이 주입   │
│   └─ 실측 방어력: 1,067 / 1,067 (사살률 100.0%), 전체 코퍼스 163.77 ms 검증  │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 비공개 연구 하네스 벤치마크 및 검증 원장 요약

| 연구 하네스 | 버전 | 시스템 연구 범위 (비영리 개인 연구) | 실측 결정론적 벤치마크 |
| :--- | :--- | :--- | :--- |
| **`omega-citadel`** | `v1.1.2` | **인메모리 5단계 통합 검증 하네스:** 프롬프트/스키마 계약 검사, AST 근거 대조, MDE 노이즈 플로어 필터링, 홀드아웃 격리, SHA-256 영수증 서명을 단일 Python 프로세스로 통합하여 IPC/디스크 병목 제거. | 20회 반복 측정 중앙값 **14.2 ms – 19.3 ms** (5단계 순차 서브프로세스 실행 대비 **63.5배 – 79.1배** 가속) |
| **`omega-crucible`** | `v1.0.3` | **적대적 메타 검증 하네스:** 데이터 분할 누수, 스키마 이탈, 조작된 `file:line` 인용 등 합성 실패 모드를 주입하여 검증 게이트의 결정 경계를 스트레스 테스트. | **1,067 / 1,067 (100.0%)** 적대적 변이 사살률 달성 (**163.77 ms** 내 전체 코퍼스 인메모리 검증) |

---

## 🔒 검증 방법론 및 운영 원칙 (Verification Contract)

* **Fail-Closed 기본 원칙:** 검증되지 않은 코드 인용, 홀드아웃 매니페스트 누락, 또는 MDE 노이즈 플로어 이내의 무의미한 점수 변동은 즉시 비정상 종료 코드(Non-zero exit code)로 차단됩니다.
* **통합 검증 원장 (Verification Ledger):** 6개 공개 Apache-2.0 패키지와 2개 비공개 연구 하네스 전반에 걸쳐 총 **4,306 PASS**의 결정론적 테스트 및 변이 사살 어설션이 동기화되어 있습니다.
* **라이선스 및 연구 범위 고지:** 6개 공개 저장소는 **Apache-2.0** 오픈소스로 제공되며, `omega-citadel` 및 `omega-crucible`은 상업적 목적이 배제된 **순수 개인 연구용 비공개 하네스**(`All Rights Reserved`)입니다.
