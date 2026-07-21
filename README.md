# my-todo-log

매일매일의 작업 내용을 프로젝트별로 기록하는 개인 작업 로그 저장소입니다.

## 구조

```
my-todo-log/
├── OpenFoam-Optimization/    # 프로젝트 (큰 개념) 단위 폴더
│   └── 260721/               # 날짜 폴더 (YYMMDD)
│       ├── logs.md           # 간략한 데일리 로그
│       └── logs_detail.md    # 상세 보고서 (제3자도 읽을 수 있게)
├── HEAT-Optimization/
│   └── ...
└── ...
```

- **최상위 폴더** = 진행 중인 프로젝트의 큰 개념 (예: `OpenFoam-Optimization`, `HEAT-Optimization`)
- **날짜 폴더** = `YYMMDD` 형식 (예: 2026-07-21 → `260721`)
- 날짜 폴더 안에 두 종류의 기록을 둡니다:
  - **`logs.md`** — 그날 한 일 / 진행 중·이슈 / 다음 할 일 / 메모를 간략히
  - **`logs_detail.md`** — 분석 과정과 결과를 제3자도 쉽게 읽을 수 있는 보고서 형태로 상세히

## 작성 템플릿

- 간략 로그: [`_templates/daily-log.md`](_templates/daily-log.md)
