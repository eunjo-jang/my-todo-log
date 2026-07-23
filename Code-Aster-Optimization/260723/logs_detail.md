# Code_Aster 구조해석 내부 프로파일링 & MPI 최적화 — 상세 보고서

> 작성일: 2026-07-23
> 대상: KFE 다이버터 솔버 오픈소스 대체 — Code_Aster 구조(FEM) 노드

---

## 0. 세 줄 요약
- ANSYS Mechanical을 Code_Aster로 대체 평가하기 위해, "무엇을 최적화할지"를 먼저 알아내는 **비침습 4계층 내부 프로파일링**을 구현하고 실제 2.6M-DOF 케이스로 검증했다.
- 병목은 명확히 **MECA_STATIQUE 안의 MUMPS 인수분해(587초, 전체의 63%)** 단독이었고, 이를 공략하는 여러 레버 중 **MPI 병렬만이 답을 보존하며 효과적**이었다(16코어에서 1.72배, peak 소수점까지 동일). BLR 저랭크 압축은 정확도 게이트 위반 또는 오히려 감속으로 이 문제에서 부적합.
- 프로파일링 + MPI를 프로덕션에 배선하고, 지금까지 실런 3회를 모두 죽이던 post-solve 크래시(`.rmed` 인식 실패)도 함께 수정했다.

---

## 1. 배경 / 목적

KFE 다이버터 설계 자동화 파이프라인에서 상용 솔버(ANSYS CFX·Mechanical)를 오픈소스(OpenFOAM·Code_Aster)로 대체하는 과제의 일부다. HEAT·OpenFOAM에서 했던 것과 동일하게, Code_Aster 구조해석 노드도 **솔브 내부를 최대한 잘게 쪼개** 시간이 어디서 쓰이는지 실측하고, 그 결과로 **어떤 최적화 레버가 실효적인지** 판정하는 것이 목적이다.

작업 착수 시점의 상태:
- 실행 경로: Docker 컨테이너에서 `run_aster study.export` (Code_Aster 17.4.0, `simvia/code_aster:stable`). 솔버는 `MECA_STATIQUE`(선형 정적, LIAISON_MAIL 본디드, MUMPS).
- 타이밍은 **단일 불투명 숫자 하나**(`solve_seconds`, 노드가 `solve_kfe_bonded` 전체를 `perf_counter`로 감싼 값)뿐. ANSYS Mechanical 재솔브 시간은 버려지고 있었음.
- Code_Aster가 `message`(.mess) 파일에 명령별 타이밍 표를 이미 출력하지만 **아무도 파싱하지 않음**.
- `app/code_aster/`에는 OpenFOAM과 달리 `profiling.py`·`provenance.py`가 없었음.

원칙: OpenFOAM 프로파일링(커밋 `c930ff9`)의 **비침습·기본off·사후파싱** 패턴을 그대로 미러. 즉 프로파일링 off일 때 프로덕션 런은 바이트/수치 동일, 노드 fingerprint 불변.

---

## 2. 진행 내용

### 2.1 4계층 내부 프로파일링 (거친 → 세밀)

| 계층 | 무엇을 쪼개나 | 소스 | on/off |
| --- | --- | --- | --- |
| A. Host 오케스트레이션 | DPF 추출 / hex20 게이트 / ds.dat 파싱 / 케이스 빌드 / 컨테이너 cp·solve / post + **버려지던 ANSYS 재솔브 시간** | `solve_kfe_bonded`·`docker.run_case`의 `perf_counter` 구간 | 항상 |
| B. 명령별 타이밍 | DEBUT·LIRE_MAILLAGE·…·**MECA_STATIQUE**·… + TOTAL_JOB + VmPeak | `message`의 명령 타이밍 표(이미 출력됨) | 항상 |
| C. 솔버 내부 비용 | MECA_STATIQUE 내부: 기본계산+어셈블리 vs 선형계 풀이 | `DEBUT(MESURE_TEMPS=_F(NIVE_DETAIL=3))` | 크노브 |
| D. MUMPS 통계 (최세밀) | **factorize vs solve vs 어셈블리 vs RHS** + 솔버 설정 에코 | 명령레벨 `INFO=2`의 `Statistics:` 블록 | 크노브 |

