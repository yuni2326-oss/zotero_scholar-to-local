# Semantic Scholar 검색 추가 구현 플랜

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** `zotero_scholar_to_local.py`에 Semantic Scholar Public API 검색을 세 번째 소스로 추가한다.

**Architecture:** 기존 Google Scholar + OpenAlex 병렬 검색 패턴과 동일하게 `search_semantic_scholar()` 함수를 추가하고, `merge_and_deduplicate()`에 세 번째 인자를 추가하며, docx 출력에 새 섹션을 포함한다.

**Tech Stack:** Python 표준 라이브러리 (`urllib`, `json`), Semantic Scholar Academic Graph API v1 (API key 불필요)

---

### Task 1: `search_semantic_scholar()` 함수 추가

**Files:**
- Modify: `zotero_scholar_to_local.py` — OpenAlex 검색 함수 블록 아래 (`# ── OpenAlex 검색 ───` 섹션 끝, 약 277번 줄 이후)

**Step 1: 함수 코드 작성**

`search_openalex` 함수 끝난 직후 (276번째 줄 `return papers` 다음 빈 줄), 아래 코드를 삽입:

```python
# ── Semantic Scholar 검색 ──────────────────────────────────────────────────────

def search_semantic_scholar(english_query: str, limit: int,
                             years_back: Optional[int] = None) -> list[ScholarPaper]:
    params: dict[str, str] = {
        "query": english_query,
        "limit": str(limit),
        "fields": "title,authors,year,venue,externalIds,abstract",
    }
    if years_back:
        from_year = datetime.date.today().year - years_back
        params["year"] = f"{from_year}-{datetime.date.today().year}"

    url = "https://api.semanticscholar.org/graph/v1/paper/search?" + urllib.parse.urlencode(params)
    req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0"})
    try:
        with urllib.request.urlopen(req, timeout=20) as resp:
            data = json.loads(resp.read().decode("utf-8"))
    except Exception as exc:
        print(f"[WARN] Semantic Scholar 검색 실패: {exc}")
        return []

    papers: list[ScholarPaper] = []
    for work in data.get("data", []):
        title = work.get("title") or ""
        if not title:
            continue

        authors = [
            a.get("name", "")
            for a in work.get("authors", [])[:8]
            if a.get("name")
        ]
        year = str(work["year"]) if work.get("year") else None
        venue = work.get("venue") or None
        abstract = work.get("abstract") or None

        ext_ids = work.get("externalIds") or {}
        doi = ext_ids.get("DOI") or None
        url_val = f"https://doi.org/{doi}" if doi else ""

        papers.append(ScholarPaper(
            title=title, url=url_val, authors=authors,
            year=year, venue=venue, abstract=abstract,
            doi=doi, source="semantic_scholar",
        ))

    return papers
```

**Step 2: 수동 검증 — 함수 단독 실행**

Python 인터프리터에서:
```python
import sys
sys.path.insert(0, "e:/My project/Zotero_test")
from zotero_scholar_to_local import search_semantic_scholar
results = search_semantic_scholar("nanoplastics", limit=3, years_back=3)
for r in results:
    print(r.title, r.doi, r.abstract[:80] if r.abstract else None)
```
예상 결과: 제목 3개 출력, 에러 없음

---

### Task 2: `merge_and_deduplicate()` 세 번째 소스 지원

**Files:**
- Modify: `zotero_scholar_to_local.py` — `merge_and_deduplicate` 함수 (약 281~302번 줄)

**Step 1: 시그니처 및 로직 수정**

현재 코드:
```python
def merge_and_deduplicate(scholar_papers: list[ScholarPaper],
                          openalex_papers: list[ScholarPaper]) -> list[ScholarPaper]:
    """Scholar 우선, 제목 소문자 기준 중복 제거. Scholar에 없는 abstract/doi를 OpenAlex로 보완."""
    seen: dict[str, ScholarPaper] = {}

    for p in scholar_papers:
        key = re.sub(r'\s+', ' ', p.title.lower().strip())
        seen[key] = p

    for p in openalex_papers:
        key = re.sub(r'\s+', ' ', p.title.lower().strip())
        if key not in seen:
            seen[key] = p
        else:
            existing = seen[key]
            # Scholar 항목에 abstract/doi 보완
            if not existing.abstract and p.abstract:
                existing.abstract = p.abstract
            if not existing.doi and p.doi:
                existing.doi = p.doi

    return list(seen.values())
```

