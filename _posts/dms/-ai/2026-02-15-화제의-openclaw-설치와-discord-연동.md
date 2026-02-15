---
layout: post
title: "[Daily morning study] 화제의 Openclaw 설치와 discord 연동"
description: >
  #daily morning study
category: 
    - dms
    - -ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# 화제의 Openclaw 설치와 Discord 연동 가이드

Openclaw는 다양한 기능을 가진 오픈소스 봇으로, 특히 Discord와의 통합이 뛰어난 특징을 갖고 있습니다. 이 가이드는 Openclaw를 설치하고 Discord와 연결하는 방법에 대해 알아보겠습니다.

## 1. Openclaw 설치하기

Openclaw는 GitHub에서 간편하게 설치할 수 있습니다. 설치를 시작하기 전에 Git 및 Node.js가 시스템에 설치되어 있어야 하니, 먼저 확인해봅시다.

### 1.1. 의존성 설치하기

- **Git 설치**: 공식 [Git 웹사이트](https://git-scm.com/)에서 설치합니다.
- **Node.js 설치**: [Node.js 공식 웹사이트](https://nodejs.org/)에서 LTS 버전을 다운로드하여 설치합니다.

### 1.2. Openclaw 클론하기

터미널을 열고 아래 명령어를 입력하여 Openclaw 리포지토리를 클론합니다.

```bash
git clone https://github.com/username/openclaw.git
cd openclaw
```

### 1.3. 의존성 설치

클론한 디렉토리에서 아래 명령어를 입력하여 필요한 패키지를 설치합니다.

```bash
npm install
```

### 1.4. 환경 변수 설정

Openclaw 설정에 필요한 환경 변수를 .env 파일에 추가합니다. 아래 명령어로 .env 파일을 만들고 편집합니다.

```bash
touch .env
nano .env
```

.env 파일에 다음과 같은 내용을 입력합니다.

```
TOKEN=YOUR_DISCORD_BOT_TOKEN
PREFIX=!
```

`YOUR_DISCORD_BOT_TOKEN` 부분은 Discord에서 발급받은 봇 토큰으로 교체합니다.

## 2. Discord에서 봇 생성하기

Openclaw를 Discord에 연동하기 위해서는 Discord 개발자 포털에서 봇을 생성해야 합니다.

### 2.1. Discord 개발자 포털 방문하기

[Discord Developer Portal](https://discord.com/developers/applications)로 이동하고 로그인합니다.

### 2.2. 새로운 애플리케이션 생성하기

1. "New Application" 버튼을 클릭하여 새로운 애플리케이션을 생성합니다.
2. 애플리케이션 이름을 입력하고 "Create"를 클릭합니다.

### 2.3. 봇 추가하기

1. 애플리케이션 대시보드에서 "Bot" 탭을 선택합니다.
2. "Add Bot" 버튼을 클릭하여 봇을 생성합니다.
3. 생성된 봇의 TOKEN을 복사하여 `.env` 파일의 `YOUR_DISCORD_BOT_TOKEN` 부분에 붙여넣습니다.

### 2.4. 권한 설정하기

1. "OAuth2" 탭으로 이동합니다.
2. "Scopes" 섹션에서 `bot`을 선택합니다.
3. "Bot Permissions"에서 필요한 권한을 선택합니다. (예: SEND_MESSAGES, READ_MESSAGES 등)
4. 생성된 URL을 복사하여 웹 브라우저에 붙여넣고 봇을 원하는 서버에 추가합니다.

## 3. Openclaw 실행하기

모든 설정이 완료되었다면 Openclaw를 실행할 수 있습니다. 아래 명령어를 입력하여 봇을 실행합니다.

```bash
npm start
```

Openclaw가 성공적으로 실행되면, Discord 서버에서 봇이 활성화된 것을 확인할 수 있습니다.

## 4. Openclaw 사용하기

Openclaw가 활성화되면, 설정한 PREFIX (디폴트는 `!`)를 사용하여 명령어를 입력할 수 있습니다. 예를 들어, `!help`를 입력하여 도움말을 볼 수 있습니다.

```
> !help
```

## 5. 추가 설정 및 커스터마이징

Openclaw는 다양한 플러그인과 기능으로 커스터마이징이 가능합니다. 필요한 모듈을 설치하고 기능을 추가하여 나만의 봇을 만들어보세요. 

예를 들어, 특정 플러그인을 설치하고 싶다면 아래와 같이 입력합니다.

```bash
npm install plugin-name
```

그 후, Openclaw의 설정 파일을 수정하여 추가된 기능을 사용할 수 있습니다.

## 6. 마무리

이 가이드를 통해 Openclaw 설치와 Discord 연동 방법을 알아보았습니다. 이제 기본적인 설정과 실행 방법을 익혔으니, 다양한 기능을 실험해보며 더 많은 것을 배워보세요!

---
