# Code_Aster 노드 mesh-only 추출 — ANSYS 전체 재솔브 제거 — 상세 보고서

> 작성일: 2026-07-27
> 대상: KFE 다이버터 솔버 오픈소스 대체 — Code_Aster 구조(FEM) 노드
> 선행: 260723(프로파일링+MPI), 260724(PETSc 하이브리드+필드검증). 본 문서는 260724의 "다음 할 일 #2 — 노드 안 중복 ANSYS 재솔브 제거"를 실제 구현한 것으로, **솔버 밖(아키텍처) 최적화**다. 중복 최소화, 새 내용만 다룬다.
> 산출물: Jira **DIVERTOR-185**, GitHub 이슈 #198, Draft **PR #199**(브랜치 `feature-code-aster-mesh-only`, base `feature-code-aster-profiling`).

---

## 0. 세 줄 요약
- Code_Aster 구조 노드는 mesh를 얻으려고 매 런 **ANSYS Mechanical 전체 solve(~8분 factorization)**를 돌린 뒤 그 mesh로 다시 Code_Aster를 풀었다 — "ANSYS 대체" 노드가 매번 ANSYS로 푸는 자기모순.
- `ds.dat`가 이미 전체 mesh를 담고 있고 solve 전에 쓰인다는 사실을 이용, **(1) `ds.dat`에서 직접 mesh를 만들고 (2) solve 직전에 runwb2를 중단**해 ~8분을 건너뛰도록 바꿈. 플래그 `KFE_CODE_ASTER_MESH_ONLY`(기본 on).
- 호스트 실측으로 **답 보존 확정**: 덱 mesh가 DPF mesh와 **노드·요소 단위로 동일**, end-to-end 실런 peak도 DPF 경로와 일치(400.18 = 400.2).

---

## 1. 배경 / 목적

Code_Aster 구조 노드(real 모드)의 흐름은 이랬다:

    CFX 결과(온도) ─▶ [ANSYS Mechanical 전체 solve] ─▶ file.rst ─▶ [DPF로 mesh 추출] ─▶ [Code_Aster 재솔브] ─▶ peak
                       └ ~8분 factorization (버려짐)            └ mesh + ds.dat

목적은 오직 **mesh + `ds.dat`(적용 온도장 등)**를 얻는 것인데, 그걸 위해 ANSYS의 수치 solve(참고: 상용 Mech 1-core solve ≈ 680s)를 통째로 돌리고 그 결과 `file.rst`는 mesh만 뽑고 버렸다. 즉 매 런 수 분이 순수 오버헤드였고, 오픈소스로 ANSYS를 대체하겠다는 노드의 취지와 정면으로 어긋난다.

**핵심 관찰**: `ds.dat`(MAPDL 입력덱, 평문)는 이미
- `NBLOCK`(노드 좌표) + `EBLOCK`(요소 연결성) = **전체 mesh**,
- `BFBLOCK`(적용 온도장), 접촉 영역, `tref`(기준온도)
를 담고 있고, 저장소의 `mapdl_deck.parse_ds_dat`가 **이미** 이걸 `MapdlDeck.node_coords` / `.elements`로 파싱한다. 게다가 `ds.dat`는 mesh/입력 생성 단계에서 **solve 전에** 디스크에 쓰인다. 따라서 필요한 데이터는 solve 없이 이미 존재한다.

이 작업의 범위는 **(A) ANSYS 경유 mesh-only (속도)** — (B) gmsh 같은 ANSYS-free 메셔(로드맵 2단계)가 아니다.

---

## 2. 진행 내용

### 2.1 `ds.dat` → mesh 변환 (`mesh_export.mesh_from_deck`)
- DPF 경로(`export_structural_mesh`, `file.rst` 읽음)와 **동일한 shape**의 `NeutralMesh`를 덱에서 생성.
- 덱 요소 중 노드 20개짜리(SOLID186 hex volume)만 `hexahedron20` 블록으로. 8개짜리(QUAD8 접촉면)는 제외 — 현행 DPF도 그렇고, 접촉면은 하류 `attach_contact_groups`가 덱에서 append한다.
- 노드 순서(MAPDL EBLOCK)는 DPF의 요소 노드 순서와 동일(둘 다 SOLID186 규약)이라 기존 `HEX20_DPF_TO_MESHIO`(항등) 재사용. `_FIXEDSU` 고정지지 노드군도 덱에서 매핑.
- 요소·노드 id가 덱에서 직접 오므로, 하류 정렬(재질군은 요소 id, 접촉·온도장은 노드 id 기준)이 **구축상 완벽 일치**.