- 신규 파서 `app/code_aster/profiling.py` (best-effort·never-raising): `parse_command_timings`(B) / `parse_solver_phases`(C) / `parse_solver_stats`(D) / `collect_solve_profiling`.
- 결과는 신규 `app/code_aster/provenance.py`가 `provenance.json`의 `phase_timings`+`solve_profiling`으로 기록. **취약한 VTU 단계 앞에서** 기록해 실패해도 프로파일이 남게 함.
- 진단 크노브 `KFE_CODE_ASTER_PROFILING`(기본 False). 솔브 시점 env로 읽어 노드 fingerprint에 안 들어감.
- 계층 B는 기존 실제 런 3개 message로 즉시 검증. 계층 C/D는 작은 해석 케이스를 크노브 켜고 실제 컨테이너에서 돌려 출력 포맷 확보 후 파서 작성.
- **발견**: `INFO`는 `SOLVEUR=_F(...)` 안이 아니라 **명령 레벨**(`MECA_STATIQUE(..., INFO=2)`)에 넣어야 17.4에서 통과 (`SOLVEUR` 안에 넣으면 `Unauthorized keyword: 'INFO'`).

### 2.2 post-solve 크래시 수정

실런 3회(`bench_run_a`/`bench_run_b_ca`/`bench_codeaster`)가 모두 solve 성공 후 `result.rmed → .vtu` 변환에서 죽고 있었음. 원인은 meshio가 `.rmed` 확장자를 인식 못 함(`.med`만 매핑) → "Could not deduce file format". 수정: `from_med.py`에서 `meshio.read(path, file_format="med")` 명시. 실제 68MB `.rmed`로 검증(peak 368.7 MPa).

### 2.3 최적화 실험 (병목 기반)

프로파일링이 지목한 병목(MUMPS 인수분해)을 공략. 실험은 `bench_run_b_ca`의 **이미 준비된 실제 메시(80MB)+온도장(58MB)**을 재사용해 컨테이너 솔브만 재실행(ANSYS/DPF 불필요). 매 실험 peak를 baseline 368.747 MPa와 비교(답 보존 검증).

- 이미지가 **완전 MPI 병렬 빌드** 확인: OpenMPI 5.0.8, ParMETIS, SCOTCH, PETSc, MUMPS 5.6.2, `ASTER_HAVE_MPI`.
- 컨테이너는 **56 CPU / 62GB** 가용, baseline은 8코어(mpi=1, ncpus=8)만 사용 → 병렬 확장 여지 큼.
- **MPI 구동 게이트 발견**: 컨테이너가 결과 write 권한 때문에 `--user root`로 도는데, OpenMPI 5는 root의 `mpiexec`를 거부 → `OMPI_ALLOW_RUN_AS_ROOT`(+`_CONFIRM`) env로 해결.

### 2.4 MPI 프로덕션 배선

- 크노브 `KFE_CODE_ASTER_MPI_NBCPU`(기본 1) + `KFE_CODE_ASTER_NCPUS`(기본 8). node → `solve_kfe_bonded` → `build_kfe_case` → `assemble_case` → `render_export`(이미 `P mpi_nbcpu`/`P ncpus` emit)로 스레딩.
- `docker.build_create_cmd`에 OpenMPI allow-run-as-root env 추가(직렬 시 무해).
- 기본값=기존 직렬 baseline이라 프로덕션 동작 불변. perf-only라 fingerprint 제외. `solver_cores`는 이제 `mpi×ncpus` 보고.

---

## 3. 결과 및 분석

### 3.1 병목 실측 (baseline, 실제 2.6M-DOF)

MECA_STATIQUE ≈ 849s가 TOTAL_JOB 927s의 ~91%. 그 내부(INFO=2 Statistics):

