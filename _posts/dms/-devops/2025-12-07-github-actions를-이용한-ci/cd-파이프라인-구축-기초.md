---
layout: post
title: "[Daily morning study] GitHub Actions를 이용한 CI/CD 파이프라인 구축 기초"
description: >
  #daily morning study
category: 
    - dms
    - -devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# GitHub Actions를 이용한 CI/CD 파이프라인 구축 기초

GitHub Actions는 CI/CD(지속적 통합/지속적 배포) 워크플로우를 구축하는 데 유용한 도구입니다. 이 문서에서는 GitHub Actions를 활용하여 기본적인 CI/CD 파이프라인을 구축하는 방법에 대해 알아보겠습니다.

## 1. GitHub Actions의 이해

GitHub Actions는 GitHub repository에서 이벤트에 반응하여 자동으로 작업을 수행할 수 있도록 하는 기능입니다. 워크플로우를 정의하여 소프트웨어 빌드, 테스트, 배포 등의 작업을 자동화할 수 있습니다. 

### 주요 구성 요소
- **워크플로우(Workflow)**: 이벤트에 의해 트리거되는 작업의 집합입니다.
- **잡(Job)**: 하나 이상의 단계(Step)로 구성된 작업 단위입니다.
- **단계(Step)**: 특정 작업을 수행하는 명령어 또는 스크립트입니다.
- **러너(Runner)**: GitHub Actions가 작업을 실행하는 서버입니다.

## 2. 기본 워크플로우 설정하기

### 2.1. 설정 파일 생성하기

워크플로우 파일은 `.github/workflows` 디렉토리에 YAML 형식으로 생성합니다. 예를 들어, `ci.yml` 파일을 생성해 보겠습니다.

```yaml
name: CI

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v2
      
    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '14'

    - name: Install dependencies
      run: npm install

    - name: Run tests
      run: npm test
```

위의 YAML 파일에서 `name` 필드는 워크플로우의 이름을 정의합니다. `on` 필드에서는 어떤 이벤트에서 워크플로우를 트리거할지를 정의하고, `jobs` 섹션에서는 작업을 나열합니다. 이 예시에서는 `push` 이벤트가 발생할 때마다 `main` 브랜치에서 워크플로우가 실행됩니다.

### 2.2. 각 단계 설명

- **Checkout code**: 현재 repository의 소스 코드를 체크아웃합니다.
- **Setup Node.js**: Node.js 환경을 설정합니다.
- **Install dependencies**: `npm install` 명령어로 프로젝트의 의존성을 설치합니다.
- **Run tests**: `npm test` 명령어로 테스트를 실행합니다.

## 3. 배포 파이프라인 추가하기

CI 설정이 완료되었다면, 이제 지속적 배포를 추가할 차례입니다. 예를 들어, 테스트가 통과하면 자동으로 AWS S3에 배포하는 방법을 살펴보겠습니다.

```yaml
deploy:
  runs-on: ubuntu-latest
  needs: build
  steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Deploy to S3
      uses: jwalton/gh-act-s3@v1
      with:
        args: sync ./dist s3://your-bucket-name/path --delete
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### 3.1. 배포 단계 설명

- **needs: build**: 이 단계를 실행하기 전에 `build` 잡이 완료되어야 함을 명시합니다.
- **Deploy to S3**: AWS S3에 파일을 배포합니다. AWS 계정 정보는 GitHub Secrets에 저장된 값을 사용합니다.
  
위와 같은 방식으로 배포 파이프라인을 추가함으로써, CI/CD 프로세스를 완성할 수 있습니다.

## 4. GitHub Secrets 설정하기

배포할 때 사용될 AWS 키와 같은 민감한 정보를 GitHub Secrets에 저장해야 합니다. 

1. repository의 **Settings**로 이동합니다.
2. 좌측 메뉴에서 **Secrets and variables** > **Actions**를 클릭합니다.
3. **New repository secret**를 클릭하여 `AWS_ACCESS_KEY_ID`와 `AWS_SECRET_ACCESS_KEY`를 추가합니다.

## 5. 테스트 및 확인하기

모든 설정이 끝났다면, 변경사항을 `main` 브랜치에 푸시하여 작업이 잘 동작하는지 확인합니다. GitHub에서 Actions 탭으로 이동하면 워크플로우가 실행되는 것을 볼 수 있습니다.

여기서는 기본적인 CI/CD 파이프라인을 다뤘습니다. GitHub Actions는 매우 유연하고 다양한 용도로 사용할 수 있으니, 필요에 맞게 확장하거나 변형하여 사용해 보세요.
