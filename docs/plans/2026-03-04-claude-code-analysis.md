# Claude Code 연구 제안 분석 기능 구현 계획

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** API 키 없이 Claude Code + Zotero MCP로 연구 제안 타당성을 평가하는 "AI 분석" 버튼을 GUI에 추가한다.

**Architecture:** subprocess `claude -p` 방식을 제거하고, 분석 요청 JSON을 저장한 뒤 새 CMD 창에서 `claude` 인터랙티브 모드를 열어 Claude Code가 Zotero MCP로 직접 논문을 조회하도록 한다.

**Tech Stack:** Python 3, tkinter, sqlite3, subprocess (CREATE_NEW_CONSOLE), json

---

### Task 1: analyze_with_claude 함수 제거 및 docx analysis 파라미터 제거

**Files:**
- Modify: `zotero_scholar_to_local.py:439-527` (analyze_with_claude 함수 전체)
- Modify: `zotero_scholar_to_local.py:543-662` (generate_summary_docx)

**Step 1: analyze_with_claude 함수 전체 삭제**

`zotero_scholar_to_local.py`에서 아래 블록을 찾아 완전히 삭제한다 (섹션 주석 포함):

```python
# ── Claude CLI 분석 ───────────────────────────────────────────────────────────

def analyze_with_claude(papers: list[ScholarPaper], proposal: str,
                        log_fn: Callable[[str], None] = print) -> Optional[str]:
    ...
    (함수 전체 88줄)
```

삭제 후 `# ── docx 생성 ───` 섹션이 바로 이어져야 한다.

**Step 2: generate_summary_docx에서 analysis 파라미터 제거**

함수 시그니처 변경:
```python
# 변경 전
def generate_summary_docx(papers: list[ScholarPaper], keyword: str, output_path: Path,
                          log_fn: Callable[[str], None] = print,
                          analysis: str = "") -> None:

# 변경 후
def generate_summary_docx(papers: list[ScholarPaper], keyword: str, output_path: Path,
                          log_fn: Callable[[str], None] = print) -> None:
```

**Step 3: docx 내 AI 분석 섹션 삭제**

함수 내부에서 아래 블록을 찾아 삭제:
```python
    # AI 연구 트렌드 분석 섹션
    if analysis:
        doc.add_page_break()
        ph = doc.add_paragraph()
        add_run(ph, "■ AI 연구 트렌드 분석", bold=True, size=13,
                color=RGBColor(0x2E, 0x74, 0xB5))
        for line in analysis.split("\n"):
            pa = doc.add_paragraph()
            pa.paragraph_format.left_indent = Inches(0.2)
            size = 11 if line.startswith("##") else 10
            bold = line.startswith("##")
            add_run(pa, line.lstrip("# ").strip() if line.startswith("#") else line,
                    bold=bold, size=size)
        doc.add_paragraph()
        pf = doc.add_paragraph()
        pf.paragraph_format.left_indent = Inches(0.2)
        add_run(pf, f"※ 분석 모델: Claude  |  분석 기준: 수집된 {len(papers)}편 논문",
                size=8, color=RGBColor(0x80, 0x80, 0x80))
```

**Step 4: 파일을 저장하고 python으로 문법 검사**

```bash
python -c "import ast; ast.parse(open('zotero_scholar_to_local.py', encoding='utf-8').read()); print('OK')"
```
Expected: `OK`

---

### Task 2: save_analysis_request, open_claude_terminal 함수 추가

**Files:**
- Modify: `zotero_scholar_to_local.py` — `get_user_library_id` 함수 뒤, `# ── docx 생성` 섹션 앞에 삽입

**Step 1: 섹션 주석과 두 함수를 삽입**

`get_user_library_id` 함수 끝(빈 줄 2개) 바로 다음, `# ── docx 생성` 주석 바로 앞에 아래 코드를 삽입:

