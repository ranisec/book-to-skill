---
name: book-to-skill-usage
description: "책·문서(PDF/EPUB/DOCX/MD/HTML/RTF/MOBI)를 에이전트 스킬(SKILL.md + 챕터별 파일)로 변환하는 book-to-skill 사용법. 공용 박스 /workspace/tools/book-to-skill 에 설치됨. 'book-to-skill', '책을 스킬로', 'PDF를 스킬로 변환' 요청 시 사용."
---

# book-to-skill 사용법 (공용 박스용)

저장소: https://github.com/virgiliojr94/book-to-skill (MIT, v1.4.0, 공식). 설치 위치: `/workspace/tools/book-to-skill` (전용 venv `.venv`).
검토일 2026-10-28. 사칭 저장소 `Leutenegger/book-to-skill`(악성 재업로드, SECURITY-NOTICE.md 참고)은 **절대 사용 금지**.

## 1. 구조 (핵심 이해)
두 부분으로 나뉜다.
1. **추출기(로컬 Python)** — 문서 → 깨끗한 텍스트 + metadata.json. 네트워크 호출 코드 없음. 오프라인 검증됨.
2. **생성기 = 에이전트가 SKILL.md 지침을 따라 수행** — 추출 텍스트를 읽고 챕터 요약/용어집/패턴/치트시트를 *에이전트의 LLM이* 작성. 즉 **문서 내용은 그 에이전트가 쓰는 LLM(제공사)에게 전달된다.** 이 도구 자체는 별도 API 키·외부 LLM 호출이 없다. 토큰 비용은 에이전트 모델 요금에 따른다(책 1권 = 입력 수십~수백K 토큰).
→ **저작권 있는 책/민감 문서는 이 점을 사용자에게 알리고 진행.**

## 2. 추출만 실행 (CLI, 이 박스에서 바로 가능)
```bash
cd /workspace/tools/book-to-skill
export BOOK_SKILL_WORKDIR=/workspace/tmp/b2s-<이름>      # 작업 폴더 고정 권장(없으면 /tmp/book_skill_work-<pid>)
.venv/bin/book-to-skill <파일|폴더|glob>... --mode text --no-install-missing
#   --mode text      : 산문 위주(기본·빠름, pdftotext 등)
#   --mode technical : 표·코드·수식 보존(Docling 필요, 미설치. 설치 시 모델 다운로드 발생 → 사용자 확인 후)
.venv/bin/book-to-skill --check     # 형식별 추출기 설치 상태 점검
.venv/bin/book-to-skill --help
```
- 항상 `--no-install-missing`(또는 `BOOK_SKILL_INSTALL_MISSING=no`) 붙일 것. 기본 `ask`는 TTY에서 pip 설치를 제안하고, `yes`면 자동 pip install(외부 접속)함.
- 출력: `$BOOK_SKILL_WORKDIR/full_text.txt`, `metadata.json`(페이지·단어·토큰·챕터·ToC·sha256). 같은 WORKDIR 재사용은 sources sha256 일치 시에만.
- 지원: .pdf .epub .docx .md .txt .rst .adoc .html/.htm/.xhtml .rtf, .mobi/.azw/.azw3(Calibre `ebook-convert` 필요 — 이 박스엔 없음).
- 스캔본 PDF(텍스트 없음)는 초반 페이지 검사 후 중단하고 OCR 안내 → `ocrmypdf in.pdf out.pdf` 후 재시도.
- 설치된 추출기: pdf-inspector, pypdf, pdfminer.six, pdftotext(poppler), ebooklib, beautifulsoup4, python-docx, striprtf. 미설치: docling, trafilatura(HTML 경계 제거, bs4 폴백으로 동작), Calibre.

## 3. 전체 변환(스킬 생성) — 에이전트 절차
에이전트가 `/workspace/tools/book-to-skill/SKILL.md`(55KB, 원본 지침)를 읽고 단계대로 수행. 요약:
- Mode 1 전체 변환(기본) / Mode 2 analyze-only(Step 0–3까지, 보고서만) / Mode 3 기존 분석에서 생성 / Mode 4 기존 스킬에 fold-in(업데이트).
- Step 0 범위 확인·재실행 가드(같은 슬러그 이미 있으면 사용자에게 선택 질문) → Step 1 입력 검증 → 1.5 유형 질문(technical/text/모름) → 2 추출(`--mode`) → 2.5 **비용 추정 후 사용자 확인** → 3 구조 분석 → 4 목적 질문(→ DEPTH study/reference) → 5 이름 → 6~9 생성(`SKILL.md` ~4K토큰, `chapters/chNN-*.md` ~1K, `glossary.md`, `patterns.md`, `cheatsheet.md`) → **9.5 보안 스캔** → 10 정리·보고 → 11 GitHub 게시(선택).
- 생성물 위치: 기본 `~/.agents/skills/<slug>/`. **공용 박스에서는 홈 대신 작업용 폴더(예: /workspace/skills-out/<slug>/)를 명시**하고, 최종 스킬 등록은 사용자 승인 후 부모 에이전트가 저장. 프로젝트 로컬은 `BOOK_TO_SKILL_SCOPE=project`.
- 대용량(>50K 토큰) 책은 `full_text.txt`를 통째로 읽지 말고 grep/offset·limit로 챕터 단위 읽기(Step 2.6).

## 4. 생성된 스킬 검사 도구
```bash
python3 tools/scan_generated_skill.py <스킬폴더>                  # 프롬프트 인젝션/위험 권한 스캔(비0 종료 시 사람 검토 전엔 로드·게시 금지)
python3 tools/validate_skill.py <SKILL.md> --lens claude|copilot|amp|hermes|openclaw|opencode
python3 tools/discovery_tax.py ...                                # 토큰 절감 측정(선택)
```

## 5. 사용 예 (생성된 스킬 호출)
`/book-to-skill ./book.pdf` · `/book-to-skill ./docs/ project-knowledge` · `/book-to-skill "~/books/*.epub" my-library` · `/book-to-skill new.pdf ~/.claude/skills/project-knowledge`(fold-in)
생성 후: `/<slug>` (핵심 모델) · `/<slug> replication` (주제) · `/<slug> ch05` (챕터).

## 6. 주의·금지 (사용자 상시 규칙 반영)
- **Step 11 GitHub 게시(`gh repo create`, `git push`, `npx skills add`)는 외부 전송 — 사용자가 명시 요청·승인한 경우에만.** 기본 비공개(`--private`)이며 `public`만 정확히 답해야 공개. 평소엔 건너뛴다(skip).
- `npx skills add`·`pip install`·docling 모델 다운로드는 외부 접속 — 임의로 실행 금지.
- 저작권 책으로 만든 스킬은 비공개 유지.
- 사용자 계정·키·쿠키 접근 불필요 — 건드리지 말 것. 유료 API 키 불필요(비용은 에이전트 LLM 토큰).
- 업데이트 확인/텔레메트리 없음(검토 시점). 업그레이드는 `git pull` 후 이 문서의 코드 검사를 다시 할 것.
- 출력 끝에 후원 안내 문구(sponsors 링크)가 출력됨 — 단순 텍스트.

## 7. 시험 기록 (2026-10-28)
w3.org 공개 dummy.pdf(1쪽)를 **네트워크 차단(unshare -n)** 상태로 변환 → 추출 성공(pdftotext, 9단어, 챕터 0). 생성(LLM) 단계는 박스 CLI 범위 밖이라 미시험.
