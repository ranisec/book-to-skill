---
name: book-to-skill-usage
description: "책·문서(PDF/EPUB/DOCX/MD/HTML/RTF/MOBI)를 에이전트 스킬(SKILL.md + 챕터별 파일)로 변환하는 book-to-skill 한국어 사용법. 'book-to-skill', '책을 스킬로', 'PDF를 스킬로 변환' 요청 시 사용."
---

# book-to-skill 한국어 사용 가이드

> 문서 하나(또는 여러 개)를 읽어 **에이전트가 필요할 때 꺼내 쓰는 스킬 폴더**로 바꿔 주는 도구입니다.
> 공식 저장소: [virgiliojr94/book-to-skill](https://github.com/virgiliojr94/book-to-skill) (MIT, 이 가이드는 v1.4.0 기준, 2026-10-28 검토)

> [!WARNING]
> 공식 저장소만 쓰세요. 업스트림 [SECURITY-NOTICE.md](SECURITY-NOTICE.md)에 따르면 `Leutenegger/book-to-skill`은 이 프로젝트와 **무관한** 저장소입니다.

## 목차
1. [어떻게 동작하나요](#1-어떻게-동작하나요)
2. [설치](#2-설치)
3. [추출만 실행하기 (CLI)](#3-추출만-실행하기-cli)
4. [전체 변환 (에이전트가 수행)](#4-전체-변환-에이전트가-수행)
5. [생성된 스킬 검사](#5-생성된-스킬-검사)
6. [생성된 스킬 사용 예](#6-생성된-스킬-사용-예)
7. [주의사항](#7-주의사항)
8. [시험 기록](#8-시험-기록)

## 1. 어떻게 동작하나요
book-to-skill은 두 부분으로 나뉩니다.

| 단계 | 누가 하나 | 네트워크 | 결과 |
|---|---|---|---|
| **추출** | 로컬 Python CLI `book-to-skill` | 없음 (오프라인 확인) | `full_text.txt`, `metadata.json` |
| **생성** | 에이전트가 [SKILL.md](SKILL.md) 지침대로 수행 | 에이전트가 쓰는 LLM | `SKILL.md`, `chapters/`, `glossary.md`, `patterns.md`, `cheatsheet.md` |

> [!IMPORTANT]
> 생성 단계에서는 **문서 내용이 에이전트의 LLM 제공사로 전달됩니다.** 도구 자체에는 API 키나 별도 LLM 호출이 없고,
> 비용은 에이전트 모델의 토큰 요금입니다(책 한 권 = 입력 수십~수백K 토큰). 저작권 있는 책이나 민감한 문서라면 이 점을 먼저 확인하세요.

## 2. 설치
```bash
git clone https://github.com/virgiliojr94/book-to-skill.git
cd book-to-skill
python3 -m venv .venv
.venv/bin/pip install -e ".[pdf,epub,docx,rtf]"   # 필요한 형식만 골라도 됨
.venv/bin/book-to-skill --check                    # 형식별 추출기 설치 상태 점검
```

| extra | 설치되는 것 | 용도 |
|---|---|---|
| `pdf` | pdf-inspector, pypdf, pdfminer.six | PDF |
| `epub` | ebooklib, beautifulsoup4 | EPUB |
| `docx` | python-docx | Word |
| `rtf` | striprtf | RTF |
| `html` | trafilatura | HTML 본문 추출 (없으면 beautifulsoup4로 대체) |
| `technical` | docling | 표·코드·수식 보존 (설치 시 모델 다운로드 발생) |

- `pdftotext`(poppler-utils)가 있으면 PDF 추출에 함께 씁니다.
- `.mobi`, `.azw`, `.azw3`는 Calibre의 `ebook-convert`가 필요합니다.

## 3. 추출만 실행하기 (CLI)
```text
book-to-skill <문서|폴더|glob>... [--mode technical|text] [--install-missing ask|yes|no]
book-to-skill --check
```
```bash
export BOOK_SKILL_WORKDIR=./work/my-book      # 작업 폴더 고정 (없으면 <임시폴더>/book_skill_work-<pid>)
.venv/bin/book-to-skill book.pdf --mode text --install-missing no
.venv/bin/book-to-skill "./docs/*.md" --mode text --install-missing no
```

| 옵션 | 설명 |
|---|---|
| `--mode text` | 산문 위주. 기본값이며 빠름 |
| `--mode technical` | 표·코드·수식 보존 (docling 필요) |
| `--install-missing no` | 빠진 추출기를 **자동 설치하지 않음** (권장). `ask`는 물어보고, `yes`는 바로 `pip install` — 외부 접속 발생 |
| 환경 변수 `BOOK_SKILL_INSTALL_MISSING` | 위 옵션과 같은 역할 (`ask` 기본) |

- **출력**: `$BOOK_SKILL_WORKDIR/full_text.txt`, `metadata.json`(페이지·단어·토큰 수, 챕터, 목차, 원본별 sha256).
  같은 작업 폴더를 다시 쓰는 것은 원본의 sha256이 일치할 때만 안전합니다.
- **지원 형식**: `.pdf` `.epub` `.docx` `.md` `.txt` `.rst` `.adoc` `.html` `.htm` `.xhtml` `.rtf` `.mobi` `.azw` `.azw3`
- **스캔본 PDF**(텍스트 없음)는 앞쪽 페이지를 검사한 뒤 멈추고 OCR을 안내합니다 → `ocrmypdf in.pdf out.pdf` 후 다시 실행.

## 4. 전체 변환 (에이전트가 수행)
에이전트에게 [SKILL.md](SKILL.md)(원본 지침)를 따르게 합니다. 모드는 네 가지입니다.

| 모드 | 내용 |
|---|---|
| Mode 1 | 전체 변환 (기본) |
| Mode 2 | 분석만 (Step 0–3, 보고서만) |
| Mode 3 | 기존 분석 결과에서 생성 |
| Mode 4 | 기존 스킬에 새 문서 합치기 (fold-in) |

단계 요약:
1. **Step 0** 범위 확인·재실행 방지 (같은 이름의 스킬이 있으면 사용자에게 선택을 물음)
2. **Step 1–2** 입력 검증 → 문서 유형 질문 → 추출 (`--mode`)
3. **Step 2.5** 토큰 비용 추정 후 **사용자 확인**
4. **Step 3–5** 구조 분석 → 목적 질문(study/reference) → 이름 정하기
5. **Step 6–9** 생성: `SKILL.md`(~4K 토큰), `chapters/chNN-*.md`(~1K씩), `glossary.md`, `patterns.md`, `cheatsheet.md`
6. **Step 9.5** 보안 스캔 → **Step 10** 정리·보고 → **Step 11** GitHub 게시(선택)

팁:
- 기본 설치 위치는 `~/.agents/skills/<slug>/`입니다. 프로젝트 안에 두려면 `BOOK_TO_SKILL_SCOPE=project`.
- 여러 사람이 함께 쓰는 서버라면 홈 대신 별도 작업 폴더를 지정하고, 확인 후에 스킬 폴더로 옮기세요.
- 큰 책(>50K 토큰)은 `full_text.txt`를 통째로 읽지 말고 챕터 단위로 나눠 읽게 하세요(Step 2.6).

## 5. 생성된 스킬 검사
```bash
python3 tools/scan_generated_skill.py <스킬폴더>      # 프롬프트 인젝션·위험 권한 스캔
python3 tools/validate_skill.py <스킬폴더>/SKILL.md --lens claude   # claude|copilot|amp|hermes|openclaw|opencode
python3 tools/discovery_tax.py ...                     # (선택) 토큰 절감 측정
```
`scan_generated_skill.py`가 0이 아닌 값으로 끝나면, 사람이 확인하기 전에는 그 스킬을 불러오거나 게시하지 마세요.

## 6. 생성된 스킬 사용 예
에이전트 안에서:
```text
/book-to-skill ./book.pdf
/book-to-skill ./docs/ project-knowledge
/book-to-skill "~/books/*.epub" my-library
/book-to-skill new.pdf ~/.claude/skills/project-knowledge     # 기존 스킬에 합치기
```
생성 후 호출:
```text
/<slug>                 # 핵심 개념 요약
/<slug> replication     # 특정 주제
/<slug> ch05            # 특정 챕터
```

## 7. 주의사항
- **Step 11 GitHub 게시**(`gh repo create`, `git push`, `npx skills add`)는 외부로 내보내는 작업입니다. 원할 때만 하세요.
  기본은 비공개(`--private`)이며, 정확히 `public`이라고 답해야 공개됩니다.
- `npx skills add`, `pip install`, docling 모델 다운로드는 외부 접속이 생깁니다.
- 저작권 있는 책으로 만든 스킬은 비공개로 두세요.
- 계정·API 키·쿠키가 필요 없습니다. 유료 API 키도 필요 없습니다(비용은 에이전트 LLM 토큰).
- 검토 시점(v1.4.0) 기준 텔레메트리·업데이트 확인 코드는 없습니다. `git pull`로 올린 뒤에는 다시 확인하세요.
- CLI 출력 끝에 후원 안내 문구가 나옵니다(단순 텍스트).

## 8. 시험 기록
- 2026-10-28: w3.org의 공개 `dummy.pdf`(1쪽)를 **네트워크 차단(`unshare -n`)** 상태에서 추출 → 성공(pdftotext, 9단어, 챕터 0).
- 생성(LLM) 단계는 에이전트 환경에 따라 다르므로 이 기록에는 포함하지 않았습니다.