```python
# ── Claude Code 분석 연동 ──────────────────────────────────────────────────────

def save_analysis_request(keyword: str, proposal: str, output_dir: Path) -> Path:
    """분석 요청 JSON 저장. Claude Code + Zotero MCP 분석 시 사용."""
    safe_kw = re.sub(r'[\\/:*?"<>|]', "_", keyword)
    json_path = output_dir / f"{safe_kw}_analysis.json"
    instructions = (
        f"Zotero MCP에서 컬렉션 '{keyword}'의 논문들을 zotero_search_items로 검색하고, "
        f"각 논문의 메타데이터와 초록을 확인한 뒤, 아래 연구 제안의 타당성을 한국어로 평가해주세요.\n\n"
        f"[연구 제안]\n{proposal}\n\n"
        f"[평가 항목]\n"
        f"1. 주요 연구 트렌드와의 관계\n"
        f"2. 연구 공백(gap) 및 차별성\n"
        f"3. 타당성 종합 평가"
    )
    data = {
        "keyword": keyword,
        "collection": keyword.strip(),
        "proposal": proposal,
        "created_at": datetime.date.today().isoformat(),
        "instructions": instructions,
    }
    json_path.write_text(json.dumps(data, ensure_ascii=False, indent=2), encoding="utf-8")
    return json_path


def open_claude_terminal(json_path: Path,
                         log_fn: Callable[[str], None] = print) -> None:
    """새 CMD 창에서 Claude Code를 열고 JSON 분석 요청을 시작한다."""
    import subprocess
    import shutil

    if shutil.which("claude") is None:
        log_fn("[WARN] claude CLI를 찾을 수 없습니다.")
        log_fn(f"[INFO] 직접 Claude Code를 열고 파일을 참조하세요: {json_path}")
        return

    abs_path = str(json_path.resolve()).replace("\\", "/")
    prompt = f"Read {abs_path} and follow the instructions field"
    try:
        subprocess.Popen(
            ["cmd", "/k", "claude", prompt],
            creationflags=subprocess.CREATE_NEW_CONSOLE,
            cwd=str(json_path.parent),
        )
        log_fn(f"[INFO] Claude Code 터미널을 열었습니다.")
        log_fn(f"[INFO] 분석 파일: {json_path.name}")
    except Exception as exc:
        log_fn(f"[WARN] 터미널 실행 실패: {exc}")
        log_fn(f"[INFO] 수동으로 Claude Code를 열고 아래 파일을 참조하세요: {json_path}")

```

**Step 2: 문법 검사**

```bash
python -c "import ast; ast.parse(open('zotero_scholar_to_local.py', encoding='utf-8').read()); print('OK')"
```
Expected: `OK`

---

### Task 3: run_pipeline 수정

**Files:**
- Modify: `zotero_scholar_to_local.py` — `run_pipeline` 함수 내 AI 분석 블록

**Step 1: AI 분석 블록을 JSON 저장으로 교체**

기존:
```python
    # AI 분석
    analysis = ""
    if proposal.strip():
        analysis = analyze_with_claude(merged, proposal, log_fn=log_fn) or ""

    # docx 생성
    if output_path is not None:
        log_fn(f"[INFO] docx 생성 중: {output_path}")
        generate_summary_docx(merged, keyword, output_path, log_fn=log_fn,
                              analysis=analysis)
```

변경 후:
```python
    # 분석 요청 JSON 저장 (proposal 있을 때)
    if proposal.strip():
        save_dir = output_path.parent if output_path else Path(".")
        json_path = save_analysis_request(keyword, proposal, save_dir)
        log_fn(f"[INFO] 분석 요청 파일 저장됨: {json_path.name}")
        log_fn("[INFO] GUI에서 'AI 분석' 버튼을 클릭하거나 Claude Code에서 파일을 열어주세요.")

    # docx 생성
    if output_path is not None:
        log_fn(f"[INFO] docx 생성 중: {output_path}")
        generate_summary_docx(merged, keyword, output_path, log_fn=log_fn)
```

**Step 2: 문법 검사**

```bash
python -c "import ast; ast.parse(open('zotero_scholar_to_local.py', encoding='utf-8').read()); print('OK')"
```
Expected: `OK`

---

### Task 4: run_gui 수정 — AI 분석 버튼 추가 및 이모지 수정

**Files:**
- Modify: `zotero_scholar_to_local.py` — `run_gui` 함수

**Step 1: 버튼 영역에 AI 분석 버튼 추가**

기존 버튼 블록:
```python
    # ── 버튼 ───────────────────────────────────────────────────────
    btn_search = tk.Button(root, text="  검색 시작  ", font=("Malgun Gothic", 11, "bold"),
                           bg="#2E74B5", fg="white", relief="flat", cursor="hand2")
    btn_search.pack(pady=(0, 8))
```

