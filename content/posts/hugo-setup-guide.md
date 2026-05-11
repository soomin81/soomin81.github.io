+++
date = '2026-05-11T21:51:39+09:00'
draft = false
title = "WSL에서 Hugo 블로그 설치부터 GitHub Pages 배포까지"
categories = ["개발"]
tags = ["hugo", "github-pages", "wsl", "blog"]
+++
## 개요

WSL(Windows Subsystem for Linux) 환경에서 Hugo를 설치하고 GitHub Pages에 배포하는 과정을 정리했습니다.

---

## 1. WSL 환경 준비

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 2. Hugo 설치

apt 기본 버전은 구버전일 수 있어서 직접 설치합니다.
PaperMod 테마 사용 시 **extended** 버전이 필요합니다.

```bash
HUGO_VERSION="0.161.1"

wget https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb

sudo dpkg -i hugo_extended_${HUGO_VERSION}_linux-amd64.deb

hugo version  # 설치 확인
```

---

## 3. 사이트 생성 및 테마 설치

```bash
hugo new site my-blog
cd my-blog
git init

# PaperMod 테마 설치
git submodule add https://github.com/adityatelange/hugo-PaperMod themes/PaperMod
```

---

## 4. hugo.toml 설정

```toml
baseURL = 'https://<username>.github.io/'
languageCode = 'ko-kr'
title = '내 블로그'
theme = "PaperMod"
paginate = 10

[outputs]
  home = ["HTML", "RSS", "JSON"]  # 검색 기능에 필요

[taxonomies]
  category = "categories"
  tag = "tags"
  series = "series"

[frontmatter]
  date = [":git", "date"]           # 작성일 git 커밋 날짜 자동 적용
  lastmod = [":git", "lastmod", "date"]  # 수정일 git 커밋 날짜 자동 적용

[params]
  defaultTheme = "auto"           # 시스템 다크/라이트 모드 따라감
  ShowReadingTime = true          # 읽기 시간 표시
  ShowPostNavLinks = true         # 다음글/이전글
  ShowCodeCopyButtons = true      # 코드 복사 버튼
  ShowToc = true                  # 목차
  TocOpen = false                 # 목차 기본 접힘
  ShowBreadCrumbs = true          # 경로 표시
  ShowCategories = true           # 카테고리 표시
  favicon = "/favicon.ico"        # 파비콘

  [params.homeInfoParams]
    Title = "안녕하세요 👋"
    Content = "개발 블로그입니다"

[[menus.main]]
  name = "글 목록"
  url = "/posts/"
  weight = 1

[[menus.main]]
  name = "카테고리"
  url = "/categories/"
  weight = 2

[[menus.main]]
  name = "태그"
  url = "/tags/"
  weight = 3

[[menus.main]]
  name = "검색"
  url = "/search/"
  weight = 4
```

---

## 5. 검색 페이지 생성

```bash
cat > content/search.md << 'EOF'
+++
title = "Search"
layout = "search"
summary = "search"
placeholder = "검색어를 입력하세요"
+++
EOF
```

---

## 6. GitHub 리포지토리 생성

GitHub에서 **`<username>.github.io`** 이름으로 public 리포지토리 생성합니다.
(README 등 아무것도 체크하지 않기)

---

## 7. GitHub Actions 배포 설정

`.github/workflows/deploy.yml` 생성:

```yaml
name: Deploy Hugo to GitHub Pages

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0        # git 날짜 자동 적용에 필요

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "0.161.1"
          extended: true

      - name: Build
        run: hugo --minify

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

---

## 8. GitHub Pages 설정

리포지토리 **Settings → Pages → Source** 를 **`GitHub Actions`** 로 변경합니다.

---

## 9. Personal Access Token 발급

GitHub 비밀번호 인증이 막혀있어서 PAT가 필요합니다.

`https://github.com/settings/tokens/new` 에서 발급:
- scope: `repo`, `workflow` 체크

```bash
git remote set-url origin https://<token>@github.com/<username>/<username>.github.io.git
```

---

## 10. 첫 배포

```bash
git add .
git commit -m "init: hugo blog"
git push -u origin main
```

push 후 `https://github.com/<username>/<username>.github.io/actions` 에서 배포 진행 확인.
완료되면 `https://<username>.github.io` 접속!

---

## 글 작성 방법

```bash
# 파일명은 영어 추천 (URL 인코딩 문제)
hugo new posts/my-first-post.md
```

front matter 예시:

```yaml
+++
title = "첫 번째 글"
date = 2026-05-11
draft = false
categories = ["개발"]
tags = ["hugo", "blog"]
+++

본문 내용...
```

> `draft = false` 로 변경해야 배포됩니다.

글 작성 후 배포:

```bash
git add .
git commit -m "post: 글 제목"
git push
```

push 하면 GitHub Actions가 자동으로 빌드 및 배포합니다.
