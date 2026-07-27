# Code_Aster 노드 mesh-only — 개념 · Before/After · 실행 관점 정리

> 작성일: 2026-07-28
> 대상: KFE 다이버터 솔버 오픈소스 대체 — Code_Aster 구조(FEM) 노드
> 260727 상세 보고서(mesh-only 추출, DIVERTOR-185)를 처음 보는 사람 기준으로 용어와 흐름부터 정리한 문서.

---

## 1. 이 노드가 하는 일 (한 문단)

`code_aster_structural` 노드는 **ANSYS Mechanical이 하던 열응력 해석을 오픈소스 Code_Aster로 똑같이 재현**하는 노드다. 입력은 CFX가 계산한 온도장, 출력은 구조물의 **최대 등가응력(peak von Mises, P18)**이다. 목표는 "언젠가 ANSYS 없이도 같은 답을 내는 것"이다.

이번 작업(260727)은 그 노드가 **정작 mesh를 얻으려고 매번 ANSYS를 전체로 돌리던 모순**을 없앤 것이다.

---

## 2. 용어 정리 (이 문서를 읽는 데 필요한 최소한)

| 용어 | 뜻 | 비유 |
| --- | --- | --- |
| **mesh(메시)** | 구조물을 잘게 쪼갠 격자(노드+요소). 해석의 기본 입력. | 3D 모델을 레고 블록으로 쪼갠 것 |
| **ANSYS Mechanical** | 상용 구조해석 프로그램(Workbench 안의 한 시스템) | — |
| **MAPDL** | Mechanical 내부의 실제 계산 엔진(솔버) | Mechanical이 "이거 풀어" 하고 넘기는 계산기 |
| **solve / factorization** | mesh+조건으로 실제 방정식을 푸는 무거운 수치 계산 | 가장 오래 걸리는 단계(우리 케이스 ~8분) |
| **ds.dat** | Mechanical이 MAPDL에게 넘기는 **입력 파일**(평문 텍스트). mesh+온도+접촉+재질이 다 들어있음 | **문제지** (아직 답 아님) |
| **file.rst** | MAPDL이 solve를 끝내고 내놓는 **결과 파일**(응력/변형 등) | **정답지** |
| **DPF** | ANSYS의 결과 후처리 라이브러리(Data Processing Framework). `file.rst` 같은 결과 파일을 읽는 도구. ANSYS 세션/라이선스 필요 | 정답지를 읽어주는 전용 리더기 |
| **Code_Aster** | 우리가 쓰는 오픈소스 구조해석 솔버(도커 컨테이너) | ANSYS 대체품 |

핵심 대비: **`ds.dat` = 문제지(mesh 포함, 빨리 만들어짐)**, **`file.rst` = 정답지(solve 8분 후에 나옴)**.

---

## 3. `ds.dat`은 언제 · 어디서 · 누가 만드나

ANSYS Mechanical이 한 번 해석("Solve")을 돌릴 때 내부 순서는 이렇다:

    [1] 형상(Geometry) 생성
    [2] mesh 생성 (구조물을 격자로 쪼갬)
    [3] 온도 등 조건 붙이기 (CFX 온도를 구조 mesh 위로 가져옴)
    [4] Solution 시작:
          (4a) ds.dat 를 디스크에 저장   ← 여기! MAPDL에게 줄 "문제지" 완성
          (4b) MAPDL 실행 → 8분 factorization
          (4c) file.rst 저장            ← "정답지" 완성

- **누가**: 사용자가 만드는 게 아니라, Mechanical이 solve 과정에서 **자동으로** 만든다.
- **언제**: 무거운 계산(4b)을 **시작하기 바로 직전**(4a). 즉 `ds.dat`은 `file.rst`보다 **~8분 먼저** 완성된다.
- **어디서**: Workbench 프로젝트 폴더 안 `..._files/dp0/SYS/MECH/ds.dat`.

이 "8분 먼저 완성된다"는 사실이 이번 최적화의 열쇠다.

---

## 4. Before — 기존 방식 (바꾸기 전)

    CFX 온도 ─▶ ANSYS Mechanical 전체 Solve ─▶ file.rst ─▶ DPF로 mesh 추출 ─▶ Code_Aster로 다시 풀기 ─▶ 최대응력
                     └ ds.dat + file.rst              └ (정답지에서 mesh만 꺼냄)
                     └ 이 중 8분짜리 factorization은 결국 안 씀

무엇이 낭비였나:
- 우리가 진짜 필요한 건 **mesh + 온도(=`ds.dat` 안에 이미 있음)**뿐이다.
- 그런데 그 `ds.dat`을 얻으려고 굳이 뒤의 **8분짜리 solve까지 다 돌려 `file.rst`(정답지)를 만든 다음**, 거기서 DPF로 mesh만 꺼냈다.
- 그 `file.rst`가 담은 ANSYS의 응력 답은 **버려진다**(어차피 Code_Aster로 다시 풀 거라서).
- 즉 "ANSYS를 대체하겠다"는 노드가 매 실행마다 ANSYS를 통째로 돌리는 자기모순 + 8분 낭비.

