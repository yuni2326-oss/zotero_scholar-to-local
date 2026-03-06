# NotebookLM 통합 구현 플랜

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** `zotero_scholar_to_local.py` GUI에 "AI 분석 (NLM)" 버튼을 추가하여, Zotero SQLite에서 컬렉션 논문을 읽어 NotebookLM으로 연구 제안 타당성을 평가하고 결과를 로그창에 출력한다.

**Architecture:** 3개의 신규 함수 추가 (`read_collection_papers_from_zotero`, `analyze_with_notebooklm` async, `open_notebooklm_analysis` 동기 래퍼) + GUI에 버튼 1개 추가. 기존 Claude 분석 버튼은 그대로 유지하며 텍스트만 변경.

**Tech Stack:** Python 표준 라이브러리 (`sqlite3`, `asyncio`, `threading`), `notebooklm-py` (pip install), tkinter GUI (기존)

**전제 조건:**
```bash
pip install "notebooklm-py[browser]"
notebooklm login    # 브라우저 열림 → Google 계정 로그인 → 1회만
```

---

### Task 1: `read_collection_papers_from_zotero()` 함수 추가

**Files:**
- Modify: `zotero_scholar_to_local.py` — `save_analysis_request()` 함수 앞 (약 538번 줄 위)

**Step 1: 함수 삽입 위치 확인**

`zotero_scholar_to_local.py`에서 538번 줄 근처의 `save_analysis_request` 함수를 찾는다.
아래 함수를 `save_analysis_request` 정의 바로 위에 삽입한다.

**Step 2: 코드 삽입**

```python
# ── NotebookLM 통합 ───────────────────────────────────────────────────────────

def read_collection_papers_from_zotero(db_path: Path,
                                        collection_name: str) -> str:
    """Zotero SQLite에서 컬렉션 논문 title + abstract를 텍스트로 반환 (최대 20편)."""
    try:
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
    except Exception as exc:
        return ""
```

**Step 3: 수동 검증**

Python 인터프리터에서 (Zotero 앱 종료 상태):
```python
import sys
sys.path.insert(0, "e:/My project/Zotero_test")
from zotero_scholar_to_local import read_collection_papers_from_zotero, resolve_zotero_paths
from pathlib import Path
db = resolve_zotero_paths().db_path
text = read_collection_papers_from_zotero(db, "나노플라스틱")  # 실제 컬렉션명으로 변경
print(text[:300])
```
예상: 논문 제목과 초록이 번호 매겨져 출력됨 (없으면 빈 문자열)

---

### Task 2: `analyze_with_notebooklm()` + `open_notebooklm_analysis()` 추가

**Files:**
- Modify: `zotero_scholar_to_local.py` — `read_collection_papers_from_zotero()` 함수 끝 바로 아래

**Step 1: 두 함수 삽입**

`read_collection_papers_from_zotero` 함수의 마지막 `return ""` 다음 빈 줄 이후에 삽입:

```python
async def analyze_with_notebooklm(db_path: Path, keyword: str,
                                   proposal: str,
                                   log_fn: Callable[[str], None]) -> None:
    """NotebookLM에서 Zotero 컬렉션 논문으로 연구 제안 타당성을 평가한다."""
    try:
        from notebooklm import NotebookLMClient
    except ImportError:
        log_fn("[ERROR] notebooklm-py 미설치: pip install 'notebooklm-py[browser]'")
        return

    text = read_collection_papers_from_zotero(db_path, keyword)
    if not text.strip():
        log_fn(f"[WARN] Zotero 컬렉션 '{keyword}'에 논문이 없습니다.")
        return

    log_fn("[NLM] NotebookLM 연결 중...")
    try:
        async with await NotebookLMClient.from_storage() as client:
            notebook = await client.create_notebook(title=f"[분석] {keyword}")
            log_fn(f"[NLM] 노트북 생성: {notebook.title}")
            await notebook.add_source(text=text)
            log_fn(f"[NLM] 소스 추가 완료 ({len(text)}자)")

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


def open_notebooklm_analysis(db_path: Path, keyword: str,
                              proposal: str,
                              log_fn: Callable[[str], None]) -> None:
    """analyze_with_notebooklm의 동기 래퍼 (GUI 백그라운드 스레드에서 호출)."""
    import asyncio
    asyncio.run(analyze_with_notebooklm(db_path, keyword, proposal, log_fn))
```

**Step 2: 수동 검증 — ImportError 경로**

`notebooklm` 패키지 없는 상태에서:
```bash
python -c "
import sys; sys.path.insert(0, 'e:/My project/Zotero_test')
from zotero_scholar_to_local import open_notebooklm_analysis
from pathlib import Path
open_notebooklm_analysis(Path('dummy.db'), 'test', 'test proposal', print)
"
```
예상 출력: `[ERROR] notebooklm-py 미설치: pip install 'notebooklm-py[browser]'`

---

### Task 3: GUI 수정 — 버튼 추가 및 상태 관리

**Files:**
- Modify: `zotero_scholar_to_local.py` — `run_gui()` 함수 (약 780~900번 줄)

**Step 1: `btn_analyze` 텍스트 변경 (824번 줄 근처)**

현재:
```python
btn_analyze = tk.Button(frm_btn, text="  AI 분석  ", font=("Malgun Gothic", 11, "bold"),
                        bg="#217346", fg="white", relief="flat", cursor="hand2",
                        state="disabled")
btn_analyze.pack(side="left", padx=4)
```

