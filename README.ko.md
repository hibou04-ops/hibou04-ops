# AI 에이전트 및 LLM 파이프라인을 위한 결정론적 검증 게이트

로컬 코드 인용, 평가 설정, 측정 노이즈, 홀드아웃 판정, 재현 가능한 반례를 검사하는 Python 도구입니다. 실행 가능한 예제로 출력을 확인한 뒤, 문서에 명시된 계약을 자신의 입력에 적용할 수 있습니다.

[![License: Apache-2.0](https://img.shields.io/badge/Public_Packages-Apache_2.0-2ea44f?style=flat-square)](https://opensource.org/licenses/Apache-2.0)
[![Six public packages](https://img.shields.io/badge/Public_Packages-6-0969da?style=flat-square)](#packages)
[![Research scope](https://img.shields.io/badge/Research-Personal_R%26D-7057ff?style=flat-square)](#architecture)
[English](./README.md)

## 설치된 패키지로 실패 예제 실행

두 데모 모두 설치된 패키지에 포함됩니다. 작성된 fixture에 실제 검사를 적용한 요약이며, 모델 품질이나 일반화를 측정하지 않습니다. 아래 출력의 `-`와 `+`는 diff 강조 표시이며 stdout 문자는 아닙니다.

```bash
python -m pip install mini-antemortem-cli==0.11.3
python -m mini_antemortem_cli.quickstart_demo
```

```diff
- [FAIL] overlapping train/test IDs: gate exit 1
+ [PASS] disjoint train/test IDs: gate exit 0
+ [PASS] checked 11 local traps per fixture
```

설정 검사 CLI의 명령은 `check`와 `list-traps`입니다. 실제 규칙 매핑은 `OMG008 = train_test_id_overlap`, `OMG003 = variants_homogeneous`, `OMG007 = no_held_out_slice`를 포함합니다.

```bash
python -m pip install antemortem-structural-lab==0.8.2
python -m tomolab.quickstart_demo
```

```diff
- [COUNTEREXAMPLE] point=0 actual=1 expected=0 confirmations=3
+ [PASS] fixed point=0 actual=0 expected=0
+ [PASS] replay matched all 3 failure observations
```

알려진 실패 함수, 명시적인 참조 함수, 수정 함수를 비교하는 재현 예제입니다. 자율적 결함 발견이나 완전한 정확성을 입증하지 않습니다.

<a id="packages"></a>
## 패키지 선택

공개 패키지 6종의 라이선스는 Apache-2.0입니다. 각 저장소에서 실제 CLI 계약과 실행 가능한 quickstart를 확인할 수 있습니다.

| 저장소 | PyPI 설치 | CLI 확인 | 검사 범위 |
|---|---|---|---|
| [antemortem-cli](https://github.com/hibou04-ops/antemortem-cli) | `pip install antemortem` | `uvx antemortem --help` | 로컬 파일·행 근거, 지원하는 AST 검사, CI 판정 |
| [mini-antemortem-cli](https://github.com/hibou04-ops/mini-antemortem-cli) | `pip install mini-antemortem-cli` | `uvx mini-antemortem-cli --help` | 평가 설정의 결정론적 검사 11종, `OMG001–OMG011` |
| [mini-omega-lock](https://github.com/hibou04-ops/mini-omega-lock) | `pip install mini-omega-lock` | `uvx --from mini-omega-lock preflight --help` | 측정 신뢰성 프로브와 경험적 노이즈 경계 |
| [omegaprompt](https://github.com/hibou04-ops/omegaprompt) | `pip install omegaprompt` | `uvx omegaprompt --help` | 프롬프트 평가·보정 및 설정 검사 |
| [omega-lock](https://github.com/hibou04-ops/omega-lock) | `pip install omega-lock` | `uvx omega-lock --help` | 홀드아웃 감사, 점수 저하 검사, 무결성 영수증 |
| [antemortem-structural-lab](https://github.com/hibou04-ops/antemortem-structural-lab) | `pip install antemortem-structural-lab` | `uvx antemortem-structural-lab --help` | `tomolab`의 실험적 계약·Oracle 기반 반례와 재현 |

`antemortem-cli` 저장소의 배포명은 `antemortem`이고, mini-omega-lock의 실행 명령은 `preflight`입니다. 4개 저장소의 quickstart helper는 소스 checkout이 필요하므로 해당 README의 clone 단계를 따라야 합니다. 위의 두 모듈 데모는 wheel에 포함됩니다.

MCP 설정에는 각 저장소에 명시된 extra와 실행 명령을 사용합니다. 로컬 검증은 설치 후 모델 서비스 없이 실행할 수 있으며, 평가·추론 도구는 설정된 provider를 호출할 수 있습니다.

<a id="architecture"></a>
## 공개 도구와 비공개 개인 연구

독립적인 개인 소프트웨어 엔지니어링 연구 공간이며, 다음 두 범위로 구분합니다.

| 범위 | 구성 | 목적 |
|---|---|---|
| 공개 패키지 · Apache-2.0 | 위의 6개 패키지 | 개별 사용 가능한 CLI·Python 및 일부 MCP 인터페이스 |
| 비공개 연구 하네스 · All Rights Reserved | `omega-citadel` v1.1.2; `omega-crucible` v1.0.3 | 인메모리 파이프라인 측정과 합성 변이 실험 |

### 과거 벤치마크 기록

아래 수치는 원래 측정한 버전에 귀속됩니다. 현재 비공개 버전을 새로 측정한 결과가 아닙니다.

| 기록 버전 | 워크로드와 결과 | 해석 범위 |
|---|---|---|
| `omega-citadel` v1.0.1 | 교대 쌍 20회: warm 중앙값 **14.2 ms**, cold **1,122.6 ms**, 비율 **79.1x** | 고정 입력의 5개 cold 서브프로세스와 단일 warm Python 프로세스 비교 |
| `omega-citadel` v1.1.1 | warm 중앙값 **19.3654 ms**, cold **1,229.61405 ms**, 비율 **63.50x** | 같은 비교 설계; 과거의 `14.2 ms – 19.3 ms` 표시는 뒤 수치를 절삭한 요약 |
| `omega-crucible` v1.0.0 | seed-42 코퍼스: 합성 변이 **1,067 / 1,067**건 검출, **163.77 ms**, 정상 대조군 통과 | 하네스 자체 방어기에 대한 유한한 작성 코퍼스 |
| `omega-crucible` v1.0.3 | 별도의 clean-wheel 기록: **1,067 / 1,067**건 검출, **173.2306 ms** | 163.77 ms 기록과 구별되는 영수증 |

지연시간 기록의 환경은 Windows x64 / CPython 3.12.10입니다. 공개 제품들의 실제 CLI 연결 실행을 비교한 결과가 아니며, 프로세스 시작과 반복 파싱도 측정 비율에 기여합니다. 원본 비공개 아티팩트는 비공개로 유지되며, 이 요약은 독립 재현된 공개 벤치마크가 아닙니다.

## 근거의 범위

- 경로·행 범위·AST·해시 일치는 도구에 명시된 검사만 충족합니다. 자연어 주장의 진실성까지 입증하지 않습니다.
- 경험적 노이즈 경계는 검정력 기반 MDE 추정치나 보정된 95% coverage 보장이 아닙니다. preflight 통과는 배포 승인이 아닙니다.
- 홀드아웃 결과는 제공된 분할·계약·측정 절차에 의존합니다. 무결성 영수증만으로 실제 환경의 일반화를 입증할 수 없습니다.
- Structural Lab과 비공개 연구는 `M0_HOLD`를 유지하며, utility와 독립 OOS는 `NOT_ESTABLISHED`입니다. 엔지니어링 테스트 통과가 이 연구 상태를 변경하지 않습니다.
