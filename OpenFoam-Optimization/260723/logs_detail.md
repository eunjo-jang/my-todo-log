# OpenFOAM CHT 강한 스케일링 상세 (8/16코어) — 최적화 마무리

> 작성일: 2026-07-23
> 대상: KFE 다이버터 솔버 오픈소스 대체 — OpenFOAM 열유동 노드(`openfoam_cht`)

---

## 0. 세 줄 요약
- 기존 MPI 병렬화(4코어 2.77배) 이후, 코어를 8·16까지 확장해 강한 스케일링 곡선을 완성했다.
- 병렬효율이 62%→42%→29%로 급락하고 Amdahl 점근 하한이 ≈650~800초라, 어떤 코어 수로도 CFX(4코어 237초)를 따라잡지 못한다.
- OpenFOAM은 "ANSYS보다 빠르게 가속" 기준을 충족하지 못하고 코드 레벨 비효율도 없다 → **OpenFOAM 최적화 종료.** 다음은 Code-Aster를 동일 기준으로 분석.

---

## 1. 목적

기존 MPI 병렬화 결과(4코어 2.77배, end-to-end 77분→28분) 이후 추가 최적화 여지를 두 기준으로 검증한다: ① 매쉬가 속도 병목인지, ② 병목 계산을 ANSYS 대비 유의미하게 가속할 수 있는지. 이를 위해 코어를 8·16까지 확장해 강한 스케일링 곡선을 완성했다.

## 2. 실험 방법 (재현 절차)

- 시나리오: `kfe_openfoam_cht_real.yaml` (real 모드, tilting=0 기준 형상).
- 상류 노드(ansys_dm / blender / heat)는 콘텐츠 주소 캐시(CAS) 히트로 즉시 복원 → ANSYS 라이선스 소모 없이 `openfoam_cht`만 실제 재계산.
- 코어 수 = 환경변수 `KFE_OPENFOAM_CORES`. 이 값은 fingerprint 입력이 아니므로, 코어만 바꾸면 CAS가 이전 결과를 복원해 재계산이 일어나지 않음. 따라서 매 코어마다 `graph_cache/openfoam_cht` CAS 엔트리를 비워 강제 재계산 후 측정, 완료 후 원복.
- 분해: `system/decomposeParDict`가 `method simple`, `n (N 1 1)` 1-D 분할로 렌더.
- profiling OFF(production 경로) → byte-stable, 오버헤드 없음. 기존 clean 수치와 동일 지표.
- 실행 예:
  ```
  PYTHONUTF8=1 KFE_OPENFOAM_CORES=8 uv run --directory backend --no-sync \
    kfe-divertor-run --scenario workflows/scenarios/kfe_openfoam_cht_real.yaml \
    --run-id bench_openfoam_8core --cores 8
  ```

## 3. 원자료

### 3-1. solve phase (`chtMultiRegionSimpleFoam` 최종 ExecutionTime)
| 코어 | ExecutionTime (s) | 노드 wall* (s) | solve_seconds** (s) |
|---|---|---|---|
| 1 | 4008.73 | ~4620 (77분) | — |
| 4 | 1605.77 | 1693 (28.2분) | — |
| 8 | 1195.29 | 1246 (20.8분) | 1232.4 |
| 16 | 864.13 | 917 (15.3분) | 901.8 |

\* 노드 wall = 컨테이너 기동 + decompose + solve + reconstruct + 추출 포함.
\*\* solve_seconds = 노드가 추출한 솔버 시간 metric. 세 지표 모두 동일 추세.

### 3-2. speedup / 병렬효율 (ExecutionTime 기준)
| 코어 | speedup | 효율(=speedup/코어) | 직전 대비(코어 2배당) |
|---|---|---|---|
| 1 | 1.00× | 100% | — |
| 4 | 2.50× | 62.4% | (1→4, 4배) 2.50× |
| 8 | 3.35× | 41.9% | (4→8) 1.34× |
| 16 | 4.64× | 29.0% | (8→16) 1.38× |

### 3-3. 결과 보존 (max(T), °C)
| | monoblock | tube | fluid |
|---|---|---|---|
| 4코어(기준) | 435.390 | 142.568 | 85.816 |
| 8코어 | 435.389 | 142.569 | 85.817 |
| 16코어 | 435.396 | 142.577 | 85.825 |