변경:
```python
btn_analyze = tk.Button(frm_btn, text="  AI 분석 (Claude)  ", font=("Malgun Gothic", 11, "bold"),
                        bg="#217346", fg="white", relief="flat", cursor="hand2",
                        state="disabled")
btn_analyze.pack(side="left", padx=4)

btn_nlm = tk.Button(frm_btn, text="  AI 분석 (NLM)  ", font=("Malgun Gothic", 11, "bold"),
                    bg="#7B2D8B", fg="white", relief="flat", cursor="hand2",
                    state="disabled")
btn_nlm.pack(side="left", padx=4)
```

**Step 2: 상태 변수 추가**

현재 코드에 `last_json_path: list[Optional[Path]] = [None]` 바로 아래에 추가:
```python
last_keyword: list[str] = [""]
last_db_path: list[Optional[Path]] = [None]
```

**Step 3: `on_analyze_nlm` 핸들러 추가**

`btn_analyze.configure(command=on_analyze)` 바로 아래에 추가:

```python
def on_analyze_nlm() -> None:
    keyword = last_keyword[0]
    proposal_text = text_proposal.get("1.0", "end").strip()
    db = last_db_path[0]
    if not keyword or not proposal_text or not db:
        messagebox.showwarning("분석 불가", "먼저 검색을 실행하고 연구 제안을 입력하세요.")
        return
    btn_nlm.configure(state="disabled", text="  분석 중...  ")
    def nlm_worker() -> None:
        try:
            open_notebooklm_analysis(db, keyword, proposal_text, append_log)
        except Exception as exc:
            append_log(f"[ERROR] NLM 오류: {exc}")
        finally:
            root.after(0, lambda: btn_nlm.configure(state="normal",
                                                      text="  AI 분석 (NLM)  "))
    threading.Thread(target=nlm_worker, daemon=True).start()

btn_nlm.configure(command=on_analyze_nlm)
```

**Step 4: `worker()` 함수에서 상태 변수 저장**

`worker()` 함수 내에서 `run_pipeline(...)` 호출 직전 (또는 db_path 해석 직후):

현재 (878번 줄 근처):
```python
def worker() -> None:
    try:
        db_path = resolve_zotero_paths().db_path
        run_pipeline(db_path, keyword, limit, output_path,
                     years_back=years_back, log_fn=append_log,
                     proposal=proposal)
```

변경 — `db_path` 저장 라인 추가:
```python
def worker() -> None:
    try:
        db_path = resolve_zotero_paths().db_path
        last_db_path[0] = db_path
        last_keyword[0] = keyword
        run_pipeline(db_path, keyword, limit, output_path,
                     years_back=years_back, log_fn=append_log,
                     proposal=proposal)
```

**Step 5: NLM 버튼 활성화 로직 추가**

기존 Claude 버튼 활성화 블록 (884번 줄 근처):
```python
                if proposal:
                    jp = output_path.parent / f"{safe_kw}_analysis.json"
                    if jp.exists():
                        last_json_path[0] = jp
                        root.after(0, lambda: btn_analyze.configure(state="normal"))
                        append_log("[INFO] 'AI 분석' 버튼이 활성화되었습니다.")
```

변경 — NLM 버튼 활성화 추가 + 로그 메시지 업데이트:
```python
                if proposal:
                    jp = output_path.parent / f"{safe_kw}_analysis.json"
                    if jp.exists():
                        last_json_path[0] = jp
                        root.after(0, lambda: btn_analyze.configure(state="normal"))
                    root.after(0, lambda: btn_nlm.configure(state="normal"))
                    append_log("[INFO] 'AI 분석 (Claude)', 'AI 분석 (NLM)' 버튼이 활성화되었습니다.")
```

**Step 6: 수동 검증 — GUI 실행**

```bash
cd "e:/My project/Zotero_test"
python zotero_scholar_to_local.py
```
확인 사항:
- 버튼 3개 표시: "검색 시작", "AI 분석 (Claude)", "AI 분석 (NLM)"
- NLM 버튼은 처음에 비활성화(회색)
- 키워드 입력 + 연구 제안 입력 후 검색 → NLM 버튼 활성화
- NLM 버튼 클릭 시 로그창에 "[NLM] NotebookLM 연결 중..." 출력
  (notebooklm-py 미설치 시 "[ERROR] notebooklm-py 미설치..." 출력)

---

### Task 4: MEMORY.md 업데이트

**Files:**
- Modify: `C:/Users/yuni2/.claude/projects/e--My-project-Zotero-test/memory/MEMORY.md`

**Step 1: GUI 요소 섹션 업데이트**

현재:
```
- AI 분석 버튼 (proposal 입력 + 검색 완료 시 활성화) — `open_claude_terminal()` 호출
```

변경:
```
- AI 분석 (Claude) 버튼 (proposal 입력 + 검색 완료 시 활성화) — `open_claude_terminal()` 호출
- AI 분석 (NLM) 버튼 (proposal 입력 + 검색 완료 시 활성화) — `open_notebooklm_analysis()` 호출
```

**Step 2: 주요 함수 목록 업데이트**

현재:
```
- `save_analysis_request(keyword, proposal, output_dir)` → `{keyword}_analysis.json` 저장, Path 반환
```

아래 항목들 추가:
```
- `read_collection_papers_from_zotero(db_path, collection_name)` → Zotero SQLite에서 컬렉션 논문 텍스트 반환
- `open_notebooklm_analysis(db_path, keyword, proposal, log_fn)` → NotebookLM 분석 동기 래퍼
```

**Step 3: 의존성 섹션 업데이트**

```
- 선택: `notebooklm-py` (pip install "notebooklm-py[browser]") - NLM 분석 버튼 사용 시 필요
```
