# Semantic Scholar 검색 추가 설계

**날짜**: 2026-03-06
**파일 대상**: `zotero_scholar_to_local.py`

## 개요

현재 Google Scholar + OpenAlex 2소스 파이프라인에 Semantic Scholar Public API를 세 번째 소스로 추가한다.

## 변경 범위

### 1. 새 함수 `search_semantic_scholar()`

```python
def search_semantic_scholar(english_query: str, limit: int,
                             years_back: Optional[int] = None) -> list[ScholarPaper]:
```

- **API**: `https://api.semanticscholar.org/graph/v1/paper/search`
- **쿼리 파라미터**:
  - `query`: 검색어
  - `limit`: 최대 결과 수
  - `fields`: `title,authors,year,venue,externalIds,abstract`
  - `year`: `{from_year}-{current_year}` (years_back 있을 때)
- **DOI**: `externalIds.DOI`에서 추출
- **저자**: `authors[].name` (최대 8명)
- **source**: `"semantic_scholar"`
- API key 불필요 (100 req/5분 제한), 실패 시 빈 리스트 반환

### 2. `merge_and_deduplicate()` 시그니처 변경

```python
def merge_and_deduplicate(scholar_papers, openalex_papers,
                          semantic_scholar_papers) -> list[ScholarPaper]:
```

- 우선순위: Scholar > OpenAlex > Semantic Scholar
- 중복 기준: 제목 소문자 정규화 (기존 방식 유지)
- 보완: 기존 항목에 abstract/DOI 없으면 Semantic Scholar 데이터로 채움

### 3. `run_pipeline()` 수정

```
Scholar 검색 → OpenAlex 검색 → Semantic Scholar 검색 → 병합 → Zotero → docx
```

`merge_and_deduplicate()` 호출에 세 번째 인자 추가.

### 4. `generate_summary_docx()` 수정

- 헤더 출처 표기: `"Google Scholar + OpenAlex + Semantic Scholar"`
- 섹션 추가: `"■ Semantic Scholar 검색 결과"`
- 하단 주석 텍스트: `"Google Scholar + OpenAlex + Semantic Scholar에서 검색"`

## 변경하지 않는 것

- `ScholarPaper` 데이터클래스 (source 필드에 `"semantic_scholar"` 값만 추가)
- Zotero SQLite 쓰기 로직
- CLI/GUI 인터페이스
- 번역 함수
