# HEAT 열유속 계산 최적화 — 통합 정리 (Optical / Gyro)

> 작성일: 2026-07-21
> 대상: KFE × EnableFusion 다이버터 설계 협업의 HEAT 열유속 엔진 가속
> 종합 소스: 로컬 자료(`07_Surrogate`), `EnableFusion-AI/DesignCore`, `EnableFusion-AI/heat-svc`

이 문서는 여러 곳에 흩어져 있던 최적화 자료(설계 문서·프로파일링·보고서·리포 커밋)를 시간순으로 종합하여, **무엇을 어떻게 최적화했고, 그 결과가 무엇이며, 코드의 어디에 어떻게 반영돼 있는지**를 하나로 정리한 것이다.

---

## 0. 한눈에 보기

| 모드 | 최적화 대상 단계 | Before → After | 배율 | 시기 | 반영 위치 (리포 / 브랜치) |
| --- | --- | --- | --- | --- | --- |
| **Optical** | shadowWalk (필드라인 그림자 추적) | 518s → 30s | **17.2×** | 2026-06 | heat-svc `enf_heat` (초기 DesignCore `enf-heat-node`) |
| **Optical** | 전체 실행 (공식 stock 대비) | 931s → 159s(CPU) / 142s(GPU) | **5.9× / 6.6×** | 2026-06 | 〃 |
| **Gyro** | 자이로 궤도 추적 전체 | 271.5min → 16~18min | **~16×** | 2026-07 | heat-svc `feature/gyro-orbit-mp` |
| **Gyro** | gyroHF2 파워 재분배 루프 | — | 4.1× (해당 루프) | 2026-07 | heat-svc `feature/gyro-inmem-gc` |
| (보너스) **Radiative** | Open3D 광자 전달 | 57.4s → 36.2s | 1.6× | 2026-07 | heat-svc `feature/rad-open3d-opt` |

**핵심 결론**: Optical·Gyro 두 모드 모두 물리 결과를 보존한 채 대폭 가속했다. Optical은 "외부 프로그램(MAFOT) 호출 제거 → 인메모리 RK4 적분"이, Gyro는 "레이캐스팅 씬 1회 생성 + 배치 처리 + 멀티코어화"가 최대 기여다. 실제 엔진 코드는 `heat-svc`에 있고, `DesignCore`가 이를 원격 서비스(`enf` 모드)로 호출한다.

---

## 1. 배경 — HEAT와 세 가지 열유속 모드

**HEAT**는 토카막 다이버터 등 플라즈마 대향 부품(PFC)의 표면 열유속을 예측하는 오픈소스 코드다(`plasmapotential/HEAT`, Docker `plasmapotential/heat:v4.2.6`). 자기력선을 따라가며 각 표면 삼각형이 열을 직접 받는지(LIT) 다른 부품에 가려지는지(SHADOW)를 판정하고(shadowMask), 받는 열유속(qDiv)을 계산한다. 레퍼런스 케이스는 **KSTAR 텅스텐 다이버터, shot 38798 @ 8.7s**이다.

열유속이 표면에 도달하는 물리 경로에 따라 세 가지 모드가 있으며, 모두 HEAT v4.2에 내장돼 있다(배치 파일 `Output` 열에서 `hfOpt`/`hfGyro`/`hfRad`로 선택, 조합 가능).

| 모드 | 물리 | 특징 | 무거운 커널 |
| --- | --- | --- | --- |
| **Optical (광학)** | 열이 자기력선을 따라 **직진**. 앞이 가려지면 뒤는 열 0(그림자). | 가장 단순한 기준 모델. 비용은 그림자 판정(필드라인 추적)에 집중. | 필드라인 따라 광선-삼각형 교차 |
| **Gyro (자이로 궤도)** | 이온이 유한한 자이로(라머) 반경으로 **나선 운동** → optical이 "그림자"라 한 면에도 열이 도달. optical 위에 얹는 **보정**. | 더 현실적이나 계산이 무겁다. | 자이로 궤도 매크로입자 추적(속도·위상 샘플링) |
| **Radiative (복사)** | 플라즈마의 **광자 복사**가 자기장과 무관하게 **직선(line-of-sight)** 으로 전달 → 자기 그림자에 숨은 곳도 가열. | 별도 계산 경로(복사원 + Mitsuba3 광선추적), 복사원 데이터 필요. | Mitsuba3 광자 광선추적 |

