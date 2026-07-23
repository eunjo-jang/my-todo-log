# 2026-07-23 (260723) 작업 로그

프로젝트: KFE 다이버터 솔버 오픈소스 대체 — OpenFOAM 열유동 노드

> 0722 결론("config·코드 최적화 여지 없음")에 이어, 코어 수를 **8·16까지 확장**해 강한 스케일링(strong scaling)을 측정하고 최적화 여지를 최종 재검토. 판단 기준 2가지: ① 매쉬가 속도 병목인가, ② 병목 계산을 ANSYS보다 빠르게 가속할 수 있는가. (서버 56코어 보유)

## 오늘 한 일
- 기존 MPI 병렬화(4코어 2.77배)에서 **코어를 8·16까지 확장**해 강한 스케일링 곡선 완성.
- 코어별 real 런 측정: 상류 노드(ansys_dm/blender/heat)는 CAS 캐시 히트로 복원, **openfoam_cht만 강제 재계산**(코어 수는 fingerprint 아님 → CAS 비운 뒤 측정, 완료 후 원복).
- 결과 보존 확인 + Amdahl 피팅으로 점근 하한 추정.

## 결과 (solve phase, `chtMultiRegionSimpleFoam` ExecutionTime)
| 코어 | solve 시간 | 1코어 대비 speedup | 병렬효율 |
|---|---|---|---|
| 1 | 4008.7 s (66.8분) | 1.00× | 100% |
| 4 | 1605.8 s (26.8분) | 2.50× | 62% |
| 8 | 1195.3 s (19.9분) | 3.35× | 42% |
| 16 | 864.1 s (14.4분) | 4.64× | 29% |

- 병렬효율이 **62% → 42% → 29%로 급락**. 8→16코어(코어 2배)에서 solve 시간은 **28%만 감소**.
- Amdahl 피팅상 직렬 구간 **s ≈ 0.16~0.20** → 코어를 무한히 늘려도 solve 하한 **≈ 650~800초**. 16코어(864초)가 이미 이 하한에 근접.
- **결과 보존**: 1/4/8/16코어 max(T) 편차 ≤ **0.009°C (~0.001%)**. 코어 수는 답을 바꾸지 않는 순수 성능 레버.

## 판단 (오늘의 결론) ★
- **기준① 매쉬가 병목인가 → 아니오.** 매쉬 파이프라인(checkMesh·decompose·reconstruct·import·VTK) 합계 **<1%**, 나머지 ~99%가 solve.
- **기준② 병목(solve)을 ANSYS보다 빠르게 가속 가능한가 → 아니오.**
  - CPU 코어: 16코어 864초 vs **CFX 4코어 237초 → 3.6배 느림**. 점근 하한(650~800초)도 CFX 대비 **2.7~3.4배 느려** 어떤 코어 수로도 못 따라잡음.
  - GPU 선형솔버: 선형대수는 solve의 **26%뿐**(나머지 74%는 방정식 조립·물성 조회·다영역 커플링, 전부 CPU). 무한 가속해도 최선 ~1186초로 여전히 CFX 대비 ~5배 느림 → dead-end.
- → OpenFOAM은 **기준②를 충족 못 함**. 코드 레벨 비효율도 없음(이미 최적 솔버 사용). **OpenFOAM 최적화 종료.**

## 진행 중 / 이슈
- OpenFOAM 트랙은 종료. 실질 가속을 낼 유일한 방향은 서로게이트이나 현재 우선순위 아님.

## 다음 할 일
- **Code-Aster를 동일 기준으로 분석**: ① 매쉬(전처리)가 병목인가, ② 병목 계산을 ANSYS Mechanical(baseline solve ≈ 316초)보다 빠르게 가속 가능한가. MPI/코어 스케일링 + 병목 프로파일링을 동일 포맷으로 공유.

## 메모
- 원자료: `backend/runs/bench_openfoam_{8,16}core/` (events.ndjson, `nodes/openfoam_cht/_of_result/profiling_logs/log.chtMultiRegionSimpleFoam`, metrics json).
- CFX 4코어 baseline 237초는 07-20 벤치(`ansys-baseline-benchmark-times`) 참조.
- 재현 주의: 코어 수는 fingerprint 입력이 아니므로 `graph_cache/openfoam_cht` CAS를 비워야 재계산됨(안 비우면 이전 결과 복원 = 가짜 성공).
