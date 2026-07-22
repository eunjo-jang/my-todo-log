# 2026-07-22 (260722) 작업 로그

프로젝트: KFE 다이버터 솔버 오픈소스 대체 — OpenFOAM 열유동 노드

> 0721 오후 "다음 할 일 (A)"의 실행: 경량 native `profiling`으로 **연산별 실제 시간 트리**를 확보하고,
> 이를 근거로 **코드 최적화 vs 서로게이트 vs Ansys(CFX) 유지** 의사결정을 내리는 것이 오늘의 목표.

## 오늘 한 일
- **솔버 내부(연산별) 비침습 프로파일링을 실제로 계측·수집** — 0721에서 붙인 4계층 계측에 더해:
  - OpenFOAM 네이티브 `profiling{active true}` 스코프 타이머(방정식·선형솔버별 self-time)
  - maxT 함수객체를 25 iter마다 기록 → **max(T) 수렴 곡선** 확보
  - 파서(`app/openfoam/profiling.py`) → `provenance.solve_profiling`에 자동 적재
  - **기본 off** 게이팅(`KFE_OPENFOAM_PROFILING`)이라 production 경로는 바이트·타이밍 불변
- **실제 real 런 2회 수집**: 1코어 solve **4,009초**, 4코어 solve **1,606초**(2.44배). profiling on/off로 max(T) 소수 2자리까지 동일(비침습 실증).
- 발견·수정: 네이티브 profiler는 `<시각>/uniform/profiling`(time 디렉터리)에 쓰는데 드라이버가 copy-out 안 해 **1코어 런에서 유실** → Allrun이 `profiling_logs/`로 수집하도록 수정.
- 계측 커밋·푸시(`feature-openfoam-mpi`, `c930ff9`). 반복수(end_iterations)는 팀 정합 이슈라 **건드리지 않고 계측만** 남김.

## 핵심 측정 결과 (가설 아님 · 실측)
- **solve = 컨테이너 시간의 97.6%(4코어)/99.5%(1코어)** — 메시 변환·분해·VTK는 noise. 숨은 I/O 착시 없음.
- **solve 내부 self-time(4코어)**: 선형대수는 **26%뿐**, **74%가 방정식 조립 + 테이블 물성(`hTabulated`/`icoTabulated`) + 난류 + 영역 연성**. → 0721의 "속성 평가 비용" 가설을 뒷받침. 스톡 profiler는 이 74%를 더 쪼개지 못함(솔버 재컴파일 필요).
  - 명명된 최대 스코프: p_rgh GAMG 코스솔브 7.5%, 에너지 h(3영역) 5%, U/k/omega는 저렴.
  - 선형반복 **횟수**는 여전히 오도 지표(h 44k iter vs p_rgh 4.6k인데 self-time은 p_rgh가 큼).
- **max(T) plateau(1·4코어 동일)**: fluid @~1125, monoblock @~1400, tube @~1500 of 2000. **1500→2000(25%)은 max(T)를 0.5°C 미만으로만 변화**(fluid Δ0.2 / mb Δ0.29 / tube Δ0.47 °C).

## 판단 (오늘의 결론) ★
### 1) 솔버 코드 최적화는 사실상 종료 (headroom 소진)
| 레버 | 결과 | 판정 |
|---|---|---|
| config(GAMG smoother, 분해법) | ~1-2% (0721) | noise, 폐기 |
| 소스 재컴파일(앱 레벨) | 앱은 773줄 오케스트레이션뿐, 계산은 성숙 라이브러리 + GPLv3 + fork 부담 | ROI 낮음, 비권장 |
| **반복수 축소(temporal, 신규)** | 2000→1600 = **~20%** 단축, 지표 드리프트 <0.4°C | **유일한 유효 레버(1회성)**, 미러 정합 재검증 필요 |
| 74% 조립/물성 본체 | 물성·이산화 변경 필요 → 답 변화·CFX 이탈 / 또는 GPU(petsc4Foam) 대공사 | 답-보존 아님, 비권장 |

→ **결과를 보존하며 벽시계를 유효하게 줄이는 solver-level 레버는 반복수 축소(~20%, 1회성)가 사실상 전부.** HEAT 같은 order-of-magnitude 개선 여지는 없음.

### 2) OpenFOAM은 CFX보다 "빠를" 수 없다 — 속도는 서로게이트의 몫
| 항목 | CFX(상용) | OpenFOAM(OSS) |
|---|---|---|
| 4코어 solve | **≈237초** (07-20 실측) | **≈1,606초** (오늘, 반복수 축소 시 ~1,285초) |
| 라이선스 | 유료(동시 solve = seat 제한) | **무료** |
| 정확도(gold 대비) | 검증된 oracle | fluid +5.1% / mb +5.9% / tube −1.6% (허용오차 5% 근소 초과) |
| 배치 확장성 | seat 수에 묶임 | **56코어 호스트에서 대량 병렬(무료)** |

