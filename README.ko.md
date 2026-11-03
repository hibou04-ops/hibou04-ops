# AI 에이전트 및 LLM 파이프라인을 위한 결정론적 검증 게이트

> **"확률적 모델을 또 다른 확률적 모델로 감사하는 일을 멈추십시오."**<br>
> 배포 전에 **환각 `file:line` 인용**, **학습/평가 데이터 누수**, **LLM Judge 노이즈 착시**, **프롬프트·스윕 과적합**을 검사하는 오프라인·외부 API 비용 없는 검증 도구입니다.

![Public Primitives](https://img.shields.io/badge/Public_Primitives-6_Packages_(Apache_2.0)-2ea44f?style=flat-square)
![Verification Ledger](https://img.shields.io/badge/Historical_Ecosystem_Ledger-4,306_PASS-0969da?style=flat-square)
![Runtime Contract](https://img.shields.io/badge/Contract-Fail--Closed_%2F_Offline-d73a49?style=flat-square)
[![English Docs](https://img.shields.io/badge/Docs-English_README-7057ff?style=flat-square)](./README.md)

---

## ⚡ 10초 미리보기: 에이전트가 거짓 인용을 하거나 과적합될 때 무엇을 잡는가?

통과라는 자기보고 뒤에 잘못된 인용, 오염된 데이터 분할, 측정 노이즈보다 작은 개선이 숨어 있을 수 있습니다. 주장과 CI 판정 사이에 결정론적 검사를 두십시오.

아래 명령은 접힌 설정 예제의 재현 가능한 입력으로 실행했습니다. **색상 줄은 실제 fixture 결과의 요약이며, stdout 원문은 아닙니다.** `-` / `+`는 diff 강조 표시입니다. 한 번 설치하면 아래 검사는 API 키 없이 오프라인으로 실행됩니다.

```diff
$ python -m pip install antemortem==0.12.3 mini-antemortem-cli==0.11.3 mini-omega-lock==0.8.5 omegaprompt==2.2.5 omega-lock==0.3.12 antemortem-structural-lab==0.8.2

# 1. 파일 끝을 넘는 에이전트 인용을 잡고, 주장한 AST 심볼을 별도로 검사:
$ antemortem metrics agent_patch_claim.json --repo . --fail-over 0
- [FAIL] 허위 인용: src/core/validator.py:142 (실제 파일은 98행; exit 4)
$ python -c "from pathlib import Path; from antemortem.citations import verify_ast_symbol_grounding as verify; ok=verify(Path('src/core/validator.py'),40,75,'enforce_holdout_split'); print('AST symbol grounded:',ok); raise SystemExit(0 if ok else 1)"
- [FAIL] AST 심볼 불일치: 40–75행에 enforce_holdout_split 없음 (exit 1)

# 2. 평가 전에 숨어 있는 학습/평가 데이터 누수를 탐지:
$ mini-antemortem-cli check --target-provider openai --judge-provider anthropic --train tune_set.jsonl --test test_set.jsonl --rubric rubric.json --variants variants.json --validation-mode disjoint --fail-on-severity high
- [FAIL] 분할 누수: ID가 달라도 정규화한 평가 입력 14건이 학습 내용과 중복 (exit 1)

# 3. 저장된 delta로 경험적 노이즈 경계보다 작은 개선 주장을 차단:
$ python -c "import json; from mini_omega_lock import compute_empirical_noise_ceiling as ceiling; noise=ceiling(json.load(open('judge_deltas.json'))); gain=0.021; print(f'gain={gain:.3f}; empirical_noise_ceiling={noise:.3f}'); raise SystemExit(1 if gain <= noise else 0)"
- [BLOCKED] 주장된 개선 0.021 <= 경험적 노이즈 경계 0.038 (exit 1)

# 4. 제공한 홀드아웃 점수가 설정된 게이트를 통과하는지 검사:
$ omega-lock gate --train train_scores.json --holdout holdout_scores.json
+ [PASS] 제공된 홀드아웃 fixture가 게이트 통과 (exit 0)
```

**미리보기의 검증 계약:** `antemortem-cli`는 저장소 이름이며 배포명과 실행 명령은 `antemortem`입니다. AST 심볼 검증은 별도 Python helper입니다. mini-omega-lock의 `preflight`에는 추론 endpoint가 필요하므로 오프라인 예제는 Python API를 사용합니다. 경험적 노이즈 경계(`compute_minimum_detectable_effect`는 호환 alias)는 검정력 기반 MDE나 보정된 coverage 보장이 아닙니다. 명시적인 `gain <= noise` 비교는 이 예제의 기각 정책입니다. 마지막 PASS는 제공된 점수 배열에만 적용되며, 이 명령이 영수증을 발급하거나 독립 OOS 일반화를 입증하지는 않습니다.

<details>
<summary><strong>로컬에서 실패 재현 — 정확한 fixture 설정</strong></summary>

빈 작업 디렉터리에 아래 코드를 `make_showcase_fixture.py`로 저장하고 `python make_showcase_fixture.py`, 이어서 `cd showcase-fixture`를 실행하십시오. 위 명령은 `$`와 색상 결과 줄을 제외하고 한 줄씩 실행합니다. 의도적으로 실패하게 만든 사례는 비정상 종료가 정상 결과입니다. 정적 linter의 provider 이름은 설정 메타데이터이며, 이 검사는 provider 요청을 보내지 않습니다.

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

## 🧭 시작 가이드: 겪고 있는 실패 유형별 게이트 선택

설치할 패키지를 추측하지 말고 AI 워크플로우의 실패 유형에 맞춰 선택하십시오. 아래 **고정 버전 6종은 모두 PyPI에 배포**되어 있으며 **Apache-2.0**을 사용합니다. 설치명·버전은 아티팩트 원장과 일치합니다.

| 워크플로우에서 겪고 있는 문제 | 설치 및 실행 (`PyPI · Apache-2.0`) | 결정론적 검사 범위 | 인터페이스 |
| :--- | :--- | :--- | :--- |
| **코딩 에이전트가 없는 `file:line`이나 심볼을 인용함** | [**`antemortem-cli`**](https://github.com/hibou04-ops/antemortem-cli) · `pip install antemortem==0.12.3` | 로컬 소스의 실제 행 범위와 인용 대조; 별도 Python AST 심볼 helper | `CLI` · `Python API` · `MCP` · `GitHub Action` |
| **평가 설정에 누수나 편향 함정이 숨어 있음** | [**`mini-antemortem-cli`**](https://github.com/hibou04-ops/mini-antemortem-cli) · `pip install mini-antemortem-cli==0.11.3` | 분할 중복, 편향, 동질적 변이 등을 포함하는 **평가 함정 규칙 11종**의 사전 정적 검사 | `CLI` · `CI Gate` |
| **A/B 프롬프트 점수 상승이 Judge 노이즈일 수 있음** | [**`mini-omega-lock`**](https://github.com/hibou04-ops/mini-omega-lock) · `pip install mini-omega-lock==0.8.5` | 측정 delta의 경험적 노이즈 경계; endpoint를 이용하는 측정 preflight | `CLI: preflight` · `Python API` |
| **개발셋에 맞춘 프롬프트가 홀드아웃 검사에서 실패함** | [**`omegaprompt`**](https://github.com/hibou04-ops/omegaprompt) · `pip install omegaprompt==2.2.5` | 프롬프트 평가·보정, 설정 및 홀드아웃 분할 검사; 문서화된 `promptfoo` / `DSPy` 워크플로우 | `CLI` · `MCP` · `CI Gate` |
| **하이퍼파라미터·스윕 최적화(Optuna 등)가 과적합됨** | [**`omega-lock`**](https://github.com/hibou04-ops/omega-lock) · `pip install omega-lock==0.3.12` | 워크포워드·홀드아웃 검사, 점수 저하 방어 및 별도의 SHA-256 무결성 영수증 워크플로우 | `CLI` · `Python API` |
| **취약한 코드 경로에 반례 프로브가 필요함** | [**`antemortem-structural-lab`**](https://github.com/hibou04-ops/antemortem-structural-lab) · `pip install antemortem-structural-lab==0.8.2` | 실험적 AST 구조 프로빙, 계약·Oracle 기반 실행 피드백 및 결정론적 반례 재현 | `CLI` · `Python API` |

설치 후 로컬 검증은 모델 서비스 없이 실행할 수 있습니다. 평가·추론 경로는 설정된 provider를 사용할 수 있습니다. MCP는 각 저장소에 명시된 extra와 지원 entry point를 사용하십시오.

---

## 🏗️ 2계층 시스템 아키텍처 (공개 프리미티브 + 인메모리 R&D 코어)

이 워크스페이스는 **독립적인 개인 시스템 엔지니어링 및 검증 연구실**이며, 2계층으로 구성됩니다.

* **Tier 1 — 공개 모듈형 프리미티브 (`6종 패키지 · Apache-2.0 · PyPI Live`):** 개발 워크플로우와 CI에 사용할 수 있는 독립형 CLI, Python 및 일부 MCP 도구입니다.
* **Tier 2 — 비공개 연구 하네스 (`2종 하네스 · All Rights Reserved — 개인 R&D`):** 인메모리 파이프라인 통합(`omega-citadel`)과 적대적 경계 변이 검사(`omega-crucible`)를 연구하는 비공개 시스템입니다.

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

| 비공개 연구 하네스 | 아티팩트 고정 버전 | 시스템 연구 범위 (개인 R&D) | 실측 결정론적 벤치마크 |
| :--- | :--- | :--- | :--- |
| **`omega-citadel`** | `v1.1.2` | **인메모리 5단계 통합 검증 엔진:** 프롬프트·스키마 계약, AST 근거 대조, 노이즈 필터, 홀드아웃 잠금, SHA-256 무결성 영수증을 단일 Python 런타임으로 통합합니다. | 과거 20쌍 측정: **v1.0.1**에서 **14.2 ms / 79.1x**, **v1.1.1**에서 **19.3654 ms / 63.50x**. 과거의 **14.2 ms – 19.3 ms** 표시는 뒤 중앙값을 절삭한 요약입니다. |
| **`omega-crucible`** | `v1.0.3` | **적대적 메타 검증 하네스:** 합성 데이터 분할 누수, 스키마 이탈, 환각 `file:line` 인용을 하네스 자체 방어기에 주입합니다. | 과거 **v1.0.0:** **1,067 / 1,067 (100.0%)** 합성 변이 검출, **163.77 ms**. 별도의 **v1.0.3 clean-wheel** 기록: **1,067 / 1,067**, **173.2306 ms**. |

**벤치마크 계약:** Windows x64 / CPython 3.12.10의 고정 입력 연구 하네스입니다. 지연시간 비율은 5개 cold 서브프로세스 단계와 단일 warm 인메모리 프로세스를 비교하며, 프로세스 시작 및 파싱 비용을 포함합니다. 공개 6개 패키지의 실제 CLI 연결 실행을 측정한 결과는 아닙니다. 과거 수치는 원래 측정 버전에 귀속되며, 유한한 작성 변이 코퍼스는 보편적인 견고성의 증거가 아닙니다. 비공개 소스 아티팩트는 비공개로 유지됩니다.

**연구 상태:** Structural Lab과 비공개 연구는 **`M0_HOLD`**, utility와 독립 OOS는 **`NOT_ESTABLISHED`**를 유지합니다. 엔지니어링 PASS 수, 행·AST 검사, 무결성 해시가 자연어 주장의 진실성이나 실제 환경의 일반화를 입증하지는 않습니다.

---

## 🔐 암호학적 아티팩트 무결성 증명 및 SHA-256 해시 원장

`omega-citadel`과 `omega-crucible`은 **`All Rights Reserved` 개인 R&D 비공개 아티팩트**입니다. 아래에 기록된 wheel, 소스 압축 파일 및 컴파일된 바이트코드 `locked-runtime.zip` 번들은 접근 권한이 있는 검증자를 위한 고정 해시입니다. 공개 영역에는 6개 버전의 PyPI 파일을 각각 2개씩 기록했습니다. **아티팩트 해시 17개**를 실제 바이트와 대조했고, 공개 해시는 현재 PyPI 메타데이터와도 일치합니다(**2026-11-03**).

다음 블록을 그대로 복사해 해당 파일들과 같은 디렉터리의 `SHA256SUMS.txt`로 저장하십시오.

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

**누락된 소스 압축 파일:** `omega_crucible-1.0.3.tar.gz`는 검증한 로컬 매니페스트와 v1.0.3 GitHub Release의 첨부 파일 17개에 없습니다. 확인된 SHA-256이 없어 checksum 블록에서 제외했습니다. 비공개 파일 5개는 표준 wheel·소스 파일명의 `omega_` 접두부를 포함해 실제 기록된 이름을 그대로 유지합니다.

* **로컬 아티팩트 무결성 검증:** 나열된 파일을 모두 보유한 권한 있는 검증자는 다음 명령을 실행합니다.

  ```bash
  sha256sum -c SHA256SUMS.txt
  ```

  공개 파일만 검증하려면 Tier 1의 행만 별도 checksum 파일로 복사하십시오. 해시는 이 원장과의 바이트 동일성을 확인하며, 전자서명이나 아티팩트 안전성의 증거는 아닙니다.
* **공개 파일 다운로드:** [antemortem 0.12.3](https://pypi.org/project/antemortem/0.12.3/#files) · [mini-antemortem-cli 0.11.3](https://pypi.org/project/mini-antemortem-cli/0.11.3/#files) · [mini-omega-lock 0.8.5](https://pypi.org/project/mini-omega-lock/0.8.5/#files) · [omegaprompt 2.2.5](https://pypi.org/project/omegaprompt/2.2.5/#files) · [omega-lock 0.3.12](https://pypi.org/project/omega-lock/0.3.12/#files) · [antemortem-structural-lab 0.8.2](https://pypi.org/project/antemortem-structural-lab/0.8.2/#files).
* **생태계 검증 원장:** **4,306 PASS**는 공개 패키지 검사, Structural Lab 및 비공개 하네스 실행을 아우르는 과거 실행 누계(**2,160 + 1,561 + 230 + 355**)입니다. 이 페이지의 8개 고정 버전을 새로 동기 실행한 테스트 총계가 아닙니다. 위의 fail-closed 예제는 명시적인 기각 종료 코드를 유지합니다.
* **English Documentation:** [README.md](./README.md)