### 2.2 solve 없이 `ds.dat` 뽑기 (`mech_solve.regenerate_ds_dat_mesh_only`)
- 별도 저널을 새로 만들지 않고, **검증된 `node_mech_solve` 저널을 그대로 재사용**. 그 저널은 설계점을 밀어넣고 프로젝트 `Update()`(리메시 → 온도 재임포트 → solve)를 수행한다.
- `Update()`가 solve를 시작할 때 `ds.dat`가 **완전히** 쓰인 뒤 MAPDL이 그걸 읽어 factorization을 돈다. 즉 "`ds.dat` 완성 ↔ MAPDL 시작" 사이에 깨끗한 절단점이 있다.
- 신규 러너 `journal.run_journal_until_ready`가 runwb2를 띄우고, `ds.dat` 크기가 안정되는 순간(`_ds_dat_stable_probe`) **프로세스 트리를 종료**(Windows `taskkill /T`). 라이선스 데몬은 서비스라 자식이 아니므로 안 건드림. staged 프로젝트는 런마다 새 복사본(throwaway)이라 남는 `.lock`·부분파일 무해.
- `Model.Edit()`(이 호스트에서 OpenSSL 크래시)를 피하려고 "write-input-only" 같은 Mechanical 스크립팅 대신 이 인터럽트 방식을 채택.

### 2.3 배선 + 폴백 플래그
- `kfe_case.solve_kfe_bonded(use_deck_mesh=...)`: 덱을 먼저 파싱 → 플래그면 `mesh_from_deck`, 아니면 DPF. hex20 순서 게이트는 양쪽 동일 적용.
- `node.py::_run_real`: `KFE_CODE_ASTER_MESH_ONLY`(기본 True)면 `regenerate_ds_dat_mesh_only` + `use_deck_mesh=True`, 아니면 레거시 `regenerate_solved_rst` + DPF. `file.rst` 의존 제거(다른 출력 포트는 변화 없음).

### 2.4 검증 설계
- **오프라인(항상)**: 합성 `ds.dat`로 `mesh_from_deck` 유닛테스트 + `run_journal_until_ready`/프로브 유닛테스트.
- **호스트 opt-in**: (a) spike(solve 없이 `ds.dat` 생성), (b) 덱 mesh vs DPF mesh 동등성, (c) 덱 mesh end-to-end 실 solve. 실 엔진 테스트는 마커(`requires_ansys`/`requires_code_aster`)로 CI 기본 스위트에서 자동 skip.

---

## 3. 결과 및 분석

### 3.1 Spike — solve 없이 `ds.dat` 생성 (실측)
- 기준 설계점(tilting=0)으로 mesh-only runwb2 실행 → **799,077 노드 / 171,556 hex20 / 127 MB** `ds.dat` 생성, 유효 NBLOCK+EBLOCK, `mesh_from_deck` 파싱 성공.
- **벽시계 338s**(staging + WB 콜드 스타트 + 리메시 포함, solve 직전 중단). 종료 후 orphan MAPDL/hydra 프로세스 **없음** — MAPDL 시작 전에 중단됐다는 뜻.

### 3.2 왜 안전하게 중단되나 — 골든 node_mech 타임라인
| 시점(dp0/SYS/MECH) | 파일 | 의미 |
| --- | --- | --- |
| T+~2분 | `ds.dat` (127~140 MB) | 덱 완성·닫힘 |
| T+~2분30초 | `file.err` | MAPDL 첫 출력 = solve 시작 |
| T+~10분 | `file.rst` | solve 완료 |

`ds.dat`는 solve 완료보다 **~8분 먼저** 확정된다. 단일 flush로 쓰이고(단일 mtime, 이후 30초 조용), 프로브는 크기 안정 6초를 요구하므로 조기 중단 위험 낮음 — 실제로 799k 노드 덱이 온전히 파싱됨.

