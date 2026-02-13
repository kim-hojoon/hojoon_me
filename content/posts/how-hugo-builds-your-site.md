---
title: "Hugo는 어떻게 웹사이트를 만들어줄까?"
date: 2026-02-12
draft: false
tags: ["Hugo", "Static Site Generator", "Web", "Tutorial"]
categories: ["Technology"]
summary: "마크다운 파일 하나가 어떻게 멋진 웹페이지로 변하는지, 이 블로그를 예시로 하나하나 따라가봅니다."
ShowToc: true
TocOpen: true
---

지금 여러분이 보고 있는 이 블로그는 **Hugo**라는 도구로 만들어졌습니다.

저는 HTML을 직접 작성하지 않았습니다. 대신 이렇게 편한 마크다운(Markdown) 파일을 하나 썼을 뿐인데, Hugo가 알아서 예쁜 웹페이지로 바꿔줬습니다. 어떻게 이런 일이 가능한 걸까요?

## 정적 사이트 생성기란?

웹사이트를 만드는 방법은 크게 두 가지가 있습니다.

**동적 사이트**는 누군가 페이지를 열 때마다 서버가 실시간으로 HTML을 만들어서 보내줍니다. WordPress가 대표적이죠. 서버에 PHP, 데이터베이스 같은 것들이 필요합니다.

**정적 사이트**는 미리 만들어둔 HTML 파일을 그대로 보내줍니다. 서버가 할 일이 거의 없어서 빠르고, 보안 걱정도 적습니다.

Hugo는 **정적 사이트 생성기**(Static Site Generator)입니다. 마크다운으로 글을 쓰면, Hugo가 미리 HTML로 변환해둡니다. 방문자가 접속하면 이미 완성된 HTML을 바로 전달하기만 하면 되는 거죠.

## 이 블로그의 실제 구조

이 블로그의 프로젝트 폴더를 열어보면 이렇게 생겼습니다.

```text
hojoon_me/
├── hugo.yaml           # 사이트 설정 파일
├── content/            # 글을 쓰는 곳
│   ├── posts/          # 블로그 포스트들
│   ├── projects/       # 포트폴리오
│   └── about.md        # 소개 페이지
├── themes/
│   └── PaperMod/       # 디자인 테마
├── static/             # 이미지, 파비콘 등
└── public/             # Hugo가 만들어낸 최종 결과물
```

각 폴더가 하는 역할을 하나씩 살펴보겠습니다.

### `content/` - 글을 쓰는 곳

여러분이 신경 쓸 곳은 사실상 여기뿐입니다. 이 폴더 안에 마크다운 파일을 만들면 그게 곧 블로그 글이 됩니다.

예를 들어, 지금 이 글의 원본 파일은 `content/posts/how-hugo-builds-your-site.md`에 있습니다. 파일 맨 위에는 이런 정보가 적혀 있습니다.

```yaml
---
title: "Hugo는 어떻게 웹사이트를 만들어줄까?"
date: 2026-02-12
tags: ["Hugo", "Static Site Generator"]
---
```

이 부분을 **Front Matter**라고 부릅니다. 글의 제목, 날짜, 태그 같은 메타 정보를 적는 곳이죠. Hugo는 이 정보를 읽어서 목록 페이지를 자동으로 만들어주고, 태그별로 글을 분류해줍니다.

그 아래에는 평범한 마크다운 문법으로 글 내용을 작성합니다. 지금 여러분이 읽고 있는 이 글처럼요.

### `themes/PaperMod/` - 디자인 담당

글을 쓰면 내용은 있지만 디자인이 없겠죠? 테마가 그 역할을 합니다.

이 블로그는 **PaperMod**라는 테마를 사용하고 있습니다. 상단의 네비게이션 바, 다크 모드 토글 버튼, 글 목록의 레이아웃, 검색 기능 등 여러분이 보는 모든 디자인 요소가 이 테마에서 나옵니다.

테마를 바꾸고 싶다면? 다른 테마를 다운로드하고 설정 파일에서 테마 이름만 바꿔주면 됩니다. 글 내용은 하나도 건드릴 필요가 없습니다. 이것이 콘텐츠와 디자인을 분리하는 Hugo의 장점입니다.

### `hugo.yaml` - 사이트의 설정서

이 파일 하나에 사이트의 모든 설정이 들어있습니다. 실제 내용을 일부 살펴보면 이렇습니다.

```yaml
baseURL: "https://hojoon.me/"
title: "Hojoon Kim"
theme: "PaperMod"

params:
  defaultTheme: auto          # 시스템 설정에 따라 다크/라이트 자동 전환
  disableThemeToggle: false   # 수동 전환 버튼도 활성화

  profileMode:
    enabled: true
    title: "Hojoon Kim"
    subtitle: "Hello, I'm Hojoon Kim."
    buttons:
      - name: Blog
        url: /posts/
      - name: Projects
        url: /projects/
```