| 내부 단계 | 시간 | 전체 대비 |
| --- | --- | --- |
| **MumpsSolver.factorize** (LU 인수분해) | **587.3s** | **~63%** |
| MumpsSolver.solve (전진/후진 대입) | 175.2s | ~19% |
| _computeMatrix (강성행렬) | 29.4s | ~3% |
| _computeStress (응력 복원) | 24.5s | ~3% |
| _computeRhs | 13.1s | ~1% |
| assemble (행렬 조립) | 7.6s | ~1% |
| `. fortran` (run_aster 시작 오버헤드) | ~103s | ~11% |

→ 병목은 **MUMPS 수치 인수분해 단독**. VmPeak 15.1GB, DOF 2,602,665.

### 3.2 최적화 비교 (정답 peak = 368.747 MPa)

| 설정 | 코어 | 총(TOTAL_JOB) | factorize | solve | 메모리 | 정답 | vs baseline |
| --- | --- | --- | --- | --- | --- | --- | --- |
| baseline (mpi1 × 8스레드) | 8 | 927s | 587s | 175s | 15.1GB | ✓ | — |
| **mpi4** (4 × 4) | 16 | **538s** | 286s | 78s | 11.3GB | ✓ | **1.72×** |
| mpi8 (8 × 4) | 32 | 514s | 241s | 60s | 10.7GB | ✓ | 1.80× |
| BLR 1e-9 | 8/16 | 실패 | — | — | — | ✗ | FACTOR_57 |
| BLR 1e-12 (보수) | 16 | 780s | 558s | 50s | 17.6GB | ✓ | 0.85× |

### 3.3 레버별 판정

| 레버 | 판정 | 근거 |
| --- | --- | --- |
| **MPI 병렬** | **채택** | 16코어 1.72×, factorize 2.05×, 메모리 25%↓, peak 소수점까지 동일. 16→32코어는 +4.5%뿐 → 16코어(mpi4×4) sweet spot |
| BLR (ACCELERATION='LR') | 기각 | 1e-9: 계산 오차 5.2% > RESI_RELA 1e-4(~공학 허용치) → `FACTOR_57` 거부. 1e-12: 답 보존되나 압축 이득 없이 오버헤드만 → 오히려 감속(factorize 286→558s). 행렬이 안전 임계값에서 압축 안 됨 |
| RENUM / in-core | 미측정 | MPI로 충분, 우선순위 낮음 |

### 3.4 ANSYS 대비

ANSYS Mechanical 1-core solve = **680s**. baseline(927s)은 느리지만 **MPI 적용 시 538s로 상용 1-core보다 빠름** → OSS 대체 타당성에 긍정적.

---

## 4. 다음 할 일
1. ✓ 완료 (#197) — `feature-code-aster-profiling` 브랜치 draft PR (base = `feature-openfoam-mpi`, 스택)
2. 기본값 mpi=4 승격 여부 결정 (현재 기본 mpi=1 직렬 baseline 유지)
3. 전체 노드 end-to-end 실런(ANSYS+DPF 포함)으로 프로덕션 MPI 경로 + provenance.json 4계층 최종 확인
4. RENUM/in-core 레버는 필요 시에만 추가 탐색
5. Jira 키 확정 → 커밋 메시지 `DIVERTOR-TBD` 교체

---

## 5. 결론

Code_Aster 구조해석의 느린 원인은 **MUMPS 인수분해 단독**임을 실측으로 규명했고, 이를 공략하는 레버 중 **MPI 병렬만이 답을 보존하며 실효적**(16코어 1.72배, 결과 불변)임을 확인했다. BLR 저랭크 압축은 이 본디드 선형 시스템에서 정확도-속도 트레이드오프가 불리해 부적합. MPI 적용 시 상용 ANSYS Mechanical 1-core보다 빨라져, 구조해석 노드의 OSS 대체 타당성이 뒷받침된다. 프로파일링·MPI·크래시 수정은 모두 프로덕션에 배선(기본값=기존 동작)되어 `feature-code-aster-profiling`에 커밋됨.

---

_원본: 2026-07-23 세션 실측 데이터(`runs/bench_ca_profiled`, `runs/bench_ca_opt`) 정리. 관련 브랜치 `feature-code-aster-profiling` (커밋 `e6f7d7b`·`485dd0c`, 부모 `feature-openfoam-mpi`)._