### 3.3 답 보존 — 덱 경로 ≡ DPF 경로 (골든 프로젝트)

**(a) mesh 동등성 (오프라인 + DPF, ~62s)**
- `mesh_from_deck(골든 ds.dat)` vs `export_structural_mesh(골든 file.rst)`를 ANSYS 노드·요소 id 기준 정렬 후 비교.
- 노드 좌표 최대 오차 **< 1e-6 mm**, hex20 연결성(ANSYS id) **완전 일치**. 888,874 노드 / 192,296 hex20 동일.

**(b) end-to-end 실 solve (컨테이너, ~17.8분)**
| 항목 | 값 |
| --- | --- |
| 덱 mesh → Code_Aster peak | **P18 = 400.18 MPa** |
| DPF 경로 골든 peak (문서 Real-run 2026-07-07) | ~400.2 MPa |
| 판정 | **일치** |

- 생성된 `.comm`이 DPF 경로와 **동일**(같은 `body_mat*` 재질군, `cont_slv_no_193`, `LIAISON_MAIL` 본디드 타이, 온도장) — mesh-only가 하류에 투명함을 재확인.
- 컨테이너 solve 시간(~17.8분)은 mesh 소스와 무관한 **하류 solve**라 양 경로 동일. 절감은 전적으로 ANSYS 단계(~8분).

> **peak 값 주의**: peak는 데이터셋/설계점 의존이다. 여기 골든(888k)은 ~400 MPa. 260724 필드비교의 368.7은 다른 캐시 런(`bench_run_b_ca`) 값이라 직접 비교 대상이 아니다. 이 작업의 불변식은 절대값이 아니라 **"동일 입력에 대해 덱 경로 = DPF 경로"**이며, 그건 (a) 비트동일 mesh + (b) 일치하는 end-to-end peak로 증명됐다.

### 3.4 시간 절감
| 경로 | ANSYS 단계 | 절감 |
| --- | --- | --- |
| 레거시(full solve) | staging+WB+리메시 + **~8분 factorization** | — |
| mesh-only | staging+WB+리메시 (338s), solve 중단 | **~8분/런** |

Code_Aster 컨테이너 solve는 양쪽 동일하므로, 노드 전체 wall에서 ANSYS solve 몫이 통째로 사라진다.

---

## 4. 다음 할 일
1. 사람 `/code-review` + CI 통과 → 부모 PR #197(PETSc/프로파일링) 머지 후 #199 진행.
2. (선택) tilting≠0 설계점 end-to-end 1회로 다설계점 보존 못박기.
3. **로드맵 2단계 — ANSYS-free 메셔**: gmsh 컨테이너로 `ds.dat` 의존까지 제거(같은 `NeutralMesh` 산출 → 하류 무변). 별도 Jira.
4. 260724의 PETSc 하이브리드 프로덕션 반영(#197)은 별건으로 유지.

---

## 5. 결론

Code_Aster 구조 노드가 mesh를 얻으려고 돌리던 **ANSYS 전체 solve(~8분)**를, `ds.dat`가 이미 전체 mesh를 담고 solve 전에 쓰인다는 사실을 이용해 제거했다. `mesh_export.mesh_from_deck`로 덱에서 직접 mesh를 만들고, `run_journal_until_ready`로 `ds.dat` 완성 시점에 runwb2를 중단한다(`KFE_CODE_ASTER_MESH_ONLY`, 기본 on; false로 레거시 복귀). 호스트 실측으로 **덱 mesh가 DPF mesh와 노드·요소 단위로 동일**하고 end-to-end peak도 DPF 경로와 일치(400.18 = 400.2)함을 확인해 **답 보존을 필드가 아닌 mesh 수준에서 구조적으로 보장**했다. 이는 "ANSYS 대체" 노드의 자기모순을 없앤 것이며, 완전 ANSYS 제거(로드맵 2단계, gmsh 메셔)로 가는 디딤돌이다.

---

_원본: 2026-07-25~26 세션 실측(mesh-only spike / DPF 동등성 / end-to-end 덱 solve) 정리. 브랜치 `feature-code-aster-mesh-only`, 커밋 `f0b666a`, Draft PR #199 on `feature-code-aster-profiling`._