교체할 코드:
```python
def merge_and_deduplicate(scholar_papers: list[ScholarPaper],
                          openalex_papers: list[ScholarPaper],
                          semantic_scholar_papers: list[ScholarPaper] | None = None) -> list[ScholarPaper]:
    """Scholar 우선, 제목 소문자 기준 중복 제거. abstract/doi를 후순위 소스로 보완."""
    seen: dict[str, ScholarPaper] = {}

    for p in scholar_papers:
        key = re.sub(r'\s+', ' ', p.title.lower().strip())
        seen[key] = p

    for p in openalex_papers:
        key = re.sub(r'\s+', ' ', p.title.lower().strip())
        if key not in seen:
            seen[key] = p
        else:
            existing = seen[key]
            if not existing.abstract and p.abstract:
                existing.abstract = p.abstract
            if not existing.doi and p.doi:
                existing.doi = p.doi

    for p in (semantic_scholar_papers or []):
        key = re.sub(r'\s+', ' ', p.title.lower().strip())
        if key not in seen:
            seen[key] = p
        else:
            existing = seen[key]
            if not existing.abstract and p.abstract:
                existing.abstract = p.abstract
            if not existing.doi and p.doi:
                existing.doi = p.doi

    return list(seen.values())
```

**Step 2: 수동 검증**

```python
from zotero_scholar_to_local import merge_and_deduplicate, ScholarPaper
s = [ScholarPaper("Paper A", "", [], "2024", None, source="scholar")]
o = [ScholarPaper("Paper B", "", [], "2023", None, source="openalex")]
ss = [ScholarPaper("Paper A", "", [], "2024", None, abstract="test abstract", source="semantic_scholar")]
merged = merge_and_deduplicate(s, o, ss)
print(len(merged))            # 2
print(merged[0].abstract)     # "test abstract" (보완됨)
```

---

### Task 3: `run_pipeline()` 에 Semantic Scholar 검색 추가

**Files:**
- Modify: `zotero_scholar_to_local.py` — `run_pipeline` 함수 내 검색 블록 (약 622~633번 줄)

**Step 1: 검색 호출 및 병합 수정**

현재 코드:
```python
    log_fn("[INFO] Google Scholar 검색 중...")
    scholar_papers = search_google_scholar_recent(english_keyword, limit=limit,
                                                  years_back=years_back)
    log_fn(f"[INFO] Scholar 결과: {len(scholar_papers)}편")

    log_fn("[INFO] OpenAlex 검색 중...")
    openalex_papers = search_openalex(english_keyword, limit=limit, years_back=years_back)
    log_fn(f"[INFO] OpenAlex 결과: {len(openalex_papers)}편")

    # 병합
    merged = merge_and_deduplicate(scholar_papers, openalex_papers)
```

교체할 코드:
```python
    log_fn("[INFO] Google Scholar 검색 중...")
    scholar_papers = search_google_scholar_recent(english_keyword, limit=limit,
                                                  years_back=years_back)
    log_fn(f"[INFO] Scholar 결과: {len(scholar_papers)}편")

    log_fn("[INFO] OpenAlex 검색 중...")
    openalex_papers = search_openalex(english_keyword, limit=limit, years_back=years_back)
    log_fn(f"[INFO] OpenAlex 결과: {len(openalex_papers)}편")

    log_fn("[INFO] Semantic Scholar 검색 중...")
    ss_papers = search_semantic_scholar(english_keyword, limit=limit, years_back=years_back)
    log_fn(f"[INFO] Semantic Scholar 결과: {len(ss_papers)}편")

    # 병합
    merged = merge_and_deduplicate(scholar_papers, openalex_papers, ss_papers)
```