최대 편차: monoblock 0.006 / tube 0.009 / fluid 0.009 °C (~0.001%). 노드 허용오차(~5%) 대비 무시 수준. 코어 증설은 답을 바꾸지 않는(answer-preserving) 성능 레버.

## 4. 스케일링 해석

- 효율 62% → 42% → 29% 급락. 8→16코어에서 solve 1195→864초, **28% 감소에 그침**.
- Amdahl 피팅 `T(N) = T₁·(s + (1−s)/N)`:
  - (1c, 4c) 피팅 → s ≈ 0.20, 점근 하한 T∞ = T₁·s ≈ 800초.
  - (1c, 16c) 피팅 → s ≈ 0.16, T∞ ≈ 655초. (16코어가 예측보다 빠름 → s는 더 작은 쪽.)
  - 즉 코어를 무한히 늘려도 solve 하한 **≈ 650~800초 (11~13분)**. 16코어(864초)가 이미 하한에 근접.
- 1-D 분할 `(16 1 1)`도 decomposePar에서 빈 프로세서/에러 없이 통과 → 분해 방식이 16코어 한계 요인은 아님. 한계 요인은 **물리적 직렬 구간**(다영역 커플링·경계 갱신·리덕션).

## 5. 두 기준에 대한 판정 — OpenFOAM

**기준 ① 매쉬가 병목인가 → 아니오.** 매쉬 파이프라인(checkMesh, decomposePar, reconstructPar, mesh import, VTK 변환) 합계 end-to-end의 **<1%**, 나머지 ~99%가 solve.

**기준 ② 병목(solve)을 ANSYS보다 빠르게 가속 가능한가 → 아니오.**
- 기준값: CFX 4코어 solve ≈ 237초.
- OpenFOAM 16코어 solve = 864초 → **CFX 대비 3.6배 느림**.
- 점근 하한 ≈ 650~800초 → CFX 대비 **2.7~3.4배 느림**. 56코어 이내 어떤 구성으로도 237초 도달 불가.
- 남은 유일한 가속 레버 = **GPU 선형솔버(PETSc4FOAM / AmgX)**:
  - 프로파일링상 solve 내 선형대수 스코프 합 ≈ 26%, 나머지 74%는 방정식 조립·물성 테이블 조회·다영역 커플링(전부 CPU).
  - Amdahl 상한: 선형대수를 무한 가속(→0)해도 solve는 4코어 1606초 → 최선 ~1186초. 74%가 CPU에 남아 여전히 CFX 대비 ~5배 느림. **GPU는 26% 상한에 막혀 dead-end.**

**종합**: OpenFOAM은 두 번째 기준을 충족하지 못하며 코드 레벨 비효율도 없음(최적 솔버 사용). **OpenFOAM 최적화 종료.**

## 6. 다음 단계

- **Code-Aster를 동일 기준으로 분석**: ① 매쉬(또는 전처리)가 병목인가, ② 병목 계산을 ANSYS Mechanical(baseline solve ≈ 316초)보다 빠르게 가속 가능한가. MPI/코어 스케일링 + 병목 프로파일링을 동일 포맷으로 공유.
- OpenFOAM에서 실질 가치를 낼 유일한 방향은 **서로게이트**이나 현재 우선순위 아님.

## 7. 재현 메모

- 코어 수는 fingerprint 입력이 아니므로 CAS(`graph_cache/openfoam_cht`)를 비워야 재계산됨. 비우지 않으면 이전 결과가 복원되어 짧은 시간의 **가짜 성공**이 발생.
- 파이프(`... | tail`)로 실행 시 종료코드가 파이프 마지막 명령 기준이라 실제 크래시가 exit 0으로 보일 수 있음.
- backend 상대경로(fixture 등) 해석은 cwd 의존 → 반드시 `uv run --directory backend`로 실행.
- 원자료 위치: `backend/runs/bench_openfoam_{8,16}core/` (events.ndjson, `nodes/openfoam_cht/_of_result/profiling_logs/log.chtMultiRegionSimpleFoam`, metrics json).

---

_원본: 2026-07-23 OpenFOAM 강한 스케일링(8/16코어) 측정 세션. 브랜치 `feature-openfoam-mpi`. CFX/Mechanical baseline은 07-20 벤치 참조._
