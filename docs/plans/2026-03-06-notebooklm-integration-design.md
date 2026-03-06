# NotebookLM 통합 설계

**날짜**: 2026-03-06
**파일 대상**: `zotero_scholar_to_local.py`

## 개요

기존 "AI 분석 (Claude Code)" 버튼 옆에 "AI 분석 (NLM)" 버튼을 추가한다.
NLM 버튼은 Zotero SQLite에서 해당 컬렉션의 논문을 읽어 NotebookLM에 소스로 넣고,
연구 제안 평가 질문을 보낸 뒤 답변을 GUI 로그창에 출력한다.

## 전제 조건

```bash
pip install "notebooklm-py[browser]"
notebooklm login    # 브라우저 열림 → Google 계정 로그인 → 1회만
```

## 변경 범위

### 1. 신규 함수: `read_collection_papers_from_zotero(db_path, collection_name) -> str`

- Zotero SQLite에서 해당 컬렉션 논문의 title + abstractNote를 읽어 텍스트 포맷으로 반환
- fieldID 1 = title, fieldID 27 = abstractNote
- 최대 20편, 각 논문: `N. 제목\n초록\n\n`
- 컬렉션 없거나 논문 없으면 빈 문자열 반환

```python
def read_collection_papers_from_zotero(db_path: Path,
                                        collection_name: str) -> str:
    with sqlite3.connect(str(db_path)) as con:
        row = con.execute(
            "SELECT collectionID FROM collections WHERE collectionName = ?",
            (collection_name,)
        ).fetchone()
        if not row:
            return ""
        coll_id = row[0]
        items = con.execute(
            "SELECT ci.itemID FROM collectionItems ci "
            "JOIN items i ON ci.itemID = i.itemID "
            "WHERE ci.collectionID = ? LIMIT 20",
            (coll_id,)
        ).fetchall()
        parts = []
        for n, (item_id,) in enumerate(items, 1):
            vals = {
                fid: val
                for fid, val in con.execute(
                    "SELECT id.fieldID, idv.value "
                    "FROM itemData id JOIN itemDataValues idv "
                    "ON id.valueID = idv.valueID "
                    "WHERE id.itemID = ? AND id.fieldID IN (1, 27)",
                    (item_id,)
                ).fetchall()
            }
            title = vals.get(1, "")
            abstract = vals.get(27, "")
            if title:
                parts.append(f"{n}. {title}\n{abstract}\n")
        return "\n".join(parts)
```

### 2. 신규 함수: `async analyze_with_notebooklm(db_path, keyword, proposal, log_fn)`

- `NotebookLMClient.from_storage()` 로그인 세션 재사용 (no re-login)
- 노트북 생성 → 텍스트 소스 추가 → 연구 제안 평가 질문 → 답변 로그 출력 → 노트북 삭제

```python
async def analyze_with_notebooklm(db_path: Path, keyword: str,
                                   proposal: str,
                                   log_fn: Callable[[str], None]) -> None:
    try:
        from notebooklm import NotebookLMClient
    except ImportError:
        log_fn("[ERROR] notebooklm-py 미설치: pip install 'notebooklm-py[browser]'")
        return

    text = read_collection_papers_from_zotero(db_path, keyword)
    if not text:
        log_fn(f"[WARN] Zotero 컬렉션 '{keyword}'에 논문이 없습니다.")
        return

    log_fn("[NLM] NotebookLM 연결 중...")
    try:
        async with await NotebookLMClient.from_storage() as client:
            notebook = await client.create_notebook(title=f"[분석] {keyword}")
            log_fn(f"[NLM] 노트북 생성: {notebook.title}")
            await notebook.add_source(text=text)
            log_fn(f"[NLM] 소스 추가 완료 ({len(text)} chars)")

            question = (
                f"아래 연구 제안의 타당성을 논문들을 바탕으로 한국어로 평가해주세요.\n\n"
                f"[연구 제안]\n{proposal}\n\n"
                f"[평가 항목]\n"
                f"1. 주요 연구 트렌드와의 관계\n"
                f"2. 연구 공백(gap) 및 차별성\n"
                f"3. 타당성 종합 평가"
            )
            log_fn("[NLM] 분석 중... (수십 초 소요)")
            answer = await notebook.chat(question)
            log_fn("\n[NLM 분석 결과]\n" + answer)
            await notebook.delete()
            log_fn("[NLM] 임시 노트북 삭제 완료")
    except Exception as exc:
        if "login" in str(exc).lower() or "auth" in str(exc).lower():
            log_fn("[ERROR] NotebookLM 로그인 필요: notebooklm login 실행 후 재시도")
        else:
            log_fn(f"[ERROR] NotebookLM 오류: {exc}")
```

### 3. 신규 함수: `open_notebooklm_analysis(db_path, keyword, proposal, log_fn)`

동기 래퍼 — GUI 백그라운드 스레드에서 호출됨:

```python
def open_notebooklm_analysis(db_path: Path, keyword: str,
                              proposal: str,
                              log_fn: Callable[[str], None]) -> None:
    import asyncio
    asyncio.run(analyze_with_notebooklm(db_path, keyword, proposal, log_fn))
```

### 4. GUI 변경

| 현재 | 변경 후 |
|------|---------|
| `btn_analyze` — "  AI 분석  " | `btn_analyze` — "  AI 분석 (Claude)  " |
| _(없음)_ | `btn_nlm` — "  AI 분석 (NLM)  " (신규, disabled) |

두 버튼의 활성화 조건: proposal 입력 + 검색 완료 (동일)

NLM 버튼 클릭 핸들러:
```python
def on_analyze_nlm() -> None:
    keyword = last_keyword[0]
    proposal = text_proposal.get("1.0", "end").strip()
    db_path = last_db_path[0]
    if not keyword or not proposal or not db_path:
        messagebox.showwarning("분석 불가", "먼저 검색을 실행하고 연구 제안을 입력하세요.")
        return
    btn_nlm.configure(state="disabled", text="  분석 중...  ")
    def nlm_worker():
        try:
            open_notebooklm_analysis(db_path, keyword, proposal, append_log)
        except Exception as exc:
            append_log(f"[ERROR] {exc}")
        finally:
            root.after(0, lambda: btn_nlm.configure(state="normal",
                                                      text="  AI 분석 (NLM)  "))
    threading.Thread(target=nlm_worker, daemon=True).start()
```

GUI에 `last_keyword: list[str] = [""]`, `last_db_path: list[Optional[Path]] = [None]` 상태 변수 추가.

## 변경하지 않는 것

- `ScholarPaper`, `run_pipeline()`, docx 생성 로직
- Claude 분석 버튼 기존 동작 (`open_claude_terminal`)
- CLI 인터페이스

## 오류 처리 요약

| 상황 | 처리 |
|------|------|
| `notebooklm-py` 미설치 | ImportError 캐치 → `pip install` 안내 로그 |
| `notebooklm login` 미실행 | "login" 키워드 포함 예외 → 재시도 안내 |
| 컬렉션 없음 | 빈 문자열 반환 → WARN 로그 후 중단 |
| 기타 API 오류 | `[ERROR] NotebookLM 오류: {e}` 로그 |
