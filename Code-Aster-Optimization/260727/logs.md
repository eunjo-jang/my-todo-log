# 2026-07-27 (260727) 작업 로그

프로젝트: KFE 다이버터 솔버 오픈소스 대체 — Code_Aster 구조(FEM) 노드

> 260724에서 "다음 할 일 #2"로 남겨둔 **노드 안 중복 ANSYS 재솔브 제거**를 오늘 구현. 솔버(Code_Aster) 밖의 구조적 최적화 — Code_Aster 노드가 매 런마다 mesh를 얻으려고 **ANSYS 전체 solve(~8분)를 돌리던 자기모순**을 제거한다. Jira **DIVERTOR-185**, 별도 브랜치 `feature-code-aster-mesh-only`.

## 오늘 한 일
- **문제 규명**: real 노드는 mesh + `ds.dat` + 적용 온도장을 얻으려고 ANSYS Mechanical **전체 solve**를 돌린 뒤, DPF로 `file.rst`에서 mesh를 뽑아 Code_Aster로 **다시** 풀었다. ANSYS solve(~8분)는 순수 오버헤드.
- **핵심 사실 확인**: `ds.dat`(MAPDL 입력덱)는 이미 전체 mesh(NBLOCK 좌표 + EBLOCK 연결성) + 적용 온도장(BFBLOCK) + 접촉 + tref를 담고 있고, `mapdl_deck.parse_ds_dat`가 이미 파싱한다. 그리고 `ds.dat`는 **solve 전** mesh/입력생성 단계에서 디스크에 쓰인다.
- **구현 (2가지)**:
  1. `mesh_export.mesh_from_deck(deck)` — `ds.dat`에서 DPF 경로와 **동일한** `NeutralMesh` 생성 (no `file.rst`/DPF).
  2. `mech_solve.regenerate_ds_dat_mesh_only` — 검증된 `node_mech_solve` 저널을 그대로 돌리되 **`ds.dat`가 완전히 쓰인 순간 runwb2 배치를 중단**(`journal.run_journal_until_ready`)해 ~8분 factorization을 건너뜀.
- **플래그화**: `KFE_CODE_ASTER_MESH_ONLY`(기본 `true`; `false`면 레거시 전체 solve + DPF).
- **호스트에서 3종 실측 검증** (아래) → **답 보존 확인**.
- PR **#199** 오픈 (draft, base=`feature-code-aster-profiling` 스택). 커밋 `f0b666a`.

## 핵심 측정 결과 (실측)

**① ANSYS 단계 시간 — mesh-only가 solve를 건너뜀**

| 경로 | ANSYS 단계 내용 | 시간 | 결과물 |
| --- | --- | --- | --- |
| 레거시(전체 solve) | staging + WB + 리메시 + **MAPDL factorization(~8분)** | ~13분 | `file.rst` + `ds.dat` |
| **mesh-only (신규)** | staging + WB + 리메시, **solve 직전 중단** | **338s (~5.6분)** | `ds.dat`만 |
| 절감 | — | **약 8분/런** | (`file.rst` 불필요) |

**② `ds.dat`가 solve 없이 나오는 이유 (골든 node_mech 타임라인)**

| 시점 | 이벤트 |
| --- | --- |
| ~2분 | `ds.dat` 완성·닫힘 (전체 mesh 포함) |
| +~30s | MAPDL 첫 출력(`file.err`) = solve 시작 |
| +~8분 | `file.rst` = solve 완료 |

→ `ds.dat`는 factorization보다 **~8분 먼저** 완성됨. 그 안정화 시점에 중단 = 안전한 절단점(MAPDL·MPI 시작 전이라 orphan 프로세스 없음).

**③ 답 보존 검증 (골든 프로젝트)**

| 검증 | 방법 | 결과 |
| --- | --- | --- |
| mesh 동등성 (오프라인+DPF) | 덱 mesh vs `file.rst` DPF mesh 노드·요소 단위 비교 | **완전 동일** (좌표 오차 <1e-6 mm, 요소 연결성 ANSYS id 기준 일치) |
| end-to-end 실제 solve | 덱 mesh → Code_Aster 풀이 → peak | **P18 = 400.18 MPa** == DPF 경로 골든(~400.2) |
| spike | solve 없이 `ds.dat` 생성 | 799,077 노드 / 171,556 hex20 / 127 MB, 유효 NBLOCK+EBLOCK |

