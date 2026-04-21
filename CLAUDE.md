# CLAUDE.md — Blog Post Creation Guidelines

## Post File Naming Rules

When creating a new post file under `_posts/`, the **filename** must follow these rules to avoid `invalid path` errors on Windows during `git pull`.

### Forbidden characters in filenames

| Character | Reason | Replacement |
|-----------|--------|-------------|
| `:` (colon) | Invalid on Windows | remove or replace with `-` |
| `?` (question mark) | Invalid on Windows | remove |
| `/` (slash) | Creates unintended nested directories | replace with `-` |
| `\` (backslash) | Invalid on Windows | replace with `-` |
| `*` `<` `>` `"` `\|` | Invalid on Windows | remove |

### Example

**Title in front matter** (special characters allowed):
```
title: "Transformer 아키텍처의 핵심: 셀프 어텐션(Self-Attention)의 원리"
```

**Filename** (special characters must be removed/replaced):
```
# Bad — causes git pull error
2025-11-13-transformer-아키텍처의-핵심:-셀프-어텐션(self-attention)의-원리.md

# Good
2025-11-13-transformer-아키텍처의-핵심-셀프-어텐션(self-attention)의-원리.md
```

More examples:

| Original title fragment | Filename fragment |
|------------------------|------------------|
| `무엇인가?` | `무엇인가` |
| `하는가?)` | `하는가)` |
| `big-o 표기법: 시간 복잡도` | `big-o-표기법-시간-복잡도` |
| `CI/CD 파이프라인` | `cicd-파이프라인` or `ci-cd-파이프라인` |
| `TCP/IP 4계층` | `tcpip-4계층` or `tcp-ip-4계층` |
| `비동기 I/O` | `비동기-io` |
| `HTTP/1.1, HTTP/2` | `http-1.1-http-2` |

### Checklist before saving a new post file

- [ ] Filename contains no `:` `?` `/` `\` `*` `<` `>` `"` `|`
- [ ] Filename format: `YYYY-MM-DD-slug.md`
- [ ] `title:` in front matter may still contain the original special characters for display

## Category Slugs

The `category:` field in post front matter must use the exact slugs below.
Do NOT use the old `-ai`, `-backend`, etc. values — they cause 404 errors.

### DMS sub-categories

| Title shown in sidebar | Slug to use in `category:` |
|------------------------|---------------------------|
| AI                     | `dms-ai`                  |
| Frontend               | `dms-frontend`            |
| Backend                | `dms-backend`             |
| Devops                 | `dms-devops`              |
| Cloud                  | `dms-cloud`               |
| DSA                    | `dms-dsa`                 |
| OS                     | `dms-os`                  |
| Network                | `dms-network`             |
| Database               | `dms-database`            |

**Rule:** DMS sub-category slugs always follow the pattern `dms-<topic>`.

## Post Front Matter Template

```yaml
---
layout: post
title: "[Daily morning study] 제목 (특수문자 허용)"
description: >
  #daily morning study
category: 
    - dms
    - dms-ai        # ← use dms-<topic> slug from the table above
hide_last_modified: true
---
```

## Duplicate Post Prevention

**Before writing a new post, always check existing posts to avoid duplicate topics.**

```bash
# Check existing post titles
ls _posts/dms/-database/
ls _posts/dms/-ai/
# (check the relevant category directory)
```

### Rules

- Do NOT write a post whose core topic is already covered by an existing post.
- If a related post exists, choose a clearly different angle or a more specific subtopic.
- Redis posts already exist covering: data structures, caching strategies (Cache-Aside, Write-Through, Write-Behind), cache invalidation, TTL, eviction policy, persistence (RDB/AOF).

### Bad examples (duplicate)

| Existing post | Bad new post (too similar) |
|--------------|---------------------------|
| Redis 캐싱 전략과 핵심 자료구조 | Redis 데이터 구조와 캐싱 전략 ← same content |
| Redis 캐시 전략 (Cache-Aside...) | Redis 캐시 전략 정리 ← same content |

### Good examples (distinct angle)

| Existing post | Good new post (different angle) |
|--------------|--------------------------------|
| Redis 캐싱 전략 | Redis Pub/Sub과 Stream 활용 |
| Redis 데이터 구조 | Redis Cluster 구성과 Failover |
