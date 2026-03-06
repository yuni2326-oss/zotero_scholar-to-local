# Zotero Scholar to Local

키워드로 논문을 자동 검색하여 Zotero 라이브러리에 추가하고, 요약 문서(docx)를 생성하며, AI로 연구 제안을 평가하는 파이프라인 도구입니다.

## 주요 기능

| 기능 | 설명 |
|------|------|
| **다중 소스 검색** | Google Scholar + OpenAlex + Semantic Scholar 동시 검색 및 중복 제거 |
| **Zotero 자동 추가** | 검색 결과를 로컬 Zotero SQLite에 직접 추가 (컬렉션 자동 생성) |
| **docx 요약 생성** | 초록 한국어 번역 포함 논문 요약 Word 문서 생성 |
| **AI 분석 (Claude)** | Claude Code + Zotero MCP로 연구 제안 타당성 평가 |
| **AI 분석 (NLM)** | NotebookLM으로 연구 제안 타당성 평가 (GUI 내 로그 출력) |
| **GUI / CLI** | tkinter GUI와 명령줄 인터페이스 모두 지원 |

## 스크린샷

```
┌─────────────────────────────────────────┐
│  키워드  [나노플라스틱 유해성          ]  │
│  논문 수 [5 ▲▼]  최근 [5 ▲▼] 년       │
│  내 연구 제안                            │
│  [                                     ] │
│  [  검색 시작  ] [AI 분석(Claude)] [AI 분석(NLM)] │
│─────────────────────────────────────────│
│ [START] 키워드: 나노플라스틱 유해성      │
│ [INFO] Google Scholar 검색 중...        │
│ [INFO] Scholar 결과: 5편                │
│ [INFO] OpenAlex 결과: 5편               │
│ [INFO] Semantic Scholar 결과: 5편       │
│ [INFO] 병합 후 총 12편                  │
│ [완료] 저장 파일: 나노플라스틱 유해성.docx│
└─────────────────────────────────────────┘
```

## 설치

### 필수 요구사항

- Python 3.10 이상
- [Zotero](https://www.zotero.org/) 설치 및 로컬 라이브러리 존재

### 패키지 설치

```bash
# docx 생성 (필수)
pip install python-docx

# NotebookLM 분석 버튼 사용 시 (선택)
pip install "notebooklm-py[browser]"
playwright install chromium
```

### NotebookLM 최초 로그인 (1회만)

```bash
notebooklm login
```
브라우저가 열리면 Google 계정으로 로그인합니다. 이후 자동 세션 유지.

## 실행 방법

### GUI 모드

```bash
python zotero_scholar_to_local.py
```

### CLI 모드

```bash
# 기본 검색
python zotero_scholar_to_local.py "나노플라스틱 유해성" --limit 5 --years 3

# docx 없이 검색만
python zotero_scholar_to_local.py "microplastics" --limit 10 --no-docx

# 연구 제안 포함 (AI 분석 JSON 생성)
python zotero_scholar_to_local.py "나노플라스틱" --limit 5 --proposal "나노플라스틱이 면역계에 미치는 영향을 연구하려 한다"
```

### CLI 옵션

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `keyword` | (필수) | 검색 키워드 (한글/영문 모두 가능) |
| `--limit N` | 5 | 소스별 최대 논문 수 (1~20) |
| `--years N` | 제한 없음 | 최근 N년 논문만 검색 |
| `--no-docx` | — | docx 생성 안 함 |
| `--proposal "..."` | — | 연구 제안 텍스트 (AI 분석 활성화) |
| `--db-path <path>` | 자동 탐지 | zotero.sqlite 경로 직접 지정 |
| `--output <path>` | `{keyword}.docx` | docx 저장 경로 |

## 파이프라인 흐름

```
키워드 입력
  ↓ translate_to_english()      한글 → 영문 (Google Translate 무료 API)
  ↓ search_google_scholar()     Google Scholar 검색 (연도 필터)
  ↓ search_openalex()           OpenAlex API 검색
  ↓ search_semantic_scholar()   Semantic Scholar API 검색
  ↓ merge_and_deduplicate()     제목 기준 중복 제거 (Scholar > OpenAlex > S2 우선)
  ↓ insert to Zotero SQLite     컬렉션 생성 + 논문 추가 (doi, abstract 포함)
  ↓ save_analysis_request()     proposal 있을 때 {keyword}_analysis.json 저장
  ↓ generate_summary_docx()     초록 번역 후 docx 저장
```

## AI 분석 기능

### AI 분석 (Claude)

검색 완료 후 "AI 분석 (Claude)" 버튼 클릭 → 새 CMD 창에서 Claude Code 실행 → Zotero MCP로 컬렉션 논문 조회 → 연구 제안 타당성 평가 (한국어 리포트)

> Claude Code와 Zotero MCP가 설치되어 있어야 합니다.

### AI 분석 (NLM)

검색 완료 후 "AI 분석 (NLM)" 버튼 클릭 → Zotero SQLite에서 컬렉션 논문 읽기 → NotebookLM에 논문 소스 추가 → 연구 제안 평가 질문 → GUI 로그창에 결과 출력 → 임시 노트북 자동 삭제

> `notebooklm-py` 패키지와 최초 1회 `notebooklm login`이 필요합니다.

## 출력 결과

### docx 요약 문서

- **파일명**: `{keyword}.docx`
- **구성**: Google Scholar / OpenAlex / Semantic Scholar 섹션 분리
- **논문 항목**: 영문 원제 + 한국어 번역 초록 (최대 800자) + 저자, 저널, 연도, DOI
- **헤더**: 작성일, 총 논문 수, 출처 표기

### analysis JSON (Claude 분석용)

```json
{
  "keyword": "나노플라스틱",
  "collection": "나노플라스틱",
  "proposal": "연구 제안 내용...",
  "created_at": "2026-03-06",
  "instructions": "Zotero MCP 분석 지시문..."
}
```

## 주의사항

- **Zotero 앱 종료 필수**: 스크립트 실행 중 Zotero가 열려 있으면 DB lock 충돌 발생
- **Google Scholar 차단**: IP 기반 CAPTCHA 차단이 발생할 수 있음 → OpenAlex/Semantic Scholar가 대체 소스
- **Semantic Scholar 요청 제한**: 분당 약 100회 제한, 429 오류 시 약 5분 후 재시도 안내 출력
- **DB 직접 수정**: Zotero SQLite를 직접 수정하므로 실행 전 백업 권장

## 프로젝트 구조

```
Zotero_test/
├── zotero_scholar_to_local.py   # 메인 파이프라인 (GUI + CLI)
├── zotero_local_rw_test.py      # Zotero DB 읽기/쓰기 테스트 스크립트
├── make_summary_docx.py         # 나노플라스틱 논문 요약 docx (일회성)
└── docs/
    └── plans/                   # 설계 문서 및 구현 플랜
```

## 의존성

| 패키지 | 용도 | 설치 |
|--------|------|------|
| `python-docx` | docx 생성 | `pip install python-docx` |
| `notebooklm-py` | NLM 분석 버튼 | `pip install "notebooklm-py[browser]"` |
| `playwright` (chromium) | NLM 로그인용 브라우저 | `playwright install chromium` |

표준 라이브러리만으로 기본 기능(검색 + Zotero 추가) 동작 가능. docx/NLM 기능은 선택 설치.

## 라이선스

MIT
