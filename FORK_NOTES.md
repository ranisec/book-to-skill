# ranisec 포크 노트 — book-to-skill

| 항목 | 내용 |
|---|---|
| 업스트림 | [virgiliojr94/book-to-skill](https://github.com/virgiliojr94/book-to-skill) |
| 라이선스 | MIT (업스트림과 동일, 변경 없음) |
| 원본 코드 수정 | 없음 (문서 1개 추가만) |
| 업스트림 PR | 보내지 않음 |
| 검토한 버전 | v1.4.0 |

## [ranisec fork] - 2026-10-29

### 추가: `USAGE_SKILL.md`
한국어 사용 가이드입니다. 앞부분에 스킬용 YAML 머리말이 있어 에이전트 스킬로도 그대로 쓸 수 있습니다.

담은 내용:
1. 구조: 로컬 추출기(Python, 네트워크 없음)와 에이전트 LLM이 하는 생성 단계의 차이
2. 설치와 추출 CLI (`--mode`, `--check`, `--install-missing`, `BOOK_SKILL_WORKDIR`)
3. 전체 변환 절차 요약 (Mode 1–4, Step 0–11)
4. 생성된 스킬 검사 도구 (`tools/scan_generated_skill.py`, `tools/validate_skill.py`)
5. 개인정보·저작권·외부 접속 주의사항
6. 오프라인 시험 기록

### 사용법
GitHub에서 [USAGE_SKILL.md](USAGE_SKILL.md)를 그대로 읽으면 됩니다. 에이전트 스킬로 쓰려면 스킬 폴더에 `SKILL.md`라는 이름으로 복사하세요.
```bash
mkdir -p ~/.agents/skills/book-to-skill-usage
cp USAGE_SKILL.md ~/.agents/skills/book-to-skill-usage/SKILL.md
```
