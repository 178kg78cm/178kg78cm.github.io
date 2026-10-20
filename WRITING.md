# 작성 가이드

이 블로그에 글을 쓸 때 지키는 규칙. (사이트 빌드에서는 제외된다)

- `/` : 포트폴리오 (경력·스킬은 [`_data/portfolio.yml`](_data/portfolio.yml), 프로젝트 카드는 글의 `project:`)
- `/blog/` : 글 목록, `/categories/`, `/tags/`

## 글 파일

`_posts/<폴더>/YYYY-MM-DD-영문-제목.md` 파일을 만든다. 파일명에는 공백을 넣지 않는다.

```yaml
---
layout: post
title:  "글 제목"
date:   2025-10-20 +0900
categories: Embedded
tags: [STM32, UART]
---
```

## 카테고리

글 하나에 카테고리 하나만 쓴다. 세부 주제는 `tags`로 표시한다.

| 카테고리 | 다루는 내용 |
| --- | --- |
| `Embedded` | MCU, 주변장치, C/C++, 메모리, 최적화, 리눅스 |
| `AUTOSAR` | SWC, RTE, BSW, 통신·메모리 스택 |
| `Network` | CAN, Ethernet, 진단, CANoe, OTA |
| `RTOS` | OSEK/AUTOSAR OS, 스케줄링, 동기화 |
| `Process` | A-SPICE, ISO 26262, 요구공학, 테스팅 |
| `Security` | ISO/SAE 21434, SecOC, 시큐어 코딩 |
| `Algorithm` | 알고리즘, 코딩 테스트 |
| `Etc` | 그 외 |

## 메인 프로젝트 카드로 보여주기

front matter에 `project:`를 추가하면 메인의 프로젝트 섹션에 카드로 나온다. 카드는 그 글로 연결된다.

```yaml
project:
  name: "카드 제목"        # 없으면 글 제목
  summary: "카드 설명"     # 없으면 글 첫 문단
  tech: [TC275, CAN]      # 없으면 글 tags
  order: 1                # 표시 순서. 없으면 뒤쪽에 최신순
```

간단히 `project: true` 한 줄만 써도 된다.

## 이미지

VS Code에서 글에 이미지를 붙여넣으면(`Ctrl+V`) `assets/images/<글 파일명>/`에 저장되고 링크가 자동으로 들어간다. ([`.vscode/settings.json`](.vscode/settings.json))

```markdown
![설명](../../assets/images/2025-10-20-uart-dma/image.png)
```

## 다른 글 링크

글 주소는 `/<YYYY-MM-DD>-<파일명 제목>/` 이다. 예: `[RTOS 정리](/2025-04-07-A_RTOS_정리/)`

## 로컬에서 확인

```bash
bundle install
bundle exec jekyll serve
# http://localhost:4000
```