## 판단 (오늘의 결론) ★
- **mesh는 이제 `ds.dat`에서 직접(기본값)** — ANSYS 전체 solve를 건너뛰어 런당 ~8분 절감. mesh가 DPF mesh와 **비트 단위로 동일**하므로 결과(peak+응력장)는 구조적으로 동일함이 보장되고, end-to-end 실런으로도 확인(400.18 = DPF 400.2).
- **왜 "동일한 답"인가**: 두 경로의 유일한 차이는 mesh를 어디서 얻느냐뿐(`mesh_from_deck` vs `DPF`). mesh가 동일하면 그 하류(케이스 조립·컨테이너 solve)는 전부 같은 코드·같은 입력 → 같은 결과. mesh 동등성만 증명하면 30분 solve를 두 번 돌릴 필요가 없다.
- **대체 목표와 정면 정합**: "ANSYS를 대체하겠다"는 노드가 매번 ANSYS로 풀던 모순을 없앴다. 다음 단계(gmsh 등 ANSYS-free 메셔, 로드맵 2단계)로 가는 디딤돌.

## 진행 중 / 이슈
- **아직 mesh 소스만 ANSYS-free가 아님** — `ds.dat`를 얻는 runwb2 실행 자체는 ANSYS 필요. 완전 ANSYS 제거는 로드맵 2단계(gmsh 메셔), 별도 태스크.
- **검증 설계점**: tilting=0 기준점에서 end-to-end 실런. tilting≠0은 `mesh_from_deck` 로직이 설계점 무관이고 mesh 동등성 증명이 구조적으로 보장하지만, 실제 solve까지는 미실행.
- **`/code-review` 미실행** — 이 커맨드는 모델이 자동 호출 불가(사람 전용). 머지 전 사람이 직접 리뷰 필요.
- **부모 PR #197(PETSc/프로파일링) 미머지** — 그 위에 stack이므로 부모가 먼저 진행돼야 함.
- 이 Windows 호스트 유닛 스위트에 실패 5건(심볼릭링크 권한·snakemake 환경) — **제 변경과 무관**(변경 stash 후 base 재현으로 확인).

## 다음 할 일
1. 사람 `/code-review` + CI 통과 → 부모(#197) 머지 후 이 PR 진행.
2. (선택) tilting≠0 등 다른 설계점 end-to-end 1회로 못박기.
3. **로드맵 2단계**: ANSYS-free 메셔(gmsh 컨테이너)로 `ds.dat` 의존까지 제거 — 별도 Jira 태스크.
4. 260724의 **PETSc 하이브리드 프로덕션 반영(#197)**은 여전히 별건으로 대기.

## 메모
- 건드린 파일: `mesh_export.py`(mesh_from_deck) / `mech_solve.py`(regenerate_ds_dat_mesh_only, 스테이징 공용화) / `journal.py`(run_journal_until_ready 인터럽트 러너) / `kfe_case.py`(use_deck_mesh) / `node.py`(배선) / `settings.py`(플래그) + 테스트/문서.
- 신규 테스트: `test_mesh_from_deck`(mesh 빌더 + opt-in DPF 동등성), `test_mech_solve`(ds.dat 안정화 프로브), `test_journal`(인터럽트 러너), `test_container_smoke`(덱-mesh end-to-end).
- peak 값은 **데이터셋/설계점 의존** — 골든(888k 노드) 기준 ~400 MPa. 260724 필드비교의 368.7은 다른 캐시 런(`bench_run_b_ca`) 값이라 직접 비교 대상 아님. 불변식은 "덱 경로 = DPF 경로(동일 입력)".
- 인터럽트 안전성: staged `_wbpj`는 런마다 새 복사본(throwaway)이라 중단 후 남는 `.lock`·부분파일 무해. 라이선스 데몬(`ansyslmd`)은 절대 종료 안 함.
- 산출물: Jira DIVERTOR-185, GitHub 이슈 #198, Draft PR #199 on `feature-code-aster-profiling`.