**Step 2: 수동 검증 — CLI 실행**

```bash
cd "e:/My project/Zotero_test"
python zotero_scholar_to_local.py "microplastics" --limit 2 --years 2 --no-docx
```
예상 로그:
```
[INFO] Semantic Scholar 검색 중...
[INFO] Semantic Scholar 결과: N편
[INFO] 병합 후 총 M편 (중복 제거 완료)
```

---

### Task 4: `generate_summary_docx()` — Semantic Scholar 섹션 추가

**Files:**
- Modify: `zotero_scholar_to_local.py` — `generate_summary_docx` 함수 내 헤더/섹션/푸터 부분 (약 541~608번 줄)

**Step 1: 헤더 출처 표기 수정**

현재:
```python
    add_run(p2, f"작성일: {datetime.date.today().strftime('%Y년 %m월 %d일')}  |  "
                f"총 {len(papers)}편  |  출처: Google Scholar + OpenAlex",
            size=9, color=RGBColor(0x70, 0x70, 0x70))
```
변경:
```python
    add_run(p2, f"작성일: {datetime.date.today().strftime('%Y년 %m월 %d일')}  |  "
                f"총 {len(papers)}편  |  출처: Google Scholar + OpenAlex + Semantic Scholar",
            size=9, color=RGBColor(0x70, 0x70, 0x70))
```

**Step 2: 소스 분류 및 섹션 추가**

현재:
```python
    # Scholar / OpenAlex 분류
    scholar_papers = [p for p in papers if p.source == "scholar"]
    openalex_papers = [p for p in papers if p.source == "openalex"]
    ...
    num = 1
    num = write_section("■ Google Scholar 검색 결과", scholar_papers, num)
    if openalex_papers:
        doc.add_page_break()
    num = write_section("■ OpenAlex 검색 결과", openalex_papers, num)
```
변경:
```python
    # 소스별 분류
    scholar_papers = [p for p in papers if p.source == "scholar"]
    openalex_papers = [p for p in papers if p.source == "openalex"]
    ss_papers = [p for p in papers if p.source == "semantic_scholar"]
    ...
    num = 1
    num = write_section("■ Google Scholar 검색 결과", scholar_papers, num)
    if openalex_papers:
        doc.add_page_break()
    num = write_section("■ OpenAlex 검색 결과", openalex_papers, num)
    if ss_papers:
        doc.add_page_break()
    num = write_section("■ Semantic Scholar 검색 결과", ss_papers, num)
```

**Step 3: 푸터 주석 수정**

현재:
```python
    add_run(pn, f"※ 키워드 '{keyword}'로 Google Scholar + OpenAlex에서 검색한 결과입니다.",
            size=8, color=RGBColor(0x80, 0x80, 0x80))
```
변경:
```python
    add_run(pn, f"※ 키워드 '{keyword}'로 Google Scholar + OpenAlex + Semantic Scholar에서 검색한 결과입니다.",
            size=8, color=RGBColor(0x80, 0x80, 0x80))
```

**Step 4: 수동 검증 — docx 생성 확인**

```bash
python zotero_scholar_to_local.py "nanoplastics" --limit 2 --years 2
```
예상: `nanoplastics.docx` 생성됨, Word에서 열어 "■ Semantic Scholar 검색 결과" 섹션 확인

---

### Task 5: MEMORY.md 업데이트

**Files:**
- Modify: `C:/Users/yuni2/.claude/projects/e--My-project-Zotero-test/memory/MEMORY.md`

파이프라인 흐름 섹션을 아래와 같이 업데이트:

```markdown
  → search_semantic_scholar()    # Semantic Scholar API 검색 (연도 필터)
```

`merge_and_deduplicate()` 설명에 세 번째 인자 추가.

docx 출처 표기 `"Google Scholar + OpenAlex + Semantic Scholar"`로 업데이트.
