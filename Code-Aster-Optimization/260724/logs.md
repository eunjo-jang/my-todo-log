# 2026-07-24 (260724) 작업 로그

프로젝트: KFE 다이버터 솔버 오픈소스 대체 — Code_Aster 구조(FEM) 노드

> 어제(260723)의 4계층 프로파일링 + MPI 병렬에 이어, 오늘은 **선형 솔버 방법 자체를 바꾸는 최적화(PETSc 하이브리드)**와 **피크가 아닌 응력 필드 전체로의 정확도 검증**에 집중.

## 오늘 한 일
- **PETSc 하이브리드 솔버(FGMRES + LDLT_SP) 실측** → MUMPS 직접법보다 빠름. 병목이던 이중정밀 인수분해(587s)를 **단정밀 인수분해 전처리 + 반복**으로 대체. baseline 927s → **405~459s**
- **"피크만 비교"의 한계 교정**: 88만 노드 von Mises 필드를 MUMPS 기준과 **node-by-node** 비교. RESI_RELA `1e-4` vs `1e-6` 두 정밀도로 필드 보존 여부 판정
- 큰 그림 **최적화 지점 정리**(솔버 밖): 노드 안의 중복 ANSYS 재솔브, 스윕 병렬, 워밍 컨테이너
- PR **#197** 오픈 (draft, base=`feature-openfoam-mpi` 스택). MPI 프로덕션 반영은 어제 커밋(`485dd0c`)

## 핵심 측정 결과 (실측 · 실제 2.6M-DOF, 16코어 mpi4×4 · MUMPS 직접법 대비)

| 솔버 | RESI_RELA | 총 시간 | vs baseline | peak 차이 | 최악 노드(피크 대비) | RMS |
| --- | --- | --- | --- | --- | --- | --- |
| MUMPS 직접 (baseline, 8코어) | 직접 | 927s | — | — | — | — |
| MUMPS + MPI | 직접 | 538s | 1.72× | 0 (비트동일) | 0 | 0 |
| PETSc 하이브리드 | 1e-4 | **405s** | **2.29×** | −1.2% | 5.4% ⚠ | 0.96 MPa |
| **PETSc 하이브리드** | **1e-6** | **459s** | **2.02×** | **0.0%** | **0.06%** ✓ | **0.0075 MPa** |

- GAMG(순수 대수 멀티그리드)는 **발산 실패** — 본디드 LIAISON_MAIL의 Lagrange 승수(saddle-point) 때문.
- 참고: 이 미러가 ANSYS 대비 갖는 고유 RMS는 4.4 MPa. 하이브리드가 MUMPS 대비 더하는 RMS는 1e-6에서 0.0075 MPa로 **무시 가능 수준**.

## 판단 (오늘의 결론) ★
- **운영점 = PETSc 하이브리드 + RESI_RELA=1e-6**: 피크+필드 전체를 직접법 수준으로 보존(최악 0.06%, RMS 0.0075)하면서 **baseline 2.02× 가속**, MUMPS-MPI(538s)보다도 1.17× 빠름.
- **"피크만 보면 안 된다"가 오늘의 교훈**: 1e-4는 peak −1.2%로 통과했지만 필드 최악 노드가 5.4%로 경계였음. 필드 전체를 봐야 진짜 답 보존을 판정.
- **주의(방법론)**: 응력장은 대부분 ~0이라 "상대차 %" 지표가 near-zero 분모로 폭주(무의미). **절대차·RMS·백분위·피크정규화**로 봐야 함. (온도장은 절대온도라 상대차가 통했지만 응력은 다름)

## 진행 중 / 이슈
- PETSc 하이브리드는 **아직 프로덕션 미반영** (현재 브랜치엔 MUMPS+MPI만). 크노브화 + 커밋 필요
- PETSc `1e-4`의 최악 노드 5.4%는 응력 집중부인데, 그 지점은 이미 ANSYS 대비 +26% 아티팩트(DIVERTOR-174) 영역이라 거기서의 5%는 큰 의미 없음
- PETSc @ mpi8(32코어) 미측정 (MUMPS는 16→32 포화였음)
- end-to-end 실런(ANSYS+DPF 포함)으로 프로덕션 경로 최종 검증 아직

## 다음 할 일
1. **PETSc 하이브리드를 프로덕션 솔버 옵션으로 반영** (`comm.py` SOLVEUR 토큰화 + 크노브, 기본 MUMPS 유지·PETSc opt-in) → `feature-code-aster-profiling` 커밋 → PR #197 갱신 → `/code-review`
2. **큰 그림 최적화: 노드 안 중복 ANSYS 재솔브 제거** (mesh-only 추출/캐시) — 아키텍처 변경, 별도 Jira 태스크
3. end-to-end 실런 1회로 provenance.json 4계층 + 프로덕션 MPI/PETSc 경로 검증
4. (선택) PETSc @ mpi8, RENUM/in-core 부수 측정

## 메모
- 원시 데이터: `runs/bench_ca_petsc`(PETSc 솔버 비교), `runs/bench_ca_fieldcmp`(1e-4 필드), `runs/bench_ca_fieldcmp_1e6`(1e-6 필드) — 각 `results.json`/`field_compare*.json`
- 하이브리드 원리: 단정밀 LDLT 인수분해를 전처리로만(≈218s) + FGMRES 반복으로 정확도 회복. 반복 수는 RESI_RELA로 조절(1e-4 solve 24s → 1e-6 solve 70s)
- 어제 커밋: `e6f7d7b`(프로파일링) → `485dd0c`(MPI) on `feature-code-aster-profiling`
- 레버 순위(최종): **PETSc-LDLT_SP+MPI > MUMPS+MPI > MUMPS 직렬** / BLR·GAMG 기각