---

## 5. After — 바뀐 방식 (지금)

    CFX 온도 ─▶ ANSYS mesh 단계만 실행 → ds.dat 나오는 순간 중단 ─▶ ds.dat에서 직접 mesh 만들기 ─▶ Code_Aster로 풀기 ─▶ 최대응력
                     └ 8분 solve는 건너뜀 (file.rst 안 만듦)          └ (문제지에서 바로 mesh 꺼냄, DPF 불필요)

두 가지를 새로 만들었다:
1. **`ds.dat`에서 직접 mesh 만들기** (`mesh_from_deck`) — `file.rst`나 DPF 없이 문제지(`ds.dat`)만 읽어서 mesh를 구성.
2. **solve 직전에 멈추기** (`regenerate_ds_dat_mesh_only`) — 기존 ANSYS 실행을 그대로 돌리되, **`ds.dat` 파일이 다 써진 순간을 감지해서 ANSYS를 강제 종료**. 뒤의 8분 factorization은 시작도 안 함.

`ds.dat`이 solve보다 먼저 완성되기 때문에 이 "중간에 끊기"가 안전하게 가능하다.

---

## 6. 실제로 어떻게 돌리나 (실행 관점)

**바뀐 것과 그대로인 것을 구분하는 게 중요하다.**

- **파이프라인을 돌리는 명령/방식은 그대로다.** real 모드로 시나리오(예: `kfe_code_aster_structural_real`)를 실행하면 노드가 알아서 돈다. 사용자가 새로 칠 명령은 없다.
- **바뀐 건 노드 "내부"에서 밟는 단계뿐이다.** real 모드에서 노드가 실행하는 내부 절차가 아래처럼 달라졌다:

| 단계 | Before | After (기본값) |
| --- | --- | --- |
| ANSYS 실행 | Mechanical **전체 Solve** (~13분, 8분 factorization 포함) | **mesh-only 실행** (~5.6분, solve 직전 중단) |
| mesh 얻는 법 | `file.rst`에서 **DPF로 추출** | `ds.dat`에서 **직접 구성** |
| `file.rst` | 만들어짐(그리고 버려짐) | **안 만듦** |
| Code_Aster 풀이 | 동일 | 동일 |
| 최대응력 결과 | 동일 | **동일** |

- **되돌리고 싶을 때만** 환경변수 하나로 옛 방식으로 스위치한다. `backend/.env`에:

      KFE_CODE_ASTER_MESH_ONLY=false   # 옛날 방식(전체 Solve + DPF)로 복귀

  (기본값은 `true` = 새 mesh-only 방식. 아무것도 안 하면 새 방식으로 돈다.)

정리하면 **"돌리는 법은 똑같은데, 노드가 속으로 8분짜리 헛일을 안 하게 됐다"**가 사용자 입장의 변화다.

---

## 7. 결과 (호스트에서 실제로 측정)

- **시간**: ANSYS 단계가 ~13분 → **~5.6분** (런당 **약 8분 절감**). Code_Aster 풀이 시간(~18분)은 mesh 출처와 무관하므로 그대로.
- **답 보존(가장 중요)**: 새로 만든 mesh가 기존 DPF mesh와 **노드·요소 단위로 완전히 동일**(좌표 오차 < 1e-6 mm)함을 확인. 같은 mesh를 같은 솔버에 넣으니 결과도 같다.
- **실제 끝까지 풀어서 확인**: 새 방식으로 Code_Aster를 돌린 최대응력 **P18 = 400.18 MPa** = 기존 방식 골든값(~400.2)과 일치.

---

## 8. 아직 안 된 것 / 다음

- 지금은 **mesh를 얻는 실행 자체는 여전히 ANSYS**를 쓴다(그 안의 8분 solve만 뺐을 뿐). **완전히 ANSYS를 없애는 건 다음 단계**(gmsh 같은 오픈소스 메셔로 직접 mesh 생성 — 별도 작업).
- 코드 리뷰(사람) + CI 통과 후 머지 예정. 현재는 Draft PR #199.

---

## 9. 한 장 요약

- `ds.dat` = ANSYS가 solve 직전에 자동으로 만드는 **문제지**(mesh 다 들어있음). `file.rst` = 8분 뒤 나오는 **정답지**. DPF = 정답지를 읽는 ANSYS 도구.
- 기존엔 문제지 얻자고 정답지까지(8분) 만들고 DPF로 mesh만 꺼내 썼다 → 낭비.
- 지금은 **문제지가 만들어지는 순간 ANSYS를 멈추고, 그 문제지에서 바로 mesh를 꺼내** Code_Aster로 푼다.
- 돌리는 명령은 그대로, 결과도 동일, ANSYS 단계만 ~8분 짧아졌다. 되돌리려면 `KFE_CODE_ASTER_MESH_ONLY=false`.
