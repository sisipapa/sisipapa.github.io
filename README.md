# SiSipapa 개발노트

개발하면서 마주친 문제, 새로 배운 기술, 공부한 내용을 정리하고 기록하는 개인 블로그입니다.

> 같은 문제를 두 번 검색하지 않기 위해, 그리고 배운 것을 오래 기억하기 위해 글로 남깁니다.

## 소개

이 저장소는 [GitHub Pages](https://pages.github.com/)와 [Jekyll](https://jekyllrb.com/)로 만든 정적 블로그입니다.
개발 또는 학습한 내용을 주제별로 정리해 두는 공간으로 사용하고 있습니다.

- **블로그 주소**: https://sisipapa.github.io
- **주요 관심 주제**: Spring Boot, JPA, Docker, Kubernetes(k8s)

## 기술 스택

| 구분 | 사용 기술 |
| --- | --- |
| 정적 사이트 생성기 | Jekyll |
| 테마 | Simple Texture |
| 마크업/스타일 | HTML, SCSS(Sass) |
| 마크다운 엔진 | kramdown |
| 호스팅 | GitHub Pages |

## 디렉터리 구조

```text
sisipapa.github.io
├─ _config.yml      # 블로그 전체 설정 파일
├─ _data/           # 사이트에서 사용하는 데이터 파일
├─ _includes/       # 머리글, 바닥글 등 재사용 가능한 조각 템플릿
├─ _layouts/        # 페이지의 공통 레이아웃 템플릿
├─ _posts/          # 블로그 글(포스트)이 저장되는 폴더
├─ _sass/           # 스타일(SCSS) 파일
├─ assets/          # 이미지, CSS, JS 등 정적 리소스
├─ blog/            # 블로그 목록/카테고리/태그 페이지
├─ index.html       # 메인 페이지
└─ search.json      # 사이트 내 검색용 데이터
```

## 글 작성 방법

새 글은 `_posts` 폴더에 마크다운 파일로 추가합니다.
파일 이름은 반드시 `YYYY-MM-DD-제목.md` 형식을 따라야 합니다.

```markdown
---
layout: post
title: "글 제목"
date: 2026-06-22 09:00:00 +0900
categories: [Spring Boot]
tags: [jpa, querydsl]
---

여기에 본문 내용을 작성합니다.
```

상단의 `---`로 감싼 부분은 **Front Matter**라고 부르며, 글의 제목·날짜·분류 같은 메타데이터를 적는 영역입니다.

## 로컬에서 실행하기

블로그를 내 컴퓨터에서 미리 확인하려면 아래 순서대로 진행합니다.

```bash
# 1. 필요한 라이브러리(gem) 설치
bundle install

# 2. 로컬 서버 실행
bundle exec jekyll serve

# 3. 브라우저에서 접속
# http://localhost:4000
```

> 실행하려면 [Ruby](https://www.ruby-lang.org/)와 [Bundler](https://bundler.io/)가 설치되어 있어야 합니다.

## 라이선스

이 저장소의 글 콘텐츠는 [CC BY-SA 4.0](http://creativecommons.org/licenses/by-sa/4.0/) 라이선스를 따릅니다.
테마 및 코드 라이선스는 [LICENSE](./LICENSE) 파일을 참고해 주세요.
