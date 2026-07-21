# 2026-07-21 (260721) 작업 로그

프로젝트: KFE 다이버터 솔버 오픈소스 대체 — OpenFOAM 열유동 노드

## 오늘 한 일
- OpenFOAM 열유동 노드(`openfoam_cht`)를 **직렬 전용 → N코어 MPI 옵션**으로 확장 (기본 1코어=기존 동작 그대로, `KFE_OPENFOAM_CORES>1`로 opt-in)
- 4코어 실측 검증 완료: **4,647초 → 1,679초 (2.77배 가속)**, 온도 결과는 직렬과 라운드오프 수준으로 동일 → 병렬 물리 정확
- 병렬 실패 원인 2건(root-MPI 거부, 리전별 `decomposeParDict` 필요) 규명·해결
- 지난 벤치마크 3개 세션(07-20) 결과를 하나의 보고서로 정리 (→ `logs_detail.md`)

## 진행 중 / 이슈
- OpenFOAM 온도 정확도: 유체·모노블록이 CFX 대비 +5~6%로 허용오차(5%) 근소 초과 → 캘리브레이션 필요
- Code_Aster 응력 −8.1% 차이 원인 미규명
- 벤치마크 타이머에 Workbench 시동·메시 재생성 같은 고정 오버헤드가 섞여 "순수 solve" 비교가 아직 불완전
- `from_med.py:57` meshio `.rmed` 인식 버그 (solve 성공에도 "실패" 표시) — 수정안 검증됨, PR 미반영

## 다음 할 일
- OpenFOAM 온도 캘리브레이션 (물 물성치·벽면 y+·열유속 매핑 튜닝)
- OpenFOAM 추가 가속: 분할방식 개선(`(N 1 1)`→`(2 2 1)`/scotch), 코어 확대(호스트 56코어), GPU 오프로드(`petsc4Foam`)
- Code_Aster −8.1% 원인 분석
- 타이머 범위 정규화(순수 solve만 측정)
- `from_med.py` meshio 버그 PR
- `feature-openfoam-mpi` 브랜치 `/code-review` 후 draft PR

## 메모
- 커밋 `dc00691` on `feature-openfoam-mpi`
- 동일 코어 비교 결과 격차: 20배(1 vs 1코어) → 7배(4 vs 4코어)로 좁혀짐
- 코어 수는 노드 fingerprint에서 제외(캐시 re-key 방지, ANSYS 방식과 동일)
- 원시 데이터: `runs/bench_openfoam`(1코어), `runs/bench_openfoam_4core_v4`(4코어)
