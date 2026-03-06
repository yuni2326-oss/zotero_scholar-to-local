# Claude Code 연구 제안 분석 기능 설계

**날짜:** 2026-03-04
**상태:** 승인됨

## 배경

기존 `analyze_with_claude()` 함수는 `subprocess`로 `claude -p`를 호출하는 방식이었으나,
`ANTHROPIC_API_KEY` 없이는 동작하지 않는 문제가 있었다.

## 목표

API 키 없이 Claude Code + Zotero MCP를 활용하여 연구 제안 타당성을 평가한다.

## 아키텍처

```
[검색 시작 버튼]
  → 논문 수집 (Scholar + OpenAlex)
  → Zotero SQLite 저장
  → docx 생성
  → "AI 분석" 버튼 활성화

[AI 분석 버튼]
  → {keyword}_analysis.json 저장
  → 새 CMD 창에서 claude 실행
  → Claude Code: JSON 읽고 zotero_search_items 호출
  → 컬렉션 논문 조회 → 연구 제안 타당성 평가 (한국어)
```

## JSON 파일 구조

파일명: `{safe_keyword}_analysis.json`

```json
{
  "keyword": "나노플라스틱",
  "collection": "나노플라스틱",
  "proposal": "사용자 입력 연구 제안",
  "created_at": "2026-03-04",
  "instructions": "Zotero MCP에서 컬렉션 '나노플라스틱'의 논문들을 zotero_search_items로 검색하고, 각 논문의 초록을 확인한 뒤, 아래 연구 제안의 타당성을 한국어로 평가해주세요.\n\n[연구 제안]\n...\n\n[평가 항목]\n1. 주요 연구 트렌드와의 관계\n2. 연구 공백(gap) 및 차별성\n3. 타당성 종합 평가"
}
```

## 터미널 실행

인코딩 문제 회피를 위해 cmd 인수는 영문으로 유지, 한국어 상세 프롬프트는 JSON 내부에 보관.

```python
subprocess.Popen(
    ["cmd", "/k", "claude",
     f"Read {json_path} and follow the instructions field"],
    creationflags=subprocess.CREATE_NEW_CONSOLE,
    cwd=str(project_dir),
)
```

## 변경 범위

| 항목 | 변경 내용 |
|------|-----------|
| `analyze_with_claude()` | 제거 |
| `save_analysis_request()` | 신규 추가 |
| `open_claude_terminal()` | 신규 추가 |
| `run_pipeline()` | 분석 호출 제거, JSON 저장 호출 추가 |
| `generate_summary_docx()` | `analysis` 파라미터 제거 |
| `run_gui()` | "AI 분석" 버튼 추가, 상태 관리 로직 추가 |
| `run_cli()` | `--proposal` 유지, 분석 대신 JSON 저장으로 변경 |

## 버튼 상태 관리

- 초기: "AI 분석" 버튼 `disabled`
- 검색 완료 + proposal 비어있지 않음: 버튼 활성화
- 버튼 클릭 시: 비활성화 (중복 방지) → JSON 저장 → 터미널 오픈 → 다시 활성화