물리적으로 optical → gyro → radiative 순으로 정교해지고 무거워진다. 이번 최적화의 주 대상은 **Optical**과 **Gyro** 두 모드이다.

---

## 2. 전체 타임라인 (시간순)

### 2026년 6월 — Optical 모드 최적화 (완료)

| 날짜 | 이정표 |
| --- | --- |
| 06-01 | 역할·방향 정리 |
| 06-04 | Surrogate PoC 계획 + Step 0 프로파일링 계획 |
| 06-05 | 마스터플랜(3주 스프린트) 수립 |
| 06-06 | **Phase 0 프로파일링** — shadow/FLT가 병목(~56~70%)임을 규명. "8.8초 착시"(open3d 커널만 잰 값)를 깨고, 진짜 비용은 숨은 MAFOT 추적 + `struct.dat` 디스크 I/O(686s)임을 밝힘 |
| 06-09 | **Phase 0 §4.5 타이머 분해** — shadow 블록 = MAFOT 43% + 디스크 I/O 40% + open3d 1.3%. READ 파서 교체(`genfromtxt`→`pandas`)로 329.7s→72.8s |
| 06-10 | 광학 열유속 재작성 전략 문서 |
| 06-11~12 | **Phase D** — MAFOT를 인메모리 RK4로 교체. 전체 22.1분→4.75분(4.65×) |
| 06-15 | DesignCore `enf_heat` 노드 통합 브리프(`--version old/new` 안전 비교 스캐폴딩) |
| 06-22~25 | **종합 보고서** — shadowWalk 518s→30s(17×), 공식 931s→159s/142s(~6×), 결과 비트 동일 |
| 06-25 | 다이버터 회의 — optical 위에 **gyro + radiative 추가** 결정, 3D 가능성 논의 |
| 06-26 | 자료 폴더 재정리, gyro/rad/3D가 "현재 작업"으로 전환 |

### 2026년 7월 — Gyro 모드 최적화 + Radiative + Surrogate 착수 (heat-svc)

