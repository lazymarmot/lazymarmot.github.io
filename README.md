# Lazymarmot's Blog

This repository contains the source code for **https://lazymarmot.github.io**.  
It is a Jekyll site based on the `plainwhite` theme, customized for:

- Personal study notes (primarily about **AMBA AXI4**)
- A simple **portfolio** section for selected projects

---

## Structure

- `_config.yml`  
  Global site configuration (title, author, navigation, theme options).

- `_layouts/`  
  - `default.html` – Base layout (profile sidebar, global styles, home button).  
  - `home.html` – Blog index that lists study posts.  
  - `portfolio.html` – Custom layout that lists posts categorized as `portfolio`.  
  - `post.html`, `page.html` – Individual post/page templates.

- `_posts/AXI/`  
  AMBA AXI4 related notes:
  - `2026-03-01-axi4-notes.markdown` – **AXI4 Notes** index (links to all AXI articles).
  - `2026-03-04-...` – Detailed posts (Architecture, Channel signaling, SIGNAL*, Cache, Glossary, Memory Type, etc.).

- `_posts/Portfolio/`  
  Portfolio entries (each post with `categories: portfolio`).

- `study.md`  
  Entry point for study notes. Uses `layout: home` and lists normal blog posts.

- `portfolio.md`  
  Entry point for portfolio. Uses `layout: portfolio` and lists posts tagged with `portfolio`.

- `assets/posts_image/AXI/`  
  Images used in AXI posts, organized by topic (e.g. `AXI Architecture`, `Channel Signaling Requirements`, `SIGNAL *`, `AXI Glossary`, etc.).

---

## Running locally

```bash
bundle install
bundle exec jekyll serve
```

Then open:

- `http://localhost:4000/` → automatically redirects to `/study/`
- `http://localhost:4000/study/` → study posts
- `http://localhost:4000/portfolio/` → portfolio posts

---

## Writing a new study post

1. 새 글 파일을 `_posts/` (또는 `_posts/AXI/` 같은 서브 디렉토리) 아래에 만든다.

   ```markdown
   _posts/2026-03-10-my-new-note.markdown
   ```

2. 다음과 같은 front matter 를 사용한다.

   ```markdown
   ---
   layout: post
   title:  "My New Note"
   date:   2026-03-10 12:00:00 +0900
   categories: AXI4 AMBA   # 원하는 태그들
   show_on_home: true      # 홈(/study)에 숨기고 싶으면 false 로 설정
   ---
   ```

3. 아래에 내용을 마크다운으로 작성한다.

이렇게 하면, `show_on_home` 이 `false` 가 아닌 한 `/study/` 의 포스트 리스트에 나타난다.

---

## Adding a new portfolio item

1. `_posts/Portfolio/` 아래에 새 포스트를 만든다.

   ```markdown
   _posts/Portfolio/2026-03-10-my-project.markdown
   ```

2. 다음 front matter 를 사용한다.

   ```markdown
   ---
   layout: post
   title: "My Project"
   date: 2026-03-10 12:00:00 +0900
   categories: portfolio something-else
   show_on_home: false
   ---
   ```

3. 프로젝트 설명, 사용 기술, 스크린샷 링크 등을 본문에 작성한다.

`categories` 에 `portfolio` 가 포함된 포스트는 자동으로 `/portfolio/` 페이지의 리스트에 나타난다.

---

## Deployment

이 사이트는 **GitHub Pages** 로 빌드/호스팅된다.

- `main` 브랜치가 소스 브랜치이다.
- `main` 에 push 할 때마다 GitHub 가 Jekyll 을 실행하고 결과를 `https://lazymarmot.github.io` 에 배포한다.

별도의 CI/CD 설정 없이, 수정 후 다음 명령으로 배포할 수 있다.

```bash
git add .
git commit -m "Update blog content"
git push origin main
```
