---
layout: post
title: "[Python] uv - 차세대 Python 패키징 도구"
subtitle: "uv - 차세대 Python 패키징 도구"
category: devlog
tags: python
---

![Image](https://github.com/user-attachments/assets/7dcb8cef-f0ee-4dad-b425-d2295783f6df)

---

# Python uv — 차세대 Python 패키징 도구

Python 생태계는 pip, poetry, conda 등 다양한 도구로 인해 복잡했습니다.  
**uv**는 이 모든 것을 통합하고 더 빠르게 실행되도록 설계된 최신 도구입니다.

## uv란?

- Rust로 작성된 고속 의존성 해석 엔진
- 패키지 관리, 가상환경 생성, 빌드 등을 통합
- pip 대비 8배 빠른 설치 속도

## uv 사용법

### 설치
```bash
uv venv
uv sync
```

### 패키지 설치 및 실행

```bash
uv add requests
uv run hello.py
```

### 린터 및 포매터 실행

```bash
uvx ruff check .
uvx ruff format *.py
```

## 결론

uv는 Python 개발을 더 쉽고 빠르게 만들어주는 차세대 표준 도구로 급부상하고 있습니다.
