# 2026-07-23 (260723) 작업 로그

프로젝트: KFE 다이버터 솔버 오픈소스 대체 — Code_Aster 구조(FEM) 노드

## 오늘 한 일
- Code_Aster 구조해석 솔브에 **비침습 4계층 내부 프로파일링** 구현 (OpenFOAM 프로파일링 패턴 미러, 기본 off = 프로덕션 바이트 동일). 진단 크노브 `KFE_CODE_ASTER_PROFILING`
- 프로파일링으로 병목 실측: **MECA_STATIQUE가 전체의 ~91%**, 그 내부는 **MUMPS 인수분해(factorize) 587초가 단독 지배** (실제 2.6M-DOF 케이스)
- 병목 기반 최적화 실행·비교: **MPI 병렬이 유일하게 효과적인 답 보존 레버** — `mpi=4×ncpus=4`(16코어)에서 **총 927s → 538s (1.72배)**, peak von Mises 368.747 MPa 완전 동일
- MPI를 프로덕션에 배선 (`KFE_CODE_ASTER_MPI_NBCPU` / `KFE_CODE_ASTER_NCPUS`, 기본값=기존 직렬 baseline)
- 지금까지 실제 런 3회를 모두 죽인 post-solve 크래시 원인 규명·수정 (`from_med.py` meshio `.rmed` 포맷 인식 실패)

## 핵심 측정 결과 (가설 아님 · 실측)

실제 KFE 888k-노드 / 2.6M-DOF 본디드 MECA_STATIQUE, 정답 = peak von Mises **368.747 MPa** (on/off 소수점까지 동일).

| 설정 | 코어 | 총(TOTAL_JOB) | factorize | solve | 메모리 | 정답 | vs baseline |
| --- | --- | --- | --- | --- | --- | --- | --- |
| baseline (mpi1 × 8스레드) | 8 | 927s | 587s | 175s | 15.1GB | ✓ | — |
| **mpi4** (4 × 4) | 16 | **538s** | 286s | 78s | 11.3GB | ✓ | **1.72×** |
| mpi8 (8 × 4) | 32 | 514s | 241s | 60s | 10.7GB | ✓ | 1.80× |
| BLR 1e-9 | 8/16 | 실패 | — | — | — | ✗ | FACTOR_57 (오차 5.2%) |
| BLR 1e-12 (보수) | 16 | 780s | 558s | 50s | 17.6GB | ✓ | 0.85× (더 느림) |

- 참고: 컨테이너는 **56 CPU / 62GB** 가용인데 baseline은 8코어만 사용 (48코어 유휴).
- 비교 기준선: ANSYS Mechanical 1-core solve = **680s** → MPI 적용 시(538s) OSS가 더 빠름.

## 판단 (오늘의 결론) ★
- **MPI가 유일한 실효 레버.** 16코어(mpi4)가 sweet spot — 32코어(mpi8)는 +4.5%뿐이라 MUMPS 병렬 효율이 16코어 근처에서 포화.
- **BLR(저랭크 압축)은 이 문제에서 막다른 길.** 공격적(1e-9)이면 잔차 게이트(RESI_RELA 1e-4, ~공학 허용치) 위반으로 거부, 보수적(1e-12)이면 정답은 보존되나 압축 이득 없이 오버헤드만 붙어 오히려 느려짐 → 행렬이 안전 임계값에서 압축 안 됨. (OpenFOAM의 "답 보존 설정 레버는 유의미하게 도움 안 됨" 결론과 동일 패턴)

## 진행 중 / 이슈
- MPI 프로덕션 경로는 배선 완료했으나 **실측 검증은 스크래치 러너로만** 수행 (프로덕션 코드가 emit하는 export/env와 동일한 조합). 전체 노드 end-to-end(ANSYS+DPF 포함) 실런은 미수행
- `mpi_nbcpu>1` 시 컨테이너가 `--user root`라 OpenMPI 5가 root 실행 거부 → `OMPI_ALLOW_RUN_AS_ROOT` env로 해결 (드라이버에 반영)
- 32코어 이상 스케일링 미탐색 (16코어에서 이미 포화 확인, 실익 낮음)
- RENUM / in-core(GESTION_MEMOIRE) 레버는 미측정 (MPI로 충분해 우선순위 낮음)

## 다음 할 일
- ✓ 완료 (#197) — `feature-code-aster-profiling` 브랜치 draft PR (base = `feature-openfoam-mpi`, 스택)
- 기본값 mpi=4로 승격할지 결정 (현재 기본 mpi=1 직렬 baseline 유지 중)
- 전체 노드 end-to-end 실런으로 provenance.json 4계층 최종 확인
- Jira 키 확정 후 커밋 메시지 `DIVERTOR-TBD` → 실제 키로 교체

## 메모
- 커밋: `e6f7d7b`(프로파일링) → `485dd0c`(MPI 병렬) on `feature-code-aster-profiling` (부모 `feature-openfoam-mpi` 위 스택)
- 코어 수는 노드 fingerprint에서 제외 (perf-only, ANSYS/OpenFOAM 방식과 동일)
- 원시 데이터: `runs/bench_ca_profiled`(baseline+프로파일), `runs/bench_ca_opt/{mpi4,mpi8,mpi4_blr12}`(최적화 실험, `results.json`/`results_r2.json`)
- 크래시 수정: `from_med.py` `meshio.read(path, file_format="med")` — meshio가 `.rmed` 확장자를 못 알아봄 (`.med`만 매핑)
- MPI 발견: `run_aster`가 `mpiexec -n N`로 내부 구동 → export의 `P mpi_nbcpu N`로 제어
- 테스트: code_aster + node 스위트 125 passed, 1 skipped