- `baseURL`은 사이트의 주소입니다.
- `theme`은 어떤 테마를 쓸지 알려줍니다.
- `profileMode`는 메인 페이지에 프로필 카드를 보여주는 설정입니다.

이 설정 파일을 수정하면 사이트의 동작이 바뀝니다. 코드를 작성하는 게 아니라, 옵션을 고르는 것에 가깝습니다.

### `public/` - 최종 결과물

여기가 핵심입니다. 터미널에서 `hugo` 명령어를 실행하면 이런 일이 벌어집니다.

```text
$ hugo

Start building sites …

                   | EN
-------------------+-----
  Pages            | 16
  Paginator pages  |  0
  Non-page files   |  0
  Static files     |  0
  Processed images |  0
  Aliases          |  2

Total in 58 ms
```

58밀리초. 0.058초 만에 16개 페이지가 생성됩니다. Hugo가 빠르다고 유명한 이유가 이것입니다.

이 명령이 실행되면 `public/` 폴더 안에 완성된 HTML, CSS, JavaScript 파일이 생깁니다. 웹 서버는 이 폴더만 공개하면 됩니다.

## 빌드 과정을 그림으로 보면

글을 쓰고 사이트에 반영되기까지의 흐름을 정리하면 이렇습니다.

```text
마크다운 파일 작성
       │
       ▼
  content/posts/my-post.md    ← 여러분이 쓰는 글
       │
       ▼
   hugo 명령어 실행
       │
       ├─ hugo.yaml 설정을 읽고
       ├─ themes/PaperMod/ 디자인을 입히고
       ├─ Front Matter 정보로 목록, 태그 페이지를 만들고
       │
       ▼
   public/ 폴더에 HTML 생성    ← 완성된 웹사이트
       │
       ▼
   Caddy 웹 서버가 public/ 을 서빙
       │
       ▼
   https://hojoon.me 접속 가능!
```

즉, 여러분이 하는 일은 **마크다운 파일 하나 작성하고 `hugo` 한 번 실행하는 것**이 전부입니다.

## 새 글은 어떻게 추가하나요?

정말 간단합니다.

```bash
hugo new posts/my-new-post.md
```

이 명령을 실행하면 `content/posts/my-new-post.md` 파일이 자동으로 만들어집니다. Front Matter가 미리 채워져 있어서, 본문만 작성하면 됩니다.

글을 다 썼다면 빌드하고 확인합니다.

```bash
hugo server
```

이 명령은 로컬에서 미리보기 서버를 띄워줍니다. 브라우저에서 `http://localhost:1313`으로 접속하면 결과를 바로 확인할 수 있습니다. 파일을 수정하면 자동으로 새로고침까지 됩니다.

만족스러우면 `hugo` 명령으로 최종 빌드하면 끝입니다.

## 왜 Hugo를 선택했나요?

이 사이트는 원래 `index.html` 파일 하나와 `style.css` 파일 하나로 이루어진 아주 단순한 페이지였습니다. 하지만 블로그 글을 추가하려면 매번 HTML을 직접 작성해야 하고, 목록 페이지도 수동으로 관리해야 합니다. 글이 10개만 넘어가도 감당하기 어려워집니다.

Hugo를 도입하면서 얻은 것들을 정리하면 이렇습니다.

| 전 (정적 HTML) | 후 (Hugo) |
|---|---|
| 글마다 HTML 직접 작성 | 마크다운으로 내용만 작성 |
| 목록 페이지 수동 관리 | 자동 생성 |
| 디자인 변경 시 모든 파일 수정 | 테마 교체 한 번으로 해결 |
| 검색 기능 없음 | 내장 검색 지원 |
| 다크 모드 직접 구현 필요 | 설정 한 줄로 활성화 |

서버 비용도 동일합니다. 똑같이 정적 파일을 서빙하는 것이니까요. 이 사이트는 Proxmox 위의 Ubuntu 서버에서 Caddy로 서빙하고 있는데, Hugo 전환 전후로 서버 설정은 거의 달라지지 않았습니다.

## 마무리

Hugo의 핵심 철학은 단순합니다.

> **글쓰기에만 집중하세요. 나머지는 Hugo가 해결합니다.**

마크다운 파일을 만들고, `hugo` 명령어를 실행하면, 완성된 웹사이트가 나옵니다. 데이터베이스도 없고, 서버 사이드 언어도 없고, 복잡한 배포 과정도 없습니다.

궁금한 점이 있다면 [Hugo 공식 문서](https://gohugo.io/documentation/)를 참고해보세요. 그리고 이 블로그의 소스 코드도 곧 GitHub에 공개할 예정이니, 실제 구조를 직접 살펴보실 수 있을 겁니다.
