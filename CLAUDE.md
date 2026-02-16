# hojoon.me - Hugo 블로그

## 프로젝트 개요
- Hugo + PaperMod 테마 기반 개인 블로그
- 글은 `content/posts/` 에 마크다운으로 작성

## 블로그 글 작성 워크플로우

### 소스 자료 위치
학업 노트가 `~/mnt/nas/icloud/` 에 학기별로 정리되어 있다. 블로그 글을 쓸 때는 이 디렉토리의 lecture note와 정리 노트를 참고해서 작성한다.

```
~/mnt/nas/icloud/
├── 2021-Spring/    # 일반화학, 미적분학, 일반물리학, 일반생물학 등
├── 2021-Fall/      # CS101, MAS102, MAS109, PH142 등
├── 2022-Spring/    # CS204, CS206, EE201, EE202, MAS201 등
├── 2022-Fall/      # CS230, CS300, EE304, HSS112 등
├── 2023-Spring/    # CS202, CS270, CS311, CS320, CS492 등
├── 2023-Fall/      # CS330, CS360, CS479 등
├── Additional Study/  # cs231n 등 자율 학습
├── Research/       # 2023-CASYS(Jongse-Park)
└── Dropped/        # 드랍한 과목
```

### 글 작성 순서
1. `~/mnt/nas/icloud/` 에서 해당 과목 디렉토리의 자료를 먼저 읽는다
2. lecture note + 정리 노트 내용을 파악한다
3. 블로그 글 형태로 재구성하여 `content/posts/` 에 작성한다

### Front Matter 형식
```yaml
---
title: "[과목코드] N. 제목"
date: YYYY-MM-DD
draft: false
tags: ["과목코드", "주제", "KAIST", ...]
categories: ["Computer Science"]
summary: "한두 문장 요약"
ShowToc: true
TocOpen: true
---
```

### 작성 규칙
- 시리즈 글은 `[CS230] 1. 제목` 형식으로 번호를 매긴다
- 파일명은 `{과목코드 소문자}-{번호}-{slug}.md` (예: `cs230-1-data-representation.md`)
- 글 첫 부분에 blockquote로 과목 정보와 교재를 명시한다
- 코드 블록에는 언어 힌트를 반드시 붙인다 (` ```c `, ` ```text ` 등)
- 한국어로 작성하되, 전공 용어는 영어를 병기한다
