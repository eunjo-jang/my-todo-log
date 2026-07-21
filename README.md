# my-todo-log

매일매일의 작업 내용을 프로젝트별로 기록하는 개인 작업 로그 저장소입니다.

## 구조

```
my-todo-log/
├── OpenFoam-Optimization/   # 프로젝트 (큰 개념) 단위 폴더
│   ├── logs/                # 날짜별 작업 로그 (YYYY-MM-DD.md)
│   └── ...                  # 코드, 노트, 자료 등
├── HEAT-Optimization/
│   ├── logs/
│   └── ...
└── ...
```

- **최상위 폴더** = 진행 중인 프로젝트의 큰 개념 (예: `OpenFoam-Optimization`, `HEAT-Optimization`)
- **폴더 하위** = 해당 프로젝트의 코드, 그리고 대부분은 날짜별 작업 로그 마크다운

## 작업 로그 작성법

각 프로젝트의 `logs/` 폴더 안에 `YYYY-MM-DD.md` 형식으로 하루치 로그를 남깁니다.
템플릿은 [`_templates/daily-log.md`](_templates/daily-log.md)를 참고하세요.
