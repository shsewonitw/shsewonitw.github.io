---
layout: post
title: "[ETC] SVN to Git 마이그레이션"
subtitle: "#svn #git #migration"
category: devlog
tags: etc
---

# SVN to Git 마이그레이션 — 실전 방법과 명령어 정리

SVN에서 Git으로 이전하는 것은 많은 기업과 개발자에게 중요한 작업입니다.

## SVN 사용자 목록 추출

```bash
svn log --quiet URL | grep '^r' | awk '{print $3}' | sort | uniq > svn_authors.txt
```

## Git으로 마이그레이션

```bash
git svn clone http://svn.repo.url --no-metadata -A users.txt -T trunk -b branches -t tags ./git-repo
```

## 결론

git svn을 활용하면 이력과 브랜치를 모두 유지한 상태로 쉽게 마이그레이션이 가능합니다.
