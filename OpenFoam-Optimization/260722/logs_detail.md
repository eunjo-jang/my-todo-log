# 요약 및 분석 (260722 상세)

## 프로젝트 핵심

핵융합로 다이버터 설계 자동화에서 상용 CFX를 오픈소스 OpenFOAM으로 대체하려는 작업의 일부.
0721에 "코드 최적화 여지가 없으면 Ansys가 나을 수도 있으니 판단부터"라는 방향에 따라, 오늘은
**솔버 내부를 연산 단위로 실측 분해**하여 그 판단의 근거 데이터를 확보한다. (0721 "다음 할 일 (A)"의 실행)

방법론은 HEAT 최적화와 동일: **비침습 계측으로 쪼개 측정 → 병목 원인 규명 → 답-보존 해결안 판단.**

---

## 1. 계측 방법 (실행에 영향 없음)

기존 계측(0721: 단계별 Allrun 스탬프 + 파이썬 perf_counter)에 **솔버 내부 2종**을 추가.
전부 **기본 off** — `KFE_OPENFOAM_PROFILING=1`일 때만 켜지고, production 경로는 바이트·타이밍 불변.

| 신규 계측 | 무엇을 재나 | 비침습 근거 |
|---|---|---|
| OpenFOAM 네이티브 `profiling{active true}` | 방정식·선형솔버 스코프별 누적 self-time | 솔버에 이미 컴파일된 타이머를 켜기만 함 → 선형대수 산술 불변 |
| maxT 함수객체 `writeInterval=25` | 25 iter마다 영역별 max(T) → 수렴 곡선 | read-only max 리덕션, 필드 write 없음 |

**비침습 실증 (핵심):** profiling on 상태에서 추출된 max(T)가 코어 수만 다를 뿐 라운드오프 수준으로 일치.

| 응답 | 1코어 (profiling on) | 4코어 (profiling on) | 차이 |
|---|---|---|---|
| fluid max(T) | 85.8137 °C | 85.8161 °C | 0.0024 |
| monoblock max(T) | 435.3949 °C | 435.3903 °C | 0.0046 |
| tube max(T) | 142.5673 °C | 142.5676 °C | 0.0003 |
| Σq·A 입열 | 6039.685 W | 6039.685 W | 0 |

→ 계측이 결과를 바꾸지 않음. 파서 결과는 `provenance.json`의 `solve_profiling`에 자동 적재.

**코드 변경(계측만, 커밋 `c930ff9`):** settings(노브) / controlDict(profiling 블록·토큰) / case.py(`_profiling_tokens`) /
Allrun(`uniform/profiling` 수집 버그 수정) / profiling.py(파서, 신규) / provenance·node 연결 / 단위테스트(신규, 통과).

---

## 2. 측정 데이터 (전문)

### 2-1. 엔드투엔드 단계 분해
| 단계 | 1코어 (s) | 1코어 % | 4코어 (s) | 4코어 % |
|---|---|---|---|---|
| **solve** | **4008.73** | **99.5** | **1605.77** | **97.6** |
| checkmesh | 7.62 | 0.2 | 11.12 | 0.7 |
| reconstruct | — | — | 7.62 | 0.5 |
| decompose | — | — | 6.81 | 0.4 |
| split | 6.22 | 0.2 | 6.67 | 0.4 |
| import(fluent3DMeshToFoam) | 4.20 | 0.1 | 4.08 | 0.2 |
| vtk(foamToVTK) | 1.55 | 0.0 | 1.55 | 0.1 |
| changedict | 1.33 | 0.0 | 1.28 | 0.1 |
| scale | 0.06 | 0.0 | 0.02 | 0.0 |
| **합계** | **4029.70** | | **1644.92** | |

→ 메시 변환·분해·재조립·VTK는 전부 noise. **최적화 대상은 오직 solve.** (HEAT의 "숨은 I/O 착시" 같은 건 여기 없음.)
→ 4코어 가속 = 4008.73 / 1605.77 = **2.44배** (0721의 2.77배와 유사; profiling 오버헤드·noise 포함치).

### 2-2. solve 내부 self-time (4코어 네이티브 profiler)
`<시각>/uniform/profiling` 스코프 트리에서 self-time(= totalTime − childTime) 상위:

| 스코프 | calls | self (s) | self % | 성격 |
|---|---|---|---|---|
| **time.run() 본체 (비계측 잔여)** | 2000 | **1185.66** | **73.9** | 방정식 조립 + 테이블 물성 + 난류 + 영역 연성 |
| lduMatrix p_rgh GAMG coarsestLevelCorr | 2000 | 119.91 | 7.5 | 압력 멀티그리드 코스솔브 |
| lduMatrix omega | 2000 | 33.98 | 2.1 | 난류 |
| lduMatrix k | 2000 | 33.82 | 2.1 | 난류 |
| lduMatrix h (fluid) | 2000 | 32.51 | 2.0 | 에너지 |
| lduMatrix Ux | 2000 | 32.38 | 2.0 | 운동량 |
| lduMatrix h (tube) | 2000 | 31.40 | 2.0 | 에너지 |
| lduMatrix Uy | 2000 | 28.52 | 1.8 | 운동량 |
| lduMatrix Uz | 2000 | 28.49 | 1.8 | 운동량 |
| fvMatrix fluid::U 조립·솔브 | 2000 | 20.68 | 1.3 | 운동량 조립 |
| lduMatrix h (mb) | 2000 | 18.48 | 1.1 | 에너지 |
| functionObject yPlus | 1999 | 11.09 | 0.7 | 진단(계측용) |
| fvOption limitT | 2000 | 11.02 | 0.7 | 온도 제한 |
| lduMatrix p_rgh | 2000 | 6.79 | 0.4 | 압력 |
| fvMatrix fluid::p_rgh 조립 | 2000 | 1.85 | 0.1 | 압력 조립 |

**명명 스코프 합 ≈ 411s (≈26%) / 시간루프 본체 잔여 ≈ 1186s (≈74%).**

> 해석: **선형대수는 solve의 26%뿐이고, 74%는 스톡 profiler가 명명하지 않는 영역** — 방정식 조립(`fvm::div/laplacian`),
> 테이블 기반 물성 평가(`hTabulated`/`icoTabulated`/`tabulated` transport: 매 iter 셀마다 표 검색+보간),
> 난류 모델, 3영역 켤레 연성. 이 74%를 더 쪼개려면 **솔버 재컴파일**이 필요(스톡 profiler 한계).
> 계획 단계의 "p_rgh 압력식이 지배" 가설은 측정으로 **뒤집힘**(p_rgh 선형솔브 자체는 0.4%; GAMG 코스솔브만 7.5%).

### 2-3. 방정식별 선형솔버 부하 (수렴 로그)
| field | solves | inner iters (1코어) | inner iters (4코어) | 선형솔버 |
|---|---|---|---|---|
| h (에너지, 3영역) | 6000 | 37,726 | 44,027 | PBiCGStab+DILU |
| p_rgh (압력) | 2000 | 4,235 | 4,615 | GAMG |
| Ux/Uy/Uz (운동량) | 각 2000 | 각 2000 | 각 2000 | PBiCGStab+DILU (1 스윕) |
| k, omega (난류) | 각 2000 | 각 2000 | 각 2000 | PBiCGStab+DILU (1 스윕) |

> **반복 횟수는 벽시계의 오도 지표:** h가 iter 수로 압도(44k)하지만 self-time은 82s(5%)뿐. 반면 p_rgh는 iter 4.6k인데
> GAMG 코스솔브 self-time이 120s(7.5%)로 더 큼 (GAMG 1회 ≫ PBiCGStab 1회 비용). ⇒ 0721의 "count는 나쁜 대리지표" 재확인.

### 2-4. max(T) 수렴 곡선 (plateau)
25 iter 간격 실측(4코어, 총 2000 iter). 각 영역 max(T)가 사실상 수렴(plateau)하는 지점과, 마지막 25%(1500→2000 iter)의 변화폭:

| 영역 | plateau 진입 (대략) | 1500→2000 (마지막 25%) 변화 |
|---|---|---|
| fluid | ~1125 / 2000 iter | Δ0.20 °C |
| monoblock | ~1400 / 2000 iter | Δ0.29 °C |
| tube | ~1500 / 2000 iter | Δ0.47 °C |

→ 세 영역 모두 1500 iter 근처에서 사실상 수렴하며, 마지막 500 iter(전체의 25%)이 max(T)에 주는 변화는 **0.5°C 미만**. 1·4코어 곡선은 동일.
→ 이것이 판단의 **"반복수 2000→1600 축소(~20%) 시 지표 드리프트 <0.4°C"** 근거다. (25 iter 간격 전체 곡선 원자료는 `provenance.json`의 `solve_profiling`에 수록.)

---

## 3. 판단 & 결론

측정에 기반한 판단(코드 최적화 종료 · CFX 대비 per-run 6.8배 열세 · 속도는 서로게이트의 몫 · CFX vs OpenFOAM 생성기 선택 3조건)은
**같은 날짜의 [`logs.md`](logs.md) "판단(오늘의 결론)" 절**에 정리했다. 본 문서는 그 판단의 **근거 실측 데이터**를 담는다.

---

_원본: 2026-07-22 OpenFOAM 연산별 프로파일링 세션. 계측 커밋 `c930ff9` on `feature-openfoam-mpi`. 전략적 판단·권고는 동일 폴더 `logs.md` 참조._