| 날짜 | 커밋 | 이정표 |
| --- | --- | --- |
| 06-23 | (PR #1 merged) | HEAT를 서버측 vending 서비스 + CLI로 래핑 → **`heat-svc` 독립 서비스 탄생** |
| 07-01 | `73f5875` | **Gyro ~4.4×** — 레이캐스팅 씬 1회 생성 + 배치 raycast + O(N²)→O(N) 호이스팅. 271.5→61.5분 |
| 07-08 | `6145840` | **Gyro ~16×** — 멀티코어 helix 빌드 + 영속 Pool 캐시. 271.5→16~18분 |
| 07-12 | `6a04fc5` | Gyro 인메모리 RK4 guiding-center(`ENF_GC_INMEM=1`, opt-in), MAFOT 대체 |
| 07-12 | `9664211` | `gyroHF2` 파워 재분배 `np.add.at` 벡터화(해당 루프 4.1×), 비트 동일 |
| 07-14 | `1c2e90a` | Radiative Open3D 사전할당 버퍼(1.6×, `ENF_RAD_OPT`) + 정확성 수정 2건 |
| 07-16 | `409f86a` 등 | 프로파일러 + ML export/Surrogate 스켈레톤 착수 |

---

## 3. Optical 모드 최적화 (2026-06, 완료)

### 3.1 병목 진단

Optical의 비용은 열유속 수학이 아니라 **그림자 판정(필드라인 추적, FLT)** 에 있었다. 결정적 발견은 "8.8초 착시"였다 — 로그에 찍히는 교차 커널 시간(open3d)은 8.8초에 불과했지만, 실제 비용은 그 안에 숨은 **MAFOT 외부 프로그램 호출 + `struct.dat` 파일 왕복 I/O(686s, 전체의 55.9%)** 였다. old 코드는 필드라인을 한 스텝 전진시킬 때마다 MAFOT 프로세스를 새로 띄우고 좌표를 디스크로 주고받았고, 이를 364회 반복했다.

### 3.2 단계별 최적화 (shadowWalk 기준)

| 단계 | 기법 | shadowWalk | 누적 배율 |
| --- | --- | --- | --- |
| 0. old | MAFOT 외부 프로세스(스텝·방향마다 실행 + 디스크 왕복) | **518s** | 1.0× |
| 1 | **인메모리 RK4 적분** — 축대칭 필드라인 ODE를 numpy 메모리에서 전면 벡터화(외부 프로세스·디스크 I/O 제거) | 113s | 4.6× |
| 2 | **B-field 보간 교체** — `scipy.RectBivariateSpline.ev` → `scipy.ndimage.map_coordinates`(동일 큐빅, C 루틴) | 49s | 10.6× |
| 3 | GPU 보간(cupy) — 진단용 징검다리 | 34s | — |
| 4 | **RK4+보간 전체를 GPU 상주**(`_traceStep_gpu`) + vtpMesh 출력 캐시 | **30s** | **17.2×** |

보조 최적화: vtpMesh 출력 시 삼각형 좌표 배열을 메쉬별로 1회 생성·캐시(24s→14s, 파일 비트 동일). Phase D 트랙에서는 출력 포맷 축소(GLB/VTP off, CSV만)만으로 출력 656s→24s.

### 3.3 종합 결과 (공식 측정)

공개 stock HEAT v4.2.6과 최적화 버전을 **동일 데이터·동일 출력 조건(vtpMesh)** 으로 비교:

| 구성 | 벽시계 | vs stock |
| --- | --- | --- |
| Stock 오리지널 + vtpMesh | 931s (15.5분) | 1.0× |
| 최적화 CPU | **159s (2.7분)** | **5.9×** |
| 최적화 GPU | **142s (2.4분)** | **6.6×** |

GPU가 CPU 대비 +12%에 그치는 이유는 광선추적(raycast)이 CPU 전용(open3d/Embree)이기 때문이다. 즉 이득의 대부분은 GPU가 아니라 **CPU 알고리즘 최적화**에서 나왔다. (전체 실행 기준으로는 mesh load ~60s가 미최적화로 남아 배율이 희석됨.)

### 3.4 정확도 검증

모든 최적화 단계에서 shadowMask·물리량은 **비트 단위로 동일**하다(max qDiv = 59.634174 불변). 유일한 미세 차이는 old(MAFOT) vs new(RK4)에서 **2,046,192면 중 161면(0.0079%)** 의 그림자 분류가 뒤집힌 것으로, RK4와 MAFOT가 같은 ODE를 미세하게 다르게 적분해 180스텝 누적 ~28µm(mm급 메쉬의 1/100) 벌어진 결과다. 경계 근처 grazing 면에서만 발생하고 전체 열유속 영향은 ~0.05%, 시스템 피크 열유속은 불변이다. (전용 검증 하네스 `phaseD_validate.py`로 스텝당 오차 max 0.36µm 확인.)

### 3.5 코드 반영 위치 (`enf_heat/source/`)

- `MHDClass.py` — `traceFieldLineStep_inmemory()`(벡터화 RK4), `_fastBfield_inmemory()`, `_traceStep_gpu`(GPU 상주)
- `pfcClass.py` — `findOpticalShadows`(optical 그림자 워크). MAFOT 왕복을 인메모리 호출로 대체
- `engineClass.py` — `HF_PFC`(optical 열유속), `MHD.flt_inmemory` 플래그(False = MAFOT로 롤백)
- `toolsClass.py` — `readStructOutput` 파서 pandas 화(Phase 0 트랙)
- `phaseD_validate.py` — 검증 하네스
- 플래그: `--version old|new`(`ENF_VERSION`), GPU: `HEAT_GPU=1 ENF_GPU_INTERP=1`

---

## 4. Gyro 모드 최적화 (2026-07, 완료)

Optical 완료 후, 회의 결정에 따라 gyro 모드를 대상으로 동일한 "프로파일 → 병목 제거" 루틴을 적용했다. 기준 자이로 궤도 추적은 **271.5분**으로 매우 무거웠다.

### 4.1 커밋별 최적화

| 커밋 | 날짜 | 기법 | 결과 | 브랜치 |
| --- | --- | --- | --- | --- |
| `73f5875` | 07-01 | ① Open3D RaycastingScene 1회만 생성 ② 전 면 단일 배치 `cast_rays` ③ 루프 내 per-face 팬시 인덱싱 배열을 밖으로 호이스팅(O(N²)→O(N)) | 271.5→61.5분 (**~4.4×**) | `feature/gyro-orbit-optim` |
| `6145840` | 07-08 | 멀티코어 helix 빌드(`_enfHelixWorker` 정적 메서드, self 피클 회피) + 영속 Pool 캐시(`_enfGetPool`) | 271.5→~16~18분 (**~16×**) | `feature/gyro-orbit-mp` |
| `6a04fc5` | 07-12 | 인메모리 RK4 guiding-center 추적(`traceMultiStep_inmemory`), MAFOT 대체, `struct.dat` 레이아웃 재현. `ENF_GC_INMEM=1` opt-in | opt-in | `feature/gyro-inmem-gc` |
| `9664211` | 07-12 | `gyroHF2` 파워 재분배를 `np.add.at`으로 벡터화 | 해당 루프 4.1×, 비트 동일 | `feature/gyro-inmem-gc` |

Gyro 역시 모든 최적화가 물리 결과를 보존한다(bit-identical). 큰 흐름은 Optical과 동일한 철학이다: **불필요한 반복 작업(씬 재생성·프로세스·디스크 I/O) 제거 + 벡터화 + 멀티코어**.

### 4.2 코드 반영 위치 (`enf_heat/source/`)

- `gyroClass.py` — `multipleGyroTraceOpen3D`(최적화된 핫 커널)
- `engineClass.py` — `gyroOrbitIntersects`, `gyroOrbitHF`
- `pfcClass.py` — `findGuidingCenterPaths`, `findHelicalPathsOpen3D`
- `heatfluxClass.py` — `gyroHF` / `gyroHF2`(파워 재분배)
- `MHDClass.py` — `traceMultiStep_inmemory`(GC 인메모리 추적)

---

## 5. (보너스) Radiative 모드 최적화 (2026-07)

Optical/Gyro만큼 핵심은 아니지만, radiative 경로도 함께 손봤다.

- `1c2e90a`(07-14) — `radClass.calculatePowerTransferOpen3D_opt`, 광선 버퍼 사전할당으로 **57.4s→36.2s(~1.6×)**, `ENF_RAD_OPT` opt-in
- `de9ebab` — BYOM/STL combinedMesh 쓰기 크래시 수정
- `d496f0d` — 복사 결과가 비결정적으로 전부 0이 되던 문제 수정(해시 랜덤화 집합 순회 → ROI 우선 씬 순서)

반영: `radClass.py`, `engineClass.radPower`, 브랜치 `feature/rad-open3d-opt`.

---

## 6. 어디에 어떻게 반영돼 있나 — 리포 구조

최적화는 **두 리포에 걸쳐** 반영돼 있다. 실제 최적화 엔진은 `heat-svc`에 있고, `DesignCore`는 이를 소비만 한다.

### 6.1 heat-svc — 최적화 엔진의 본체

`EnableFusion-AI/heat-svc`는 최적화된 HEAT 엔진(`plasmapotential/HEAT`의 사내 포크)을 **독립 서비스(FastAPI vending API + `heat` CLI)** 로 래핑한 것이다. 엔진 소스는 `custom_solvers/enf_heat/source/` 아래에 벤더링돼 있고, **Optical·Gyro·Radiative 물리와 모든 최적화가 여기에 산다**.

- 서비스 래핑: PR #1(2026-06-23 merged) — dev로 병합됨
- **주의**: 위 최적화(gyro 4.4×/16×, 인메모리 GC, rad, surrogate)는 아직 **feature 브랜치 상태**이며 별도 PR·dev 병합 전이다. 브랜치별 정리는 §2·§4 표 참조.
- 엔진 파일 지도는 §3.5 / §4.2 참조.

### 6.2 DesignCore — 오케스트레이터(소비 측)

`EnableFusion-AI/DesignCore`는 KFE 다이버터 설계 워크플로우(FUSE·SOLPS·CATIA·HEAT·ANSYS·OpenFOAM·Code_Aster)를 fingerprint 노드 DAG로 엮는 오케스트레이터다. HEAT 관련 실작업은 `origin/dev` 브랜치에 있다.

- HEAT 노드: `backend/app/node/heat/node.py` — `mock` / `real` / `enf` 세 모드
  - `real`: `plasmapotential/heat:v4.2.6`를 Docker로 직접 실행
  - **`enf`: 원격 EnableFusion heat-service(=heat-svc)를 호출**(`enf_client.py`로 submit/poll/download). PR #152(2026-06-23)에서 추가. → **이것이 DesignCore ↔ heat-svc 연결고리**다.
- 현재 파이프라인은 **Optical만 완전 배선**돼 있다. Gyro·Radiative 입력 블록은 `blender_stl_to_heat_input`의 `KSTAR_input.csv.template`에 존재하나 gold 기본값으로 고정돼 있고, 출력을 수확하는 코드는 아직 없다(latent).
- 부수 최적화: HEAT 노드는 Windows(WSL2 9p) 바인드 마운트의 느린 I/O를 피하려 **컨테이너 내부 저장소 + `docker create`/`docker cp` 스테이징**을 쓴다(DIVERTOR-126).

### 6.3 연결 관계 요약

```
[DesignCore]  HEAT 노드(enf 모드)  ──enf_client──▶  [heat-svc]  vending API/CLI
                                                          │
                                                    custom_solvers/enf_heat/source/
                                                    (Optical/Gyro/Rad 물리 + 최적화)
```

---

## 7. 다음 단계 — Surrogate (예정)

최적화의 다음 가속 레버는 **ML 서로게이트**다. PPPL의 HEAT-ML(Corona et al., *Fusion Engineering and Design* 217 (2025) 115010)을 재현하여, 몇 개의 평형 파라미터(Ip, q95, 자속 각도)로 **그림자 마스크를 예측**해 수십 분을 밀리초로 줄이는 것이 목표다.

- 스켈레톤 착수: `409f86a`(07-16) — `mlExport.py`를 `HF_PFC`의 그림자 마스크 계산 뒤에 훅(`ENF_ML_EXPORT=1`), 독립 패키지 `custom_solvers/enf_surrogate/`. 브랜치 `feature-surrogate-build`.
- 위치: 프로젝트 로드맵상 Phase 4(Scaling). 3D 64-카세트 확장(축대칭 1/64만 풀던 것을 전량 → 최대 ~64× 계산량)을 감당하기 위한 핵심 수단.
- 상태: **아직 미구축(PoC/연구)**. 평형 데이터셋 확보(KFE 요청 또는 FreeGS 합성)와 3D 확장 일정에 게이팅됨.

---

## 부록 A. 참조 데이터·환경

- 테스트 데이터: KSTAR `260522_optis_heat_kstar_wdiv_share`, shot 38798 @ 8.7s, 다이버터 메쉬 PFC.001 = **2,046,192면**(그림자 141,135 / 노출 1,905,057)
- 하드웨어: Intel 56코어 CPU + NVIDIA RTX 6000 Ada(49GB)
- 실행: WSL2 Ubuntu + docker, 이미지 `plasmapotential/heat:v4.2.6`
- GPU 광선추적(OptiX)은 WSL2에서 불가 → 광선추적은 CPU Embree/open3d 상한

## 부록 B. 주요 환경 플래그

| 플래그 | 의미 |
| --- | --- |
| `--version old\|new`, `ENF_VERSION` | old = stock MAFOT, new = 인메모리 RK4 (Optical) |
| `HEAT_GPU=1`, `ENF_GPU_INTERP=1` | Optical GPU 보간/상주 경로 |
| `ENF_GC_INMEM=1` | Gyro 인메모리 RK4 guiding-center (opt-in) |
| `ENF_RAD_OPT` | Radiative Open3D 최적화 경로 (opt-in) |
| `ENF_ML_EXPORT=1` | Surrogate 학습용 데이터 export (opt-in) |
| `HEAT_PROFILE` / `HEAT_AB` / `ENF_DEBUG` | 프로파일링·A/B·픽스처 비교 하네스 |

---

_종합: 로컬 자료 `07_Surrogate`(계획·프로파일링·6월 보고서) + `EnableFusion-AI/heat-svc`(7월 최적화 커밋) + `EnableFusion-AI/DesignCore`(오케스트레이션·소비 측). 2026-07-21 정리._