→ per-run 속도는 **CFX가 ~6.8배 빠름**. 최적화를 다 해도 이 격차는 안 뒤집힘. **"더 빠른 CFX 대체"는 달성 불가.**

### 3) 그래서 전략적 결론
원래 목표는 (a) 라이선스 비용 절감 위해 CFX→OpenFOAM 대체, (b) 느리니 **서로게이트로 가속**. **속도 문제의 해답은 솔버 최적화가 아니라 서로게이트다** (추론 시 solver 종류 무관, 1000배급 가속은 서로게이트에서 나옴). 솔버는 "서로게이트 학습데이터 생성기"로서 **충분히 싸고 배치 가능하면 됨.**

- **데이터 생성기로 무엇을 쓸지 = 3개 조건에 달림**:
  1. 필요한 학습 포인트 수(DOE 크기)
  2. CFX 라이선스 seat 수/비용
  3. "OpenFOAM의 5% bias"를 서로게이트의 ground truth로 수용 가능한지
- **OpenFOAM 채택이 유리한 경우**: 포인트 多 + seat 제한 + 5% bias 수용 → 56코어에서 동시 다발 배치(무료)가 seat에 묶인 CFX보다 **총 처리량↑ + 비용 0**. (예: 56코어에서 4코어 solve 14개 동시 = 시간당 CFX 대비 경쟁력.)
- **CFX 유지가 유리한 경우**: 포인트 少, 또는 CFX 정확도가 필수 ground truth, 또는 seat이 넉넉/저렴 → per-run 6.8배 빠르고 검증된 CFX가 단순·안전. **OpenFOAM 대체의 실익 없음.**

### 4) 권고 (현재 데이터 기준)
1. **solver-level 최적화 중단** (반복수 축소 20%는 OpenFOAM을 생성기로 확정할 때만, DOE 전체에 곱해져 의미 있을 때 적용).
2. **노력을 서로게이트화로 피벗** — 여기가 실제 속도 이득처.
3. **CFX vs OpenFOAM 생성기 선택은 위 3조건을 팀이 수치로 확정 후 결정.** 라이선스 비용·DOE 크기가 크면 OpenFOAM(무료·배치), 아니면 CFX 유지.
4. 캘리브레이션(5% 정확도 갭)은 **속도가 아닌 정확도** 이슈이며 샘플 종속(일반화 약함) → OpenFOAM을 ground truth로 쓸 때만 착수.

## 진행 중 / 이슈
- 반복수 축소(2000→1600)는 2000-반복 캘리브레이션과 **비트 동일 아님**(<0.4°C 드리프트) → golden 재현 테스트 · CFX 미러 허용치 재확인 필요(팀 정책 결정).
- 74% 조립/물성 본체는 스톡 profiler로 더 못 쪼갬 → 더 깊은 분해는 솔버 재컴파일(GPLv3·유지부담) 필요.
- OpenFOAM 정확도 5~6% 갭 미해소(캘리브레이션 보류 상태 유지).

## 다음 할 일
- **의사결정 미팅용 입력 확정**: DOE 크기 / CFX seat 수·비용 / 5% bias 수용 여부 3개 수치 수집.
- 결정에 따라: (OpenFOAM 채택 시) end_iterations 2000→1600 재검증 후 적용 + 서로게이트 데이터 파이프라인 설계 / (CFX 유지 시) OpenFOAM 트랙 동결.
- 서로게이트 방식 스코핑(입력=8 KFE 파라미터 → 출력=3 max(T)): 데이터량·모델(GP/NN)·정확도 목표.
- `feature-openfoam-mpi` `/code-review` 후 draft PR.

## 메모
- 계측 커밋 `c930ff9` on `feature-openfoam-mpi` (10파일, 557+/17−, 단위테스트 105 passed).
- 기준 응답(오늘 실측): fluid 85.82 / 모노블록 435.39 / 튜브 142.57 °C (0721과 동일).
- 원시 데이터(scratchpad): `of_prof_1core/`, `of_prof_4core/` — 각 `openfoam/provenance.json`의 `solve_profiling`에 단계·스코프·수렴·max(T)곡선 수록.
- CFX 4코어 baseline 237초는 07-20 벤치(`ansys-baseline-benchmark-times`) 참조.
- 계측은 기본 off라 production 무영향 — 필요 시 `KFE_OPENFOAM_PROFILING=1`로만 재현.