변경 후 (frm_btn 프레임으로 두 버튼 나란히 배치):
```python
    # ── 버튼 ───────────────────────────────────────────────────────
    frm_btn = tk.Frame(root)
    frm_btn.pack(pady=(0, 8))

    btn_search = tk.Button(frm_btn, text="  검색 시작  ", font=("Malgun Gothic", 11, "bold"),
                           bg="#2E74B5", fg="white", relief="flat", cursor="hand2")
    btn_search.pack(side="left", padx=4)

    btn_analyze = tk.Button(frm_btn, text="  AI 분석  ", font=("Malgun Gothic", 11, "bold"),
                            bg="#217346", fg="white", relief="flat", cursor="hand2",
                            state="disabled")
    btn_analyze.pack(side="left", padx=4)

    last_json_path: list[Optional[Path]] = [None]
```

**Step 2: on_analyze 핸들러 추가 및 btn_analyze에 바인딩**

`on_search` 함수 정의 바로 앞에 삽입:
```python
    def on_analyze() -> None:
        jp = last_json_path[0]
        if not jp or not jp.exists():
            from tkinter import messagebox
            messagebox.showwarning("분석 불가", "먼저 검색을 실행하고 연구 제안을 입력하세요.")
            return
        btn_analyze.configure(state="disabled")
        open_claude_terminal(jp, log_fn=append_log)
        btn_analyze.configure(state="normal")

    btn_analyze.configure(command=on_analyze)
```

**Step 3: worker 함수 수정 — 이모지 제거 + 분석 버튼 활성화**

기존 worker:
```python
        def worker() -> None:
            try:
                db_path = resolve_zotero_paths().db_path
                run_pipeline(db_path, keyword, limit, output_path,
                             years_back=years_back, log_fn=append_log,
                             proposal=proposal)
                append_log(f"\n✅ 완료!  저장 파일: {output_path.resolve()}")
            except Exception as exc:
                append_log(f"[ERROR] {exc}")
            finally:
                btn_search.configure(state="normal", text="  검색 시작  ")
```

변경 후:
```python
        def worker() -> None:
            try:
                db_path = resolve_zotero_paths().db_path
                run_pipeline(db_path, keyword, limit, output_path,
                             years_back=years_back, log_fn=append_log,
                             proposal=proposal)
                append_log(f"\n[완료] 저장 파일: {output_path.resolve()}")
                # 분석 버튼 활성화
                if proposal:
                    jp = output_path.parent / f"{safe_kw}_analysis.json"
                    if jp.exists():
                        last_json_path[0] = jp
                        btn_analyze.configure(state="normal")
                        append_log("[INFO] 'AI 분석' 버튼이 활성화되었습니다.")
            except Exception as exc:
                append_log(f"[ERROR] {exc}")
            finally:
                btn_search.configure(state="normal", text="  검색 시작  ")
```

**Step 4: 문법 검사**

```bash
python -c "import ast; ast.parse(open('zotero_scholar_to_local.py', encoding='utf-8').read()); print('OK')"
```
Expected: `OK`

**Step 5: GUI 실행하여 버튼 레이아웃 육안 확인**

```bash
python zotero_scholar_to_local.py
```

- "검색 시작" + "AI 분석" 버튼이 나란히 보임
- "AI 분석"은 회색(disabled) 상태
- 검색 + proposal 입력 후 실행 시 "AI 분석" 버튼이 초록색으로 활성화됨

---

### Task 5: 메모리 업데이트 및 최종 확인

**Step 1: MEMORY.md 업데이트**

`C:\Users\yuni2\.claude\projects\e--My-project-Zotero-test\memory\MEMORY.md` 의
`analyze_with_claude` 관련 항목을 새 아키텍처로 교체한다.

**Step 2: 전체 임포트 확인**

`subprocess` 는 `open_claude_terminal` 내부에서 지역 import 하므로 상단 import 블록 변경 불필요.
`Optional[Path]` 타입은 이미 `from typing import Callable, Optional` 로 임포트됨 — 확인.

**Step 3: CLI 모드 확인**

```bash
python zotero_scholar_to_local.py "test" --limit 1 --no-docx --proposal "테스트 제안"
```

Expected 로그:
```
[INFO] 분석 요청 파일 저장됨: test_analysis.json
[INFO] GUI에서 'AI 분석' 버튼을 클릭하거나 Claude Code에서 파일을 열어주세요.
```
